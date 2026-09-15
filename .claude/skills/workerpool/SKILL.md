---
name: workerpool
description: >-
  Паттерн реализации production-ready worker pool на Go: единый тип задачи в
  канале, graceful shutdown с работающим force-stop, panic recovery на задачу,
  per-task context, backpressure (block/fail-fast/drop), неблокирующая доставка
  ошибок, динамический resize. Implementing your own goroutine pool in Go:
  channel skeleton, invariants, shutdown and cancellation wiring.
when_to_use: >-
  Когда в проекте нужно написать свой пул горутин с очередью задач и корректным
  завершением — или отревьюить существующий. Триггеры: "worker pool", "пул
  горутин", "воркер пул", "очередь задач", "graceful shutdown пула", "force
  stop", "panic recovery в горутине", "backpressure", "errgroup или свой пул",
  "resize пула", "goroutine pool", "task queue", "drain queue".
---

# Паттерн: как писать worker pool на Go

Навык — про **реализацию** пула горутин с очередью задач, а не про вызов
готовой библиотеки. Описывает структуру, инварианты и решения, которые делают
пул корректным под `-race`. Адаптируй имена и набор фич под проект — бери
ровно то, что нужно, не тащи всё.

Go 1.27, ниже не поддерживается (политика версий — скилл `go-quality`).

## 0. Сначала реши — нужен ли свой пул

- **I/O-bound фан-аут, фиксированное число задач** → не пиши пул, бери
  `errgroup.Group` + `semaphore.Weighted` (см. §0.1). Короче и идиоматичнее.
- **Готовое в проде** → `sourcegraph/conc`, `alitto/pond`.
- **Свой пул оправдан**, когда даёт что-то поверх семафора: **реюз воркера**
  (дорогая инициализация на воркер), долгоживущая очередь, управляемый
  backpressure, рантайм-resize. Только тогда — реализуй по паттерну ниже.

### 0.1. Идиоматичная альтернатива: errgroup + semaphore

Для **конечного известного набора задач** с ограничением параллелизма пул
обычно избыточен. `golang.org/x/sync/errgroup` даёт fan-out + сбор первой
ошибки + отмену остальных через общий `ctx`.

**Простой случай — `errgroup.SetLimit`.** Ограничивает число одновременных
горутин; `g.Go` блокируется, когда лимит выбран. Это встроенный backpressure
без отдельного семафора. Метод есть с **`golang.org/x/sync` v0.1.0** — это
отдельный модуль, а не стандартная библиотека, поэтому смотри версию в
`go.mod`, версия языка тут ни при чём:

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8) // не больше 8 задач параллельно

for _, item := range items {
	g.Go(func() error {
		return process(ctx, item) // первая ошибка отменит ctx для остальных
	})
}
if err := g.Wait(); err != nil { // ждёт всех, вернёт первую ошибку
	return err
}
```

`errgroup.WithContext`: как только любая `g.Go`-функция вернёт ошибку, `ctx`
отменяется — остальные задачи, уважающие `ctx`, останавливаются (fail-fast).
`g.Wait()` дожидается завершения всех и возвращает первую ненулевую ошибку.
Есть и `g.TryGo` — неблокирующий запуск (вернёт `false`, если лимит выбран).
`SetLimit` нельзя вызывать, пока в группе есть активные горутины — паникует.

**Когда нужен `semaphore.Weighted`** (`golang.org/x/sync/semaphore`):

- **взвешенные задачи** — тяжёлая весит больше одного слота
  (`Acquire(ctx, 4)`), `SetLimit` так не умеет;
- **backpressure до спавна горутины** — `Acquire(ctx, n)` блокирует
  продьюсера и **уважает отмену `ctx`** (возвращает ошибку при cancel), в
  отличие от простого канала-семафора;
- нужно отделить лимит ресурса от группы ошибок.

```go
sem := semaphore.NewWeighted(int64(maxConcurrent))
g, ctx := errgroup.WithContext(ctx)

for _, item := range items {
	// блокируемся, пока не освободится слот; ctx-отмена прерывает ожидание
	if err := sem.Acquire(ctx, item.Weight); err != nil {
		break // ctx отменён (другой задачей или снаружи)
	}
	g.Go(func() error {
		defer sem.Release(item.Weight)
		return process(ctx, item)
	})
}
err := g.Wait() // дождаться запущенных; вернёт первую ошибку
```

**Чем это отличается от worker pool (и почему обычно лучше):**

| | errgroup (+semaphore) | свой worker pool |
|---|---|---|
| Горутины | по горутине на задачу, без реюза | N долгоживущих воркеров, реюз |
| Что ограничиваем | параллелизм (слоты) | очередь + число воркеров |
| Workload | конечный, известный заранее | поток/долгоживущая очередь |
| Ошибки | первая отменяет всех (fail-fast) | каждая в канал, пул живёт дальше |
| Отмена | встроена через общий `ctx` | пишешь сам (§5, §7) |
| Паники | **роняют процесс** — нет recovery | recover на задачу (если сделал) |

## 1. Минимальный скелет

Базис, на который наслаиваются фичи. Каждое поле — под конкретный инвариант.
В канале лежит **`task`, а не голая функция**: задаче нужен собственный `ctx`
(§5), а протащить его отдельно от `fn` некуда.

```go
type task struct {
	ctx context.Context
	fn  func(ctx context.Context) error
}

type Pool struct {
	tasks         chan task
	errs          chan error
	stop          chan struct{}      // close в начале Shutdown — «дренируй и выходи»
	forceCtx      context.Context    // отменяется на force-stop — «бросай очередь, выходи сейчас»
	forceStop     context.CancelFunc // единственный способ отменить forceCtx
	wg            sync.WaitGroup     // ждём завершения всех воркеров
	once          sync.Once          // защита от повторного close(stop) в Shutdown
	onFull        OnFull             // политика при полной очереди, §6
	droppedTasks  atomic.Uint64      // §6
	droppedErrors atomic.Uint64      // §4
}

const defaultErrBuf = 64 // почему не 0 — §4

func New(workers, queueSize int, onFull OnFull) (*Pool, error) {
	if workers <= 0 || queueSize < 0 {
		return nil, fmt.Errorf("%w: workers=%d, queueSize=%d", ErrBadConfig, workers, queueSize) // §9
	}
	forceCtx, forceStop := context.WithCancel(context.Background())
	p := &Pool{
		tasks:     make(chan task, queueSize),
		errs:      make(chan error, defaultErrBuf),
		stop:      make(chan struct{}),
		forceCtx:  forceCtx,
		forceStop: forceStop,
		onFull:    onFull,
	}
	for range workers {
		p.wg.Go(p.worker) // wg.Go (Go 1.25) — без Add/defer Done
	}
	return p, nil
}
```

Политика `onFull` фиксируется в конструкторе: менять её на живом пуле — гонка с
`Submit`. Поля под опциональный resize (§8) в базисе не заведены. Sentinel-ошибки
(`ErrBadConfig`, `ErrNilTask`, `ErrPoolClosed`, `ErrQueueFull`,
`ErrAlreadyShutdown`, `ErrResizeNonPositive`) объяви через `errors.New` — по
одной на отказ, чтобы вызывающий матчил их через `errors.Is`.

`forceCtx` в поле структуры — сознательное исключение из «контекст не хранят в
структурах»: это lifetime-контекст пула, а не per-request. Он нужен именно
контекстом, а не каналом, чтобы от него можно было **произвести `ctx` задачи**
(§5) — без этого force-stop ничего не прерывает.

## 2. Инвариант №1 — канал задач НЕ закрывается

Самая частая ошибка новичка: закрыть `tasks` в `Shutdown`, пока продьюсер ещё
шлёт → паника `send on closed channel`. **Никогда не закрывай `tasks`.** Сигнал
остановки — отдельный канал `stop` (close). Воркеры выходят по `stop`, а не по
закрытию очереди.

```go
func (p *Pool) worker() {
	for {
		select {
		case <-p.forceCtx.Done(): // force-stop: выходим немедленно
			return
		case t := <-p.tasks:
			p.runTask(t)
		case <-p.stop: // graceful: дорезаем остаток и выходим
			p.drainAndExit()
			return
		}
	}
}

func (p *Pool) drainAndExit() {
	for {
		select {
		case <-p.forceCtx.Done():
			return
		case t := <-p.tasks:
			p.runTask(t)
		default: // очередь пуста — выходим
			return
		}
	}
}
```

## 3. Инвариант №2 — panic recovery НА КАЖДУЮ задачу

`recover` должен стоять в функции, вызываемой на каждую задачу, а не один раз
на весь воркер — иначе первая же паника убьёт воркера навсегда. Выноси
исполнение в отдельный метод с `defer recover`. Панику доставляй потребителю
как типизированную ошибку (со стеком), а не теряй.

```go
type PanicError struct {
	Recovered any
	Stack     []byte
}

func (e *PanicError) Error() string { return fmt.Sprintf("panic в задаче: %v", e.Recovered) }

func (p *Pool) runTask(t task) {
	defer func() {
		if r := recover(); r != nil {
			p.sendErr(&PanicError{Recovered: r, Stack: debug.Stack()})
		}
	}()

	if err := t.ctx.Err(); err != nil { // отменена, пока лежала в очереди
		p.sendErr(fmt.Errorf("задача не запущена: %w", err))
		return
	}

	ctx, release := p.taskContext(t.ctx) // §5: ctx задачи + force-stop
	defer release()

	if err := t.fn(ctx); err != nil {
		p.sendErr(err)
	}
}
```

Задача исполняется с `t.ctx`, а не с `context.Background()`. `Background()`
здесь — молчаливая отмена per-task дедлайнов: `Submit` принял `ctx`, а до `fn`
он не доехал.

## 4. Инвариант №3 — доставка ошибок неблокирующая

Если воркер блокируется на отправке в `errs`, он встаёт — и пул деградирует.
Шли через `select/default` и считай потерянные. Канал `errs` **не закрывай**
на `Shutdown`: in-flight задачи могут писать после возврата `Shutdown`.

```go
func (p *Pool) sendErr(err error) {
	select {
	case p.errs <- err:
	default:
		p.droppedErrors.Add(1) // atomic.Uint64; экспонируй как метрику
	}
}

func (p *Pool) Errors() <-chan error  { return p.errs }
func (p *Pool) DroppedErrors() uint64 { return p.droppedErrors.Load() }
func (p *Pool) DroppedTasks() uint64  { return p.droppedTasks.Load() }
```

**Буфер `errs` обязан быть ненулевым.** С нулевым отправка проходит, только
если потребитель стоит на `<-p.errs` ровно в этот момент: пока он занят
предыдущей ошибкой, каждая новая уходит в `default`. Буфер = «сколько ошибок
придержим, пока потребитель занят»; бери не меньше числа воркеров, дефолт 64.

Потребитель читает `Errors()` в отдельной горутине весь жизненный цикл и
различает панику через `errors.As(err, &pe)`. Не полагайся на закрытие канала
как на сигнал конца.

## 5. Per-task context и проводка force-stop

Каждая задача получает свой `ctx` (как `http.Request.Context()`): отмена и
дедлайн на конкретную задачу. Поэтому `ctx` лежит в `task` (§1), а не в полях
пула, и `Submit` кладёт в очередь `task{ctx: ctx, fn: fn}` (§6).

`close`/`cancel` сигнала force-stop **сам по себе не прерывает выполняющуюся
задачу**: воркер сидит внутри `t.fn(ctx)` и вернётся в `select` только когда
`fn` закончит. Чтобы force-stop работал, нужны обе половины:

1. `ctx` задачи производен от `forceCtx` — тогда отмена долетает внутрь `fn`;
2. сама `fn` уважает `ctx.Done()` — иначе отменять некому и нечего.

Первую половину даёт `context.AfterFunc` — без горутины на задачу:

```go
// taskContext: ctx задачи = ctx вызывающего, дополнительно отменяемый force-stop.
func (p *Pool) taskContext(parent context.Context) (context.Context, func()) {
	ctx, cancel := context.WithCancel(parent)
	stopAfter := context.AfterFunc(p.forceCtx, cancel)
	return ctx, func() {
		stopAfter() // снять регистрацию: она висит на forceCtx до конца жизни пула
		cancel()
	}
}
```

`release()` вызывать обязательно (в `runTask` он под `defer`): без `stopAfter()`
каждая выполненная задача оставляет запись в `forceCtx`, и на долгоживущем пуле
это чистая утечка памяти.

## 6. Submit + backpressure

Перед отправкой проверь `stop`/`ctx` (fast-path отказа). Поведение при полной
очереди делай **явным выбором политики**, а не зашитым:

```go
type OnFull int

const (
	OnFullBlock      OnFull = iota // ждать слот / ctx / stop (default, zero-value)
	OnFullFailFast                 // вернуть ErrQueueFull сразу
	OnFullDropNewest               // дропнуть + DroppedTasks++, вернуть nil
)
```

```go
func (p *Pool) Submit(ctx context.Context, fn func(context.Context) error) error {
	if fn == nil {
		return ErrNilTask
	}
	select {
	case <-p.stop:
		return ErrPoolClosed
	case <-ctx.Done():
		return ctx.Err()
	default:
	}
	t := task{ctx: ctx, fn: fn}
	switch p.onFull {
	case OnFullFailFast:
		select {
		case p.tasks <- t:
			return nil
		default:
			return ErrQueueFull
		}
	case OnFullDropNewest:
		select {
		case p.tasks <- t:
			return nil
		default:
			p.droppedTasks.Add(1)
			return nil
		}
	}
	// OnFullBlock
	select {
	case p.tasks <- t:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	case <-p.stop:
		return ErrPoolClosed
	}
}
```

Ключевые решения:
- Zero-value политики = `Block` → дефолт безопасный и обратносовместимый.
- `DropNewest` возвращает `nil` (fire-and-forget) — это **ловушка**: потеря
  видна только через `DroppedTasks()`. Всегда экспонируй счётчик как метрику.
- `Block` всегда селектит и `ctx`, и `stop` — иначе `Submit` повиснет навечно.
- `nil` от `Submit` ≠ «задача выполнится»: если `Shutdown` пришёл между
  fast-path и отправкой, задача ляжет в очередь, которую уже никто не читает.
  Нужна гарантия исполнения — считай принятые и завершённые и сверяй.

## 7. Graceful shutdown с force-stop

`Shutdown` идемпотентен (`once`), даёт время на drain и форсит остановку по
дедлайну `ctx`:

```go
func (p *Pool) Shutdown(ctx context.Context) error {
	called := false
	p.once.Do(func() { called = true; close(p.stop) })
	if !called {
		return ErrAlreadyShutdown
	}

	drained := make(chan struct{})
	go func() { p.wg.Wait(); close(drained) }()

	select {
	case <-drained:
		return nil
	case <-ctx.Done():
		p.forceStop() // отменяет forceCtx → и воркеры, и ctx in-flight задач
		<-drained     // дождаться фактического выхода
		return fmt.Errorf("drain не успел: %w", ctx.Err())
	}
}
```

Два сигнала с разной семантикой: `stop` = «дорежь и выходи», `forceCtx` =
«выходи сейчас». Не путай их и не схлопывай в один. `forceStop` — обычный
`context.CancelFunc`, повторный вызов безопасен (в отличие от `close`), так что
отдельная защита ему не нужна.

Force-stop прерывает только те задачи, которые селектят свой `ctx.Done()`.
Задача, которая его игнорирует, доработает до конца, и `<-drained` будет её
ждать — `Shutdown` вернётся позже своего дедлайна. Это не баг пула, это
контракт для авторов задач (§11).

## 8. (Опционально) динамический resize

Нужен только если число воркеров реально меняется в рантайме. Делай синхронно
под `sync.Mutex` и возвращай **фактическое** число воркеров:

```go
// (n, nil) | (current, ErrPoolClosed) | (0, ErrResizeNonPositive)
func (p *Pool) Resize(n int) (int, error)
```

Подводные камни (каждый — реальный баг под `-race`):
- **Shrink через `close` нельзя** — это broadcast, погасит всех. Нужно ровно
  K выходов → шли K раз в unbuffered `quitOne chan struct{}` (K send = K exit).
  Какой воркер получит сигнал — undefined, и это нормально (нет affinity).
- **Eager-decrement счётчика воркеров** под mutex сразу после успешного send в
  `quitOne`, а не в `defer` воркера. Иначе два конкурентных shrink прочитают
  один stale `current`, посчитают одинаковую дельту → пул усохнет вдвое.
- При shrink параллельно селекти `<-p.stop`: если пошёл `Shutdown`, воркеры
  уйдут через drain-ветку и твой send в unbuffered `quitOne` повиснет навсегда.
- `Resize(0)` ≠ `Shutdown`: `n<=0` → ошибка, не остановка (Shutdown дренит,
  Resize — нет; смешивать = терять задачи).

Поля под resize (`mu sync.Mutex`, `quitOne chan struct{}` без буфера, счётчик
`current int`) в базисе §1 не заведены — добавь вместе с методом. И добавь 4-й
case `<-p.quitOne` в `select` воркера для кооперативного выхода.

## 9. Конструктор возвращает ошибку, паника — только в Must-обёртке

Правило про `panic` принадлежит скиллу `go-code-style` — не переопределяй, а
сверься: конструктор с валидацией возвращает `(*T, error)` и не паникует;
`panic` — только в `Must*`-обёртке над литералами, в тестовых фикстурах и на
старте до приёма трафика, и **никогда** на значении из конфига, запроса или БД.

- `New` → `(*Pool, error)` (§1): `workers`/`queueSize` в живом сервисе приезжают
  из конфига, а на значении из конфига паниковать нельзя.
- `MustNew(8, 128, OnFullBlock)` — тонкая обёртка над `New` для литералов в
  `main` и тестов: там невалидное значение = опечатка в исходнике.
- Невалидная рантайм-команда (`Resize(0)`, `Submit` после `Shutdown`) → error:
  это нормальный поток в долгоживущем процессе.

## 10. Машина состояний (держать в голове при реализации)

```
New → ACTIVE ──Shutdown(close stop)──> DRAINING ──drained──> CLOSED (nil)
        │  Submit/Resize → ErrPoolClosed      └──ctx.Done()──> FORCE-STOP
        │                                        (forceStop(), wrapped err)
        └ Submit→tasks, Resize→grow/shrink
```

## 11. Чеклист реализации

- [ ] В канале один тип — `task{ctx, fn}`; `tasks` **никогда** не закрывается, остановка через `stop` (close).
- [ ] `runTask` исполняет `t.fn` с `t.ctx`, а не с `context.Background()`; перед запуском проверяет `t.ctx.Err()`.
- [ ] `recover` на каждую задачу (`runTask`), паника → типизированная ошибка со стеком.
- [ ] `errs` буферизован (не меньше числа воркеров), шлётся через `select/default`, не закрывается.
- [ ] `Submit` при `Block` селектит `ctx` и `stop` — не виснет.
- [ ] `Shutdown` идемпотентен (`once`), есть force-stop по дедлайну (`forceStop`).
- [ ] ctx задачи производен от `forceCtx` (`context.AfterFunc`), регистрация снимается после задачи.
- [ ] Задачи уважают `ctx.Done()` — без этой половины force-stop ничего не прерывает.
- [ ] `New` возвращает `(*Pool, error)`; `panic` — только в `MustNew` над литералами (§9).
- [ ] `Resize` (если есть): mutex, K send в `quitOne` вместо close, eager-decrement, селект `stop`.
- [ ] счётчики (`DroppedTasks/DroppedErrors`) экспонированы как метрики.

## 12. Тестировать обязательно под race

Конкурентный код без race-детектора и многократного прогона недопроверен:

```bash
go test -race -count=1 ./<pkg>/
go test -count=10 ./<pkg>/ # ловит flaky-гонки
```

Минимальный набор сценариев: graceful drain (все задачи выполнились),
shutdown по таймауту (force-stop фактически прерывает задачу, которая ждёт
`ctx.Done()`, и не оставляет утечки горутин), panic recovery (соседи живы),
per-task cancel (задача, отменённая в очереди, не запускается), submit после
shutdown, double shutdown, переполнение `errs`, и — если есть resize —
concurrent resize и resize-vs-shutdown гонка.

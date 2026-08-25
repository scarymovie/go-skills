---
name: workerpool
description: >-
  Паттерн реализации production-ready worker pool на Go: структура на каналах,
  graceful shutdown с force-stop, panic recovery на задачу, per-task context,
  backpressure (block/fail-fast/drop), неблокирующая доставка ошибок,
  динамический resize. Применять, когда в проекте нужно написать свой worker
  pool / пул горутин с очередью задач и корректным завершением.
---

# Паттерн: как писать worker pool на Go

Навык — про **реализацию** пула горутин с очередью задач, а не про вызов
готовой библиотеки. Описывает структуру, инварианты и решения, которые делают
пул корректным под `-race`. Адаптируй имена и набор фич под проект — бери
ровно то, что нужно, не тащи всё.

## 0. Сначала реши — нужен ли свой пул

- **I/O-bound фан-аут, фиксированное число задач** → не пиши пул, бери
  `errgroup.Group` + `semaphore.Weighted`. Короче и идиоматичнее.
- **Готовое в проде** → `sourcegraph/conc`, `alitto/pond`.
- **Свой пул оправдан**, когда даёт что-то поверх семафора: **реюз воркера**
  (дорогая инициализация на воркер), долгоживущая очередь, управляемый
  backpressure, рантайм-resize. Только тогда — реализуй по паттерну ниже.

## 1. Минимальный скелет

Базис, на который наслаиваются фичи. Каждое поле — под конкретный инвариант.

```go
type Pool struct {
    tasks chan func(ctx context.Context) error
    errs  chan error
    stop  chan struct{}   // close в начале Shutdown — сигнал «дренируй и выходи»
    done  chan struct{}   // close на force-stop — «бросай очередь, выходи сейчас»
    wg    sync.WaitGroup  // ждём завершения всех воркеров
    once  sync.Once       // защита от double-close в Shutdown
}

func New(workers, queueSize int) *Pool {
    p := &Pool{
        tasks: make(chan func(ctx context.Context) error, queueSize),
        errs:  make(chan error, /* errBuf */ 0),
        stop:  make(chan struct{}),
        done:  make(chan struct{}),
    }
    p.wg.Add(workers)
    for range workers {
        go p.worker()
    }
    return p
}
```

## 2. Инвариант №1 — канал задач НЕ закрывается

Самая частая ошибка новичка: закрыть `tasks` в `Shutdown`, пока продьюсер ещё
шлёт → паника `send on closed channel`. **Никогда не закрывай `tasks`.** Сигнал
остановки — отдельный канал `stop` (close). Воркеры выходят по `stop`, а не по
закрытию очереди.

```go
func (p *Pool) worker() {
    defer p.wg.Done()
    for {
        select {
        case <-p.done:               // force-stop: выходим немедленно
            return
        case t := <-p.tasks:
            p.runTask(t)
        case <-p.stop:               // graceful: дорезаем остаток и выходим
            p.drainAndExit()
            return
        }
    }
}

func (p *Pool) drainAndExit() {
    for {
        select {
        case <-p.done:
            return
        case t := <-p.tasks:
            p.runTask(t)
        default:                     // очередь пуста — выходим
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

func (p *Pool) runTask(fn func(ctx context.Context) error) {
    defer func() {
        if r := recover(); r != nil {
            p.sendErr(&PanicError{Recovered: r, Stack: debug.Stack()})
        }
    }()
    if err := fn(context.Background()); err != nil {
        p.sendErr(err)
    }
}
```

## 4. Инвариант №3 — доставка ошибок неблокирующая

Если воркер блокируется на отправке в `errs`, он встаёт — и пул деградирует.
Шли через `select/default` и считай потерянные. Канал `errs` **не закрывай**
на `Shutdown`: in-flight задачи могут писать после возврата `Shutdown`.

```go
func (p *Pool) sendErr(err error) {
    select {
    case p.errs <- err:
    default:
        p.droppedErrors.Add(1)   // atomic.Uint64; экспонируй как метрику
    }
}

func (p *Pool) Errors() <-chan error { return p.errs }
func (p *Pool) DroppedErrors() uint64 { return p.droppedErrors.Load() }
```

Потребитель читает `Errors()` в отдельной горутине весь жизненный цикл и
различает панику через `errors.As(err, &pe)`. Не полагайся на закрытие канала
как на сигнал конца.

## 5. Per-task context

Каждая задача получает свой `ctx` (как `http.Request.Context()`): отмена/дедлайн
на конкретную задачу. Храни `ctx` в структуре задачи, а не в полях пула.

```go
type task struct {
    ctx context.Context
    fn  func(ctx context.Context) error
}
```

Перед запуском проверяй `t.ctx.Err()` — если уже отменён, не запускай, отдай
ошибку. `worker_id` прокидывай через `context.WithValue` с **приватным
тип-ключом** (`type workerIDKey struct{}`), чтобы снаружи не перезаписали:

```go
func WorkerID(ctx context.Context) (int64, bool) {
    v, ok := ctx.Value(workerIDKey{}).(int64)
    return v, ok
}
```

## 6. Submit + backpressure

Перед отправкой проверь `stop`/`ctx` (fast-path отказа). Поведение при полной
очереди делай **явным выбором политики**, а не зашитым:

```go
const (
    OnFullBlock      = iota // ждать слот / ctx / stop (default, zero-value)
    OnFullFailFast          // вернуть ErrQueueFull сразу
    OnFullDropNewest        // дропнуть + DroppedTasks++, вернуть nil
)
```

```go
func (p *Pool) Submit(ctx context.Context, fn func(context.Context) error) error {
    if fn == nil { return ErrNilTask }
    select {
    case <-p.stop:  return ErrPoolClosed
    case <-ctx.Done(): return ctx.Err()
    default:
    }
    t := task{ctx: ctx, fn: fn}
    switch p.onFull {
    case OnFullFailFast:
        select {
        case p.tasks <- t: return nil
        default: return ErrQueueFull
        }
    case OnFullDropNewest:
        select {
        case p.tasks <- t: return nil
        default: p.droppedTasks.Add(1); return nil
        }
    }
    // OnFullBlock
    select {
    case p.tasks <- t: return nil
    case <-ctx.Done(): return ctx.Err()
    case <-p.stop: return ErrPoolClosed
    }
}
```

Ключевые решения:
- Zero-value политики = `Block` → дефолт безопасный и обратносовместимый.
- `DropNewest` возвращает `nil` (fire-and-forget) — это **ловушка**: потеря
  видна только через `DroppedTasks()`. Всегда экспонируй счётчик как метрику.
- `Block` всегда селектит и `ctx`, и `stop` — иначе `Submit` повиснет навечно.

## 7. Graceful shutdown с force-stop

`Shutdown` идемпотентен (`once`), даёт время на drain и форсит остановку по
дедлайну `ctx`:

```go
func (p *Pool) Shutdown(ctx context.Context) error {
    called := false
    p.once.Do(func() { called = true; close(p.stop) })
    if !called { return ErrAlreadyShutdown }

    drained := make(chan struct{})
    go func() { p.wg.Wait(); close(drained) }()

    select {
    case <-drained:
        return nil
    case <-ctx.Done():
        close(p.done)                 // force-stop: воркеры бросают очередь
        <-drained                     // дождаться фактического выхода
        return fmt.Errorf("drain не успел: %w", ctx.Err())
    }
}
```

Два канала-сигнала с разной семантикой: `stop` = «дорежь и выходи», `done` =
«выходи сейчас». Не путай их и не схлопывай в один.

## 8. (Опционально) динамический resize

Нужен только если число воркеров реально меняется в рантайме. Делай синхронно
под `sync.Mutex` и возвращай **фактическое** число воркеров:

```go
func (p *Pool) Resize(n int) (int, error)   // (n,nil)/(current,ErrPoolClosed)/(0,ErrResizeNonPositive)
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

Добавь 4-й case `<-p.quitOne` в `select` воркера для кооперативного выхода.

## 9. Валидация конфига = паника, рантайм-ошибка = error

- Невалидный конфиг в `New` (`workers<=0` и т.п.) → **panic**: это программерская
  ошибка на старте, чинится в коде, а не обрабатывается.
- Невалидная рантайм-команда (`Resize(0)`, Submit после Shutdown) → **error**:
  это нормальный поток в долгоживущем процессе.

## 10. Машина состояний (держать в голове при реализации)

```
New → ACTIVE ──Shutdown(close stop)──> DRAINING ──drained──> CLOSED (nil)
        │  Submit/Resize → ErrPoolClosed      └──ctx.Done()──> FORCE-STOP
        │                                          (close done, wrapped err)
        └ Submit→tasks, Resize→grow/shrink
```

## 11. Чеклист реализации

- [ ] `tasks` **никогда** не закрывается; остановка — через `stop` (close).
- [ ] `recover` на каждую задачу (`runTask`), паника → типизированная ошибка со стеком.
- [ ] `errs` шлётся через `select/default`, есть счётчик `DroppedErrors`; канал не закрывается.
- [ ] `Submit` при `Block` селектит `ctx` и `stop` — не виснет.
- [ ] `Shutdown` идемпотентен (`once`), есть force-stop по дедлайну (`done`).
- [ ] in-flight задачи уважают `ctx.Done()`, иначе force-stop их не прервёт.
- [ ] `Resize` (если есть): mutex, K send в `quitOne` вместо close, eager-decrement, селект `stop`.
- [ ] счётчики (`DroppedTasks/DroppedErrors`) экспонированы как метрики.

## 12. Тестировать обязательно под race

Конкурентный код без race-детектора и многократного прогона недопроверен:

```bash
go test -race -count=1 ./<pkg>/
go test -count=10 ./<pkg>/   # ловит flaky-гонки
```

Минимальный набор сценариев: graceful drain (все задачи выполнились),
shutdown по таймауту (force-stop, нет утечки горутин), panic recovery (соседи
живы), per-task cancel, submit после shutdown, double shutdown, переполнение
`errs`, и — если есть resize — concurrent resize и resize-vs-shutdown гонка.

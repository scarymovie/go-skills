---
name: eventbus
description: >-
  Паттерн реализации in-process event bus на Go по actor-модели: типизированный
  publish/subscribe с тотальным порядком событий, единый pump-goroutine, тонкий
  Foo[T]-фасад над non-generic ядром (без per-T раздувания), backpressure через
  ограниченную очередь, переиспользуемые примитивы (worker, stopFlag, queue[T],
  Monitor), инспекция состояния без локов, тестовый харнесс. Применять, когда в
  проекте нужно написать свою шину событий / pub-sub поверх каналов. Триггеры:
  "event bus", "шина событий", "pub/sub", "publish/subscribe", "actor model",
  "in-process events", "событийная шина".
---

# Паттерн: как писать in-process event bus на Go

Навык — про **реализацию** шины событий (а не вызов готовой библиотеки).
Описывает структуру, инварианты и решения, делающие шину корректной под `-race`
и дешёвой по дженерикам. За основу взята архитектура `tailscale.com/util/eventbus`.
Адаптируй имена и набор фич под проект — **бери ровно то, что нужно, не тащи всё.**

## 0. Сначала реши — нужна ли своя шина

Шина — это развязка: издатель не знает подписчиков, подписчик не знает издателя.
Цена развязки — асинхронность, потеря стек-трейсов, отдельный дебаг. Не плати её
зря.

- **Два-три компонента, прямой вызов понятен** → зови напрямую или передай
  колбэк/`chan`. Шина тут — оверинжиниринг.
- **Фан-аут одного события немногим получателям внутри пакета** → хватит
  `chan T` + список подписчиков, без типизированного роутера.
- **Готовое** → `asaskevich/EventBus`, `cloudevents`, NATS (если межпроцессно).

**Своя шина оправдана**, когда нужно одновременно: развязать много модулей,
получить **единый порядок** всех событий (actor-модель), типобезопасный
publish/subscribe по `reflect.Type`, и встроенную интроспекцию (что в очереди,
кто тормозит). Только тогда — реализуй по паттерну ниже.

## 1. Actor-модель и гарантии порядка

Фундамент: **один true timeline** всех событий, и **один goroutine-роутер**
(`pump`), который владеет таблицей маршрутизации и сериализует доставку. Никаких
локов на горячем пути роутинга — состояние принадлежит одной горутине.

Гарантии, которые даёшь подписчику:
- события публикуются в тотальном порядке (две публикации не происходят «в один
  момент»);
- один клиент получает события в порядке timeline, «перепрыгивая» неинтересные
  типы — его взгляд всегда движется **вперёд** по времени;
- но клиенты **не** синхронизированы между собой: C1 может уйти вперёд C2.

Главное правило для пользователей шины (записать в doc-комментарий пакета):

> Не дёргай другой компонент напрямую сразу после получения события — его «сейчас»
> может отставать от твоего или никогда не совпасть. Взаимодействуй только через
> события: получил → обновил своё локальное состояние → при необходимости
> эмитнул своё событие.

Это и есть actor-модель: у тебя есть состояние, над которым ты единоличный
владелец; наружу — только события.

## 2. Ключевой паттерн: типизированный фасад над non-generic ядром

Самое ценное решение в коде. Публичный API типизирован (`Publish[T]`,
`Subscribe[T]`), но **в мапах и интерфейсах шины не должно быть `Foo[T]`** —
иначе каждый новый тип события платит за свой itab, dictionary и стенсел
(раздувание бинаря и рантайма на ровном месте).

Приём: `Publisher[T]`/`Subscriber[T]` — тонкие фасады, всё состояние и логика в
non-generic ядре, в коллекции шины кладётся **интерфейс, реализованный ядром**.

```go
// Публичный фасад: несёт T только в сигнатуре, состояния не дублирует.
type Publisher[T any] struct {
    core *publisherCore // вся реальная работа здесь
}

// Non-generic ядро: реализует package-private интерфейс publisher.
type publisherCore struct {
    client *Client
    stop   stopFlag
    typ    reflect.Type // кэш reflect.TypeFor[T]() — один раз при создании
}

type publisher interface { // в наборах клиента/шины храним ЭТО, не Publisher[T]
    publishType() reflect.Type
    Close()
}

func Publish[T any](c *Client) *Publisher[T] {
    p := &Publisher[T]{core: &publisherCore{client: c, typ: reflect.TypeFor[T]()}}
    c.addPublisher(p.core) // регистрируем ядро, не фасад
    return p
}

// Единственная per-T работа в Publish — боксинг v в any. Весь канальный
// танец живёт в non-generic publish().
func (p *Publisher[T]) Publish(v T) { publish(p.core, v) }

func publish(c *publisherCore, v any) {
    select { // closed publisher → no-op
    case <-c.stop.Done():
        return
    default:
    }
    select {
    case c.client.bus.write <- PublishedEvent{Event: v, From: c.client}:
    case <-c.stop.Done():
    }
}
```

То же с подписчиком: `Subscriber[T]` хранит только типизированный канал
доставки, а таймер «медленного подписчика», `unregister`, логгер и кэш имени
типа — в `subscriberCore`. В мапе шины лежит интерфейс `subscriber`, не
`Subscriber[T]`.

**Когда generic оставить осознанно.** Типизированная отправка `case s.read <- t:`
обязана лексически стоять внутри `select` — её нельзя вынести в non-generic
функцию без бриджа через отдельную горутину на каждое событие. В tailscale
замерили: такой бридж дал **~2.7x просадку throughput**. Вывод: оставляешь
per-shape стенсел только вокруг типизированного `select`, а всё окружение
(таймер, имя типа, логгер) тянешь из non-generic ядра.

```go
// Per-T остаётся ТОЛЬКО из-за case s.read <- t. Остальные cases совпадают
// с non-generic pump — держи их в синхроне.
func (s *Subscriber[T]) dispatchTyped(...) bool {
    t := vals.Peek().Event.(T) // распаковка any → T
    for {
        select {
        case s.read <- t:
            vals.Drop(); return true
        case val := <-acceptCh():   // продолжаем принимать, пока подписчик занят
            vals.Add(val)
        case <-ctx.Done():
            return false
        case <-s.core.slow.C:       // окружение — из non-generic core
            s.core.logf("subscriber for %s is slow", s.core.typeName)
            s.core.slow.Reset(slowTimeout)
        }
    }
}
```

Для колбэк-подписчика (`SubscribeFunc[T]`) бридж допустим: вызываешь `f(t)` в
отдельной горутине и ждёшь `callDone` в non-generic `select`. Колбэк синхронный —
**он не должен блокироваться надолго**, иначе тормозит все подписки клиента.

## 3. Переиспользуемые concurrency-примитивы

Эти четыре кусочка самодостаточны — таскай их в любой конкурентный код.

### `worker` — обёртка жизненного цикла горутины

```go
type worker struct {
    ctx     context.Context
    stop    context.CancelFunc
    stopped chan struct{}
}

func runWorker(fn func(context.Context)) *worker {
    ctx, stop := context.WithCancel(context.Background())
    w := &worker{ctx: ctx, stop: stop, stopped: make(chan struct{})}
    go func() { defer close(w.stopped); fn(w.ctx) }()
    return w
}

func (w *worker) Stop()        { w.stop() }
func (w *worker) Done() <-chan struct{} { return w.stopped }
func (w *worker) StopAndWait() { w.stop(); <-w.stopped }
```

### `stopFlag` — одноразовый идемпотентный сигнал остановки

Легче `context.Context`, когда нужен только «однократно щёлкнуть выключатель».
Ленивое создание канала, `Stop` можно звать многократно.

```go
type stopFlag struct {
    mu             sync.Mutex
    stopped        chan struct{}
    alreadyStopped bool
}

func (s *stopFlag) Stop() {
    s.mu.Lock(); defer s.mu.Unlock()
    if s.alreadyStopped { return }
    s.alreadyStopped = true
    if s.stopped == nil { s.stopped = make(chan struct{}) }
    close(s.stopped)
}

func (s *stopFlag) Done() <-chan struct{} {
    s.mu.Lock(); defer s.mu.Unlock()
    if s.stopped == nil { s.stopped = make(chan struct{}) }
    return s.stopped
}
```

### `queue[T]` — generic ring-buffer

Очередь под select-петлю: `Peek` (посмотреть голову, не снимая), `Drop` (снять
голову), `Add`, `Snapshot`. Bounded (для backpressure) или unbounded
(`capacity == 0`). Компактизация сдвигом, без аллокаций на стабильном размере.

```go
type queue[T any] struct {
    vals     []T
    start    int
    capacity int // 0 = безлимит
}

func (q *queue[T]) Empty() bool { return q.start == len(q.vals) }
func (q *queue[T]) Full() bool  { return q.start == 0 && !q.canAppend() }
func (q *queue[T]) Peek() T     { if q.Empty() { var z T; return z }; return q.vals[q.start] }

func (q *queue[T]) Add(v T) {
    if !q.canAppend() {
        if q.start == 0 { panic("Add on a full queue") }
        n := copy(q.vals, q.vals[q.start:]) // сдвигаем хвост в начало
        clear(q.vals[n:]); q.vals = q.vals[:n]; q.start = 0
    }
    q.vals = append(q.vals, v)
}

func (q *queue[T]) Drop() {
    if q.Empty() { return }
    var z T; q.vals[q.start] = z // обнуляем — не держим ссылку (GC)
    q.start++
    if q.Empty() { q.start = 0; q.vals = q.vals[:0] }
}
```

> Всегда обнуляй снятый слот (`var z T; q.vals[i] = z`) — иначе очередь держит
> ссылку на выданное значение и мешает GC.

### `Monitor` — zero-value-valid ожидание горутины

```go
type Monitor struct {
    cli  *Client
    done <-chan struct{}
}

func (c *Client) Monitor(f func(*Client)) Monitor {
    done := make(chan struct{})
    go func() { defer close(done); f(c) }()
    return Monitor{cli: c, done: done}
}

// Zero value валиден: Close/Wait возвращаются сразу, Done() — закрытый канал.
func (m Monitor) Wait() { if m.done != nil { <-m.done } }
```

## 4. Select-петля доставки с backpressure

Сердце шины — `pump`: тянет события из входного канала в очередь и раздаёт
подписчикам, **не вставая** при занятом подписчике. Два приёма.

**Nil-channel трюк.** `nil`-канал в `select` навсегда блокирует свой case — то
есть «выключает» его. Делаешь функцию, возвращающую либо канал, либо `nil`:

```go
acceptCh := func() chan PublishedEvent {
    if vals.Full() { return nil } // очередь полна → перестаём принимать новое
    return b.write
}
```

**Оппортунистический приём во время доставки.** Пока упёрлись в отправку одному
подписчику, продолжаем принимать новые события (если есть место) и отвечать на
snapshot-запросы — не простаиваем:

```go
for _, d := range dests {
    evt := DeliveredEvent{Event: val.Event, From: val.From, To: d.client}
    for {
        select {
        case d.write <- evt:
            break // доставили — к следующему получателю
        case <-d.closed():
            break // подписчик закрыт — не блокируемся, идём дальше
        case in := <-acceptCh():
            vals.Add(in) // пользуемся паузой, принимаем входящее
        case <-ctx.Done():
            return
        case ch := <-b.snapshot:
            ch <- vals.Snapshot() // инспекция (см. §5)
        }
    }
}
```

**Ограниченная входная очередь ловит баги.** Делай publish-очередь bounded
(например 16). Если она заполнилась — значит downstream-подписчик завис и создаёт
backpressure: это **баг медленного подписчика**, и ограниченная очередь делает
его видимым, а не маскирует безграничным буфером.

> Набор cases в `pump` и в `dispatch` подписчика должен совпадать (различие —
> только сама доставка). Расхождение = подвисший snapshot или пропущенный
> `ctx.Done()`. Пометь это комментарием в обоих `select`.

## 5. Инспекция состояния без локов

Состояние принадлежит горутине — снаружи лочить нельзя. Решения:

**Snapshot через `chan chan []T`** — запрос/ответ к владельцу. Внешний код шлёт
канал-ответ; `pump` в своём `select` отвечает в него своим срезом:

```go
func (b *Bus) snapshotPublishQueue() []PublishedEvent {
    resp := make(chan []PublishedEvent)
    select {
    case b.snapshot <- resp:   // отдаём запрос в pump
        return <-resp          // pump кладёт ответ
    case <-b.router.Done():    // шина закрыта
        return nil
    }
}
```

**Таймер медленного подписчика.** Засекай доставку; если подписчик не принял за
N секунд — логируй (а в CI можно дампить стеки горутин). Не убивай — диагностируй.

**Replace-on-write для lock-free чтения.** `pump` читает срез подписчиков топика
**без лока**. Значит при отписке нельзя мутировать срез на месте — заменяй
целиком (`slices.Clone` + `Delete`). Отписки редки, копия дешевле гонки:

```go
func (b *Bus) unsubscribe(t reflect.Type, q *subscribeState) {
    b.topicsMu.Lock(); defer b.topicsMu.Unlock()
    i := slices.Index(b.topics[t], q)
    if i < 0 { return }
    b.topics[t] = slices.Delete(slices.Clone(b.topics[t]), i, i+1)
}
```

**Дешёвые debug-хуки.** Дебаг-путь не должен стоить ничего, когда выключен.
Проверяй `active()` атомиком до любой работы по сборке debug-события:

```go
if b.routeDebug.active() { // atomic.Bool под капотом — почти бесплатно
    b.routeDebug.run(RoutedEvent{Event: val.Event, From: val.From, To: clients})
}
```

Интроспекцию (`Debugger`: кто в очереди, кто медленный, список клиентов) держи в
**отдельном типе**, а не в публичном API клиента — чтобы соблазн «подсмотреть
отправителя события» не протёк в продовый код.

## 6. Тестирование событийного кода

Шину тестируют **по наблюдаемым событиям**, а не по внутренним полям. Паттерн
харнесса (по образцу `eventbustest`):

```go
func NewBus(t testing.TB) *Bus {
    bus := New()
    t.Cleanup(bus.Close) // lifecycle привязан к тесту
    return bus
}

// Watcher подписывается на «все маршрутизированные события» через Debugger
// и копит их в буферный канал.
func NewWatcher(t *testing.T, bus *Bus) *Watcher {
    tw := &Watcher{mon: bus.Debugger().WatchBus(), events: make(chan any, 100)}
    t.Cleanup(tw.done)
    go tw.watch()
    return tw
}
```

API проверок:
- `Expect(tw, matchers...)` — заданные события встречаются как **подпоследовательность**
  (можно с чужими событиями между ними).
- `ExpectExactly(tw, matchers...)` — поток ровно из этих событий, без лишних.
- `Type[T]()` — матчер «событие типа T, содержимое неважно»:

```go
func Type[T any]() func(T) { return func(T) {} }
```

```go
bus := eventbustest.NewBus(t)
tw  := eventbustest.NewWatcher(t, bus)
somethingThatEmitsFoo()
if err := eventbustest.Expect(tw, eventbustest.Type[EventFoo]()); err != nil {
    t.Error(err)
}
```

**Проверка отсутствия событий — через `testing/synctest`.** Чтобы не ждать
реальные таймеры, оборачивай в `synctest.Test` + `synctest.Wait()`:

```go
synctest.Test(t, func(t *testing.T) {
    bus := eventbustest.NewBus(t)
    tw  := eventbustest.NewWatcher(t, bus)
    somethingThatShouldNotEmit()
    synctest.Wait()
    if err := eventbustest.ExpectExactly(tw); err != nil { // ждём пустой поток
        t.Errorf("ожидали тишину, получили %v", err)
    }
})
```

## 7. Чеклист реализации

- [ ] Один `pump`-goroutine владеет роутингом; горячий путь — без локов.
- [ ] Публичный API типизирован, но в мапах/интерфейсах шины лежит **non-generic
      ядро**, не `Foo[T]`. Регистрируешь `core`, не фасад.
- [ ] `reflect.TypeFor[T]()` и `typ.String()` кэшируются один раз при создании,
      не зовутся на каждое событие.
- [ ] Per-T стенсел — только вокруг типизированного `case ch <- t`; окружение из
      ядра.
- [ ] Входная очередь bounded; `acceptCh()` отдаёт `nil` при полной (nil-channel).
- [ ] Наборы cases в `pump` и `dispatch` синхронизированы (помечено комментарием).
- [ ] `queue.Drop` обнуляет снятый слот (GC).
- [ ] Отписка делает replace-on-write среза (`Clone`+`Delete`), не мутирует на месте.
- [ ] Snapshot/инспекция — через `chan chan []T`, не через локи чужого состояния.
- [ ] Debug-путь под `active()`-атомиком: выключенный дебаг бесплатен.
- [ ] `Close` шины каскадно закрывает клиентов/издателей/подписчиков; идемпотентно.
- [ ] Медленный подписчик логируется (не роняет шину); backpressure — это его баг.

## 8. Тестировать обязательно под race

Конкурентный код без race-детектора и многократного прогона недопроверен:

```bash
go test -race -count=1 ./<pkg>/
go test -count=10  ./<pkg>/   # ловит flaky-гонки
```

Минимальный набор сценариев: доставка в порядке публикации; подписчик получает
только свой тип; медленный подписчик создаёт backpressure, но не дедлок;
`Close` во время доставки (нет утечки горутин); отписка во время роутинга
(replace-on-write); snapshot под нагрузкой; проверка отсутствия событий через
`synctest`.

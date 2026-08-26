# Переиспользуемые concurrency-примитивы шины

Развёртка `SKILL.md` §3: код, инварианты и грабли. На шину примитивы не завязаны —
таскай их в любой конкурентный код. Читать, когда пишешь их вживую; для понимания
архитектуры шины хватает однострочников в §3.

## `worker` — обёртка жизненного цикла горутины

Горутина без владельца — утечка. `worker` связывает в одном значении три вещи,
которые иначе разъезжаются по структуре: контекст, отмену и факт завершения.

```go
type worker struct {
	ctx     context.Context
	stop    context.CancelFunc
	stopped chan struct{}
}

func runWorker(fn func(context.Context)) *worker {
	ctx, stop := context.WithCancel(context.Background())
	w := &worker{ctx: ctx, stop: stop, stopped: make(chan struct{})}
	go func() {
		defer close(w.stopped)
		fn(w.ctx)
	}()
	return w
}

func (w *worker) Stop() { w.stop() }

func (w *worker) Done() <-chan struct{} { return w.stopped }

func (w *worker) StopAndWait() {
	w.stop()
	<-w.stopped
}
```

`StopAndWait` — то, что зовёт `Bus.Close`: без ожидания `stopped` тест на утечку
горутин будет флачить.

## `stopFlag` — одноразовый идемпотентный сигнал остановки

Легче `context.Context`, когда нужно только «однократно щёлкнуть выключатель»: нет
дерева, нет значений, нет `Err()`. Канал создаётся лениво, поэтому нулевое значение
валидно; `Stop` можно звать многократно из разных горутин.

```go
type stopFlag struct {
	mu             sync.Mutex
	stopped        chan struct{}
	alreadyStopped bool
}

func (s *stopFlag) Stop() {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.alreadyStopped {
		return
	}
	s.alreadyStopped = true
	if s.stopped == nil {
		s.stopped = make(chan struct{})
	}
	close(s.stopped)
}

func (s *stopFlag) Done() <-chan struct{} {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.stopped == nil {
		s.stopped = make(chan struct{})
	}
	return s.stopped
}
```

Отдельный флаг `alreadyStopped` нужен именно из-за ленивого канала: по `s.stopped ==
nil` уже закрытое состояние от несозданного не отличить.

## `queue[T]` — generic ring-buffer под select-петлю

Очередь, из которой удобно отдавать голову в `select`: `Peek` смотрит, не снимая
(отправка может не состояться — тогда элемент остаётся), `Drop` снимает после
успешной отправки. Bounded (для backpressure) или unbounded (`capacity == 0`).
Компактизация сдвигом, без аллокаций на стабильном размере.

Методы согласованы через один предикат `canAppend` — он единственное место, где
живёт лимит.

```go
type queue[T any] struct {
	vals     []T
	start    int
	capacity int // 0 = безлимит
}

func (q *queue[T]) canAppend() bool { return q.capacity == 0 || len(q.vals) < q.capacity }

func (q *queue[T]) Empty() bool { return q.start == len(q.vals) }

func (q *queue[T]) Full() bool { return q.start == 0 && !q.canAppend() }

func (q *queue[T]) Peek() T {
	if q.Empty() {
		var zero T
		return zero
	}
	return q.vals[q.start]
}

func (q *queue[T]) Add(v T) {
	if !q.canAppend() {
		if q.start == 0 {
			panic("Add on a full queue") // недостижимо: вызывающий проверил Full()
		}
		n := copy(q.vals, q.vals[q.start:]) // сдвигаем хвост в начало
		clear(q.vals[n:])
		q.vals = q.vals[:n]
		q.start = 0
	}
	q.vals = append(q.vals, v)
}

func (q *queue[T]) Drop() {
	if q.Empty() {
		return
	}
	var zero T
	q.vals[q.start] = zero // обнуляем — не держим ссылку (GC)
	q.start++
	if q.Empty() {
		q.start = 0
		q.vals = q.vals[:0]
	}
}

func (q *queue[T]) Snapshot() []T { return slices.Clone(q.vals[q.start:]) }
```

Про `panic` в `Add`: правило владеет скилл `go-code-style`, и по нему это не «валидация
входных данных» — аргумент приходит не из запроса или конфига, а сама ветка недостижима,
пока вызывающий проверяет `Full()` (в шине это делает `select` с nil-каналом). Не
переноси такую панику на код, который принимает данные снаружи.

Три инварианта, которые легко сломать правкой:

- `Full()` — это «нельзя дописать И нельзя скомпактить», то есть `start == 0`. Если
  считать полнотой один `!canAppend()`, `Add` начнёт зря паниковать при непустом
  `start`, хотя место после сдвига есть.
- `Drop` обнуляет снятый слот. Иначе очередь держит ссылку на уже выданное значение
  и мешает GC — на событиях с большими payload это видно.
- `Snapshot` отдаёт копию (`slices.Clone`), а не `q.vals[q.start:]`: срез наружу
  утечёт в чужую горутину, а `Add` перезапишет его при следующем сдвиге.

## `Monitor` — zero-value-valid ожидание горутины

Обёртка над «запустил функцию, потом дождался». Смысл — в валидном нулевом значении:
код, который получил `Monitor` от выключенной шины, не обязан проверять его на nil.

```go
type Monitor struct {
	done <-chan struct{}
}

func (c *Client) Monitor(f func(*Client)) Monitor {
	done := make(chan struct{})
	go func() {
		defer close(done)
		f(c)
	}()
	return Monitor{done: done}
}

// Нулевое значение валидно: done == nil, и Wait на нём возвращается сразу. Канал
// наружу не отдаём — nil-канал у вызывающего блокировал бы навсегда.
func (m Monitor) Wait() {
	if m.done != nil {
		<-m.done
	}
}
```

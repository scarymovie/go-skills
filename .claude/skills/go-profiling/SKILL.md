---
name: go-profiling
description: >
  Профилирование и диагностика производительности Go-сервисов: подключение и защита pprof,
  CPU/heap/goroutine/mutex-профили, execution trace и flight recorder, бенчмарки с benchstat,
  PGO, pprof-метки, runtime/metrics. Готовые runbook'и под конкретный симптом.
  Profiling Go services: pprof, traces, benchmarks, PGO, leak hunting.
when_to_use: >
  Сервис ест CPU или память, растут горутины, скачет latency, надо померить эффект правки
  или поймать утечку. Триггеры: "профилирование", "pprof", "медленно работает", "утечка
  памяти", "горутины растут", "бенчмарк", "производительность", "тормозит", "profiling",
  "memory leak", "goroutine leak", "benchmark", "flame graph", "PGO", "latency spike".
---

# Профилирование Go-сервисов

Go 1.27, ниже не поддерживается (политика версий — скилл `go-quality`).

Порядок всегда один: снять профиль под нагрузкой → найти горячее место → починить →
доказать улучшение сравнением. Пункт «доказать» пропускают чаще всего, а без него
оптимизация неотличима от совпадения.

Флаги сборки и линтеры — в `go-quality`, тесты и `-race` — в `go-testing`.

## Подключение pprof

`net/http/pprof` регистрирует свои хендлеры в `http.DefaultServeMux` побочным эффектом
импорта. Поэтому его вешают на **отдельный сервер на отдельном порту**, а не в основной
mux: иначе `/debug/pprof/*` уедет наружу вместе с публичным API.

```go
import (
	"net/http"
	_ "net/http/pprof"
	"time"
)

func startPprof(addr string) {
	srv := &http.Server{
		Addr:              addr,
		Handler:           http.DefaultServeMux,
		ReadHeaderTimeout: 5 * time.Second,
	}
	go func() { _ = srv.ListenAndServe() }()
}
```

`ReadHeaderTimeout` здесь не формальность: без него сервер уязвим к Slowloris, а gosec
ругается на голый `http.ListenAndServe` (G114).

## Защита в проде

`/debug/pprof/*` отдаёт стеки, аргументы командной строки и куски памяти. Наружу — нельзя.
По возрастанию удобства:

1. **SSH-туннель** — `ssh -L 6060:localhost:6060 user@host`, слушаем только localhost.
   Ноль кода, ноль новой поверхности атаки. **Дефолт для прода.**
2. **Только внутренняя сеть** — слушать на внутреннем интерфейсе, наружу закрыто файрволом.
3. **Мидлварь с токеном** — если туннель невозможен. Оборачивать надо каждый хендлер
   (`pprof.Index`, `Cmdline`, `Profile`, `Symbol`, `Trace`), а не только корень: остальные
   зарегистрированы отдельно и мимо `Index` не проходят.
4. **Переключатель через конфиг** — включать по флагу, а не держать всегда открытым.

Сам факт наличия эндпоинта не стоит почти ничего. Стоит — снятие профиля: CPU-профиль
это несколько процентов на время съёмки, а вот `SetMutexProfileFraction(1)` и
`SetBlockProfileRate(1)` означают «сэмплировать каждое событие блокировки» и под нагрузкой
обходятся дорого. Их включают точечно и выключают обратно.

## Профили

| Профиль | Снять | Когда |
|---|---|---|
| CPU | `curl 'localhost:6060/debug/pprof/profile?seconds=30' -o cpu.prof` | высокое CPU, медленные запросы |
| Heap | `curl localhost:6060/debug/pprof/heap -o heap.prof` | рост памяти, OOM |
| Goroutine | `curl localhost:6060/debug/pprof/goroutine -o gr.prof` | утечка горутин, дедлок |
| Mutex | `curl localhost:6060/debug/pprof/mutex -o mu.prof` | конкуренция за локи |
| Block | `curl localhost:6060/debug/pprof/block -o blk.prof` | ожидание на каналах и I/O |
| Trace | `curl 'localhost:6060/debug/pprof/trace?seconds=5' -o trace.out` | паузы GC, задержки планировщика |

```bash
go tool pprof -http=:8080 cpu.prof   # веб-интерфейс: flame graph, граф вызовов
go tool pprof -top cpu.prof          # топ функций текстом
go tool pprof -base=old.prof new.prof # разница между двумя снимками
go tool trace trace.out              # для trace.out — своя смотрелка
```

Четыре величины в heap-профиле значат разное, и путать их дорого: `inuse_space` и
`inuse_objects` — что занято **сейчас** (ищем утечку), `alloc_space` и `alloc_objects` —
что аллоцировано **за всё время**, включая освобождённое (ищем давление на GC). Утечку
ищут по `inuse`, мусорящий горячий путь — по `-alloc_space`.

Текстовый дамп всех стеков горутин: `curl 'localhost:6060/debug/pprof/goroutine?debug=2'`.
Первая строка `?debug=1` — просто счётчик, ей удобно мониторить рост.

## Flight recorder — для того, что не поймать по запросу

Обычный trace надо успеть снять во время проблемы. Со скачком latency раз в час это
не работает. Flight recorder (GA с Go 1.25) держит в памяти скользящее окно трейса и
сбрасывает его на диск **по факту**, когда проблема уже случилась.

```go
import "runtime/trace"

fr := trace.NewFlightRecorder(trace.FlightRecorderConfig{
	MinAge:   5 * time.Second, // сколько истории держать
	MaxBytes: 8 << 20,         // потолок окна; имеет приоритет над MinAge
})
if err := fr.Start(); err != nil {
	return fmt.Errorf("flight recorder: %w", err)
}
defer fr.Stop()

// ... в обработчике, заметившем нарушение SLO:
var buf bytes.Buffer
if _, err := fr.WriteTo(&buf); err == nil {
	_ = os.WriteFile("slo-breach.trace", buf.Bytes(), 0o600)
}
```

Одновременно активен может быть только один flight recorder. Нулевые `MinAge`/`MaxBytes`
означают «на усмотрение рантайма» (порядка секунд).

## Метки pprof — привязать профиль к запросу

Профиль показывает функции, а вопрос обычно про «какой запрос/арендатор/звонок это ест».
`pprof.Do` вешает на горутину метки, и они наследуются порождёнными горутинами. Метки
видны и в CPU-профиле, и в профиле горутин.

```go
import "runtime/pprof"

pprof.Do(ctx, pprof.Labels("endpoint", "/call/accept", "tenant", tenantID),
	func(ctx context.Context) {
		handle(ctx)
	})
```

Разрез по метке в веб-интерфейсе — фильтр `tag`. Держи метки **низкокардинальными**:
`endpoint` — да, `request_id` — нет.

## Бенчмарки

```go
func BenchmarkParse(b *testing.B) {
	data := loadFixture(b)
	for b.Loop() {
		Parse(data)
	}
}
```

`for b.Loop()` (Go 1.24) — теперь единственная правильная форма. Она решает две проблемы
старого `for i := 0; i < b.N; i++`: сама сбрасывает таймер при первом вызове и
останавливает при выходе, поэтому **`b.ResetTimer()` больше не нужен**; и удерживает
аргументы и результат живыми, поэтому компилятор не имеет права выкинуть вызов целиком.
Не смешивать `b.Loop()` с циклом по `b.N` в одном бенчмарке.

```bash
go test -bench=. -benchmem ./...
go test -bench=BenchmarkParse -benchtime=10s ./pkg/
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof ./pkg/
```

`-benchmem` — практически обязателен: в Go аллокации чаще решают, чем такты.

Одиночный прогон ничего не доказывает — шум между запусками легко перекрывает эффект
правки. Сравнивать через benchstat, по нескольким прогонам:

```bash
go install golang.org/x/perf/cmd/benchstat@latest
go test -bench=. -benchmem -count=10 ./pkg/ > old.txt
# ... правка ...
go test -bench=. -benchmem -count=10 ./pkg/ > new.txt
benchstat old.txt new.txt
```

## PGO — профиль как вход компилятора

Собранный CPU-профиль полезен не только глазам. Положи его в корень main-пакета файлом
`default.pgo`, и `go build` подхватит его сам (`-pgo=auto` — дефолт с Go 1.21):
компилятор агрессивнее заинлайнит и девиртуализирует горячие пути. Типичный выигрыш —
единицы процентов без единой правки кода.

Профиль брать с прода под реальной нагрузкой; синтетический даст компилятору неверную
картину горячего.

## Метрики рантайма

Для чисел рантайма (GC, планировщик, куча) есть `runtime/metrics` — поддерживаемый
интерфейс со стабильными именами, в отличие от разбора `runtime.MemStats`:

```go
import "runtime/metrics"

samples := []metrics.Sample{
	{Name: "/gc/pauses:seconds"},
	{Name: "/sched/goroutines:goroutines"},
	{Name: "/memory/classes/heap/objects:bytes"},
}
metrics.Read(samples)
```

Наружу их отдавать через тот же экспортер, что и прикладные метрики (Prometheus/OTel).
`expvar` с `/debug/vars` — запасной вариант на ноль зависимостей: он не умеет гистограмм,
а p99 без гистограммы не посчитать.

Непрерывное профилирование (Pyroscope и аналоги) ловит регрессии, которые точечная
съёмка пропускает — профиль всегда есть за любой момент прошлого:

```go
import "github.com/grafana/pyroscope-go"

_, err := pyroscope.Start(pyroscope.Config{
	ApplicationName: cfg.App.Name,
	ServerAddress:   cfg.Pyroscope.Addr,
	ProfileTypes: []pyroscope.ProfileType{
		pyroscope.ProfileCPU,
		pyroscope.ProfileInuseSpace,
		pyroscope.ProfileAllocSpace,
	},
})
```

## Runbook по симптому

**Высокое CPU.** CPU-профиль 30 с под нагрузкой → `-http`. Смотреть горячие циклы,
неэффективные алгоритмы, лишние аллокации в горячем пути.

**Растёт память.** Heap-профиль сейчас и через интервал → `-base=heap1.prof heap2.prof`.
Разница по `inuse_space` показывает, что именно накапливается. Частые виновники:
неограниченный кеш, срез, который растят и никогда не переиспользуют, замыкание,
удерживающее большой объект.

**Растут горутины.** `curl '.../goroutine?debug=1' | head -1` — счётчик; если растёт
монотонно, снять профиль и посмотреть, на чём они стоят. Почти всегда: горутина без
способа остановиться, незакрытый канал, отсутствие отмены по контексту.

**Скачет latency.** Flight recorder, сброс по факту нарушения SLO. Если его нет — trace
5 с и надежда попасть. Искать паузы GC, задержки планировщика, блокирующие syscall.

**Конкуренция за локи.** `runtime.SetMutexProfileFraction(1)`, снять `mutex`, **вернуть
обратно в 0**. Смотреть, какой мьютекс держат дольше всего.

**Блокировки на каналах и I/O.** `runtime.SetBlockProfileRate(1)`, снять `block`,
вернуть в 0.

## Как не обмануть себя

- **Профилировать под нагрузкой и в среде, похожей на прод.** Профиль простаивающего
  приложения на ноутбуке не показывает ничего.
- **Сначала мерить, потом чинить.** 99% времени живёт в 1% кода, и это не тот код,
  на который думаешь.
- **Доказывать улучшение** через `-base` или benchstat, а не «стало быстрее по ощущениям».
- **Не верить микробенчмаркам** как модели реальной нагрузки: они не воспроизводят
  ни кеш-профиль, ни конкуренцию, ни давление на GC.
- **Смотреть на аллокации**, а не только на такты: давление на GC — самая частая
  причина «непонятных» просадок.

## Нагрузка

Профиль нужен под нагрузкой, значит нагрузку надо чем-то дать. Рабочие варианты:
`vegeta` (сценарии, несколько эндпоинтов, отчёты и графики), `k6` (сценарии на JS,
пороги прохождения), `oha` и `bombardier` (быстрый одиночный эндпоинт). `hey` встречается
в старых инструкциях, но давно не обновляется.

```bash
echo "GET http://localhost:8080/health" | vegeta attack -duration=30s -rate=100 | vegeta report
```

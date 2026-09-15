---
name: go-application-architecture
description: >
  Сборка Go-сервиса: тонкий main.go, структура Application как единственное место
  связывания зависимостей, пара Run/Shutdown с корректным кодом возврата и HTTP-стек
  проекта — net/http + http.ServeMux, мидлварь func(http.Handler) http.Handler.
  Wiring a Go service: main.go, Application struct, DI, Run/Shutdown lifecycle,
  HTTP stack and middleware shape.
when_to_use: >
  Когда пишешь или ревьюишь точку входа, связывание зависимостей, graceful shutdown,
  запуск воркеров, выбор HTTP-роутера или форму мидлвари. Триггеры: "main.go",
  "application struct", "dependency injection", "DI", "graceful shutdown",
  "run shutdown", "wire dependencies", "app wiring", "http router", "middleware",
  "запуск приложения", "точка входа", "связывание зависимостей", "роутер",
  "мидлварь", "код возврата".
---

# Архитектура Go-приложения

Go 1.27, ниже не поддерживается (политика версий — скилл `go-quality`).

HTTP-стек проекта — `net/http` + `http.ServeMux`, мидлварь `func(http.Handler) http.Handler`.
Это правило живёт здесь, остальные скиллы на него ссылаются.

## `cmd/app/main.go` — тонкий, без бизнес-логики

Обязанности: загрузить конфиг, поднять логгер, собрать `Application`, поймать сигналы,
оркестрировать `Run`/`Shutdown`.

`automaxprocs` не нужен: с Go 1.25 рантайм сам берёт `min(логические CPU, cgroup-лимит,
affinity mask)` и перечитывает лимит на ходу (выключено только при `go 1.24` и ниже в go.mod).

```go
func main() {
	cfg, err := config.Load("config.yaml")
	if err != nil {
		fmt.Fprintln(os.Stderr, "load config:", err) // логгера ещё нет: он настраивается из конфига
		os.Exit(1)
	}

	log := initLogger(cfg) // инициализация логгера вынесена в хелпер
	scarylog.SetDefault(log)
	log.Info("starting application")

	application, err := app.NewApplication(cfg, log)
	if err != nil {
		log.Error(fmt.Errorf("create application: %w", err))
		os.Exit(1)
	}

	rootCtx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	rootCtx = scarylog.ToContext(rootCtx, log)

	// Run возвращается сам: сигнал отменяет rootCtx, а Run гасит по нему сервер и воркеров.
	runErr := application.Run(rootCtx)
	if errors.Is(runErr, context.Canceled) {
		runErr = nil // штатная остановка по сигналу, не отказ
	}

	// Shutdown — только после возврата Run: он закрывает пул БД, нужный воркерам до конца.
	shutdownCtx, cancel := context.WithTimeout(context.WithoutCancel(rootCtx), cfg.App.ShutdownTimeout)
	shutdownErr := application.Shutdown(shutdownCtx)
	cancel()

	if err := errors.Join(runErr, shutdownErr); err != nil {
		log.Error(fmt.Errorf("application terminated: %w", err))
		os.Exit(1)
	}

	log.Info("application shutdown completed")
}
```

Правила:

- Любой отказ старта завершается `os.Exit(1)`. Голый `return` из `main` отдаёт код
  возврата 0, и упавшее приложение выглядит для оркестратора успешно завершённым:
  Kubernetes не считает это падением контейнера, CI красит деплой зелёным.
- `os.Exit` не выполняет `defer`. Поэтому всё, что нужно закрыть, закрывается в
  `Shutdown`, а не в `defer` внутри `main`; если `defer`-ов в `main` становится больше
  одного `stop()`, вынеси тело в `run() error` и оставь в `main` только `os.Exit(1)`.
- `panic` на старте формально тоже даёт ненулевой код, но правило про `panic` — в
  скилле `go-code-style`, и в лог вместо структурированной записи летит стек рантайма.
- Логгер — `*scarylog.Logger` (скилл `go-scarylog`). У `Error` ошибка идёт ПЕРВЫМ
  аргументом и её текст становится сообщением: `log.Error(fmt.Errorf("create application: %w", err))`,
  а не `log.Error("failed to create application", err)`. Стабильное сообщение для
  группировки — `ErrorMsg(msg, err, ...)`.
- `SetDefault` вызывается сразу после создания логгера, иначе `scarylog.FromContext`
  в местах без контекстного логгера отдаёт голый дефолт без атрибутов сервиса.
- `ShutdownTimeout` — из конфига, не константа в коде.
- Shutdown-контекст строится через `context.WithoutCancel(rootCtx)`: `rootCtx` в этот
  момент уже отменён, и таймаут, унаследованный от него, истёк бы мгновенно.

---

## `internal/app/` — единственное место связывания зависимостей

`Application` — обычная структура с полями верхнего уровня. `NewApplication` — единственное
место, где собираются зависимости: явные конструкторы, без глобальных фабрик и `init()`.

```go
type Application struct {
	server          *http.Server // либо несколько: httpServer, grpcServer
	workers         []worker.Worker
	db              *pgxpool.Pool
	shutdownTimeout time.Duration
}

func NewApplication(cfg *config.Config, log *scarylog.Logger) (*Application, error) {
	// 1. Инфраструктура: пул БД, внешние клиенты
	// 2. Репозитории (получают пул)
	// 3. Usecase (получают репозитории)
	// 4. Хендлеры (получают usecase)
	// 5. http.ServeMux, регистрация хендлеров
	// 6. http.Server поверх мидлварей
	// 7. Воркеры
	// 8. Вернуть Application: всё, что потом нужно останавливать, плюс shutdownTimeout
}
```

Правила:

- Зависимости текут сверху вниз через конструкторы: инфраструктура → репозиторий →
  usecase → хендлер. Обратных ссылок нет.
- Ни глобальных переменных, ни `init()`, ни реестров фабрик: порядок инициализации
  должен читаться сверху вниз в одной функции.
- Всё, что нужно останавливать или закрывать, лежит полем в `Application` — иначе
  до него не дотянется `Shutdown`.
- `shutdownTimeout` заполняется из `cfg.App.ShutdownTimeout`. Забыть его — тихая
  поломка: с нулевым значением стоппер в `Run` даёт серверу нулевой дедлайн, ловит
  `context.DeadlineExceeded` и роняет процесс с кодом 1 на каждом штатном SIGTERM.

---

## HTTP-стек: net/http + http.ServeMux

Роутер — `http.ServeMux` из stdlib. С Go 1.22 метод и wildcard пишутся прямо в паттерне,
а это и было единственной причиной тащить chi или gin; фреймворк добавляет зависимость
и собственный тип контекста, не давая взамен ничего.

```go
mux := http.NewServeMux()
mux.HandleFunc("POST /v1/orders/create", h.Create) // только POST: скилл openapi-rpc-conventions
mux.HandleFunc("POST /v1/orders/get", h.Get)       // чужой метод ServeMux сам закроет 405
```

Сам `http.Server` строит `config.NewServer(cfg.HTTP, handler)` из скилла `go-config` —
он же ставит таймауты, а `handler` здесь `chain(mux, scaryhttp.Middleware(log), authMW)`.
Таймаут на чтение не опционален: без него одно медленное соединение держит горутину и
дескриптор сколько угодно долго.

Мидлварь — только `func(http.Handler) http.Handler`, любая другая форма не собирается
в цепочку и не переиспользуется между сервисами:

```go
func chain(h http.Handler, mw ...func(http.Handler) http.Handler) http.Handler {
	for i := len(mw) - 1; i >= 0; i-- {
		h = mw[i](h)
	}
	return h
}
```

Первый в списке — внешний: он первым видит запрос и последним — ответ.

- Логирующая мидлварь — `scaryhttp.Middleware` из `go-scarylog`, не своя: она уже кладёт
  в контекст request-scoped логгер и переживает hijack для вебсокетов.
- Кодоген по OpenAPI — `std-http-server` + `strict-server` (настройка — в скилле
  `openapi-codegen`), чтобы сгенерированный сервер ложился на этот же стек.
- Если проект уже на echo — не мигрируем ради миграции: `echo.WrapMiddleware` принимает
  ту же `func(http.Handler) http.Handler`. Новые сервисы пишутся на `net/http`.

---

## `Run` и `Shutdown`

```go
func (a *Application) Run(ctx context.Context) error {
	g, ctx := errgroup.WithContext(ctx)

	g.Go(func() error {
		if err := a.server.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
			return err
		}
		return nil
	})

	// ListenAndServe про ctx не знает — гасим сервер отсюда.
	g.Go(func() error {
		<-ctx.Done()

		shutdownCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), a.shutdownTimeout)
		defer cancel()

		return a.server.Shutdown(shutdownCtx)
	})

	for _, w := range a.workers {
		g.Go(func() error { return w.Run(ctx) })
	}

	return g.Wait()
}

func (a *Application) Shutdown(ctx context.Context) error {
	var errs error

	if err := a.server.Shutdown(ctx); err != nil {
		errs = errors.Join(errs, err)
	}

	for i := len(a.workers) - 1; i >= 0; i-- {
		if err := a.workers[i].Stop(ctx); err != nil {
			errs = errors.Join(errs, err)
		}
	}

	a.db.Close()

	return errs
}
```

Правила:

- Горутина-стоппер обязательна. `ListenAndServe` блокируется до `Shutdown`/`Close` и про
  контекст ничего не знает: без неё отмена `ctx` — снаружи по сигналу или изнутри группы
  упавшим воркером — гасит воркеров, а `g.Wait()` продолжает ждать сервер, и `Run` не
  возвращается никогда. Тот же эффект даёт `context.AfterFunc`, но форма с `g.Go`
  нагляднее и её ошибка видна в `g.Wait()`.
- `Shutdown` вызывается только после возврата `Run`. Параллельно нельзя: `a.db.Close()`
  выдернет пул из-под ещё живых воркеров, те получат `closed pool`, и штатный SIGTERM
  превратится в ненулевой код возврата.
- Повторный `Shutdown` сервера (из стоппера и из `main`) безопасен: на уже остановленном
  сервере он возвращается сразу; заново выполняются только хуки `RegisterOnShutdown`.
- `Shutdown` останавливает воркеров в порядке, обратном инициализации, и закрывает пул БД
  последним: репозитории могут понадобиться воркерам на выходе.
- `Shutdown` собирает все ошибки через `errors.Join` и не выходит на первой: иначе
  недоостановленные компоненты утекут вместе с процессом.

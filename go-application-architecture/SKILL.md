---
name: go-application-architecture
description:
  Use this skill when writing or reviewing Go application wiring, main.go, Application struct, DI setup, Run/Shutdown lifecycle, or worker orchestration. Triggers on: "main.go", "application struct", "dependency injection", "DI", "graceful shutdown", "run shutdown", "wire dependencies", "app wiring", "запуск приложения".
---

# Go Application Architecture

## `cmd/app/main.go` — minimal, no business logic

Responsibilities: load config, init logger, build Application, handle OS signals, orchestrate Run/Shutdown.

```go
func main() {
cfg, err := config.LoadConfig("config.yaml")
if err != nil {
panic(err)
}

log := initLogger(cfg) // logger init extracted to a helper
log.Info("starting application")

application, err := app.NewApplication(cfg, log)
if err != nil {
log.Error("failed to create application", err)
return
}

rootCtx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()

rootCtx = logger.ToContext(rootCtx, log)

g, ctx := errgroup.WithContext(rootCtx)

g.Go(func () error {
return application.Run(ctx)
})

g.Go(func () error {
<-ctx.Done()
log.Info("shutdown signal received")

shutdownCtx, cancel := context.WithTimeout(context.Background(), cfg.App.ShutdownTimeout)
defer cancel()

return application.Shutdown(shutdownCtx)
})

if err = g.Wait(); err != nil && !errors.Is(err, context.Canceled) {
log.Error("application terminated with error", err)
return
}

log.Info("application shutdown completed")
}
```

**Key rules:**

- No business logic, no DI wiring in `main.go`
- `ShutdownTimeout` comes from config, not hardcoded
- Logger injected into context after creation, before passing to app
- `Run` and `Shutdown` are separate methods, orchestrated via errgroup

---

## `internal/app/` — DI wiring

`Application` is a plain struct that holds all top-level dependencies. `NewApplication` is the only place where
dependencies are wired — explicit constructors, no global factories or init functions.

```go
type Application struct {
server  *http.Server // or multiple: httpServer, grpcServer
workers []worker.Worker
db      *pgxpool.Pool // closed in Shutdown
}

func NewApplication(cfg *config.Config, log *slog.Logger) (*Application, error) {
// 1. Init infrastructure (db, external clients)
// 2. Init repositories (pass db)
// 3. Init usecases (pass repositories)
// 4. Init handlers (pass usecases)
// 5. Build router, attach handlers
// 6. Build http.Server
// 7. Build workers
// Return Application with all of the above
}
```

**Key rules:**

- Dependencies flow top-down via constructors: infra → repository → usecase → handler
- No global variables, no `init()`, no factory registries
- All components that need stopping are stored in `Application` fields

---

## `Run` and `Shutdown`

```go
func (a *Application) Run(ctx context.Context) error {
g, ctx := errgroup.WithContext(ctx)

g.Go(func () error {
if err := a.server.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
return err
}
return nil
})

for _, w := range a.workers {
w := w
g.Go(func () error { return w.Run(ctx) })
}

return g.Wait()
}

func (a *Application) Shutdown(ctx context.Context) error {
var errs error

if err := a.server.Shutdown(ctx); err != nil {
errs = errors.Join(errs, err)
}

// Stop workers in reverse order
for i := len(a.workers) - 1; i >= 0; i-- {
if err := a.workers[i].Stop(ctx); err != nil {
errs = errors.Join(errs, err)
}
}

a.db.Close()

return errs
}

```

**Key rules:**

- `Run` starts all components concurrently via errgroup
- `Shutdown` stops workers in reverse init order
- DB pool closed last, in `Shutdown`, not in `Run`
- `Shutdown` collects all errors via `errors.Join`, does not stop on first failure
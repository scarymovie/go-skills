# Устойчивое подключение к PostgreSQL на старте

Читать, когда пишешь инициализацию пула и хочешь, чтобы приложение переживало
недоступность базы в момент запуска (перезапуск пода раньше базы, failover, сетевой блип).

`pgxpool.NewWithConfig` **не подключается сразу** — пул создаётся лениво (prefill и
health-check уходят в фоновую горутину), фактический дозвон форсирует **первый `Ping()`**
(или первый запрос). Значит единственная «жадная» точка отказа — стартовый `Ping`.

- **В рантайме** пул самовосстанавливается: при следующем `Acquire` он передозванивается,
  а мёртвые соединения выбрасываются (`PrepareConn`/`AfterRelease`, см. ниже). Специальная
  обработка тут не нужна.
- **На старте** без повтора один временный `connection refused` (failover БД, под
  стартовал раньше базы) роняет процесс. Чинить нужно **только старт**.

Две вещи: (1) выставить таймаут одного дозвона — без него pgx подставляет дефолт
**2 минуты**, и «чёрная дыра» повесит воркер надолго; (2) повторять `Ping` с ограниченным
backoff'ом — переждать блип, но не висеть вечно (бюджет исчерпан → выходим, оркестратор
перезапустит под — прежнее поведение для реально лежащей БД).

```go
const (
	connectDialTimeout    = 5 * time.Second // потолок одного дозвона (вместо дефолтных 2 мин)
	connectInitialBackoff = 1 * time.Second
	connectMaxBackoff     = 5 * time.Second
	connectMaxWait        = 90 * time.Second // общий бюджет; обычный failover укладывается
)

func InitDB( /* cfg, logger */ ) (*pgxpool.Pool, error) {
	poolCfg, err := pgxpool.ParseConfig(dsn)
	// ... MaxConns/MinConns/MaxConnLifetime/... ...
	poolCfg.ConnConfig.ConnectTimeout = connectDialTimeout

	// NewWithConfig не дозванивается; Background — чтобы фоновый prefill/health-check
	// не отменился, когда InitDB вернётся (не передавай сюда короткий ctx с defer cancel).
	pool, err := pgxpool.NewWithConfig(context.Background(), poolCfg)
	if err != nil {
		return nil, err
	}
	if err := pingWithRetry(pool, logger); err != nil {
		pool.Close()
		return nil, err
	}
	return pool, nil
}

func pingWithRetry(pool *pgxpool.Pool, logger *scarylog.Logger) error {
	deadline := time.Now().Add(connectMaxWait)
	delay := connectInitialBackoff
	for attempt := 1; ; attempt++ {
		ctx, cancel := context.WithTimeout(context.Background(), connectDialTimeout)
		err := pool.Ping(ctx)
		cancel()
		if err == nil {
			return nil
		}
		if !isRetryableConnErr(err) { // неверный пароль/БД — падаем сразу, не жжём бюджет
			return err
		}
		if time.Now().Add(delay).After(deadline) {
			return err // бюджет исчерпан
		}
		logger.Warn("db connect failed, retrying", "attempt", attempt, "delay", delay, "error", err)
		time.Sleep(delay)
		delay = min(delay*2, connectMaxBackoff)
	}
}
```

**Классификатор временных ошибок** — повторяем только сеть/дозвон; на ошибках конфигурации
(логин/пароль/имя БД) падаем сразу, иначе бюджет 90с сгорит на опечатке в пароле:

```go
func isRetryableConnErr(err error) bool {
	var pgErr *pgconn.PgError
	if errors.As(err, &pgErr) {
		switch pgErr.Code {
		case "28P01", "28000", "3D000", "3F000": // пароль / authorization / БД / схема
			return false
		}
		return strings.HasPrefix(pgErr.Code, "08") || pgErr.Code == "57P03" // conn_exception / cannot_connect_now
	}
	var connErr *pgconn.ConnectError // оборачивает дозвон, в т.ч. "connection refused"
	if errors.As(err, &connErr) {
		return true
	}
	var netErr net.Error
	if errors.As(err, &netErr) {
		return true
	}
	return errors.Is(err, context.DeadlineExceeded)
}
```

> Ошибка вида `failed to connect to user=… database=…: … connect: connection refused` —
> это `*pgconn.ConnectError` (форматируется именно так). Ловится через `errors.As`.

**Гигиена соединений в пуле** — выбрасывает мёртвые соединения (напр. после failover'а)
на каждом `Acquire`, поэтому длинный `MaxConnLifetime` безопасен (pgx **v5.7.6+**):

```go
poolCfg.PrepareConn = func(ctx context.Context, conn *pgx.Conn) (bool, error) {
	if conn.IsClosed() {
		return false, nil // false = выбросить соединение, пул создаст новое
	}
	if err := conn.Ping(ctx); err != nil {
		return false, nil
	}
	return true, nil
}
poolCfg.AfterRelease = func(conn *pgx.Conn) bool {
	return !conn.IsClosed()
}
```

> Если в коде встретится `pgxpool.Config.BeforeAcquire` — это устаревший хук, его
> заменил `PrepareConn`. Когда заданы оба, `BeforeAcquire` игнорируется.

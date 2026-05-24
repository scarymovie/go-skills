---
name: go-config
description: >
  Конфигурация Go-приложений через YAML-файлы. Используй этот скилл при создании конфигов,
  загрузке настроек, добавлении новых параметров или при вопросах о конфигурации приложения.
  Триггеры: "config", "конфиг", "настройки", "yaml", "config.yaml", "загрузка конфига",
  "параметры приложения", "environment", "env".
---

# Go Config: YAML-файлы вместо переменных окружения

## Принцип

Конфигурация хранится в `config.yaml` и загружается в типизированные Go-структуры.
Никаких `os.Getenv`, никаких env-переменных, никаких `.env` файлов.

**Почему:**
- Типизация — ошибки видны при старте, а не в рантайме когда `os.Getenv` вернёт пустую строку
- Вложенность — YAML естественно группирует параметры, env требует плоских имён вроде `APP_HTTP_SERVER_READ_TIMEOUT`
- Один файл — все настройки в одном месте, не размазаны по десяткам переменных
- Дефолты — видны прямо в структуре, не надо писать `getEnvOrDefault` хелперы
- В контейнере `config.yaml` монтируется как volume

---

## Структура проекта

```
app/
├── config/
│   └── config.go              # Структуры и функция Load
├── config.example.yaml        # Шаблон конфига с плейсхолдерами (коммитится в git)
├── config.yaml                # Локальный конфиг для разработки (в .gitignore)
```

- `config.yaml` — рабочий конфиг, содержит реальные значения, **не коммитится** (добавить в `.gitignore`)
- `config.example.yaml` — шаблон с плейсхолдерами вместо секретов, **коммитится в git**
- При клонировании проекта: `cp config.example.yaml config.yaml` и заполнить реальные значения

В production `config.yaml` монтируется снаружи через docker volume:

```yaml
volumes:
  - /var/project/config.yaml:/app/config.yaml:ro
```

---

## Структуры конфига

Каждая секция — отдельная структура. Корневая структура `Config` собирает всё вместе.

```go
package config

import (
    "fmt"
    "os"
    "time"

    "gopkg.in/yaml.v3"
)

type Config struct {
    App        App        `yaml:"app"`
    HTTP       HTTP       `yaml:"http"`
    PostgreSQL PostgreSQL `yaml:"postgresql"`
}

type App struct {
    Name            string        `yaml:"name"`
    Env             string        `yaml:"env"`
    ShutdownTimeout time.Duration `yaml:"shutdownTimeout"`
}

type HTTP struct {
    Addr         string        `yaml:"addr"`
    ReadTimeout  time.Duration `yaml:"readTimeout"`
    WriteTimeout time.Duration `yaml:"writeTimeout"`
    IdleTimeout  time.Duration `yaml:"idleTimeout"`
}

type PostgreSQL struct {
    DSN               string        `yaml:"dsn"`
    MaxConns          int32         `yaml:"maxConns"`
    MinConns          int32         `yaml:"minConns"`
    MaxConnLifetime   time.Duration `yaml:"maxConnLifetime"`
    MaxConnIdleTime   time.Duration `yaml:"maxConnIdleTime"`
    HealthCheckPeriod time.Duration `yaml:"healthCheckPeriod"`
}
```

### Правила для структур

- Поля публичные — config не доменный объект, геттеры избыточны
- Теги `yaml:"camelCase"` — единый стиль в YAML-файле
- `time.Duration` — yaml.v3 парсит `30s`, `5m`, `1h` автоматически
- Вложенные структуры — `PostgreSQL`, `HTTP`, а не плоский список полей

---

## Загрузка

```go
func Load(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("read config: %w", err)
    }

    var cfg Config
    if err := yaml.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }

    return &cfg, nil
}
```

Использование в `main.go`:

```go
cfg, err := config.Load("config.yaml")
if err != nil {
    panic(err)
}
```

`panic` здесь допустим — приложение не может работать без конфига.

---

## config.example.yaml (коммитится в git)

Содержит плейсхолдеры вместо секретов, реальные значения для несекретных параметров:

```yaml
app:
  name: my-service
  env: development
  shutdownTimeout: 15s

http:
  addr: ":8080"
  readTimeout: 10s
  writeTimeout: 30s
  idleTimeout: 120s

postgresql:
  dsn: "postgres://USER:PASSWORD@HOST:5432/DATABASE?sslmode=disable"
  maxConns: 25
  minConns: 5
  maxConnLifetime: 1h
  maxConnIdleTime: 30m
  healthCheckPeriod: 1m
```

## config.yaml (локальная разработка, в .gitignore)

```yaml
app:
  name: my-service
  env: development
  shutdownTimeout: 15s

http:
  addr: ":8080"
  readTimeout: 10s
  writeTimeout: 30s
  idleTimeout: 120s

postgresql:
  dsn: "postgres://db_user:db_password@localhost:5432/db_database?sslmode=disable"
  maxConns: 25
  minConns: 5
  maxConnLifetime: 1h
  maxConnIdleTime: 30m
  healthCheckPeriod: 1m
```

### Правила для YAML

- Ключи в `camelCase` — совпадают с yaml-тегами в Go
- Длительности в человеко-читаемом формате: `30s`, `5m`, `1h`
- `config.yaml` — в `.gitignore`, никогда не коммитить
- `config.example.yaml` — всегда коммитить, плейсхолдеры вместо секретов
- При добавлении нового параметра — обновить оба файла

---

## Добавление новой секции

1. Создать структуру в `config.go`:

```go
type Redis struct {
    Addr     string `yaml:"addr"`
    Password string `yaml:"password"`
    DB       int    `yaml:"db"`
}
```

2. Добавить поле в `Config`:

```go
type Config struct {
    App        App        `yaml:"app"`
    HTTP       HTTP       `yaml:"http"`
    PostgreSQL PostgreSQL `yaml:"postgresql"`
    Redis      Redis      `yaml:"redis"`
}
```

3. Добавить секцию в `config.yaml`:

```yaml
redis:
  addr: "localhost:6379"
  password: ""
  db: 0
```

4. Использовать через `cfg.Redis.Addr` — типизированный доступ, автокомплит в IDE.

---

## Передача конфига в приложение

Конфиг передаётся в `NewApplication` и далее по цепочке зависимостей:

```go
func NewApplication(cfg *config.Config, log *scarylog.Logger) (*Application, error) {
    pool, err := postgres.NewPool(ctx, cfg.PostgreSQL)
    // ...
}
```

Каждый компонент получает только свою секцию:

```go
func NewPool(ctx context.Context, cfg config.PostgreSQL) (*pgxpool.Pool, error) {
    // ...
}

func NewServer(cfg config.HTTP, handler http.Handler) *http.Server {
    return &http.Server{
        Addr:         cfg.Addr,
        Handler:      handler,
        ReadTimeout:  cfg.ReadTimeout,
        WriteTimeout: cfg.WriteTimeout,
        IdleTimeout:  cfg.IdleTimeout,
    }
}
```

---

## Антипаттерны

```go
// ❌ os.Getenv — нет типизации, нет дефолтов, нет структуры
port := os.Getenv("APP_PORT")
if port == "" {
    port = "8080"
}

// ❌ Глобальная переменная конфига
var GlobalConfig Config

// ❌ Передача всего конфига туда, где нужна одна секция
func NewPool(cfg *config.Config) // нужен только cfg.PostgreSQL

// ❌ .env файлы и godotenv
godotenv.Load(".env")
os.Getenv("DB_HOST")

// ❌ Конфиг через флаги для десятков параметров
flag.String("db-host", "localhost", "")
flag.Int("db-port", 5432, "")
flag.String("db-user", "postgres", "")
// ...20 строк флагов
```

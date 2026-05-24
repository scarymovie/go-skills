---
name: docker-go
description: >
  Docker-образы и docker-compose для Go-проектов. Используй этот скилл при написании Dockerfile,
  docker-compose, настройке production/development окружений, или оптимизации Docker-сборки.
  Триггеры: "dockerfile", "docker", "docker-compose", "контейнер", "образ", "деплой",
  "production image", "сборка образа".
---

# Docker для Go-проектов

## Структура

```
docker/
├── images/
│   ├── app/
│   │   └── Dockerfile.production
│   └── codegen/
│       └── Dockerfile
├── docker-compose.production.yml
└── docker-compose.development.yml
```

---

## Production Dockerfile (multi-stage)

```dockerfile
ARG GO_VERSION=1.26
ARG ALPINE_VERSION=3.21

# --- Builder ---
FROM golang:${GO_VERSION}-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 go build -trimpath \
    -ldflags="-s -w" \
    -o /go/bin/app ./cmd/app

RUN CGO_ENABLED=0 go build -trimpath \
    -ldflags="-s -w" \
    -o /go/bin/migrate ./cmd/migration

# --- Runtime ---
FROM alpine:${ALPINE_VERSION}

RUN apk add --no-cache ca-certificates

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --from=builder /go/bin/app .
COPY --from=builder /go/bin/migrate .
COPY --from=builder /app/migrations ./migrations

USER appuser

EXPOSE 8080

CMD ["./app"]
```

### Если нужен Yandex Cloud CA-сертификат

Когда приложение подключается к Managed PostgreSQL или другим сервисам Yandex Cloud,
нужен их корневой сертификат. Добавить перед `USER appuser`:

```dockerfile
# --- Yandex Cloud CA (добавить перед USER appuser) ---
RUN wget -q "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
        -O /usr/local/share/ca-certificates/yandex-ca.crt && \
    update-ca-certificates
```

Если Yandex Cloud не используется — этот блок не нужен.

### automaxprocs

В контейнере Go по умолчанию видит все CPU хоста, а не лимиты из cgroup. Это приводит к избыточному
количеству goroutine и потреблению CPU сверх лимита. Всегда добавлять в `main.go`:

```go
import _ "go.uber.org/automaxprocs"
```

Пакет автоматически выставляет `GOMAXPROCS` по CPU-лимиту контейнера.

### Ключевые правила

| Правило | Зачем |
|---|---|
| `_ "go.uber.org/automaxprocs"` в main.go | Go видит все CPU хоста, не лимиты контейнера — без этого GOMAXPROCS завышен |
| `CGO_ENABLED=0` | Статическая линковка, бинарник работает в `alpine`/`scratch` без libc |
| `-trimpath` | Убирает абсолютные пути из бинарника — безопасность + воспроизводимость |
| `-ldflags="-s -w"` | Убирает debug info и symbol table — уменьшает размер |
| `COPY go.mod go.sum` перед `COPY . .` | Кеширование слоя с зависимостями — `go mod download` не перезапускается при изменении кода |
| `ALPINE_VERSION` через ARG | Фиксированная версия — воспроизводимые сборки |
| `USER appuser` один раз в конце | Контейнер не работает от root, не переключать user туда-обратно |

### Версия приложения через build arg

```dockerfile
ARG VERSION=0.0.0

RUN CGO_ENABLED=0 go build -trimpath \
    -ldflags="-s -w -X 'project/internal/version.Version=${VERSION}'" \
    -o /go/bin/app ./cmd/app
```

```bash
docker build --build-arg VERSION=$(git describe --tags --always) ...
```

---

## Codegen Dockerfile

Простой образ для кодогенерации, не для продакшена:

```dockerfile
ARG GO_VERSION=1.24

FROM golang:${GO_VERSION}-alpine

RUN go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest

WORKDIR /app

ENTRYPOINT ["oapi-codegen"]
```

---

## docker-compose.production.yml

```yaml
name: project

services:
  app:
    image: registry.example.com/project:${TAG}
    build:
      context: ../app
      dockerfile: ../docker/images/app/Dockerfile.production
      args:
        VERSION: ${VERSION}
    container_name: project-app
    restart: always
    volumes:
      - /var/project/config.yaml:/app/config.yaml:ro
    networks:
      - internal
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    logging:
      options:
        max-size: 10m
        max-file: "3"

  migrate:
    image: registry.example.com/project:${TAG}
    container_name: project-migrator
    command: ["./migrate", "up"]
    volumes:
      - /var/project/config.yaml:/app/config.yaml:ro
    networks:
      - internal
    depends_on:
      postgres:
        condition: service_healthy
    restart: "no"
    logging:
      options:
        max-size: 10m
        max-file: "3"

networks:
  internal:
    name: project_internal
```

### Ключевые правила

| Правило | Зачем |
|---|---|
| `healthcheck` на app | Позволяет другим сервисам ждать реальной готовности через `condition: service_healthy` |
| `migrate` зависит от `postgres` | Мигратору нужна БД, а не приложение |
| `condition: service_healthy` | Ждёт пока PostgreSQL примет подключения, а не просто стартует контейнер |
| Volumes с `:ro` | Config монтируется read-only — контейнер не может его перезаписать |
| `restart: "no"` для migrate | Мигратор выполняется один раз и завершается |
| `max-file: "3"` в кавычках | docker-compose ожидает строку, без кавычек может интерпретировать как число |

---

## docker-compose.development.yml

```yaml
name: project-development

services:
  postgres:
    image: postgres:17-alpine
    container_name: project-postgres-dev
    restart: unless-stopped
    environment:
      POSTGRES_USER: db_user
      POSTGRES_PASSWORD: db_password
      POSTGRES_DB: db_database
    networks:
      - internal
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U db_user -d db_database"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  internal:
    name: project_internal

volumes:
  postgres_data:
```

Только инфраструктурные зависимости. Приложение запускается локально через `go run`.

---

## Антипаттерны

```dockerfile
# ❌ Сборка без CGO_ENABLED=0 — может не работать в alpine
RUN go build -o /go/bin/app ./cmd/app

# ❌ alpine:latest — невоспроизводимые сборки
FROM alpine:latest

# ❌ Переключение USER туда-обратно
USER appuser
WORKDIR /app
USER root
RUN ...
USER appuser

# ❌ git в production builder (если не нужен для приватных модулей)
RUN apk add --no-cache git

# ❌ Миграция собирается без флагов оптимизации, а основное приложение — с ними
RUN go build -o /go/bin/migrate ./cmd/migration

# ❌ migrate зависит от app вместо postgres
depends_on:
  app:
    condition: service_healthy
```

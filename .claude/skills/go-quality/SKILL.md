---
name: go-quality
description: >
  Use this skill whenever a Go developer asks about code quality checks, linting,
  testing, build commands, or production builds for Go projects. Trigger on phrases like
  "как проверить код", "как собрать для продакшена", "go vet", "go build", "линтер",
  "race condition", "production build", "release build", "go fix", or any question about
  Go toolchain commands. Always apply Go 1.26 conventions and flags.
---

# Go 1.26 — Code Quality & Production Build

> Все команды рассчитаны на **Go 1.26**. Запускать из корня модуля.

---

## 1. Базовые проверки (перед каждым коммитом)

```bash
# Форматирование — приводит код к стандарту gofmt
go fmt ./...

# Авто-исправление устаревших API (безопасно, делает минимальные правки)
go fix ./...

# Статический анализ встроенным анализатором
go vet ./...

# Сборка всех пакетов — ловит ошибки компиляции без создания бинарника
go build ./...
```

---

## 2. Проверка гонок (race detector)

```bash
go build -race ./...
go test -race ./...
```

**Важно:**
- Требует CGO (`CGO_ENABLED=1`) и установленного GCC (на Windows — MinGW)
- Замедляет выполнение в 2–20x — только для dev/CI, не для production
- Увеличивает потребление памяти в 5–10x
- В Go 1.26 race detector поддерживает linux/amd64, darwin/amd64, darwin/arm64, windows/amd64

---

---

## 3. Production build

### Минимальный (рекомендуемый)

```bash
CGO_ENABLED=0 go build \
  -trimpath \
  -ldflags="-w" \
  -o ./bin/app \
  ./cmd/app
```

### Максимально оптимизированный (strip all debug)

```bash
CGO_ENABLED=0 go build \
  -trimpath \
  -ldflags="-w -s" \
  -o ./bin/app \
  ./cmd/app
```

### Пояснение флагов

| Флаг | Эффект | Когда использовать |
|------|--------|-------------------|
| `CGO_ENABLED=0` | Статическая линковка, нет зависимости от libc | Всегда для Docker/контейнеров |
| `-trimpath` | Убирает абсолютные пути из бинарника | Всегда — безопасность + воспроизводимые билды |
| `-ldflags="-w"` | Убирает DWARF debug info | Всегда в production |
| `-ldflags="-s"` | Убирает symbol table | Осторожно: теряются stack traces в логах |

> **Совет по `-s`:** если у тебя есть централизованный сбор логов (sentry, loki) и ты читаешь
> stack traces — подумай дважды перед `-s`. Потеря символов усложняет диагностику в prod.
> Для call center приложения с критичными SIP/RTP сессиями лучше оставить символы.

### Кросс-компиляция (пример: Linux бинарник из Windows)

```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build \
  -trimpath \
  -ldflags="-w -s -X main.version=$(git describe --tags --always)" \
  -o ./bin/app-linux \
  ./cmd/app
```

---

## 4. Рекомендуемый порядок перед релизом

```bash
go fmt ./...
go fix ./...
go vet ./...
go build -race ./...        # тесты с race detector
CGO_ENABLED=0 go build -trimpath -ldflags="-w -s" ./...
```
---
name: postgres-migrations
description: >
  Конвенции SQL-миграций asterisk-connector: goose-именование, обязательный .down.sql,
  запрет IF EXISTS-гардов, явные имена constraint'ов, индексы на каждый FK и — главное —
  правила размерности VARCHAR(N) вместо голого TEXT со справочником по enum-колонкам схемы.
  Используй этот скилл, когда пишешь или ревьюишь миграцию, добавляешь колонку/таблицу,
  выбираешь тип строковой колонки или откатываешь схему. Триггеры: "миграция", "migration",
  "goose", "up.sql", "down.sql", "VARCHAR", "TEXT колонка", "constraint", "индекс на FK",
  "откат схемы", "ALTER TABLE", "новая таблица".
---

# Миграции PostgreSQL

Файлы лежат в `migrations/` и следуют goose-именованию:
`NNN_description.up.sql` / `NNN_description.down.sql`.

## Конвенции

- **У каждого `.up.sql` обязан быть парный `.down.sql`**, даже если откат — no-op:
  создай пустой файл с комментарием, иначе раннер падает.
- Не используй гарды `IF NOT EXISTS` / `IF EXISTS` внутри миграций — каждая миграция
  выполняется ровно один раз, гарды прячут баги.
- Всегда именуй constraint'ы явно:
  `CONSTRAINT fk_<table>_<column> FOREIGN KEY ...`,
  `CONSTRAINT chk_<table>_<column> CHECK ...`.
  Имена таблиц в именах constraint'ов — в единственном числе.
- Добавляй B-tree индекс на каждую FK-колонку. PostgreSQL, в отличие от MySQL,
  **не** создаёт их автоматически; без индекса DELETE и JOIN уходят в seq scan.

## Размерность строковых колонок — `VARCHAR(N)`, никогда голый `TEXT`

| Случай | Правило | Пример |
|--------|---------|--------|
| UUID строкой | `VARCHAR(36)` | `accepted_by VARCHAR(36)` |
| Enum (с CHECK) | `VARCHAR(длина_самого_длинного_значения + 10)` | 'ONLINE'(6), 'OFFLINE'(7), 'PAUSE'(5) → `VARCHAR(17)` |
| Неизвестное / свободная форма | `VARCHAR(255)` по умолчанию — **спроси, если известна более тесная граница** | `name VARCHAR(255)` |

Ревьюя или сочиняя миграцию, отмечай каждый `VARCHAR(255)` на колонке, где вероятна
меньшая граница (`phone`, `device_username`, `external_id`), и спрашивай у пользователя
реальный максимум до коммита.

### Справочник размерностей enum'ов текущей схемы

| Колонка | Значения | VARCHAR |
|---------|----------|---------|
| `operator.status` | ONLINE(6) OFFLINE(7) PAUSE(5) | `VARCHAR(17)` |
| `*.status` (звонки) | pending(7) accepted(8) active(6) ended(5) canceled(8) | `VARCHAR(18)` |
| `active_call.call_direction` | inbound(7) outbound(8) | `VARCHAR(18)` |
| `*.cancel_reason` (звонки) | client_hangup(13) … answered_elsewhere(18) remote_unavailable(18) — полный набор в `domain.AllCancelReasons()` | `VARCHAR(28)` |

## Откаты, теряющие данные

Если `.down.sql` восстанавливает колонку фиктивным значением (пустой строкой, `id::text`),
это не полноценный откат — задокументируй потерю прямо в файле миграции комментарием,
чтобы её не приняли за обратимую.

---
name: go-project-structure
description:
  Use this skill when creating a new Go project, scaffolding directories, or when the user asks how to organize a Go project. Triggers on: "create go project", "scaffold", "project structure", "new service", "create file", "создай файл", "структура проекта".
---

# Go Project Structure

## Root Layout

```
project/
├── api/                  # OpenAPI/proto specs + generated code
├── app/                  # All Go source code
├── docker/               # All Dockerfiles (deploy, codegen, tooling, etc.)
├── test/                 # Test files (e.g. http/ for .http client files)
├── Makefile
└── .ci.yaml
```

**Key conventions:**

- Go code lives in `app/`, not in root
- `docker/` is for ALL Dockerfiles — not just deployment (codegen containers, base images, etc.)
- `api/` holds source-of-truth specs; generated code goes alongside specs

---

## `app/` — Go Source

```
app/
├── cmd/
│   ├── app/              # Main entrypoint
│   └── migration/        # Migration runner entrypoint
├── config/               # Config structs and loading
├── internal/
│   ├── app/              # App wiring (DI, startup)
│   ├── domain/
│   │   ├── api/          # Shared usecase types: filter, pagination, sort, search_result
│   │   │   └── filter/   # Split into subfolders when entities grow
│   │   ├── model/        # Pure domain entities, no dependencies
│   │   └── repository/   # Repository interfaces (implementations in infrastructure)
│   ├── usecase/          # Business logic. May contain model/ and mapper/ subfolders
│   ├── infrastructure/
│   │   ├── http/
│   │   │   └── <name>/   # One folder per server module, named after the api yaml file
│   │   │       ├── api/      # Generated server interfaces
│   │   │       ├── handler/  # HTTP handlers implementing the interfaces
│   │   │       └── mapper/   # Request/response mappers
│   │   ├── clients/
│   │   │   └── <name>/   # External HTTP clients, free internal structure
│   │   ├── postgres/     # Repository implementations
│   │   └── db/           # Connection to database
│   ├── version/
│   └── worker/
└── migrations/           # SQL migration files
```

### Key conventions

- `domain/api/` holds shared input types used across usecases (filters, pagination, sorting). Split into subfolders per
  type when the project grows.
- `domain/model/` — pure entities, zero external dependencies
- `domain/repository/` — interfaces only; implementations live in `infrastructure/postgres/`
- `usecase/` replaces "service" — more accurate name, can have its own dto/ and mapper/
- `infrastructure/http/<name>/` — for HTTP server modules. Name matches the OpenAPI yaml filename in `api/`
- `infrastructure/clients/<name>/` — for external HTTP clients. Minimal structure, free to grow as needed

---

## `docker/` — All Dockerfiles

```
docker/
└── images/
    ├── app/              # Production app image (often Alpine-based)
    ├── codegen/          # Code generation container
    │   ├── Dockerfile
    │   └── oapi-config/  # oapi-codegen configs live here, next to the Dockerfile
    └── <other>/          # redis, nextjs, etc. — one subfolder per image
```

**Convention:** one subfolder per image under `docker/images/`. Config files for a Dockerfile (e.g. codegen configs)
live in the same subfolder.

---

## `api/` — Specs

```
api/                      # May contains subfolders for pb/openapi files
├── <name>.yaml           # OpenAPI spec. Name matches infrastructure/http/<name>/
└── <name>.pb             # protobaff spec. Name matches infrastructure/grpc/<name>/
```

---

## `test/` — Tests

```
test/
└── http/                 # .http files for API testing (JetBrains HTTP Client, Bruno, etc.)
```
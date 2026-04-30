---
name: openapi-rpc-conventions
description: Use this skill when writing or reviewing OpenAPI specs, designing HTTP endpoints, naming request/response schemas, or structuring API contracts. Triggers on: "openapi", "api spec", "endpoint", "operation", "request schema", "response schema", "openapi conventions", "апи", "эндпоинт".
---

# OpenAPI RPC Conventions

## Core Philosophy

All HTTP endpoints follow an **RPC-over-HTTP** style. No GET, PUT, PATCH, DELETE. No query parameters, no path parameters — if input exists, it always goes in the request body.

---

## Endpoints

- **All endpoints use POST**
- Request body is optional — endpoints without input are valid
- No query parameters, no path parameters

---

## Naming Conventions

### Operations
OperationId should be a clear verb+noun with first letter in lower case: `createDialog`, `getUserList`, `searchMessages`.

### Request / Response schemas
Every endpoint has:
- `<OperationId>Input` — request body schema
- `<OperationId>Output` — response body schema

```yaml
# Example
/dialog/create:
  post:
    operationId: CreateDialog
    requestBody:
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/CreateDialogInput'
    responses:
      '200':
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateDialogOutput'
```

### Reusable parameter structs — `*Params` suffix
Shared input structures that represent a group of related parameters use a `Params` suffix.

```yaml
PaginationParams:
  type: object
  properties:
    page: { type: integer }
    perPage: { type: integer }

SearchParams:
  type: object
  properties:
    query: { type: string }

UserSearchParams:       # entity-specific variant is also valid
  type: object
  properties:
    query: { type: string }
    role: { type: string }
```

These are embedded inside `*Input` schemas, not used as top-level request bodies.

### Network DTO structs — `Nwk*` prefix
All other shared schemas that represent data transferred over the wire use a `Nwk` (Network) prefix.

```yaml
NwkDialog:
  type: object
  properties:
    id: { type: string }
    title: { type: string }

NwkUser:
  type: object
  properties:
    id: { type: string }
    name: { type: string }
```

**Key rule:** `Nwk*` types are wire-format DTOs only. They must not be used in usecase or domain layers — mappers in `infrastructure/http/<n>/mapper/` are responsible for converting between `Nwk*` and domain models.

---

## Error handling

All endpoints return errors in a unified format using `NwkError`:

```yaml
NwkError:
  type: object
  required:
    - code
    - message
  properties:
    code:
      type: string
      description: Machine-readable error code
    message:
      type: string
      description: Human-readable error message
```

Standard HTTP status codes:
- `400` — validation error, bad input (use `NwkError`)
- `500` — internal server error (use `NwkError`)

Example endpoint with error responses:

```yaml
/dialog/create:
  post:
    operationId: CreateDialog
    requestBody:
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/CreateDialogInput'
    responses:
      '200':
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateDialogOutput'
      '400':
        description: Validation error
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/NwkError'
      '500':
        description: Internal server error
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/NwkError'
```

---

## Schema naming summary

| Schema type | Convention | Example |
|---|---|---|
| Request body | `<OperationId>Input` | `CreateDialogInput` |
| Response body | `<OperationId>Output` | `CreateDialogOutput` |
| Reusable param group | `<Entity?>Params` | `PaginationParams`, `UserSearchParams` |
| Wire DTO | `Nwk<Entity>` | `NwkDialog`, `NwkUser` |
| Error response | `NwkError` | Always the same for 400/500 |
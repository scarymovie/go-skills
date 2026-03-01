---
name: go-code-style
description:
    Use this skill when writing, reviewing, or refactoring Go code. Covers idiomatic Go patterns, error handling, goroutines, struct initialization, and style conventions. Triggers on: "code review", "go style", "refactor", "idiomatic go", "code conventions", "напиши го код", "код ревью".
---

# Go Code Style

This skill defines code style conventions for Go projects. Full reference with examples is in `references/style-guide.md` — read it when you need examples or deeper context on a specific rule.

## Quick Reference

### Interfaces
- Never use pointer to interface — pass interfaces as values
- Verify interface compliance at compile time: `var _ http.Handler = (*Handler)(nil)`

### Globals and Init
- No mutable globals — use dependency injection
- Avoid `init()` — use explicit constructors instead
- No goroutines in `init()`

### Panics
- No `panic` in production code — return errors instead
- Exception: program startup for truly unrecoverable states

### Errors
- Use `errors.New` for static messages, `fmt.Errorf` for dynamic
- Use `%w` to wrap errors (allows `errors.Is` / `errors.As`), `%v` to hide underlying error
- Keep error messages concise: `"new store: %w"` not `"failed to create new store: %w"`
- Exported error vars: `ErrNotFound`; unexported: `errNotFound`; custom types: `NotFoundError`
- Handle each error once — don't log and return at the same time

### Goroutines and Channels
- Every goroutine must have a way to stop and a way to wait for it to exit
- Channel size: unbuffered or 1 — any other size needs justification
- Use `sync.WaitGroup` for multiple goroutines, `chan struct{}` for one

### Time
- Always use `time.Time` and `time.Duration`, never raw `int`
- If forced to use int (e.g. JSON), include unit in field name: `IntervalMillis`

### Performance
- Specify capacity for slices and maps when size is known: `make([]T, 0, size)`
- Convert `[]byte("string")` once outside loops, not on every iteration

### Enums
- Start int enums at 1 with `iota + 1` to avoid collision with zero value
- Exception: when zero value is a meaningful default

### Style
- Soft line length limit: 99 characters
- Group similar declarations with `()` — const, var, type, import
- Import groups: 3 groups separated by blank lines — stdlib / external / internal
- Package names: lowercase, no underscores, singular, no "util/common/shared"
- Don't shadow built-in names: `error`, `string`, `len`, etc.
- Sort functions in rough call order, group by receiver, exported first

### Nesting and Control Flow
- Use early return for errors and edge cases — reduce nesting
- Remove unnecessary `else` when `if` block has a `return`

### Structs and Variables
- Always use field names when initializing structs: `User{Name: "John"}` not `User{"John"}`
- Omit zero value fields unless they add meaningful context
- Use `var user User` for zero value structs instead of `user := User{}`
- Use `&T{Name: "foo"}` instead of `new(T)` for struct references
- Use `make(map[K]V)` for empty maps; map literal for fixed elements
- `nil` is a valid slice — return `nil` not `[]T{}`, check with `len(s) == 0`
- Declare variables close to their use, reduce scope where possible
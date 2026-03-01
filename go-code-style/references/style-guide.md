# Go Style Guide Reference

Based on Uber Go Style Guide, adapted with project-specific conventions.

---

## Interfaces

### Pointers to Interfaces
Never use a pointer to an interface. Pass interfaces as values — the underlying data can still be a pointer.

### Verify Interface Compliance
```go
// Bad
type Handler struct{}
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {}

// Good
type Handler struct{}
var _ http.Handler = (*Handler)(nil)
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {}
```

---

## Globals and Init

### Avoid Mutable Globals
```go
// Bad
var _timeNow = time.Now
func sign(msg string) string {
    now := _timeNow()
    return signWithTime(msg, now)
}

// Good
type signer struct {
    now func() time.Time
}
func newSigner() *signer {
    return &signer{now: time.Now}
}
func (s *signer) Sign(msg string) string {
    now := s.now()
    return signWithTime(msg, now)
}
```

### Avoid init()
```go
// Bad
var _config Config
func init() {
    raw, _ := os.ReadFile("config.yaml")
    yaml.Unmarshal(raw, &_config)
}

// Good
func loadConfig() Config {
    raw, err := os.ReadFile("config.yaml")
    // handle err
    var config Config
    yaml.Unmarshal(raw, &config)
    return config
}
```

---

## Panics

```go
// Bad
func run(args []string) {
    if len(args) == 0 {
        panic("an argument is required")
    }
}

// Good
func run(args []string) error {
    if len(args) == 0 {
        return errors.New("an argument is required")
    }
    return nil
}
```

Panic only on truly unrecoverable startup errors. In tests use `t.Fatal` instead of panic.

---

## Errors

### Error Types

| Error matching? | Message   | Use                                      |
|-----------------|-----------|------------------------------------------|
| No              | static    | `errors.New`                             |
| No              | dynamic   | `fmt.Errorf`                             |
| Yes             | static    | top-level `var` with `errors.New`        |
| Yes             | dynamic   | custom `error` type                      |

### Error Wrapping
```go
// Bad — "failed to" piles up in stack traces
return fmt.Errorf("failed to create new store: %w", err)
// produces: failed to x: failed to y: failed to create new store: the error

// Good
return fmt.Errorf("new store: %w", err)
// produces: x: y: new store: the error
```

Use `%w` when caller should be able to match with `errors.Is`/`errors.As`.
Use `%v` to hide underlying error from callers.

### Error Naming
```go
// Exported error variables
var (
    ErrBrokenLink   = errors.New("link is broken")
    ErrCouldNotOpen = errors.New("could not open")
)

// Unexported
var errNotFound = errors.New("not found")

// Custom type — suffix Error
type NotFoundError struct {
    File string
}
func (e *NotFoundError) Error() string {
    return fmt.Sprintf("file %q not found", e.File)
}
```

### Handle Errors Once
```go
// Bad — logs and returns
u, err := getUser(id)
if err != nil {
    log.Printf("Could not get user %q: %v", id, err)
    return err
}

// Good — wrap and return
u, err := getUser(id)
if err != nil {
    return fmt.Errorf("get user %q: %w", id, err)
}

// Good — log and degrade gracefully (when operation is non-critical)
if err := emitMetrics(); err != nil {
    log.Printf("Could not emit metrics: %v", err)
}
```

---

## Goroutines and Channels

### Channel Size
```go
// Bad
c := make(chan int, 64)

// Good
c := make(chan int, 1) // or unbuffered
c := make(chan int)
```

### Don't Fire-and-Forget
```go
// Bad — no way to stop
go func() {
    for {
        flush()
        time.Sleep(delay)
    }
}()

// Good
var (
    stop = make(chan struct{})
    done = make(chan struct{})
)
go func() {
    defer close(done)
    ticker := time.NewTicker(delay)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            flush()
        case <-stop:
            return
        }
    }
}()
close(stop)
<-done
```

### Wait for Goroutines
```go
// Multiple goroutines
var wg sync.WaitGroup
for i := 0; i < N; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        // ...
    }()
}
wg.Wait()

// Single goroutine
done := make(chan struct{})
go func() {
    defer close(done)
    // ...
}()
<-done
```

---

## Time

```go
// Bad
func poll(delay int) {
    time.Sleep(time.Duration(delay) * time.Millisecond)
}
poll(10) // seconds or milliseconds?

// Good
func poll(delay time.Duration) {
    time.Sleep(delay)
}
poll(10 * time.Second)

// When forced to use int in JSON
type Config struct {
    IntervalMillis int `json:"intervalMillis"` // unit in name
}
```

---

## Performance

### Container Capacity
```go
// Slice
data := make([]int, 0, size) // pre-allocate

// Map
m := make(map[string]os.DirEntry, len(files))
```

### Byte Conversion
```go
// Bad
for i := 0; i < b.N; i++ {
    w.Write([]byte("Hello world"))
}

// Good
data := []byte("Hello world")
for i := 0; i < b.N; i++ {
    w.Write(data)
}
```

---

## Enums

```go
// Bad — Add=0 collides with zero value
type Operation int
const (
    Add Operation = iota
    Subtract
)

// Good
const (
    Add Operation = iota + 1
    Subtract
    Multiply
)

// Exception: zero value is meaningful default
type LogOutput int
const (
    LogToStdout LogOutput = iota // 0 is fine here
    LogToFile
)
```

---

## Style

### Import Groups
Three groups separated by blank lines:
```go
import (
    "fmt"
    "os"

    "github.com/go-chi/chi/v5"
    "github.com/jackc/pgx/v5"

    "myproject/internal/domain"
    "myproject/internal/usecase"
)
```

### Group Similar Declarations
```go
const (
    a = 1
    b = 2
)

var (
    a = 1
    b = 2
)
```

### Function Ordering
```go
// Good
type something struct{ ... }

func newSomething() *something { ... }  // constructor after type

func (s *something) Cost() int { ... }  // methods grouped by receiver

func (s *something) Stop() { ... }

func calcCost(n []int) int { ... }      // utility functions last
```

### Reduce Nesting
```go
// Bad
for _, v := range data {
    if v.F1 == 1 {
        v = process(v)
        if err := v.Call(); err == nil {
            v.Send()
        } else {
            return err
        }
    } else {
        log.Printf("Invalid v: %v", v)
    }
}

// Good
for _, v := range data {
    if v.F1 != 1 {
        log.Printf("Invalid v: %v", v)
        continue
    }
    v = process(v)
    if err := v.Call(); err != nil {
        return err
    }
    v.Send()
}
```

### Unnecessary Else
```go
// Bad
var a int
if b {
    a = 100
} else {
    a = 10
}

// Good
a := 10
if b {
    a = 100
}
```

---

## Structs and Variables

### Initialization
```go
// Always use field names
k := User{
    FirstName: "John",
    LastName:  "Doe",
}

// Zero value struct
var user User  // not user := User{}

// Struct reference
sptr := &T{Name: "bar"}  // not new(T)
```

### Maps
```go
// Empty map — use make
m := make(map[string]int)

// Fixed elements — use literal
m := map[string]int{
    "a": 1,
    "b": 2,
}
```

### Slices
```go
// Return nil, not empty slice
if x == "" {
    return nil  // not []int{}
}

// Check empty with len
if len(s) == 0 { ... }  // not s == nil

// Zero value slice
var nums []int  // not nums := []int{}
```

### Zero-value Mutexes
```go
// Bad
mu := new(sync.Mutex)

// Good
var mu sync.Mutex
```

### Don't Embed Types in Public Structs
```go
// Bad — leaks implementation details
type ConcreteList struct {
    *AbstractList
}

// Good — explicit delegation
type ConcreteList struct {
    list *AbstractList
}
func (l *ConcreteList) Add(e Entity) {
    l.list.Add(e)
}
```
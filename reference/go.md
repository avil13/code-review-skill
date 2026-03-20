# Go Code Review Guide

A code review checklist based on the official Go documentation, Effective Go, and community best practices.

## Quick Review Checklist

### Must-check items
- [ ] Errors handled correctly (not ignored, with context)
- [ ] Goroutines have a shutdown path (avoid leaks)
- [ ] `context` propagated and cancelled correctly
- [ ] Receiver type choice is sound (value vs pointer)
- [ ] Code formatted with `gofmt`

### Common issues
- [ ] Loop variable capture (Go < 1.22)
- [ ] Nil checks complete
- [ ] Maps initialized before use
- [ ] `defer` inside loops
- [ ] Variable shadowing

---

## 1. Error handling

### 1.1 Never ignore errors

```go
// ❌ Bad: ignoring the error
result, _ := SomeFunction()

// ✅ Good: handle the error
result, err := SomeFunction()
if err != nil {
    return fmt.Errorf("some function failed: %w", err)
}
```

### 1.2 Wrapping errors and context

```go
// ❌ Bad: loses context
if err != nil {
    return err
}

// ❌ Bad: %v drops the error chain
if err != nil {
    return fmt.Errorf("failed: %v", err)
}

// ✅ Good: use %w to preserve the chain
if err != nil {
    return fmt.Errorf("failed to process user %d: %w", userID, err)
}
```

### 1.3 Use errors.Is and errors.As

```go
// ❌ Bad: direct comparison (doesn't work with wrapped errors)
if err == sql.ErrNoRows {
    // ...
}

// ✅ Good: errors.Is (works with wrapped errors)
if errors.Is(err, sql.ErrNoRows) {
    return nil, ErrNotFound
}

// ✅ Good: errors.As for a specific type
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    log.Printf("path error: %s", pathErr.Path)
}
```

### 1.4 Custom error types

```go
// ✅ Recommended: sentinel errors
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
)

// ✅ Recommended: custom error with context
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}
```

### 1.5 Handle each error in one place

```go
// ❌ Bad: log and return (duplicate handling)
if err != nil {
    log.Printf("error: %v", err)
    return err
}

// ✅ Good: return only; let the caller decide
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}

// ✅ Or: log and handle locally (no return)
if err != nil {
    log.Printf("non-critical error: %v", err)
    // continue with fallback logic
}
```

---

## 2. Concurrency and goroutines

### 2.1 Avoid goroutine leaks

```go
// ❌ Bad: goroutine never exits
func bad() {
    ch := make(chan int)
    go func() {
        val := <-ch // blocks forever; nothing sends
        fmt.Println(val)
    }()
    // function returns; goroutine leaks
}

// ✅ Good: use context or a done channel
func good(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-ctx.Done():
            return // graceful shutdown
        }
    }()
}
```

### 2.2 Channel usage

```go
// ❌ Bad: send on nil channel (blocks forever)
var ch chan int
ch <- 1 // blocks forever

// ❌ Bad: send on closed channel (panic)
close(ch)
ch <- 1 // panic!

// ✅ Good: sender closes the channel
func producer(ch chan<- int) {
    defer close(ch) // sender owns close
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

// ✅ Good: receiver detects close
for val := range ch {
    process(val)
}
// or
val, ok := <-ch
if !ok {
    // channel closed
}
```

### 2.3 Using sync.WaitGroup

```go
// ❌ Bad: Add inside the goroutine
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1) // race!
        defer wg.Done()
        work()
    }()
}
wg.Wait()

// ✅ Good: Add before starting the goroutine
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        work()
    }()
}
wg.Wait()
```

### 2.4 Avoid capturing the loop variable (Go < 1.22)

```go
// ❌ Bad (Go < 1.22): captures loop variable
for _, item := range items {
    go func() {
        process(item) // all goroutines may see the same item
    }()
}

// ✅ Good: pass as argument
for _, item := range items {
    go func(it Item) {
        process(it)
    }(item)
}

// ✅ Go 1.22+: per-iteration variables fix this by default
```

### 2.5 Worker pool pattern

```go
// ✅ Recommended: cap concurrency
func processWithWorkerPool(ctx context.Context, items []Item, workers int) error {
    jobs := make(chan Item, len(items))
    results := make(chan error, len(items))

    // start workers
    for w := 0; w < workers; w++ {
        go func() {
            for item := range jobs {
                results <- process(item)
            }
        }()
    }

    // enqueue jobs
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    // collect results
    for range items {
        if err := <-results; err != nil {
            return err
        }
    }
    return nil
}
```

---

## 3. Using context

### 3.1 Context as the first parameter

```go
// ❌ Bad: context not first
func Process(data []byte, ctx context.Context) error

// ❌ Bad: storing context on a struct
type Service struct {
    ctx context.Context // don't do this
}

// ✅ Good: first parameter, named ctx
func Process(ctx context.Context, data []byte) error
```

### 3.2 Propagate; don't invent a new root context

```go
// ❌ Bad: new root context in the call chain
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := context.Background() // loses request context!
        process(ctx)
        next.ServeHTTP(w, r)
    })
}

// ✅ Good: take from the request and propagate
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        ctx = context.WithValue(ctx, key, value)
        process(ctx)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 3.3 Always call cancel

```go
// ❌ Bad: cancel not called
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
// missing cancel() can leak resources

// ✅ Good: defer cancel
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel() // call even on timeout
```

### 3.4 Respect context cancellation

```go
// ✅ Recommended: check context in long work
func LongRunningTask(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err() // context.Canceled or context.DeadlineExceeded
        default:
            // do a small chunk of work
            if err := doChunk(); err != nil {
                return err
            }
        }
    }
}
```

### 3.5 Distinguish cancellation reasons

```go
// ✅ Use ctx.Err() to tell why it stopped
if err := ctx.Err(); err != nil {
    switch {
    case errors.Is(err, context.Canceled):
        log.Println("operation was canceled")
    case errors.Is(err, context.DeadlineExceeded):
        log.Println("operation timed out")
    }
    return err
}
```

---

## 4. Interface design

### 4.1 Accept interfaces, return concrete types

```go
// ❌ Not ideal: accept a concrete type
func SaveUser(db *sql.DB, user User) error

// ✅ Recommended: accept an interface (decoupled, testable)
type UserStore interface {
    Save(ctx context.Context, user User) error
}

func SaveUser(store UserStore, user User) error

// ❌ Not ideal: return an interface
func NewUserService() UserServiceInterface

// ✅ Recommended: return a concrete type
func NewUserService(store UserStore) *UserService
```

### 4.2 Define interfaces at the consumer

```go
// ❌ Not ideal: huge interface in the implementation package
// package database
type Database interface {
    Query(ctx context.Context, query string) ([]Row, error)
    // ... 20 methods
}

// ✅ Recommended: minimal interface in the consumer package
// package userservice
type UserQuerier interface {
    QueryUsers(ctx context.Context, filter Filter) ([]User, error)
}
```

### 4.3 Keep interfaces small and focused

```go
// ❌ Not ideal: kitchen-sink interface
type Repository interface {
    GetUser(id int) (*User, error)
    CreateUser(u *User) error
    UpdateUser(u *User) error
    DeleteUser(id int) error
    GetOrder(id int) (*Order, error)
    CreateOrder(o *Order) error
    // ... more methods
}

// ✅ Recommended: small, focused interfaces
type UserReader interface {
    GetUser(ctx context.Context, id int) (*User, error)
}

type UserWriter interface {
    CreateUser(ctx context.Context, u *User) error
    UpdateUser(ctx context.Context, u *User) error
}

// compose
type UserRepository interface {
    UserReader
    UserWriter
}
```

### 4.4 Avoid abusing the empty interface

```go
// ❌ Not ideal: overusing interface{}
func Process(data interface{}) interface{}

// ✅ Recommended: generics (Go 1.18+)
func Process[T any](data T) T

// ✅ Recommended: a concrete interface
type Processor interface {
    Process() Result
}
```

---

## 5. Choosing receiver type

### 5.1 When to use pointer receivers

```go
// ✅ Mutating the receiver
func (u *User) SetName(name string) {
    u.Name = name
}

// ✅ Receiver holds sync primitives (e.g. sync.Mutex)
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// ✅ Large struct (avoid copy cost)
type LargeStruct struct {
    Data [1024]byte
    // ...
}

func (l *LargeStruct) Process() { /* ... */ }
```

### 5.2 When to use value receivers

```go
// ✅ Small, immutable struct
type Point struct {
    X, Y float64
}

func (p Point) Distance(other Point) float64 {
    return math.Sqrt(math.Pow(p.X-other.X, 2) + math.Pow(p.Y-other.Y, 2))
}

// ✅ Alias of a basic type
type Counter int

func (c Counter) String() string {
    return fmt.Sprintf("%d", c)
}

// ✅ Receiver is map, func, or chan (already reference-like)
type StringSet map[string]struct{}

func (s StringSet) Contains(key string) bool {
    _, ok := s[key]
    return ok
}
```

### 5.3 Stay consistent

```go
// ❌ Not ideal: mixed receiver styles
func (u User) GetName() string   // value
func (u *User) SetName(n string) // pointer

// ✅ Recommended: if any method needs a pointer, use pointers for all
func (u *User) GetName() string { return u.Name }
func (u *User) SetName(n string) { u.Name = n }
```

---

## 6. Performance

### 6.1 Preallocate slices

```go
// ❌ Not ideal: repeated growth
var result []int
for i := 0; i < 10000; i++ {
    result = append(result, i) // many reallocations and copies
}

// ✅ Recommended: preallocate when size is known
result := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    result = append(result, i)
}

// ✅ Or allocate length directly
result := make([]int, 10000)
for i := 0; i < 10000; i++ {
    result[i] = i
}
```

### 6.2 Avoid unnecessary heap allocation

```go
// ❌ May escape to heap
func NewUser() *User {
    return &User{} // escapes to heap
}

// ✅ Consider returning by value when appropriate
func NewUser() User {
    return User{} // may stay on stack
}

// Inspect escape analysis:
// go build -gcflags '-m -m' ./...
```

### 6.3 Reuse objects with sync.Pool

```go
// ✅ Recommended: sync.Pool for hot allocate/free paths
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func ProcessData(data []byte) string {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()

    buf.Write(data)
    return buf.String()
}
```

### 6.4 String building

```go
// ❌ Not ideal: += in a loop
var result string
for _, s := range strings {
    result += s // new string each time
}

// ✅ Recommended: strings.Builder
var builder strings.Builder
for _, s := range strings {
    builder.WriteString(s)
}
result := builder.String()

// ✅ Or strings.Join
result := strings.Join(strings, "")
```

### 6.5 Avoid interface{} type assertions on hot paths

```go
// ❌ interface{} on hot path
func process(data interface{}) {
    switch v := data.(type) { // assertion cost
    case int:
        // ...
    }
}

// ✅ Generics or concrete types on hot paths
func process[T int | int64 | float64](data T) {
    // type fixed at compile time; no runtime assertion
}
```

---

## 7. Testing

### 7.1 Table-driven tests

```go
// ✅ Recommended: table-driven tests
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 1, 2, 3},
        {"with zero", 0, 5, 5},
        {"negative numbers", -1, -2, -3},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

### 7.2 Parallel tests

```go
// ✅ Recommended: parallelize independent subtests
func TestParallel(t *testing.T) {
    tests := []struct {
        name  string
        input string
    }{
        {"test1", "input1"},
        {"test2", "input2"},
    }

    for _, tt := range tests {
        tt := tt // copy required on Go < 1.22
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // eligible for parallel run
            result := Process(tt.input)
            // assertions...
        })
    }
}
```

### 7.3 Mock via interfaces

```go
// ✅ Define interfaces for testing
type EmailSender interface {
    Send(to, subject, body string) error
}

// production implementation
type SMTPSender struct { /* ... */ }

// test mock
type MockEmailSender struct {
    SendFunc func(to, subject, body string) error
}

func (m *MockEmailSender) Send(to, subject, body string) error {
    return m.SendFunc(to, subject, body)
}

func TestUserRegistration(t *testing.T) {
    mock := &MockEmailSender{
        SendFunc: func(to, subject, body string) error {
            if to != "test@example.com" {
                t.Errorf("unexpected recipient: %s", to)
            }
            return nil
        },
    }

    service := NewUserService(mock)
    // test...
}
```

### 7.4 Test helpers

```go
// ✅ Mark helpers with t.Helper()
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper() // failures report caller line
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}

// ✅ Clean up with t.Cleanup()
func TestWithTempFile(t *testing.T) {
    f, err := os.CreateTemp("", "test")
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() {
        os.Remove(f.Name())
    })
    // test...
}
```

---

## 8. Common pitfalls

### 8.1 Nil slice vs empty slice

```go
var nilSlice []int     // nil, len=0, cap=0
emptySlice := []int{}  // not nil, len=0, cap=0
made := make([]int, 0) // not nil, len=0, cap=0

// ✅ JSON encoding differs
json.Marshal(nilSlice)   // null
json.Marshal(emptySlice) // []

// ✅ Recommended: initialize explicitly when you need `[]` in JSON
if slice == nil {
    slice = []int{}
}
```

### 8.2 Map initialization

```go
// ❌ Bad: uninitialized map
var m map[string]int
m["key"] = 1 // panic: assignment to entry in nil map

// ✅ Good: use make
m := make(map[string]int)
m["key"] = 1

// ✅ Or a literal
m := map[string]int{}
```

### 8.3 Defer in loops

```go
// ❌ Pitfall: defers run when the function returns
func processFiles(files []string) error {
    for _, file := range files {
        f, err := os.Open(file)
        if err != nil {
            return err
        }
        defer f.Close() // all files close at end of function!
        // process...
    }
    return nil
}

// ✅ Good: closure or extract a function
func processFiles(files []string) error {
    for _, file := range files {
        if err := processFile(file); err != nil {
            return err
        }
    }
    return nil
}

func processFile(file string) error {
    f, err := os.Open(file)
    if err != nil {
        return err
    }
    defer f.Close()
    // process...
    return nil
}
```

### 8.4 Slice backing array sharing

```go
// ❌ Pitfall: subslices share backing array
original := []int{1, 2, 3, 4, 5}
slice := original[1:3] // [2, 3]
slice[0] = 100         // mutates original!
// original is now [1, 100, 3, 4, 5]

// ✅ Good: copy when you need independence
slice := make([]int, 2)
copy(slice, original[1:3])
slice[0] = 100 // original unchanged
```

### 8.5 Substring memory retention

```go
// ❌ Pitfall: substring keeps whole backing array alive
func getPrefix(s string) string {
    return s[:10] // still references full s backing store
}

// ✅ Good: independent copy (Go 1.18+)
func getPrefix(s string) string {
    return strings.Clone(s[:10])
}

// ✅ Before Go 1.18
func getPrefix(s string) string {
    return string([]byte(s[:10]))
}
```

### 8.6 Interface nil gotcha

```go
// ❌ Gotcha: nil interface value
type MyError struct{}
func (e *MyError) Error() string { return "error" }

func returnsError() error {
    var e *MyError = nil
    return e // returned error is not nil!
}

func main() {
    err := returnsError()
    if err != nil { // true! interface{type: *MyError, value: nil}
        fmt.Println("error:", err)
    }
}

// ✅ Good: return nil explicitly
func returnsError() error {
    var e *MyError = nil
    if e == nil {
        return nil // explicit nil
    }
    return e
}
```

### 8.7 Comparing time

```go
// ❌ Not ideal: == on time.Time
if t1 == t2 { // can fail due to monotonic clock differences
    // ...
}

// ✅ Recommended: Equal
if t1.Equal(t2) {
    // ...
}

// ✅ Ordering
if t1.Before(t2) || t1.After(t2) {
    // ...
}
```

---

## 9. Code organization

### 9.1 Package naming

```go
// ❌ Not ideal
package common   // too vague
package utils    // too vague
package helpers  // too vague
package models   // grouped by kind, not behavior

// ✅ Recommended: name by feature
package user     // user-related behavior
package order    // order-related behavior
package postgres // PostgreSQL implementation
```

### 9.2 Avoid import cycles

```go
// ❌ Cycle
// package a imports package b
// package b imports package a

// ✅ Fix 1: shared types in a separate package
// package types (shared)
// package a imports types
// package b imports types

// ✅ Fix 2: decouple with interfaces
// package a defines interface
// package b implements it
```

### 9.3 Exported identifiers

```go
// ✅ Export only what callers need
type UserService struct {
    db *sql.DB // unexported
}

func (s *UserService) GetUser(id int) (*User, error) // exported
func (s *UserService) validate(u *User) error         // unexported

// ✅ internal/ limits importers to same module
// internal/database/... only importable from this module
```

---

## 10. Tools and checks

### 10.1 Essential tools

```bash
# formatting (required)
gofmt -w .
goimports -w .

# static analysis
go vet ./...

# race detector
go test -race ./...

# escape analysis
go build -gcflags '-m -m' ./...
```

### 10.2 Recommended linters

```bash
# golangci-lint (bundles many linters)
golangci-lint run

# Common checks
# - errcheck: unhandled errors
# - gosec: security
# - ineffassign: useless assignments
# - staticcheck: static analysis
# - unused: dead code
```

### 10.3 Benchmarks

```go
// ✅ Performance benchmark
func BenchmarkProcess(b *testing.B) {
    data := prepareData()
    b.ResetTimer() // reset after setup

    for i := 0; i < b.N; i++ {
        Process(data)
    }
}

// Run benchmarks:
// go test -bench=. -benchmem ./...
```

---

## References

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Go Common Mistakes](https://go.dev/wiki/CommonMistakes)
- [100 Go Mistakes](https://100go.co/)
- [Go Proverbs](https://go-proverbs.github.io/)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)

---
name: go
description: >
  Expert Go programming skill. Covers idiomatic Go — package design, error handling, interfaces, concurrency, testing, and
  project layout, current through Go 1.25. Use whenever Go code is written, reviewed, debugged, or
  refactored — any .go file, go.mod, CLI tool, or web service, and whenever the user mentions
  Go or golang, even if they don't ask for "idiomatic" code.
---

# Idiomatic Go: The Go Way

Idiomatic Go patterns for robust, efficient, maintainable apps.

## When to Activate

- Writing, reviewing, auditing, or refactoring Go code, especially Java/Spring Boot-like code
- Designing Go packages, modules, or APIs
- Choosing stdlib vs third-party libraries
- Questions about Go structure, errors, concurrency, or testing

## Core Principles

### 1. Clear is Better than Clever

Go favors readable, simple, obvious code over abstraction and cleverness. Rewrite functions whose control flow needs repeated reading.

```go
// Idiomatic: Direct, linear control flow. Note: no "Get" prefix — Go omits it.
func LookupUser(id string) (*User, error) {
    user, err := db.FindUser(id)
    if err != nil {
        return nil, fmt.Errorf("finding user %s: %w", id, err)
    }
    return user, nil
}
```

### 2. Make the Zero Value Useful

Design types with immediately usable zero values. Avoid constructor boilerplate. `sync.Mutex` and `bytes.Buffer` exemplify this.

```go
// Idiomatic: Ready to use immediately
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}
```

### 3. Return Early, Keep the Happy Path Left

Handle errors and edge cases immediately. Return early; avoid `else` for main logic. Keep happy path unindented.

### 4. Separate Logical Sections with Blank Lines

Use blank lines between logical sections: setup, work phases, cleanup. Group related statements; don't cram unrelated steps or separate every statement.

```go
func Run(cfg config.Config, deps Deps) error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
	defer stop()

	log := logging.Get("server")

	// Set up OpenTelemetry.
	otelShutdown, err := observability.Start(ctx, observability.Config{
		MetricsExportInterval: 5 * time.Second,
	})
	if err != nil {
		return fmt.Errorf("setup observability: %w", err)
	}
	defer func() {
		if shutdownErr := otelShutdown(context.Background()); shutdownErr != nil {
			log.Error("otel shutdown failed", "error", shutdownErr)
		}
	}()

	router := NewRouter(deps)
	handler := requestTimeoutHandler(router, cfg.Server.RequestTimeout)

	srv := &http.Server{
		Addr:         cfg.Server.Listen,
		Handler:      handler,
		ReadTimeout:  cfg.Server.ReadTimeout,
		WriteTimeout: cfg.Server.WriteTimeout,
	}

	srvErr := make(chan error, 1)
	go func() {
		log.Info("starting server", "listen", cfg.Server.Listen)
		srvErr <- srv.ListenAndServe()
	}()

	select {
	case err := <-srvErr:
		return fmt.Errorf("listen and serve: %w", err)
	case <-ctx.Done():
		log.Info("server terminated: stopping")
	}

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("shutdown server: %w", err)
	}

	return nil
}
```

Each blank line marks next code chunk's purpose. Keep cleanup `defer` beside its setup.

## Package Organization: Domain-First, Dependency-Grouped

**Anti-Pattern:** Grouping by technical layer/type: `models/`, `controllers/`, `handlers/`, `services/`, `repository/`. Go separates at package level, so shared technical packages mix domains and obscure ownership. This invites circular dependencies. Reject `utils/`, `helpers/`, and `common/` too.

Packages define bounded contexts. Delineate domains by top-level package, not filename.

### 1. Domain Types Depend on Nothing

Domain types and interfaces import nothing from app or adapters: no database driver, HTTP, or third-party SDK.

```go
// order/order.go — pure domain type + the interface consumers need. No imports
// from db, http, or any adapter package.
type Order struct {
    ID         int
    CustomerID int
    Total      Money
}

type Repository interface {
    Order(id int) (*Order, error)
    CreateOrder(o *Order) error
}
```

### 2. One Package Per Domain

Single-domain apps may keep types in root. Multiple distinct domains get separate top-level packages: `order` and `user` are separate bounded contexts, each import-free of the other.

```
mystore/
├── order/           # order domain types + interfaces, imports nothing internal
│   └── order.go
├── user/            # user domain types + interfaces, imports nothing internal
│   └── user.go
├── postgres/        # adapters, import order and user
├── http/            # adapters, import order and user
└── cmd/
    └── mystore/
        └── main.go
```

Split a domain when it can be described in one sentence without other domains. Domain packages never import each other sideways; use an interface for needed dependencies (see #4).

### 3. Group Adapters by Dependency, Not by Domain

Push database, HTTP API, or third-party dependencies into subpackages named after wrapped dependency: `postgres`, `stripe`, `http`. Subpackage implements domain interface:

```go
// postgres/order.go
package postgres

import (
    "database/sql"
    "mystore/order"
)

// Repository is a PostgreSQL implementation of order.Repository.
type Repository struct {
    DB *sql.DB
}

func (r *Repository) Order(id int) (*order.Order, error) {
    var o order.Order
    row := r.DB.QueryRow(`SELECT id, customer_id, total FROM orders WHERE id = $1`, id)
    if err := row.Scan(&o.ID, &o.CustomerID, &o.Total); err != nil {
        return nil, fmt.Errorf("scanning order %d: %w", id, err)
    }
    return &o, nil
}
```

Dependencies become swappable and implementations can layer, e.g. in-memory cache wrapping `postgres.Repository`, both satisfying `order.Repository`. Put `net/http` code in `http` adapter package, not domains.

One `http/` or `postgres/` package can span domains because `cmd/` wires adapters. Split per-domain (`http/order/`, `postgres/order/`) when large. Domains require package separation; adapters don't.

### 4. Wire Dependencies Behind Domain Interfaces, Not Concretely

When implementation needs another dependency, depend on domain interface, not concrete adapter:

```go
type Repository struct {
    DB                 *sql.DB
    TransactionService order.TransactionService // interface, not stripe.Service
}
```

Adapters stay independently swappable.

### 5. A Shared `mock` Subpackage for Testing

Hand-write simple domain-interface mocks in one `mock` subpackage; avoid mocking frameworks:

```go
// mock/order.go
package mock

import "mystore/order"

// Repository is a mock implementation of order.Repository.
type Repository struct {
    OrderFn      func(id int) (*order.Order, error)
    OrderInvoked bool
}

func (r *Repository) Order(id int) (*order.Order, error) {
    r.OrderInvoked = true
    return r.OrderFn(id)
}
```

Inject `&mock.Repository{OrderFn: ...}` wherever `order.Repository` is expected. Complements fakes/stubs and table-driven tests.

### 6. `main` Lives Under `cmd/`, and Only Wires

Use `cmd/<binary-name>/main.go` even for one binary; leaves room for more. `main` selects concrete adapters and wires domain interfaces; no business logic.

```go
// cmd/mystore/main.go
func main() {
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    orderRepo := &postgres.Repository{DB: db}

    var h http.Handler
    h.OrderRepository = orderRepo
    // start http server...
}
```

### Dependency Direction

Domains never import adapters or `cmd`. Adapters import domains. `cmd` imports and wires everything. Compiler rejects import cycles, catching domain→adapter mistakes:

```
+-----------+     +-----------+
|   order   |     |   user    |
+-----------+     +-----------+
       ^                ^
       |                |
+------------------------------+
| http              postgres   |
+------------------------------+
               ^
               |
          +---------+
          |   cmd   |
          +---------+
```

### On `internal/`

Use `internal/` when package must be shared within your module but hidden from outside modules. For standalone executables, skip extra depth unless boundary is needed.

## Interface Design

### 1. Interfaces are Discovered, Not Designed Upfront

Write concrete types first. Define interfaces when a consumer needs interchangeable implementations.

### 2. Define Interfaces Where They Are Used

Interfaces belong in consuming package, not implementing package. Decouples packages.

```go
// processor/processor.go

// Idiomatic: The consumer defines exactly what it needs.
// The concrete 'UserStore' doesn't even need to know this interface exists.
type UserFetcher interface {
    FetchUser(id string) (*User, error)
}

type Processor struct {
    fetcher UserFetcher
}
```

### 3. Accept Interfaces, Return Structs

Accept smallest useful interface (e.g. `io.Reader`, not `*os.File`); return concrete structs so callers avoid type assertions.

## Library API Design

### Domain Object as Entry Point

For libraries wrapping stateful resources (vault, database connection, config store), make resource struct primary object. Methods return domain sub-objects. **Avoid:**

- Passing a config/resource struct as the first argument to every package-level function
- Package-level global state (like pflag's default `FlagSet`) for library code that may be embedded

**Do this instead:**

```go
// Open the primary resource once
v, err := vault.Open(path)

// Domain operations are methods on the primary object
// Each call is stateless (reloads fresh) — no cached state on the struct
idx, err := v.People()             // returns *people.Index, error
note, err := v.Daily(time.Now())   // returns *daily.Note, error
mtgs, err := v.Meetings()          // returns *meetings.Index, error

// Domain types live in sub-packages — callers use type inference
p, err := idx.FindOne("Steve")     // *people.Person
```

**Why:**

- Primary struct (`Vault`) is single entry point; callers need one import
- Subpackages define rich domain types (`people.Person`, `daily.Note`) with owning logic
- No global state supports concurrency, multiple instances, and testing
- Stateless calls reload fresh, avoiding cache invalidation
- Type inference (`:=`) usually avoids explicit subpackage imports for declarations

**Global instance:** CLI-only tools (like `pflag`) with one instance where ease of use outweighs library correctness.

## Concurrency Patterns

**Anti-Pattern:** Heavy static worker pools. Go scheduler is efficient; don't manage workers like OS threads.

### 1. Share Memory by Communicating

Prefer channels for shared data when possible. Channels orchestrate; mutexes serialize.

### 2. Bounded Concurrency

Limit concurrency with `errgroup` and `SetLimit`; don't hand-roll semaphore channels or rigid pools.

```go
func FetchAll(ctx context.Context, urls []string, maxConcurrent int) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(maxConcurrent)

    for _, url := range urls {
        g.Go(func() error {
            return fetch(ctx, url)
        })
    }

    return g.Wait()
}
```

Loop variables are per-iteration since Go 1.22. Never emit old `url := url` capture.

Without error propagation, `sync.WaitGroup.Go` (Go 1.25) removes Add/Done boilerplate:

```go
var wg sync.WaitGroup
for _, url := range urls {
    wg.Go(func() { process(url) })
}
wg.Wait()
```

### 3. Never Start a Goroutine Without Knowing How It Stops

Every `go func()` needs clear exit condition, usually `context.Context` or closed channel.

## Context: Pass It, Don't Store It

`context.Context` carries cancellation, deadlines, and request-scoped values across API boundaries. Follow these strict rules:

### 1. `ctx` Is Always the First Parameter, Never Stored in a Struct

```go
// Idiomatic
func (s *Service) FetchUser(ctx context.Context, id string) (*User, error) {
    return s.db.QueryRowContext(ctx, "SELECT ...", id).Scan(...)
}

// Anti-pattern: storing ctx on a struct hides its lifetime and makes every
// method call use whatever context was current when the struct was built,
// not the context of the call actually being made.
type Service struct {
    ctx context.Context // never do this
}
```

The one narrow exception is `http.Request.Context()` — the request itself is the struct doing the storing, and that's a stdlib convention, not a pattern to copy elsewhere.

### 2. Propagate the Context You Were Given

If function receives `ctx`, pass it or derived context (`context.WithCancel`, `WithTimeout`, `WithDeadline`, `context.WithValue`) to downstream calls. Never replace it with `context.Background()`; that severs cancellation and deadlines.

```go
// Bad: caller's cancellation/deadline never reaches the query
func (s *Service) FetchUser(ctx context.Context, id string) (*User, error) {
    return s.db.QueryRowContext(context.Background(), "SELECT ...", id).Scan(...)
}

// Idiomatic: the ctx that was passed in keeps flowing
func (s *Service) FetchUser(ctx context.Context, id string) (*User, error) {
    return s.db.QueryRowContext(ctx, "SELECT ...", id).Scan(...)
}
```

Use `context.Background()` or `context.TODO()` only at call-chain roots: `main`, incoming request start, background-job start. Never inside function already receiving `ctx`.

### 3. Use `context.WithoutCancel` to Deliberately Detach

When background task must outlive spawning request, detach explicitly so values propagate but cancellation doesn't:

```go
// The background job should keep running even after the HTTP request context cancels.
bgCtx := context.WithoutCancel(requestCtx)
go doBackgroundWork(bgCtx)
```

Call site documents intent instead of hiding possible context bug.

### 4. `context.Value` Is for Request-Scoped Data, Not Optional Parameters

Use `WithValue` only for cross-boundary request data absent from natural signature: request ID, trace span, middleware-set auth principal. Regular dependencies belong as explicit parameters.

```go
// Idiomatic: request-scoped, cross-cutting, set by middleware, read deep in the call stack
ctx = context.WithValue(ctx, requestIDKey{}, reqID)

// Anti-pattern: this is a real parameter, not incidental request metadata
ctx = context.WithValue(ctx, userIDKey{}, userID) // FetchUser(ctx, userID) instead
```

Always use unexported key type (`type requestIDKey struct{}`), never `string` or exported type; prevents package collisions.

### 5. A `context.Context` Cancellation Doesn't Stop Execution — Check It

Passing `ctx` doesn't abort work automatically. Check `ctx.Err()` or select `ctx.Done()` before expensive work, in loops, and around blocking calls. Most stdlib I/O already respects it, e.g. `QueryRowContext` and context-carrying `http.Client` requests.

```go
for _, item := range items {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }
    process(item)
}
```

## Configuration and Struct Design

### Functional Options for Complex Initialization

For many optional config parameters, avoid massive constructors. Use Functional Options.

```go
type Server struct {
    addr    string
    timeout time.Duration
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:    addr,
        timeout: 30 * time.Second, // Sane default
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

## Error Handling

### 1. Errors are Values

Errors are values, not exceptions. Check explicitly.

### 2. Wrap for Context, Not for Stack Traces

Wrap returned errors with operation context.

```go
// Idiomatic
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("loading config file %s: %w", path, err)
}
```

### 3. Log or Return — Never Both

Logging handles an error. Log or return, never both at same layer; otherwise one failure pollutes logs at every layer.

```go
// Bad: logged here, then logged again (and again) by every caller that also logs before returning
data, err := os.ReadFile(path)
if err != nil {
    log.Error("failed to read config", "error", err)
    return fmt.Errorf("loading config file %s: %w", path, err)
}

// Good: wrap with context and return — let the one place that actually stops
// the error (handles it, doesn't propagate it) do the logging.
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("loading config file %s: %w", path, err)
}
```

Log where error is handled, not propagated: request-handler top, background-job outer loop, `main`. Below boundary, wrap and return. Exception: log lower-level nonfatal warnings while continuing, e.g. cache-write failure.

## Testing Patterns

**Anti-Pattern:** Heavy BDD frameworks (like Ginkgo) or generated mocks. Go tests should stay Go.

### 1. Table-Driven Tests

Go unit-test standard: table of inputs/expected outputs, iterated with `t.Run()`.

```go
func TestParseConfig(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {"valid config", "port=8080", false},
        {"invalid format", "port=abc", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := ParseConfig(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("ParseConfig() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### 2. Meaningful Helpers with `t.Helper()`

Call `t.Helper()` in assertion helpers so failures point to test case, not helper.

### 3. Fakes and Stubs over Heavy Mocks

Use implicit interfaces and manual fakes. Keeps dependencies light and tests clear.

### 4. Golden Files and the `testdata` Directory

Use `testdata` for complex inputs or large outputs; `go test` ignores it.

### 5. Filesystem Abstraction (The Afero Pattern)

Don't hardcode `os` calls in business logic. Accept filesystem interface for in-memory tests. `github.com/spf13/afero` is standard.

```go
import "github.com/spf13/afero"

type FileProcessor struct {
    fs afero.Fs
}

func NewFileProcessor(fs afero.Fs) *FileProcessor {
    return &FileProcessor{fs: fs}
}
```

Inject `afero.NewMemMapFs()` in tests to eliminate disk I/O and flaky, slow tests.

### 6. `cmp` over DeepEqual

Compare complex structs/maps with `github.com/google/go-cmp/cmp`, not strict `reflect.DeepEqual`.

### 7. Modern `testing` Additions (Go 1.24+)

```go
// Test-scoped context — canceled automatically when the test ends.
// Use instead of context.Background() in tests.
func TestFetch(t *testing.T) {
    ctx := t.Context()
    ...
}

// Benchmarks: b.Loop() replaces the classic b.N loop — more accurate,
// prevents the compiler from optimizing the benchmarked call away.
func BenchmarkParse(b *testing.B) {
    for b.Loop() {
        Parse(input)
    }
}

// t.Chdir(dir) changes working directory for the test and restores it after.
```

### 8. `testing/synctest` for Concurrent Code (Go 1.25)

Test goroutines and timeouts deterministically with a fake clock — no `time.Sleep` in tests, no flaky timing assumptions:

```go
func TestTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        ctx, cancel := context.WithTimeout(t.Context(), time.Second)
        defer cancel()
        time.Sleep(2 * time.Second) // instant — fake clock advances when all goroutines block
        if ctx.Err() == nil {
            t.Fatal("expected timeout")
        }
    })
}
```

Never write `time.Sleep(100 * time.Millisecond)` to "wait for a goroutine" in a test. Use synctest, channels, or explicit synchronization.

## Generics (Go 1.18+)

Generics eliminate duplicated algorithms, not create type hierarchies. Inheritance/polymorphism thinking means writing Java.

### When to Use Generics

Use generics for the **same algorithm** across **multiple concrete types**:

```go
// Good: generic algorithm, concrete types as inputs
func Map[S, T any](slice []S, f func(S) T) []T {
    result := make([]T, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}

// Good: constraint expresses a meaningful requirement
func Min[T cmp.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

### When NOT to Use Generics

```go
// Bad: generic interface for polymorphism — this is Java
type Repository[T any] interface {
    Find(id string) (T, error)
    Save(entity T) error
}

// Good: a concrete interface for what you actually need
type UserStore interface {
    FindUser(id string) (*User, error)
    SaveUser(u *User) error
}
```

- **Do not** create generic base types, services, or repositories.
- **Do not** use `any` constraint to mean "I don't know the type yet." Design smell.
- **Do** use `comparable` for map keys/equality.
- **Do** use `cmp.Ordered` for `<`, `>`, `<=`, `>=`.
- Start concrete. Generify only when same logic repeats across 3+ types.
- Generic type aliases supported since Go 1.24.

## Pointers to Values: `new(expr)` (Go 1.26)

`new` now accepts an expression, not just a type — it allocates a variable initialized to that expression's value and returns its address:

```go
p := new(42)        // *int pointing to 42
q := new("debug")   // *string pointing to "debug"
r := new(true)       // *bool pointing to true
age := new(yearsSince(born)) // *int, works with any expression, not just literals
```

This replaces the generic `Ptr[T any](v T) *T` helper (and one-off `stringPtr`, `intPtr`, `boolPtr` functions) that Go code has carried for years to work around not being able to take the address of a literal. Delete those helpers and call `new(v)` directly — this is the idiomatic way to get a pointer to a literal or expression result now, including for optional struct fields and protobuf/JSON-style `*T` fields:

```go
// Old: needed a helper
type Config struct {
    MaxRetries *int
}
cfg := Config{MaxRetries: ptr(3)}

// Current: no helper needed
cfg := Config{MaxRetries: new(3)}
```

`new(Type)` with a type argument still works exactly as before (zero-value allocation) — `new(expr)` is purely additive.

## Standard Library: Use the New Packages

LLMs suggest third-party utilities or manual helpers already in stdlib since Go 1.21. **Always check stdlib first.**

### `slices` package (Go 1.21)

```go
import "slices"

// Searching and testing
slices.Contains(s, "value")
slices.Index(s, "value")           // returns -1 if not found
slices.ContainsFunc(s, func(v string) bool { return v == "x" })

// Sorting
slices.Sort(s)                     // sorts in place, works on any ordered type
slices.SortFunc(s, func(a, b T) int { return cmp.Compare(a.Name, b.Name) })
slices.IsSorted(s)

// Manipulation
slices.Reverse(s)
slices.Compact(s)                  // removes consecutive duplicates
slices.Delete(s, i, j)            // removes elements [i, j)
slices.Clone(s)                   // shallow copy
slices.Concat(s1, s2, s3)         // concatenate multiple slices (1.22)

// Iterator bridging (1.23)
slices.Collect(it)                // iterator -> slice
slices.Sorted(it)                 // iterator -> sorted slice
slices.Values(s)                  // slice -> iterator
```

Never write `sort.Slice(s, func(i, j int) bool { return s[i] < s[j] })` when `slices.Sort(s)` exists.

### `maps` package (Go 1.21; iterators 1.23)

```go
import "maps"

maps.Keys(m)        // iterator over keys — collect with slices.Collect / slices.Sorted
maps.Values(m)      // iterator over values
maps.Clone(m)       // shallow copy
maps.Copy(dst, src) // copies all entries from src into dst
maps.DeleteFunc(m, func(k K, v V) bool { ... })  // delete entries matching predicate
maps.Equal(m1, m2)  // reports whether two maps are equal

// Common idiom: sorted keys in one line
keys := slices.Sorted(maps.Keys(m))
```

### `cmp` package (Go 1.21)

```go
import "cmp"

cmp.Compare(a, b)    // returns -1, 0, or 1; works on any cmp.Ordered type
cmp.Or(a, b, c)      // returns first non-zero value — replaces ternary workarounds
min(a, b)            // built-in since Go 1.21
max(a, b)            // built-in since Go 1.21
```

Use `cmp.Or` for default values:

```go
// Instead of: if cfg.Timeout == 0 { cfg.Timeout = 30 * time.Second }
cfg.Timeout = cmp.Or(cfg.Timeout, 30*time.Second)
```

### `errors.Join` (Go 1.20)

```go
// Combine multiple errors — no third-party library needed
err := errors.Join(err1, err2, err3)

// Works correctly with errors.Is and errors.As
if errors.Is(err, ErrNotFound) { ... }
```

Use this instead of `fmt.Errorf("%w; %w", err1, err2)` or any `multierr` package.

### Iterators: `iter` and Range-over-Func (Go 1.23)

Expose sequences without slice allocation with `iter.Seq[T]` or `iter.Seq2[K, V]`; callers use plain `range`:

```go
func (idx *Index) All() iter.Seq[*Person] {
    return func(yield func(*Person) bool) {
        for _, p := range idx.people {
            if !yield(p) {
                return
            }
        }
    }
}

// Caller — just a normal range loop
for p := range idx.All() { ... }
```

Prefer iterators over slices for large/lazy sequences. Don't invent `Next()/HasNext()` iterators; that's Java.

### `math/rand/v2` (Go 1.22)

Import `math/rand/v2`, never old `math/rand`. Auto-seeded, cleaner API:

```go
import "math/rand/v2"

rand.IntN(100)            // was rand.Intn — note the capital N
rand.N(10 * time.Second)  // generic: random value in [0, n) for any integer type
```

### `encoding/json`: `omitzero` (Go 1.24)

`omitzero` omits any zero value, including `time.Time{}` and zero structs; `omitempty` doesn't.

```go
type Event struct {
    Name      string    `json:"name"`
    StartedAt time.Time `json:"started_at,omitzero"` // omitempty would emit "0001-01-01T..."
}
```

## Concurrency: Modern Patterns

### `sync/atomic` Typed Values (Go 1.19)

Use typed atomic values, not function API:

```go
// Old (still works but avoid for new code)
var count int64
atomic.AddInt64(&count, 1)
val := atomic.LoadInt64(&count)

// New — type-safe, no pointer arithmetic
var count atomic.Int64
count.Add(1)
val := count.Load()

// Other typed atomics
var flag  atomic.Bool
var ptr   atomic.Pointer[MyStruct]
var val32 atomic.Int32
var val64 atomic.Uint64
```

### `context.WithoutCancel` (Go 1.21)

See "Use `context.WithoutCancel` to Deliberately Detach" under Context above.

## HTTP: Use the Improved stdlib Router (Go 1.22)

Since Go 1.22, standard `net/http` ServeMux handles method and path-parameter routing natively; don't reflexively use gorilla/mux or chi.

```go
mux := http.NewServeMux()

// Method-scoped routes
mux.HandleFunc("GET /users", listUsers)
mux.HandleFunc("POST /users", createUser)

// Path parameters — accessed via r.PathValue
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
})

// Wildcard
mux.HandleFunc("GET /files/{path...}", serveFile)
```

Use chi or gorilla/mux only for named route generation or regex constraints. Middleware alone doesn't justify framework. Stdlib handles method + path routing.

### Production Servers: Always Set Timeouts

`http.ListenAndServe(addr, mux)` has **no timeouts**; slow clients can hold connections forever (slow-loris). Never emit production server without timeouts:

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,   // slow-loris protection
    ReadTimeout:       10 * time.Second,  // full request read
    WriteTimeout:      30 * time.Second,  // response write (covers handler time)
    IdleTimeout:       120 * time.Second, // keep-alive connections
}
```

For per-route control beyond `WriteTimeout`, use `http.TimeoutHandler` or handler-level `context.WithTimeout`. `http.DefaultClient` also has no timeout; construct one.

### Graceful Shutdown

Every production server needs shutdown path to stop accepting connections and drain in-flight requests:

```go
func run(ctx context.Context) error {
    ctx, stop := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
    defer stop()

    srv := &http.Server{ /* ... timeouts as above ... */ }

    errCh := make(chan error, 1)
    go func() { errCh <- srv.ListenAndServe() }()

    select {
    case err := <-errCh:
        return err // ListenAndServe failed at startup
    case <-ctx.Done():
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        return srv.Shutdown(shutdownCtx) // drain in-flight, then exit
    }
}
```

- `Shutdown` needs fresh context with own deadline; signal context is canceled.
- Long-lived connections (SSE, websockets) must watch `r.Context()` or hold shutdown until drain deadline.

### Middleware Is Just a Function

No framework needed. Middleware is `func(http.Handler) http.Handler`:

```go
func withRequestLog(logger *slog.Logger, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        logger.Info("request", "method", r.Method, "path", r.URL.Path, "dur", time.Since(start))
    })
}

// Compose by wrapping — outermost runs first
handler := withRequestLog(logger, withAuth(mux))
```

Don't import middleware framework for function composition.

## Syntax: Use Current Go

LLMs generate outdated syntax. Use current idioms:

### Range over Integer (Go 1.22)

```go
// Old
for i := 0; i < 10; i++ { ... }

// New
for i := range 10 { ... }
```

### Build Constraints

```go
// Old (deprecated — do not generate)
// +build linux darwin

// Current
//go:build linux || darwin
```

`//go:build` required since Go 1.17. Never emit old `// +build` syntax.

### `any` Instead of `interface{}`

```go
// Old
func Print(v interface{}) { ... }

// Current — `any` is a built-in alias for interface{} since Go 1.18
func Print(v any) { ... }
```

### Tool Dependencies in `go.mod` (Go 1.24)

Never generate `tools.go` with blank imports. Track dev tools with `tool` directive:

```bash
go get -tool golang.org/x/tools/cmd/stringer
go tool stringer -type=Color   # run it
```

## Structured Logging with `log/slog` (Go 1.21)

Common `log/slog` mistakes:

```go
// Pass a logger via context for request-scoped logging
func HandleRequest(ctx context.Context, r *Request) {
    logger := slog.With("request_id", r.ID, "user_id", r.UserID)
    // All subsequent log calls carry these fields automatically
    logger.Info("handling request")
    process(ctx, logger, r)
}

// Group related fields
logger.With(slog.Group("http",
    slog.String("method", r.Method),
    slog.String("path", r.URL.Path),
    slog.Int("status", status),
))

// Log at the right level — LLMs over-use Info
logger.Debug("cache miss", "key", key)     // internal state, high volume
logger.Info("server started", "addr", addr) // lifecycle events
logger.Warn("retrying", "attempt", n)       // recoverable problems
logger.Error("request failed", "err", err)  // needs attention
```

- **Never** use package-level `log` or `slog` global beyond `main`. Pass `*slog.Logger` as dependency.
- **Never** log and return an error; see "Log or Return — Never Both" under Error Handling.
- Use `slog.Default()` as fallback only in `main` or libraries without logger.

## Debugging: The Go Toolchain Is Not the Problem

**Go tool is reliable; almost never bug source.**

When debugging, don't suspect `go run`, `go build`, `go test`, or build cache first. Toolchain does what it says:

- `go run` always recompiles from source. It does not use a stale cached binary.
- `go build` is deterministic and correct.
- `go test` runs the actual compiled test binary.
- The build cache is keyed by source content — if the source changed, the cache is invalidated automatically.

**If error persists after edit, explanations in likelihood order:**

1. The edit did not fix the underlying logic error.
2. The edit was made in the wrong file, wrong function, or wrong package.
3. There is a second call site with the same bug that was not updated.
4. The error is coming from a different code path than the one being edited.

**Do this instead of blaming tool:**

- Re-read error; Go error messages are accurate.
- Confirm edited file is compiled (`go list -f '{{.GoFiles}}' .`).
- Add `fmt.Println` or `t.Log` at exact site to verify execution.
- Check all call sites of changed function.

Don't suggest clearing build cache (`go clean -cache`), restarting toolchain, or other tool intervention before exhausting code explanations. Tool isn't lying.

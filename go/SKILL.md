---
name: go
description: >
  Expert Go programming skill. Covers idiomatic Go — package design, error handling, interfaces, concurrency, testing, and
  project layout, current through Go 1.25. Use whenever Go code is written, reviewed, debugged, or
  refactored — any .go file, go.mod, CLI tool, or web service, and whenever the user mentions
  Go or golang, even if they don't ask for "idiomatic" code.
---

# Idiomatic Go: The Go Way

Idiomatic Go patterns and best practices for building robust, efficient, and maintainable applications.

## When to Activate

- Writing new Go code
- Reviewing or auditing existing Go code
- Refactoring Go code (especially code that looks like Java/Spring Boot patterns in Go)
- Designing Go packages, modules, or APIs
- Choosing between stdlib and third-party libraries
- Any question about Go project structure, error handling, concurrency, or testing

## Core Principles

### 1. Clear is Better than Clever

Go favors readability and simplicity over abstraction and cleverness. Code should be obvious. If you have to read a function three times to understand its control flow, it needs to be rewritten.

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

Design types so their zero value is immediately usable without initialization. This eliminates boilerplate constructors. `sync.Mutex` and `bytes.Buffer` are the gold standard for this.

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

Handle errors and edge cases immediately and return. Do not use `else` blocks for the main logic. The "happy path" of your function should never be indented.

### 4. Separate Logical Sections with Blank Lines

Inside a function, use a blank line to mark the boundary between logical sections — setup, one phase of work, the next phase, cleanup — the way you'd use paragraph breaks in prose. Don't cram unrelated steps into one unbroken block, and don't scatter blank lines between every single statement either; group the lines that belong to the same step, then break.

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

Each blank line answers "what is this next chunk of code doing." A `defer` paired with the setup it cleans up stays adjacent to that setup, not pushed apart by a section break.

## Package Organization: Domain-First, Dependency-Grouped

**Anti-Pattern:** Grouping code by technical layer or type — `models/`, `controllers/`, `handlers/`, `services/`, `repository/`. Go has no file-level separation, only package-level, so a `models/` package smashes every domain (`order`, `user`, `product`) into one namespace with no indication at the import site which domain an identifier belongs to. This causes circular dependencies the moment two domains need to reference each other. Reject it, and reject `utils/`, `helpers/`, and `common/` too — same symptom of unclear ownership.

Packages define bounded contexts, not files within a package — delineate domains by top-level package, not by file name.

### 1. Domain Types Depend on Nothing

Domain types and the interfaces that describe operations on them must not import anything else in the app — no database driver, no HTTP, no third-party SDK.

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

A single-domain app can keep its domain types in the root package. Once an app has more than one genuinely distinct domain, give each its own top-level package instead of collapsing them into one root — `order` and `user` are different bounded contexts and belong in different packages, each import-free of the other.

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

The signal to split a domain into its own package: can it be described in one sentence, with no need to know about other domains? If yes, it earns its own package. Domain packages never import each other sideways — a needed dependency between two domains means the boundary is wrong, or the dependency belongs behind an interface (see #4).

### 3. Group Adapters by Dependency, Not by Domain

Once a domain needs a database, an HTTP API, or a third-party service, push that dependency into its own subpackage named after the thing it wraps — `postgres`, `stripe`, `http`, not a domain-suffixed name. That subpackage implements the domain's interface:

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

This makes the dependency swappable and lets implementations layer (e.g. an in-memory cache wrapping `postgres.Repository`, both satisfying `order.Repository`). Applies to stdlib wrapping too — put `net/http`-specific code in your own `http` adapter package rather than letting it leak into domains.

Grouping every adapter under one `http/` or `postgres/` package is fine even across multiple domains, since they're only wired together in `cmd/`. Splitting per-domain (`http/order/`, `postgres/order/`) is also fine if an adapter layer gets large. Domains require package-level separation; adapters don't.

### 4. Wire Dependencies Behind Domain Interfaces, Not Concretely

When one domain's implementation needs another dependency (e.g. an `order` adapter needs to charge a card), depend on a domain interface, not the concrete adapter:

```go
type Repository struct {
    DB                 *sql.DB
    TransactionService order.TransactionService // interface, not stripe.Service
}
```

This keeps adapters swappable independently of one another.

### 5. A Shared `mock` Subpackage for Testing

Hand-write simple mocks that implement domain interfaces, collected in one `mock` subpackage, rather than reaching for a mocking framework:

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

Inject `&mock.Repository{OrderFn: ...}` anywhere `order.Repository` is expected. Complements, doesn't replace, the fakes/stubs and table-driven testing patterns below.

### 6. `main` Lives Under `cmd/`, and Only Wires

Use the `cmd/<binary-name>/main.go` convention even for a single binary — it leaves room for a second one later without restructuring. `main` chooses which concrete adapters to inject into which domain interfaces; it contains no business logic itself.

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

Domains never import adapters or `cmd`. Adapters import the domains they implement. `cmd` imports and wires everything. Go's compiler enforces this — import cycles won't build, so a stray domain→adapter import fails fast:

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

Reach for `internal/` only when a package needs to be importable across your own subpackages but must not be importable by outside modules. For an executable binary nobody imports anyway, `internal/` is usually just extra path depth — skip it unless a real need for that boundary shows up.

## Interface Design

### 1. Interfaces are Discovered, Not Designed Upfront

Write concrete types first. Only define an interface when you discover that multiple types need to be used interchangeably by a consumer.

### 2. Define Interfaces Where They Are Used

Interfaces belong in the package that _consumes_ them, not the package that _implements_ them. This decouples your packages.

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

Require the smallest interface possible as an input parameter (e.g., `io.Reader` instead of `*os.File`), but return a concrete struct so callers aren't forced to use type assertions to access specific fields or methods.

## Library API Design

### Domain Object as Entry Point

When designing a Go library that wraps a stateful resource (a vault, a database connection, a config store), make that resource struct the primary object. Its methods return domain-typed sub-objects. **Avoid:**

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

**Why this pattern:**

- The primary struct (`Vault`) is the single entry point — callers need just one import to get started
- Sub-packages define the rich domain types (`people.Person`, `daily.Note`) — each type lives with the logic that owns it
- No global state means the library is safe for concurrent use, multiple instances, and testing
- Stateless method calls (reload fresh each time) keep the struct simple — no cache invalidation logic needed
- Type inference (`:=`) means callers rarely need to explicitly import sub-packages for variable declarations

**When to use a global instance instead:** Only for CLI-only tools (like `pflag` itself) where there is truly only ever one instance and ease of use for end-users outweighs library correctness.

## Concurrency Patterns

**Anti-Pattern:** Heavy, static Worker Pools. Go's scheduler is incredibly efficient; you don't need to manually manage pools of workers like OS threads in other languages.

### 1. Share Memory by Communicating

Don't use mutexes to protect shared data if you can pass that data over a channel instead. Channels orchestrate execution; mutexes serialize execution.

### 2. Bounded Concurrency

To limit concurrency, use `errgroup` with `SetLimit`. Don't hand-roll a semaphore channel or a rigid worker pool when this exists.

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

Loop variables are per-iteration since Go 1.22 — never emit the old `url := url` capture line.

When you don't need error propagation, `sync.WaitGroup.Go` (Go 1.25) removes the Add/Done boilerplate:

```go
var wg sync.WaitGroup
for _, url := range urls {
    wg.Go(func() { process(url) })
}
wg.Wait()
```

### 3. Never Start a Goroutine Without Knowing How It Stops

Every `go func()` must have a clear exit condition, usually governed by a `context.Context` or a closed channel.

## Context: Pass It, Don't Store It

`context.Context` carries cancellation, deadlines, and request-scoped values across API boundaries. A few rules govern it strictly — LLMs and inexperienced Go code both violate them constantly.

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

If a function receives a `ctx`, pass that same `ctx` (or a value derived from it via `context.WithCancel`, `WithTimeout`, `WithDeadline`, or `context.WithValue`) into everything it calls that also accepts one. Don't silently drop it and call `context.Background()` instead — that severs cancellation and deadline propagation for everything downstream.

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

`context.Background()` (or `context.TODO()` while a real context isn't available yet) belongs only at the true root of a call chain — `main`, the start of an incoming request, the top of a background job — never inside a function that already received a `ctx`.

### 3. Use `context.WithoutCancel` to Deliberately Detach

When a background task legitimately needs to outlive the request that spawned it, don't drop the context — detach it explicitly so values still propagate but cancellation doesn't:

```go
// The background job should keep running even after the HTTP request context cancels.
bgCtx := context.WithoutCancel(requestCtx)
go doBackgroundWork(bgCtx)
```

This documents the intent at the call site instead of leaving a reader to wonder whether dropping the context was a bug.

### 4. `context.Value` Is for Request-Scoped Data, Not Optional Parameters

Reach for `WithValue` only for things that cut across API boundaries and aren't naturally part of a function's signature — a request ID, a trace span, an auth principal set by middleware. Don't use it to pass regular arguments a function actually depends on; those belong as explicit parameters, which keeps the function's dependencies visible and the function testable without faking a context.

```go
// Idiomatic: request-scoped, cross-cutting, set by middleware, read deep in the call stack
ctx = context.WithValue(ctx, requestIDKey{}, reqID)

// Anti-pattern: this is a real parameter, not incidental request metadata
ctx = context.WithValue(ctx, userIDKey{}, userID) // FetchUser(ctx, userID) instead
```

Always use an unexported key type (`type requestIDKey struct{}`), never a `string` or other exported type, to avoid collisions between packages.

### 5. A `context.Context` Cancellation Doesn't Stop Execution — Check It

Passing a `ctx` down doesn't automatically abort work; code has to check `ctx.Err()` or select on `ctx.Done()` at the points where it matters — before expensive work, in loop iterations, around blocking calls (most stdlib I/O already respects it, like `QueryRowContext` and `http.Client` with a context-carrying request).

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

When a struct has many optional configuration parameters, avoid massive constructors. Use the Functional Options pattern.

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

Errors aren't exceptions to be caught; they are values to be handled. Check them explicitly.

### 2. Wrap for Context, Not for Stack Traces

When returning an error, add context about what you were trying to do.

```go
// Idiomatic
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("loading config file %s: %w", path, err)
}
```

### 3. Log or Return — Never Both

Logging an error counts as handling it. If you log an error, don't also return it up the stack; if you return it, don't also log it at that site. Doing both means the same failure gets logged once per layer as it propagates, polluting logs with duplicate noise for a single root cause.

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

Log at the boundary where an error is actually handled instead of propagated further — top of a request handler, a background job's outer loop, `main`. Below that boundary, wrap and return. A deliberate exception: logging at a lower level as a warning while still continuing (not returning) is handling, not duplication — e.g. a non-fatal cache-write failure that you log and move past.

## Testing Patterns

**Anti-Pattern:** Relying on heavy BDD frameworks (like Ginkgo) or complex mocking generation tools. Go testing should just be Go programming.

### 1. Table-Driven Tests

The absolute standard for unit testing in Go. Iterate over a slice of structs containing inputs and expected outputs using `t.Run()`.

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

When extracting repeated assertion logic, always call `t.Helper()` to ensure failures point to the actual test case, not the helper function line.

### 3. Fakes and Stubs over Heavy Mocks

Leverage Go's implicit interfaces to write simple, manual fakes. This keeps test dependencies lightweight and test logic transparent.

### 4. Golden Files and the `testdata` Directory

For tests requiring complex inputs or producing large outputs, use a directory named `testdata`. The `go test` tool explicitly ignores these directories.

### 5. Filesystem Abstraction (The Afero Pattern)

Do not hardcode `os` package calls deep within business logic. Accept an interface for the filesystem so tests can run in memory without touching the disk. `github.com/spf13/afero` is the industry standard for this.

```go
import "github.com/spf13/afero"

type FileProcessor struct {
    fs afero.Fs
}

func NewFileProcessor(fs afero.Fs) *FileProcessor {
    return &FileProcessor{fs: fs}
}
```

In tests, inject `afero.NewMemMapFs()` to completely eliminate disk I/O and prevent flaky, slow tests.

### 6. `cmp` over DeepEqual

For comparing complex structs or maps, use `github.com/google/go-cmp/cmp` for rich, readable diffs instead of the strict `reflect.DeepEqual`.

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

Generics exist to eliminate duplicated algorithms, not to create type hierarchies. If you are thinking about generics in terms of inheritance or polymorphism, stop — you are writing Java.

### When to Use Generics

Use generics when you have the **same algorithm** that needs to operate on **multiple concrete types**:

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

- **Do not** create generic base types, generic services, or generic repositories.
- **Do not** use `any` as a constraint to mean "I don't know the type yet." That's a design smell.
- **Do** use `comparable` when you need map keys or equality checks.
- **Do** use `cmp.Ordered` when you need `<`, `>`, `<=`, `>=`.
- Start with a concrete implementation. Generify only when you have the same logic repeated across 3+ types.
- Generic type aliases are fully supported since Go 1.24.

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

LLMs frequently suggest third-party utilities or write manual helpers that have been in the standard library since Go 1.21. **Always check stdlib first.**

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

`cmp.Or` is especially useful for default-value patterns:

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

The standard way to expose a sequence from an API without allocating a slice. Return `iter.Seq[T]` or `iter.Seq2[K, V]`; callers use plain `range`:

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

Prefer returning iterators over slices for large or lazily-produced sequences. Don't invent a custom `Next()/HasNext()` iterator type — that's Java.

### `math/rand/v2` (Go 1.22)

Always import `math/rand/v2`, never the old `math/rand`. Auto-seeded, cleaner API:

```go
import "math/rand/v2"

rand.IntN(100)            // was rand.Intn — note the capital N
rand.N(10 * time.Second)  // generic: random value in [0, n) for any integer type
```

### `encoding/json`: `omitzero` (Go 1.24)

`omitzero` omits any zero value — including `time.Time{}` and zero structs, which `omitempty` never handled correctly:

```go
type Event struct {
    Name      string    `json:"name"`
    StartedAt time.Time `json:"started_at,omitzero"` // omitempty would emit "0001-01-01T..."
}
```

## Concurrency: Modern Patterns

### `sync/atomic` Typed Values (Go 1.19)

Use the typed atomic values instead of the function-based API:

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

LLMs reflexively recommend gorilla/mux or chi for any routing beyond the trivial. Since Go 1.22, the standard `net/http` ServeMux handles method and path-parameter routing natively.

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

Reach for chi or gorilla/mux only when you need named route generation or regex constraints — middleware doesn't justify a framework (see below). For pure method + path routing, the stdlib is sufficient.

### Production Servers: Always Set Timeouts

`http.ListenAndServe(addr, mux)` ships with **no timeouts** — a single slow client can hold a connection open forever (slow-loris). LLM-generated servers almost never set these. Never emit a production server without them:

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

For per-route control beyond `WriteTimeout`, use `http.TimeoutHandler` or handler-level `context.WithTimeout`. Outbound calls need the same discipline: `http.DefaultClient` has no timeout either — construct a client with one.

### Graceful Shutdown

Every production server needs a shutdown path that stops accepting connections and drains in-flight requests. The pattern:

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

- `Shutdown` needs a fresh context with its own deadline — the signal context is already canceled.
- Long-lived connections (SSE, websockets) must watch `r.Context()` or they'll hold shutdown until the drain deadline.

### Middleware Is Just a Function

No framework needed. A middleware is `func(http.Handler) http.Handler`:

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

Don't import a middleware framework for what function composition already does.

## Syntax: Use Current Go

LLMs frequently generate outdated syntax. Know the current idioms:

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

The `//go:build` form is required since Go 1.17. Never emit the old `// +build` syntax.

### `any` Instead of `interface{}`

```go
// Old
func Print(v interface{}) { ... }

// Current — `any` is a built-in alias for interface{} since Go 1.18
func Print(v any) { ... }
```

### Tool Dependencies in `go.mod` (Go 1.24)

Never generate a `tools.go` file with blank imports. Track dev tools with the `tool` directive:

```bash
go get -tool golang.org/x/tools/cmd/stringer
go tool stringer -type=Color   # run it
```

## Structured Logging with `log/slog` (Go 1.21)

The skill note above covers basic setup. The patterns LLMs most often get wrong:

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

- **Never** use a package-level `log` or `slog` global beyond `main`. Pass `*slog.Logger` as a dependency.
- **Never** log and return an error — see "Log or Return — Never Both" under Error Handling.
- Use `slog.Default()` as the fallback only in `main` or in libraries when no logger is provided.

## Debugging: The Go Toolchain Is Not the Problem

**The Go tool is extremely reliable. It is almost never the source of a bug.**

When debugging, do not waste time suspecting `go run`, `go build`, `go test`, or the build cache. The Go toolchain does what it says:

- `go run` always recompiles from source. It does not use a stale cached binary.
- `go build` is deterministic and correct.
- `go test` runs the actual compiled test binary.
- The build cache is keyed by source content — if the source changed, the cache is invalidated automatically.

**If an error persists after you edit the code, the explanation is one of these — in order of likelihood:**

1. The edit did not fix the underlying logic error.
2. The edit was made in the wrong file, wrong function, or wrong package.
3. There is a second call site with the same bug that was not updated.
4. The error is coming from a different code path than the one being edited.

**What to do instead of blaming the tool:**

- Re-read the error message carefully. Go's error messages are accurate.
- Confirm the file you edited is actually the file being compiled (`go list -f '{{.GoFiles}}' .`).
- Add a `fmt.Println` or `t.Log` at the exact site to verify execution reaches it.
- Check that all call sites of a changed function were updated.

Do not suggest clearing the build cache (`go clean -cache`), restarting the Go toolchain, or any other tool-level intervention before first exhausting all code-level explanations. The tool is not lying to you.

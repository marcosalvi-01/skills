---
name: go
description: >
  Expert Go programming skill covering idiomatic package design, error handling, interfaces, concurrency, testing, and
  project layout, current through Go 1.27. Use whenever Go code is written, reviewed, debugged, or refactored; for any
  .go file, go.mod, CLI tool, web service, HTTP server, test, or concurrency task; and whenever the user mentions Go or golang.
---

## When to Activate

- Writing, reviewing, auditing, or refactoring Go code, especially Java/Spring Boot-like code
- Designing Go packages, modules, or APIs
- Choosing stdlib vs third-party libraries
- Questions about Go structure, errors, concurrency, or testing

## Quick scan — flag/replace if found

- `sort.Slice(...)` → `slices.Sort` / `slices.SortFunc`
- hand-written `ptr()`/`stringPtr()`/`intPtr()` → `new(v)` (Go 1.26)
- `interface{}` → `any`
- `// +build` tags → `//go:build`
- `math/rand` (not `/v2`) or `rand.Intn` → `math/rand/v2`, `rand.IntN`
- `reflect.DeepEqual` on structs → `cmp.Diff`/`cmp.Equal`
- classic `for i := 0; i < n; i++` over plain range → `for i := range n`
- package-level `log`/`slog` outside `main` → injected `*slog.Logger`
- error logged *and* returned at same layer → pick one
- manual `sort`/dedup/key-collection on maps → `maps.Keys`/`slices.Sorted`
- manual comparison/default branches → `cmp.Compare`/`cmp.Or`/`min`/`max`
- `omitempty` when zero-value omission matters → `omitzero` (Go 1.24)
- ad-hoc error concatenation → `errors.Join`
- writing a test? → check `references/testing.md`
- writing an HTTP server/handler? → check `references/http-server.md`
- adding worker pools/semaphores/goroutines? → check `references/concurrency-advanced.md`

## Workflow

- Run `make test` for tests.
- `make lint` formats before linting. Do not use gofmt on files individually, run make lint.

## Core Principles

### 1. Clear Is Better Than Clever

Prefer direct, readable control flow. Omit redundant `Get` prefixes:

```go
func LookupUser(id string) (*User, error) {
	user, err := db.FindUser(id)
	if err != nil {
		return nil, fmt.Errorf("finding user %s: %w", id, err)
	}
	return user, nil
}
```

### 2. Make the Zero Value Useful

Design usable zero values; avoid constructor boilerplate:

```go
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

Handle errors and edge cases immediately. For multiple error checks, declare the error first; use separate checks:

```go
_, err := r.GetGroupByID(ctx, groupID)
if errors.Is(err, pgx.ErrNoRows) {
	return groups.ErrNotFound
}
if err != nil {
	return fmt.Errorf("check group: %w", err)
}
```

### 4. Separate Logical Sections with Blank Lines

Separate setup, work, and cleanup. Keep cleanup `defer` beside setup it reverses; do not cram unrelated operations.

## Package Organization: Domain-First, Dependency-Grouped

Avoid technical-layer packages: `models/`, `controllers/`, `services/`, `repository/`. Reject catch-all `utils/`, `helpers/`, and `common/`. Use top-level packages for domains and dependency-named packages for adapters.

### 1. Domain Types Depend on Nothing

Domain types/interfaces import no database driver, HTTP package, third-party SDK, adapter, or `cmd`:

```go
// order/order.go
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

Use one package per bounded context. Domain packages do not import each other sideways; expose interfaces for dependencies:

```
mystore/
├── order/           # domain types + interfaces
├── user/            # domain types + interfaces
├── postgres/        # database adapters
├── http/            # HTTP adapters
└── cmd/
    └── mystore/     # wiring only
```

Split domains when each can be described in one sentence without another domain.

### 3. Group Adapters by Dependency

Put PostgreSQL, Stripe, and HTTP implementations in packages named for wrapped dependencies. Adapters import domains and implement their interfaces:

```go
// postgres/order.go
type Repository struct {
	DB *sql.DB
}

func (r *Repository) Order(id int) (*order.Order, error) {
	// Query and map database data into the domain type.
	return &order, nil
}
```

One adapter package may span domains; split only when large.

### 4. Wire Dependencies Behind Domain Interfaces

Adapters needing dependencies use domain interfaces, not concrete adapters:

```go
type Repository struct {
	DB                 *sql.DB
	TransactionService order.TransactionService
}
```

### 5. Keep `main` Under `cmd/`

Use `cmd/<binary-name>/main.go`. `main` selects adapters and wires domain interfaces; no business logic:

```go
func main() {
	db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	app := NewApp(&postgres.Repository{DB: db})
	log.Fatal(app.Run())
}
```

### Dependency Direction

Domains never import adapters or `cmd`; adapters import domains; `cmd` wires everything. Compiler-enforced direction catches domain-to-adapter mistakes.

### `internal/`

Use `internal/` to share packages inside the module while hiding them from external modules. Skip extra depth for standalone executables unless needed.

## Interface Design

- Write concrete types first. Define interfaces when a consumer needs interchangeable implementations.
- Define interfaces in the consuming package, not the implementing package.
- Accept the smallest useful interface, such as `io.Reader`; return concrete structs.

```go
// processor/processor.go
type UserFetcher interface {
	FetchUser(id string) (*User, error)
}

type Processor struct {
	fetcher UserFetcher
}
```

## Library API Design

For stateful-resource libraries, make resource struct entry point; methods return domain sub-objects:

```go
v, err := vault.Open(path)
idx, err := v.People()
person, err := idx.FindOne("Ada")
```

Avoid resource/config arguments on every package-level function and package-level state in embeddable libraries. Stateless methods support multiple instances, concurrency, and testing.

## Context: Pass It, Don't Store It

`context.Context` carries cancellation, deadlines, and request-scoped values across API boundaries.

### 1. Put `ctx` First; Never Store It

```go
func (s *Service) FetchUser(ctx context.Context, id string) (*User, error) {
	var user User
	if err := s.db.QueryRowContext(ctx, "SELECT ...", id).Scan(&user); err != nil {
		return nil, err
	}
	return &user, nil
}
```

Never store `context.Context` in service structs. `http.Request.Context()` is the narrow stdlib exception.

### 2. Propagate the Given Context

Pass `ctx` or derived context downstream. Never replace it with `context.Background()` inside a function receiving `ctx`; use `Background` or `TODO` only at call-chain roots.

### 3. Detach Deliberately

Use `context.WithoutCancel` when background work outlives its request. Values persist, cancellation does not; give task its own lifetime limit:

```go
bgCtx := context.WithoutCancel(requestCtx)
go doBackgroundWork(bgCtx)
```

Detached-work details: [references/concurrency-advanced.md](references/concurrency-advanced.md).

### 4. Use `context.Value` Only for Request Metadata

Use `WithValue` for cross-boundary request data: request IDs, trace spans, middleware-set principals. Use explicit parameters for real inputs. Use an unexported key type, never a string:

```go
type requestIDKey struct{}
ctx = context.WithValue(ctx, requestIDKey{}, requestID)
```

### 5. Check Cancellation

Passing `ctx` does not stop work. Check `ctx.Err()` or select on `ctx.Done()` before expensive work, in loops, and around blocking operations. Context-aware stdlib I/O observes it.

## Configuration and Struct Design

### Struct Literal Formatting

Struct literals with more than two attributes: multiline, one field per line:

```go
SomeStruct{
	ID:       value.ID,
	Name:     value.Name,
	Label:    value.Label,
	Position: value.Position,
}
```

### Embedded Struct Initialization (Go 1.27)

Keyed literals may initialize promoted fields from embedded value structs directly:

```go
type Address struct {
	City    string
	Country string
}

type User struct {
	Address
	Name string
}

user := User{
	City:    "Rome",
	Country: "Italy",
	Name:    "Ada",
}
```

Promoted fields must be unambiguous. Embedded pointers are not implicitly allocated; initialize explicitly:

```go
type User struct{ *Address }

user := User{Address: &Address{City: "Rome"}}
```

### Functional Options

For many optional config parameters, avoid massive constructors:

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
	s := &Server{addr: addr, timeout: 30 * time.Second}
	for _, opt := range opts {
		opt(s)
	}
	return s
}
```

## Error Handling

### 1. Errors Are Values

Check errors explicitly. Errors are values, not exceptions.

### 2. Wrap for Context

Wrap returned errors with operation context, not fake stack traces:

```go
data, err := os.ReadFile(path)
if err != nil {
	return fmt.Errorf("loading config file %s: %w", path, err)
}
```

### 3. Log or Return, Never Both

Below handling boundary, wrap and return. Log only where handled: request boundary, outer background loop, or `main`. Lower layers may log nonfatal warnings while continuing.

## Testing

Testing-specific patterns, fakes including shared `mock` subpackages, modern `testing` APIs, and deterministic concurrency tests: [references/testing.md](references/testing.md).

## Generics

Use generics for the same algorithm across multiple concrete types, not to create type hierarchies:

```go
func Map[S, T any](values []S, f func(S) T) []T {
	result := make([]T, len(values))
	for i, value := range values {
		result[i] = f(value)
	}
	return result
}
```

- Do not create generic base types, services, or repositories.
- Do not use `any` as a placeholder constraint.
- Use `comparable` for map keys/equality and `cmp.Ordered` for ordering.
- Start concrete. Generify when the same logic repeats across multiple types.

### Generic Methods and Function Inference (Go 1.27)

Concrete methods may declare their own type parameters. Use this for generic operations naturally scoped to a receiver:

```go
type Box struct{ value any }

func (b Box) As[T any]() (T, bool) {
	value, ok := b.value.(T)
	return value, ok
}

value, ok := (Box{value: 42}).As[int]()
```

Interface methods cannot declare type parameters, and generic concrete methods do not satisfy interface methods. Use a non-generic adapter when an interface contract is required. Type inference also applies when assigning or converting a generic function to a matching function type, including function-valued fields in composite literals:

```go
func Apply[T any](T) {}

type Config struct{ Apply func(int) }

var logValue func(string) = Apply // T inferred as string
cfg := Config{Apply: Apply}        // T inferred as int
```

## Core Standard Library

Use `cmp.Compare`, `cmp.Or`, `min`, and `max` for ordering and default selection. The quick scan covers modern replacements for slices, maps, errors, iterators, random numbers, pointers, JSON tags, and tool dependencies.

For Go 1.27 modules, prefer `encoding/json/v2`, `strings.CutLast`, `bytes.CutLast`, `url.URL.Clone`, `url.Values.Clone`, standard `crypto/mldsa`, and standard `uuid` when their compatibility and behavior fit the task.

## HTTP

Go 1.22 `net/http` `ServeMux` supports method and path-parameter routing. Full router, timeout, shutdown, middleware, client, and test-server patterns: [references/http-server.md](references/http-server.md).

## Concurrency Patterns

Use channels to communicate ownership and mutexes to protect shared state. Avoid heavy static worker pools; bound concurrent work with `errgroup.SetLimit`. Every goroutine needs a clear exit condition, usually context cancellation or a closed channel.

Advanced worker, semaphore, `sync.WaitGroup.Go`, atomic, and detached-context patterns: [references/concurrency-advanced.md](references/concurrency-advanced.md).

## Structured Logging

Use `log/slog` and inject `*slog.Logger` outside `main`. Use `Debug` for high-volume internals, `Info` for lifecycle events, `Warn` for recoverable problems, and `Error` for failures requiring attention. Do not use package-level logging globals in libraries.

## Debugging: The Go Toolchain Is Not the Problem

Go tools are reliable. When an error persists, re-read it, confirm the edited file is compiled, check every call site, and verify the executed path. Do not clear the build cache or restart the toolchain before exhausting code explanations.

## Before handing back Go work

- `sort.Slice(...)` → `slices.Sort` / `slices.SortFunc`
- hand-written `ptr()`/`stringPtr()`/`intPtr()` → `new(v)` (Go 1.26)
- `interface{}` → `any`
- `// +build` tags → `//go:build`
- `math/rand` (not `/v2`) or `rand.Intn` → `math/rand/v2`, `rand.IntN`
- `reflect.DeepEqual` on structs → `cmp.Diff`/`cmp.Equal`
- classic `for i := 0; i < n; i++` over plain range → `for i := range n`
- `tools.go` blank imports → `go.mod` `tool` directive
- package-level `log`/`slog` outside `main` → injected `*slog.Logger`
- error logged _and_ returned at same layer → pick one
- manual `sort`/dedup/key-collection on maps → `maps.Keys`/`slices.Sorted`
- manual comparison/default branches → `cmp.Compare`/`cmp.Or`/`min`/`max`
- custom `Next`/`HasNext` iterators → `iter.Seq`/`iter.Seq2`
- `omitempty` when zero-value omission matters → `omitzero` (Go 1.24)
- ad-hoc error concatenation → `errors.Join`
- writing a test? → check `references/testing.md`
- writing an HTTP server/handler? → check `references/http-server.md`
- adding worker pools/semaphores/goroutines? → check `references/concurrency-advanced.md`

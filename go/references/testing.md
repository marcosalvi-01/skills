# Go Testing Patterns

Read for Go tests. Keep tests in Go; avoid heavy BDD frameworks/generated mocks.

## Contents

- [Table-Driven Tests](#table-driven-tests)
- [Helpers, Fakes, and Stubs](#helpers-fakes-and-stubs)
- [Golden Files and `testdata`](#golden-files-and-testdata)
- [Filesystem Tests](#filesystem-tests)
- [Comparing Values](#comparing-values)
- [Modern `testing` APIs](#modern-testing-apis)
- [Deterministic Concurrency Tests](#deterministic-concurrency-tests)

## Table-Driven Tests

Use data cases with `t.Run()`:

```go
func TestParseConfig(t *testing.T) {
	tests := []struct {
		name    string
		input   string
		wantErr bool
	}{
		{name: "valid config", input: "port=8080"},
		{name: "invalid format", input: "port=abc", wantErr: true},
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

## Helpers, Fakes, and Stubs

- Call `t.Helper()` in assertion helpers; failures point to test cases.
- Prefer small manual fakes/stubs with implicit interfaces. Avoid mocking frameworks.
- Keep shared domain-interface mocks in a `mock` subpackage.

Shared domain-interface mock: inject behavior; record needed calls:

```go
type Repository struct {
	OrderFn       func(int) (*order.Order, error)
	OrderInvoked bool
}

func (r *Repository) Order(id int) (*order.Order, error) {
	r.OrderInvoked = true
	return r.OrderFn(id)
}
```

## Golden Files and `testdata`

Put complex inputs/large expected outputs in `testdata`; the Go tool ignores this directory during package builds. Make golden updates explicit; never silently rewrite expected output.

## Filesystem Tests

Do not hardcode `os` calls in business logic. Inject a filesystem abstraction for in-memory tests:

```go
import "github.com/spf13/afero"

type FileProcessor struct {
	fs afero.Fs
}

func NewFileProcessor(fs afero.Fs) *FileProcessor {
	return &FileProcessor{fs: fs}
}
```

Use `afero.NewMemMapFs()` to avoid disk I/O and flaky filesystem state.

## Comparing Values

Use `github.com/google/go-cmp/cmp` for complex structs/maps. Prefer `cmp.Diff` failures over strict `reflect.DeepEqual`.

## Modern `testing` APIs

Use Go 1.24+ APIs:

```go
func TestFetch(t *testing.T) {
	ctx := t.Context() // canceled when test ends
	_ = ctx
	// ...
}

func BenchmarkParse(b *testing.B) {
	for b.Loop() {
		Parse(input)
	}
}

// t.Chdir(dir) changes directory and restores it after the test.
```

## Deterministic Concurrency Tests

Use `testing/synctest` for goroutines/timeouts. Its fake clock removes wall-clock timing and `time.Sleep` coordination:

```go
func TestTimeout(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		ctx, cancel := context.WithTimeout(t.Context(), time.Second)
		defer cancel()

		synctest.Sleep(2 * time.Second)
		if ctx.Err() == nil {
			t.Fatal("expected timeout")
		}
	})
}
```

Never use `time.Sleep(100 * time.Millisecond)` to wait for goroutines. Use `synctest`, channels, or explicit synchronization.

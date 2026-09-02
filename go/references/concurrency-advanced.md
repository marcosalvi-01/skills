# Advanced Go Concurrency

Read for worker pools, semaphores, goroutines, atomics, and detached background work.

## Choose the Primitive

- Use channels for ownership transfer/stage coordination.
- Use `sync.Mutex` or `sync.RWMutex` for shared state.
- Prefer bounded `errgroup` goroutines over fixed worker pools.
- Every goroutine needs an exit condition: context cancellation or closed channel.

## Bound Concurrency with `errgroup`

Use `SetLimit`; avoid semaphore channels/rigid worker pools. Validate limits: zero blocks all work:

```go
func FetchAll(ctx context.Context, urls []string, maxConcurrent int) error {
	if maxConcurrent < 1 {
		return fmt.Errorf("maxConcurrent must be positive")
	}

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

Loop variables are per-iteration since Go 1.22. Never emit old `url := url` capture workaround.

Without error propagation, Go 1.25 `sync.WaitGroup.Go` removes `Add`/`Done` boilerplate:

```go
var wg sync.WaitGroup
for _, url := range urls {
	wg.Go(func() { process(url) })
}
wg.Wait()
```

## Typed Atomics

Use typed atomics, not function API:

```go
var count atomic.Int64
count.Add(1)
value := count.Load()

var flag atomic.Bool
var ptr atomic.Pointer[Config]
```

Use atomics for small independent state. Use a mutex when fields change together.

## Detach Deliberately

When background work outlives its request, use `context.WithoutCancel`: values propagate, cancellation does not:

```go
bgCtx := context.WithoutCancel(requestCtx)
go doBackgroundWork(bgCtx)
```

Provide a separate lifetime limit or shutdown signal. Detaching does not self-terminate goroutines.

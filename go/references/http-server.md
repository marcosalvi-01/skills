# Go HTTP Server Patterns

Read for HTTP servers, handlers, middleware, and HTTP clients.

## Use `net/http` ServeMux

Since Go 1.22, standard `net/http` routing supports methods/path parameters. Do not add chi or gorilla/mux for routing or middleware composition alone:

```go
mux := http.NewServeMux()

mux.HandleFunc("GET /users", listUsers)
mux.HandleFunc("POST /users", createUser)
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	// ...
})
mux.HandleFunc("GET /files/{path...}", serveFile)
```

Use another router only for named route generation or regex constraints.

## Set Production Timeouts

`http.ListenAndServe` and `http.DefaultClient` have no timeout. Configure server/client explicitly:

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           mux,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      30 * time.Second,
	IdleTimeout:       120 * time.Second,
}

client := &http.Client{Timeout: 30 * time.Second}
```

`ReadHeaderTimeout` protects against slow-loris clients. `WriteTimeout` includes ordinary handler execution. Use `http.TimeoutHandler` or handler-level `context.WithTimeout` for per-route limits.

## Graceful Shutdown

Stop accepting connections; drain in-flight requests. Use fresh shutdown context because signal context is canceled:

```go
func run(ctx context.Context, mux http.Handler) error {
	ctx, stop := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
	defer stop()

	srv := &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      30 * time.Second,
		IdleTimeout:       120 * time.Second,
	}

	errCh := make(chan error, 1)
	go func() {
		errCh <- srv.ListenAndServe()
	}()

	select {
	case err := <-errCh:
		return err
	case <-ctx.Done():
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		return srv.Shutdown(shutdownCtx)
	}
}
```

Long-lived SSE/WebSocket handlers must watch `r.Context()` or finish before shutdown deadline.

## Middleware

Middleware is a function. Wrap to compose; outermost runs first:

```go
func withRequestLog(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		logger.Info("request", "method", r.Method, "path", r.URL.Path, "dur", time.Since(start))
	})
}

handler := withRequestLog(logger, withAuth(mux))
```

## Test Servers

Go 1.27 `httptest.NewTestServer` uses an in-memory fake network for `testing/synctest`; prefer it for deterministic HTTP tests needing fake-clock behavior.

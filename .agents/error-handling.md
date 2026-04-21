# Error Handling in LocalAI

This guide covers patterns and best practices for error handling across the LocalAI codebase.

## General Principles

- Always wrap errors with context using `fmt.Errorf("context: %w", err)`
- Never silently swallow errors — log or propagate them
- Use structured logging with `log.Error().Err(err).Msg("description")` (zerolog style)
- Return errors to callers rather than calling `os.Exit` in library code

## HTTP Handler Errors

Handlers should return appropriate HTTP status codes with JSON error bodies:

```go
// Bad
if err != nil {
    c.Status(500)
    return
}

// Good
if err != nil {
    log.Error().Err(err).Str("model", modelName).Msg("failed to load model")
    return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
        "error": fiber.Map{
            "message": err.Error(),
            "type":    "internal_error",
            "code":    500,
        },
    })
}
```

Common status codes used:
- `400 Bad Request` — invalid input, missing required fields
- `404 Not Found` — model or resource not found
- `500 Internal Server Error` — backend failures, unexpected panics
- `503 Service Unavailable` — model not loaded yet, backend initializing

## Backend / Model Loading Errors

When a backend fails to load, wrap the error with the model name and backend type:

```go
if err := backend.Load(opts); err != nil {
    return fmt.Errorf("loading model %q with backend %q: %w", modelName, backendName, err)
}
```

See `LoadModel` in `.agents/debugging-backends.md` for the full loading flow.

## Sentinel Errors

Define package-level sentinel errors for conditions callers may want to check:

```go
var (
    ErrModelNotFound   = errors.New("model not found")
    ErrBackendNotReady = errors.New("backend not ready")
    ErrInvalidRequest  = errors.New("invalid request")
)
```

Callers can use `errors.Is` to branch on specific conditions:

```go
if errors.Is(err, ErrModelNotFound) {
    return c.Status(fiber.StatusNotFound).JSON(...)
}
```

## Panic Recovery

Backend goroutines should recover from panics to avoid crashing the whole server:

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Error().Interface("panic", r).Msg("backend goroutine panicked")
        }
    }()
    // backend work here
}()
```

## Logging Conventions

Use zerolog fields consistently:

```go
log.Error().
    Err(err).
    Str("model", modelName).
    Str("backend", backendName).
    Msg("inference failed")
```

Avoid `log.Fatal` outside of `main()` — prefer returning the error up the stack.

## Context Cancellation

Respect context cancellation in long-running operations:

```go
select {
case result := <-resultCh:
    return result, nil
case <-ctx.Done():
    return nil, fmt.Errorf("request cancelled: %w", ctx.Err())
}
```

## Testing Errors

In tests, prefer `require.NoError` for fatal assertions and `assert.ErrorIs` to check specific error types:

```go
require.NoError(t, err, "model load should not fail")
assert.ErrorIs(t, err, ErrModelNotFound)
```

# Logging and Observability in LocalAI

This guide covers how to add structured logging, metrics, and tracing to LocalAI components.

## Logging

LocalAI uses [zerolog](https://github.com/rs/zerolog) for structured logging. Import it via:

```go
import (
    "github.com/rs/zerolog/log"
)
```

### Log Levels

Use the appropriate level for your messages:

```go
log.Trace().Str("model", modelName).Msg("loading model")
log.Debug().Str("backend", backendName).Int("threads", threads).Msg("backend config")
log.Info().Str("address", addr).Msg("server started")
log.Warn().Err(err).Str("model", modelName).Msg("model file not found, skipping")
log.Error().Err(err).Str("request_id", reqID).Msg("inference failed")
log.Fatal().Err(err).Msg("failed to initialize backend")
```

### Contextual Fields

Always attach relevant context to log entries:

```go
log.Info().
    Str("model", modelName).
    Str("backend", backendName).
    Float64("duration_ms", elapsed.Milliseconds()).
    Int("tokens", tokenCount).
    Msg("inference complete")
```

Avoid bare `log.Info().Msg("done")` — always include enough context to debug without reading surrounding code.

### Request-scoped Logging

When handling HTTP requests, attach the request ID to a child logger and pass it through context:

```go
func MyHandler(c *fiber.Ctx) error {
    reqID := c.Get("X-Request-ID", uuid.New().String())
    logger := log.With().Str("request_id", reqID).Logger()
    ctx := logger.WithContext(c.Context())
    _ = ctx // pass ctx to downstream calls
    // ...
}
```

## Error Logging Patterns

Don't swallow errors silently. Always log with context:

```go
if err != nil {
    log.Error().
        Err(err).
        Str("model", modelName).
        Str("operation", "load").
        Msg("failed to load model")
    return fmt.Errorf("loading model %s: %w", modelName, err)
}
```

See `.agents/error-handling.md` for the full error wrapping conventions.

## Startup and Shutdown Logging

Log key lifecycle events at `Info` level:

```go
log.Info().Str("version", build.Version).Msg("LocalAI starting")
log.Info().Str("galleries", strings.Join(galleryURLs, ",")).Msg("galleries configured")
log.Info().Msg("LocalAI shutdown complete")
```

## Performance Logging

For operations that may be slow (model loading, inference), log timing:

```go
start := time.Now()
// ... do work ...
log.Debug().
    Str("model", modelName).
    Dur("elapsed", time.Since(start)).
    Msg("model loaded")
```

## Log Configuration

Log level is controlled via the `--log-level` CLI flag or `LOG_LEVEL` environment variable. Valid values: `trace`, `debug`, `info`, `warn`, `error`.

Default level in production builds is `info`. Set `debug` or `trace` during development.

```bash
LOG_LEVEL=debug local-ai --address :8080
```

## What NOT to Log

- **Never log secrets, API keys, or tokens** — even at trace level.
- Avoid logging full request/response bodies by default; gate behind `trace` level.
- Don't log inside tight loops without rate-limiting — use sampling or aggregate.

```go
// BAD
for _, token := range tokens {
    log.Debug().Str("token", token).Msg("processing")
}

// GOOD
log.Debug().Int("token_count", len(tokens)).Msg("processing tokens")
```

## Metrics (Prometheus)

LocalAI exposes a `/metrics` endpoint. Register custom metrics in your feature package:

```go
import "github.com/prometheus/client_golang/prometheus"

var inferenceLatency = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "localai_inference_duration_seconds",
        Help:    "Inference latency by model and backend.",
        Buckets: prometheus.DefBuckets,
    },
    []string{"model", "backend"},
)

func init() {
    prometheus.MustRegister(inferenceLatency)
}

// In your handler:
timer := prometheus.NewTimer(inferenceLatency.WithLabelValues(modelName, backendName))
defer timer.ObserveDuration()
```

## Tracing

Distributed tracing support is opt-in via OpenTelemetry. Wrap long operations:

```go
import "go.opentelemetry.io/otel"

tracer := otel.Tracer("localai/inference")
ctx, span := tracer.Start(ctx, "RunInference")
defer span.End()

span.SetAttributes(
    attribute.String("model", modelName),
    attribute.String("backend", backendName),
)
```

Tracing is disabled by default and enabled via `--otel-endpoint`.

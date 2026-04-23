# Middleware and Hooks in LocalAI

This guide covers how to add middleware and request/response hooks to the LocalAI API layer.

## Overview

LocalAI uses [Fiber](https://gofiber.io/) as its HTTP framework. Middleware is registered on the app or route-group level and runs before/after handlers.

## Adding Global Middleware

Global middleware is registered in the main router setup, typically in `pkg/api/api.go`:

```go
app := fiber.New(fiber.Config{
    BodyLimit: config.UploadLimitMB * 1024 * 1024,
})

// Logging middleware
app.Use(logger.New(logger.Config{
    Format: "[${time}] ${status} - ${latency} ${method} ${path}\n",
}))

// Recovery middleware — catches panics and returns 500
app.Use(recover.New())

// CORS middleware
app.Use(cors.New(cors.Config{
    AllowOrigins: "*",
    AllowHeaders: "Origin, Content-Type, Accept, Authorization",
}))
```

## Authentication Middleware

When an API key is configured, the auth middleware is applied to protected route groups:

```go
func AuthMiddleware(apiKey string) fiber.Handler {
    return func(c *fiber.Ctx) error {
        if apiKey == "" {
            // No auth configured, allow all requests
            return c.Next()
        }

        token := c.Get("Authorization")
        // Support both "Bearer <key>" and raw key formats
        token = strings.TrimPrefix(token, "Bearer ")

        if token != apiKey {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": fiber.Map{
                    "message": "invalid api key",
                    "type":    "invalid_request_error",
                    "code":    "invalid_api_key",
                },
            })
        }
        return c.Next()
    }
}
```

Apply it to a group:

```go
protected := app.Group("/v1", AuthMiddleware(cfg.APIKey))
protected.Get("/models", ListModelsHandler)
```

## Request ID Middleware

Attach a unique request ID to each request for tracing:

```go
func RequestIDMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        id := c.Get("X-Request-ID")
        if id == "" {
            id = uuid.New().String()
        }
        c.Locals("requestID", id)
        c.Set("X-Request-ID", id)
        return c.Next()
    }
}
```

Access it in a handler:

```go
func MyHandler(c *fiber.Ctx) error {
    reqID := c.Locals("requestID").(string)
    log.Printf("[%s] handling request", reqID)
    return c.Next()
}
```

## Model Loading Hook

Before inference, a pre-request hook can ensure the model is loaded:

```go
func EnsureModelLoaded(ml *model.ModelLoader, cfg *config.BackendConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        modelName := c.Params("model", cfg.Model)
        if modelName == "" {
            return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
                "error": "model name is required",
            })
        }

        // Attempt to load if not already in memory
        if err := ml.EnsureLoaded(modelName); err != nil {
            log.Printf("failed to load model %s: %v", modelName, err)
            return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
                "error": fmt.Sprintf("could not load model: %v", err),
            })
        }

        c.Locals("modelName", modelName)
        return c.Next()
    }
}
```

## Timeout Middleware

For long-running inference requests, configure per-route timeouts:

```go
import "github.com/gofiber/fiber/v2/middleware/timeout"

// Apply a 5-minute timeout to completion endpoints
protected.Post("/chat/completions",
    timeout.New(ChatCompletionsHandler, 5*time.Minute),
)
```

## Metrics Hook

To record per-request metrics (see `logging-and-observability.md` for full setup):

```go
func MetricsMiddleware(metrics *Metrics) fiber.Handler {
    return func(c *fiber.Ctx) error {
        start := time.Now()
        err := c.Next()
        duration := time.Since(start).Seconds()

        metrics.RequestDuration.WithLabelValues(
            c.Method(),
            c.Path(),
            strconv.Itoa(c.Response().StatusCode()),
        ).Observe(duration)

        return err
    }
}
```

## Order of Middleware

Register middleware in this order for correct behavior:

1. `RequestIDMiddleware` — assign IDs first
2. `logger.New` — log with request ID
3. `recover.New` — catch panics before they propagate
4. `cors.New` — handle preflight before auth
5. `AuthMiddleware` — reject unauthorized requests early
6. `MetricsMiddleware` — measure only authenticated requests
7. Route-specific hooks (e.g., `EnsureModelLoaded`)

## Tips

- Use `c.Locals()` to pass data between middleware and handlers within a single request.
- Avoid heavy computation in middleware; delegate to handlers or background workers.
- When writing tests, you can skip auth middleware by not setting `APIKey` in the test config.

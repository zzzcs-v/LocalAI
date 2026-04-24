# Request Lifecycle in LocalAI

This document explains how a request flows through LocalAI from the HTTP layer down to the backend and back.

## Overview

A typical inference request goes through the following stages:

1. **HTTP Router** — Fiber routes the request to the correct handler
2. **Middleware chain** — Auth, request ID, metrics, logging
3. **Handler** — Parses and validates the request body
4. **Model loading** — Ensures the model is loaded via `EnsureModelLoaded`
5. **Backend dispatch** — Calls the appropriate gRPC backend
6. **Response streaming or buffering** — Returns tokens or the full response

## Detailed Flow

### 1. Routing

Routes are registered in `RegisterMyRoutes` (see `api-endpoints-and-auth.md`). Each route maps to a handler function and a middleware chain.

```go
app.Post("/v1/chat/completions", authMiddleware, requestIDMiddleware, chatCompletionHandler)
```

### 2. Middleware Execution

Middleware runs in order before the handler. Common middleware:

- `AuthMiddleware` — validates bearer tokens or API keys
- `RequestIDMiddleware` — attaches a unique request ID for tracing
- `MetricsMiddleware` — records request counts and latencies

See `middleware-and-hooks.md` for implementation details.

### 3. Handler: Parse & Validate

Handlers unmarshal the JSON body into a request struct and validate required fields.

```go
func ChatCompletionHandler(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        req := new(schema.OpenAIRequest)
        if err := c.BodyParser(req); err != nil {
            return fiber.NewError(fiber.StatusBadRequest, "invalid request body: "+err.Error())
        }
        if req.Model == "" {
            return fiber.NewError(fiber.StatusBadRequest, "model is required")
        }
        // ... continue
    }
}
```

### 4. Config Resolution

Before loading a model, LocalAI resolves the backend config:

```go
cfg, err := cl.LoadBackendConfigFileByName(req.Model, appConfig.ModelPath)
if err != nil {
    // fall back to defaults
    cfg = config.DefaultBackendConfig(req.Model)
}
// merge request-level overrides into cfg
config.MergeRequestOptions(cfg, req)
```

This allows per-model configuration (context size, temperature defaults, backend selection) defined in YAML files to be respected while still allowing per-request overrides.

### 5. Model Loading

`EnsureModelLoaded` (see `middleware-and-hooks.md`) checks whether the model is already loaded in the `ModelLoader` cache. If not, it calls `LoadModel` which:

- Selects the correct backend (llama.cpp, whisper, diffusion, etc.)
- Starts the gRPC backend process if not running
- Loads model weights into memory
- Returns a gRPC client stub

Model loading is protected by a per-model mutex to prevent duplicate loads under concurrent requests.

### 6. Backend Dispatch

Once a client stub is available, the handler calls the appropriate gRPC method:

```go
resp, err := grpcClient.Predict(ctx, &pb.PredictOptions{
    Prompt:      prompt,
    Temperature: float32(cfg.Temperature),
    Tokens:      int32(cfg.Maxtokens),
})
```

For streaming responses, the handler calls `PredictStream` and pipes tokens back to the client via SSE:

```go
stream, err := grpcClient.PredictStream(ctx, opts)
for {
    chunk, err := stream.Recv()
    if err == io.EOF {
        break
    }
    c.Write(formatSSEChunk(chunk.Message))
}
```

### 7. Response

Non-streaming responses are buffered and returned as JSON. Streaming responses use `text/event-stream` with `data: [DONE]` as the terminator, matching the OpenAI API convention.

## Error Handling at Each Stage

| Stage | Error type | HTTP status |
|---|---|---|
| Body parse | `fiber.StatusBadRequest` | 400 |
| Auth failure | `fiber.StatusUnauthorized` | 401 |
| Model not found | `fiber.StatusNotFound` | 404 |
| Model load failure | `fiber.StatusInternalServerError` | 500 |
| Backend timeout | `fiber.StatusGatewayTimeout` | 504 |

See `error-handling.md` for the full error handling strategy.

## Observability

Each request gets a `X-Request-ID` header. Logs at each stage include this ID so you can trace a full request across log lines. See `logging-and-observability.md`.

## Tips for Adding a New Endpoint

1. Define the request/response schema in `pkg/schema/`
2. Register the route in your `RegisterMyRoutes` function
3. Apply the standard middleware chain
4. Use `EnsureModelLoaded` before dispatching to the backend
5. Follow the error handling conventions from `error-handling.md`
6. Add a log entry at handler entry and exit with the request ID

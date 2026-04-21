# State Management in LocalAI

This document describes how LocalAI manages application state, model lifecycle, and backend connections.

## Core Concepts

### ApplicationState

The central `ApplicationState` struct (in `pkg/application/application.go`) holds all runtime state:

```go
type ApplicationState struct {
    ModelLoader    *model.ModelLoader
    BackendLoader  *backend.BackendLoader
    Options        *config.ApplicationConfig
    Gallery        *gallery.GalleryManager
    // ...
}
```

This is initialized once at startup and passed via Fiber's `Locals` or through closures into route handlers.

### Accessing State in Handlers

Handlers receive state through the Fiber context:

```go
func MyHandler(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) gin.HandlerFunc {
    return func(c *fiber.Ctx) error {
        // Use cl, ml, appConfig directly from closure
        cfg, err := cl.GetBackendConfig(modelName)
        if err != nil {
            return err
        }
        // ...
    }
}
```

## Model Lifecycle

### Loading

Models are loaded lazily on first request via `ModelLoader.LoadModel()`.
See `.agents/debugging-backends.md` for `LoadModel` details.

1. Request comes in with `model` parameter
2. `BackendConfigLoader` resolves config for that model
3. `ModelLoader` checks if model is already in memory
4. If not, spawns backend process and loads weights
5. Returns a handle used for inference

### Unloading / Eviction

Models can be evicted from memory:
- Manually via `DELETE /models/loaded/{model}`
- Automatically when memory pressure is detected (if `SingleActiveBackend` is set)
- On shutdown via graceful cleanup

```go
// Force unload a specific model
ml.ShutdownModel(modelName)

// Unload all models
ml.StopAllGRPC()
```

### SingleActiveBackend Mode

When `appConfig.SingleActiveBackend` is `true`, only one model stays loaded at a time.
Before loading a new model, the loader evicts the current one:

```go
if appConfig.SingleActiveBackend {
    ml.StopAllGRPC()
}
```

This is useful on memory-constrained systems.

## Configuration State

### BackendConfigLoader

Holds all resolved `BackendConfig` objects, keyed by model name.

- Configs are loaded from YAML files in the models directory
- Environment variables can override fields
- Configs are re-read on `POST /models/apply` (gallery install)

```go
// List all known configs
configs := cl.GetAllBackendConfigs()

// Get one by name
cfg, err := cl.GetBackendConfig("gpt-4")

// Add/update a config at runtime
cl.StoreBackendModelConfig("my-model", &config.BackendConfig{...})
```

### ApplicationConfig

Global settings parsed from CLI flags and environment variables.
This is **read-only** after startup — do not mutate it in handlers.

Key fields:
- `ModelPath` — directory where model files live
- `UploadDir` — where uploaded files are stored
- `SingleActiveBackend` — evict models on each new load
- `PreloadModels` — models to warm up at startup
- `ExternalGRPCBackends` — map of backend name → address

## Concurrency Considerations

- `ModelLoader` uses internal mutexes; safe for concurrent handler calls
- `BackendConfigLoader` uses `sync.RWMutex`; reads are concurrent, writes are serialized
- Do **not** store request-scoped data in application-level structs
- Use `fiber.Ctx.Locals()` for per-request state

## Startup Sequence

```
main()
  └─ app.New(opts...)          # build ApplicationState
       ├─ config loader init
       ├─ model loader init
       ├─ gallery manager init
       ├─ preload models        # warm up configured models
       └─ register routes       # see api-endpoints-and-auth.md
```

## Shutdown Sequence

On SIGINT/SIGTERM:
1. Fiber stops accepting new requests
2. In-flight requests complete (with timeout)
3. `ml.StopAllGRPC()` terminates backend processes
4. Temp files / sockets are cleaned up

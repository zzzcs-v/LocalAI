# Model Loading Lifecycle

This document describes how models are loaded, cached, and unloaded in LocalAI, and how to work with the model loading pipeline.

## Overview

LocalAI uses a lazy-loading approach: models are loaded on first request and cached in memory. The `ModelLoader` is the central component responsible for managing model instances.

## Key Components

### ModelLoader

Located in `pkg/model/loader.go`, the `ModelLoader` manages the lifecycle of all backend model instances.

```go
type ModelLoader struct {
    ModelPath string
    mu        sync.Mutex
    models    map[string]*BackendModel
    // grpc clients keyed by model name
    grpcClients map[string]grpc.Backend
}
```

### BackendModel

Represents a loaded model instance with its associated gRPC backend client.

```go
type BackendModel struct {
    ID       string
    Backend  string
    Options  *Options
    Client   grpc.Backend
    LoadedAt time.Time
}
```

## Loading Flow

1. **Request arrives** — handler calls `ml.LoadModel(modelName, options)`
2. **Cache check** — if model is already loaded and healthy, return existing client
3. **Backend selection** — determine which backend to use based on config or file extension
4. **Process spawn** — start the backend gRPC server process
5. **gRPC connect** — establish connection to the backend
6. **Model load** — call `backend.LoadModel(request)` over gRPC
7. **Cache store** — store the client in the loader's map
8. **Return client** — handler uses the client for inference

```
Request
  │
  ▼
ModelLoader.LoadModel(name, opts)
  │
  ├─► cache hit? ──yes──► return cached client
  │
  └─► cache miss
        │
        ▼
      selectBackend(name, opts)
        │
        ▼
      spawnBackendProcess(backend)
        │
        ▼
      grpc.NewClient(address)
        │
        ▼
      client.LoadModel(modelPath, opts)
        │
        ▼
      store in ml.models[name]
        │
        ▼
      return client
```

## Backend Selection

Backends are selected in priority order:

1. Explicitly set in model config (`backend: llama-cpp`)
2. Inferred from file extension (`.gguf` → `llama-cpp`)
3. Inferred from model name patterns
4. Default fallback backend

```go
func selectBackend(modelName string, cfg *config.BackendConfig) string {
    if cfg.Backend != "" {
        return cfg.Backend
    }
    // extension-based detection
    switch filepath.Ext(modelName) {
    case ".gguf", ".ggml":
        return LlamaCPP
    case ".bin":
        return Gpt4All
    }
    return DefaultBackend
}
```

## Model Options

Options passed during loading come from the merged config:

```go
type Options struct {
    // Model file path on disk
    ModelFile string
    // Number of threads to use
    Threads int
    // Context size
    ContextSize int
    // GPU layers to offload
    NGPULayers int
    // Additional backend-specific options
    BackendOptions map[string]interface{}
}
```

Options are merged from (lowest to highest priority):
- Built-in defaults
- Global server config
- Per-model config file
- Request-time overrides (where allowed)

## Concurrency Safety

The `ModelLoader` uses a mutex to protect the models map. Loading the same model concurrently is safe — the second caller will block until the first completes, then receive the cached instance.

```go
func (ml *ModelLoader) LoadModel(name string, opts *Options) (grpc.Backend, error) {
    ml.mu.Lock()
    defer ml.mu.Unlock()

    if existing, ok := ml.models[name]; ok {
        if existing.Client.HealthCheck(ctx) == nil {
            return existing.Client, nil
        }
        // unhealthy — fall through to reload
        delete(ml.models, name)
    }
    // ... load and cache
}
```

## Model Unloading

Models can be unloaded to free memory:

```go
// Unload a specific model
ml.ShutdownModel(modelName)

// Unload all models (e.g., on server shutdown)
ml.StopAllGRPC()
```

Unloading terminates the backend process and removes the entry from the cache.

## Health Checks

Before returning a cached model, LocalAI performs a lightweight health check over gRPC. If the backend process has died, it will be restarted transparently.

## Debugging Model Loading

Enable verbose logging to trace the loading pipeline:

```bash
LOCAL_AI_LOG_LEVEL=debug ./local-ai --models-path ./models
```

Key log messages to look for:
- `"loading model"` — model load initiated
- `"model loaded successfully"` — load complete
- `"reusing existing model"` — cache hit
- `"backend process started"` — gRPC server spawned
- `"health check failed, reloading"` — backend recovered from crash

## Common Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| `model not found` | Wrong model path or name | Check `--models-path` and model filename |
| `backend failed to start` | Missing binary or wrong arch | Check backend binaries in `./backends/` |
| `context deadline exceeded` | Model too large, slow load | Increase `--backend-assets-path` timeout |
| `out of memory` | Too many models loaded | Set `--single-active-backend` or reduce context size |

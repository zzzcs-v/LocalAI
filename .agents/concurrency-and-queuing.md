# Concurrency and Queuing in LocalAI

This document describes how LocalAI handles concurrent requests, model loading locks, and request queuing to avoid race conditions and resource exhaustion.

## Overview

LocalAI uses a combination of mutexes, semaphores, and per-model queues to safely handle multiple simultaneous inference requests. Since most backends (especially llama.cpp) are not thread-safe, requests must be serialized per model.

## Per-Model Mutex

Each loaded model has an associated mutex that ensures only one inference runs at a time:

```go
type ModelMutex struct {
    mu      sync.Mutex
    modelID string
}

func (m *ModelMutex) Lock() {
    log.Debug().Str("model", m.modelID).Msg("acquiring model lock")
    m.mu.Lock()
}

func (m *ModelMutex) Unlock() {
    log.Debug().Str("model", m.modelID).Msg("releasing model lock")
    m.mu.Unlock()
}
```

The `ModelLoader` maintains a map of these mutexes keyed by model name.

## Request Queuing Pattern

For high-throughput scenarios, use a buffered channel as a semaphore to limit concurrent backend calls:

```go
type InferenceQueue struct {
    sem     chan struct{}
    modelID string
}

func NewInferenceQueue(modelID string, maxConcurrent int) *InferenceQueue {
    return &InferenceQueue{
        sem:     make(chan struct{}, maxConcurrent),
        modelID: modelID,
    }
}

func (q *InferenceQueue) Acquire(ctx context.Context) error {
    select {
    case q.sem <- struct{}{}:
        return nil
    case <-ctx.Done():
        return fmt.Errorf("queue acquire cancelled for model %s: %w", q.modelID, ctx.Err())
    }
}

func (q *InferenceQueue) Release() {
    <-q.sem
}
```

Always use `defer q.Release()` after a successful `Acquire` to avoid deadlocks.

## Model Loading Lock

Model loading is expensive and must not happen concurrently for the same model. Use a `singleflight` group:

```go
import "golang.org/x/sync/singleflight"

var loadGroup singleflight.Group

func LoadModel(modelName string, loader *ModelLoader) (Model, error) {
    result, err, _ := loadGroup.Do(modelName, func() (interface{}, error) {
        return loader.loadFromDisk(modelName)
    })
    if err != nil {
        return nil, err
    }
    return result.(Model), nil
}
```

The third return value (shared bool) can be logged to observe cache coalescing.

## Context Propagation and Cancellation

All blocking operations must respect the request context:

```go
func RunInference(ctx context.Context, model Model, req *InferenceRequest) (*InferenceResponse, error) {
    done := make(chan struct{})
    var resp *InferenceResponse
    var inferErr error

    go func() {
        defer close(done)
        resp, inferErr = model.Infer(req)
    }()

    select {
    case <-done:
        return resp, inferErr
    case <-ctx.Done():
        // Signal backend to stop if it supports cancellation
        model.Cancel()
        return nil, fmt.Errorf("inference cancelled: %w", ctx.Err())
    }
}
```

## Avoiding Common Pitfalls

- **Never** hold a model mutex while doing I/O or waiting on another lock — this causes deadlocks.
- **Always** set a timeout on the context before entering a queue: `ctx, cancel = context.WithTimeout(ctx, cfg.InferenceTimeout)`.
- **Log** when a request waits more than a threshold (e.g., 5s) so operators can tune `maxConcurrent`.
- **Use `sync.RWMutex`** for the model registry (many readers loading configs, rare writers adding models).

## Configuration

Relevant settings in `LocalAIConfig`:

| Field | Default | Description |
|---|---|---|
| `SingleActiveBackend` | `false` | Serialize all backends globally (for low-RAM systems) |
| `MaxConcurrentRequests` | `0` (unlimited) | Cap on simultaneous inference calls per model |
| `InferenceTimeout` | `10m` | Per-request deadline passed to backend |

## Related Files

- `pkg/model/loader.go` — `ModelLoader`, mutex map
- `pkg/grpc/client.go` — gRPC call with context cancellation
- `.agents/state-management.md` — global state and registry patterns
- `.agents/error-handling.md` — how to surface timeout/cancellation errors to callers

# Configuration and Settings

This guide covers how LocalAI handles configuration, model settings, and application-level options.

## Application Configuration

LocalAI uses a layered configuration system:

1. **Default values** baked into the code
2. **Config file** (YAML) loaded at startup
3. **Environment variables** override config file values
4. **CLI flags** take highest precedence

```go
// core/config/application_config.go
type ApplicationConfig struct {
    Context           context.Context
    ConfigFile        string
    ModelPath         string
    UploadLimitMB     int
    UploadDir         string
    ImageDir          string
    AudioDir          string
    BackendAssetsPath string
    AssetsDestination string
    CORS              bool
    CORSAllowOrigins  string
    Threads           int
    Debug             bool
    LibDir            string
    Addr              string
    BaseURL           string
    ApiKeys           []string
    DisableWebUI      bool
    ParallelRequests  bool
    SingleActiveBackend bool
    PreloadModels     []string
    PreloadModelsFromPath string
    F16               bool
    AutoloadGalleries bool
    RemoteFallbackGrpcBackends []string
    ExternalGRPCBackends map[string]string
    Galleries         []Gallery
}
```

## Model Configuration (BackendConfig)

Each model has its own `BackendConfig` loaded from a YAML file alongside the model, or from a dedicated config directory.

```yaml
# Example: models/gpt-4.yaml
name: gpt-4
backend: llama-cpp
parameters:
  model: llama-2-7b-chat.Q4_K_M.gguf
  context_size: 4096
  threads: 4
  f16: true
  mmap: true

template:
  chat: |
    {{.Input}}
  chat_message: |
    {{if eq .RoleName "user"}}[INST] {{.Content}} [/INST]{{end}}
    {{if eq .RoleName "assistant"}}{{.Content}}{{end}}
    {{if eq .RoleName "system"}}<<SYS>>\n{{.Content}}\n<</SYS>>\n{{end}}

stoprwords:
  - "[INST]"
  - "[/INST]"

context_size: 4096
roles:
  user: "[INST]"
  assistant: "[/INST]"
```

## Loading BackendConfig

```go
// Load a single config by model name
cfg, err := cl.LoadBackendConfigFileByName(modelName, appConfig.ModelPath)
if err != nil {
    // fall back to auto-detection
    cfg = &config.BackendConfig{}
    cfg.Name = modelName
}

// Load all configs from a directory
err = cl.LoadBackendConfigsFromPath(appConfig.ModelPath)
```

## Environment Variable Mapping

| Env Var | Config Field | Default |
|---|---|---|
| `LOCALAI_MODELS_PATH` / `MODELS_PATH` | `ModelPath` | `./models` |
| `LOCALAI_THREADS` / `THREADS` | `Threads` | `runtime.NumCPU()` |
| `LOCALAI_DEBUG` / `DEBUG` | `Debug` | `false` |
| `LOCALAI_ADDR` / `ADDRESS` | `Addr` | `:8080` |
| `LOCALAI_API_KEY` / `API_KEY` | `ApiKeys` | `[]` |
| `LOCALAI_CORS` / `CORS` | `CORS` | `false` |
| `LOCALAI_UPLOAD_LIMIT` | `UploadLimitMB` | `15` |
| `LOCALAI_F16` / `F16` | `F16` | `false` |
| `LOCALAI_SINGLE_ACTIVE_BACKEND` | `SingleActiveBackend` | `false` |
| `LOCALAI_PARALLEL_REQUESTS` | `ParallelRequests` | `false` |

## Config Watchers

LocalAI watches model config directories for changes at runtime:

```go
// Configs are reloaded without restart when files change
// This allows adding new models without downtime
watcher, err := fsnotify.NewWatcher()
watcher.Add(appConfig.ModelPath)

go func() {
    for event := range watcher.Events {
        if event.Op&fsnotify.Write == fsnotify.Write {
            cl.LoadBackendConfigsFromPath(appConfig.ModelPath)
        }
    }
}()
```

## Validation

Always validate config before use:

```go
func (cfg *BackendConfig) Validate() error {
    if cfg.Name == "" {
        return fmt.Errorf("model name cannot be empty")
    }
    if cfg.Parameters.Model == "" {
        return fmt.Errorf("model file not specified for %s", cfg.Name)
    }
    return nil
}
```

## Tips

- Prefer YAML config files over env vars for model-specific settings
- Use env vars for deployment/infrastructure concerns (ports, paths, keys)
- `SingleActiveBackend` is useful on low-memory systems — only one model loaded at a time
- `ParallelRequests` allows concurrent inference on the same model (if backend supports it)
- Config files are hot-reloaded; no restart needed for model config changes

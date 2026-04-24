# Testing and Mocking in LocalAI

This guide covers how to write tests for LocalAI components, including unit tests, integration tests, and how to mock backends and services.

## Test Structure

Tests live alongside the code they test, following Go conventions:

```
core/
  backend/
    llm.go
    llm_test.go
  http/
    api.go
    api_test.go
```

## Unit Testing Handlers

Use `net/http/httptest` to test Fiber handlers without spinning up a full server:

```go
package http_test

import (
    "encoding/json"
    "io"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "github.com/gofiber/fiber/v2"
    "github.com/mudler/LocalAI/core/config"
    "github.com/mudler/LocalAI/core/http/endpoints"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestChatCompletionHandler(t *testing.T) {
    app := fiber.New()
    // Register routes with mock dependencies
    app.Post("/v1/chat/completions", endpoints.ChatCompletionHandler(mockBackendLoader, mockModelLoader, appConfig))

    body := `{"model": "test-model", "messages": [{"role": "user", "content": "hello"}]}`
    req := httptest.NewRequest(http.MethodPost, "/v1/chat/completions", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    resp, err := app.Test(req, 30000)
    require.NoError(t, err)
    assert.Equal(t, http.StatusOK, resp.StatusCode)
}
```

## Mocking the Backend Loader

Implement the `BackendLoader` interface for tests:

```go
type MockBackendLoader struct {
    // LoadModelFn lets tests override behavior per-test
    LoadModelFn func(modelName string, opts ...config.BackendOption) (ModelBackend, error)
}

func (m *MockBackendLoader) LoadModel(modelName string, opts ...config.BackendOption) (ModelBackend, error) {
    if m.LoadModelFn != nil {
        return m.LoadModelFn(modelName, opts...)
    }
    return &MockModelBackend{}, nil
}

type MockModelBackend struct {
    PredictFn func(input string, opts ...PredictOption) (string, error)
}

func (m *MockModelBackend) Predict(input string, opts ...PredictOption) (string, error) {
    if m.PredictFn != nil {
        return m.PredictFn(input, opts...)
    }
    return "mock response", nil
}
```

## Testing Configuration Loading

```go
func TestConfigLoading(t *testing.T) {
    t.Run("loads valid yaml config", func(t *testing.T) {
        dir := t.TempDir()
        cfgFile := filepath.Join(dir, "model.yaml")
        err := os.WriteFile(cfgFile, []byte(`
name: test-model
backend: llama
parameters:
  model: model.gguf
  temperature: 0.7
`), 0644)
        require.NoError(t, err)

        cfg, err := config.LoadBackendConfigFromFile(cfgFile)
        require.NoError(t, err)
        assert.Equal(t, "test-model", cfg.Name)
        assert.Equal(t, "llama", cfg.Backend)
        assert.Equal(t, float64(0.7), cfg.Parameters.Temperature)
    })

    t.Run("returns error on missing file", func(t *testing.T) {
        _, err := config.LoadBackendConfigFromFile("/nonexistent/path.yaml")
        assert.Error(t, err)
    })
}
```

## Integration Tests

Integration tests are tagged with `//go:build integration` to keep them out of the default test run:

```go
//go:build integration

package integration_test

import (
    "testing"
    "github.com/stretchr/testify/require"
)

func TestLlamaCppBackendLoadsModel(t *testing.T) {
    // Requires a real model file; set MODEL_PATH env var
    modelPath := os.Getenv("MODEL_PATH")
    if modelPath == "" {
        t.Skip("MODEL_PATH not set, skipping integration test")
    }
    // ... actual backend loading test
}
```

Run integration tests with:
```bash
go test -tags integration ./...
```

## Table-Driven Tests

Prefer table-driven tests for covering multiple input cases:

```go
func TestTokenEstimation(t *testing.T) {
    cases := []struct{
        name     string
        input    string
        expected int
    }{
        {"empty string", "", 0},
        {"single word", "hello", 1},
        {"sentence", "the quick brown fox", 4},
    }

    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            got := EstimateTokens(tc.input)
            assert.Equal(t, tc.expected, got)
        })
    }
}
```

## Running Tests

```bash
# Run all unit tests
go test ./...

# Run with race detector (recommended for concurrent code)
go test -race ./...

# Run a specific package
go test ./core/http/...

# Run with verbose output
go test -v ./core/backend/...

# Run integration tests
go test -tags integration -v ./tests/integration/...
```

## Tips

- Use `t.TempDir()` for temporary files — it auto-cleans after the test.
- Use `require` for fatal assertions (stops the test) and `assert` for non-fatal ones.
- Keep mocks in a `testutil` or `mocks` package if shared across multiple test files.
- Avoid global state in tests; pass dependencies explicitly.

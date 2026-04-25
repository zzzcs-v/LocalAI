# Image Generation in LocalAI

This guide covers how image generation endpoints work in LocalAI, including the request lifecycle, backend integration, and how to add support for new image generation backends.

## Overview

LocalAI supports image generation via the `/v1/images/generations` endpoint (OpenAI-compatible) and `/v1/images/edits` for image editing. Backends like `stablediffusion`, `diffusers`, and `tinydream` handle the actual generation.

## Endpoint Registration

```go
// core/http/routes/image.go
func RegisterImageRoutes(router *fiber.App, sl *model.ModelLoader, appConfig *config.ApplicationConfig, cl *config.BackendConfigLoader) {
    router.Post("/v1/images/generations", auth, ImageGenerationEndpoint(cl, ml, appConfig))
    router.Post("/v1/images/edits", auth, ImageEditEndpoint(cl, ml, appConfig))
}
```

## Request Structure

The image generation request mirrors the OpenAI API:

```go
type ImageGenerationRequest struct {
    Model          string `json:"model"`
    Prompt         string `json:"prompt"`
    NegativePrompt string `json:"negative_prompt"` // LocalAI extension
    Size           string `json:"size"`            // e.g. "512x512"
    N              int    `json:"n"`
    ResponseFormat string `json:"response_format"` // "url" or "b64_json"
    Step           int    `json:"step"`            // LocalAI extension
}
```

## Handler Implementation

```go
func ImageGenerationEndpoint(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        input := new(schema.ImageGenerationRequest)
        if err := c.BodyParser(input); err != nil {
            return fiber.ErrBadRequest
        }

        modelFile, cfg, err := backend.ModelForRequest(input.Model, cl, ml, appConfig)
        if err != nil {
            return err
        }

        // Resolve output size
        width, height := resolveImageSize(input.Size, cfg)

        results := []schema.ImageData{}
        for i := 0; i < resolveN(input.N); i++ {
            outFile := filepath.Join(appConfig.ImageDir, fmt.Sprintf("%s-%d.png", uuid.New().String(), i))

            err := backend.GenerateImage(backend.ImageRequest{
                Prompt:         input.Prompt,
                NegativePrompt: input.NegativePrompt,
                Width:          width,
                Height:         height,
                Step:           resolveStep(input.Step, cfg),
                Seed:           cfg.Seed,
                OutputFile:     outFile,
            }, modelFile, ml, appConfig, cfg)
            if err != nil {
                return fmt.Errorf("image generation failed: %w", err)
            }

            if input.ResponseFormat == "b64_json" {
                b64, err := fileToBase64(outFile)
                if err != nil {
                    return err
                }
                results = append(results, schema.ImageData{B64JSON: b64})
            } else {
                results = append(results, schema.ImageData{URL: fmt.Sprintf("/generated-images/%s", filepath.Base(outFile))})
            }
        }

        return c.JSON(schema.ImageGenerationResponse{
            Created: time.Now().Unix(),
            Data:    results,
        })
    }
}
```

## Backend Interface

Image generation backends implement the `ImageBackend` interface:

```go
type ImageRequest struct {
    Prompt         string
    NegativePrompt string
    Width          int
    Height         int
    Step           int
    Seed           int
    OutputFile     string
    // For edits
    InitImage string
    MaskImage string
}

// Backends call the gRPC GenerateImage method
func GenerateImage(req ImageRequest, modelFile string, ml *model.ModelLoader, ...) error {
    model, err := ml.Load(modelFile, ...)
    if err != nil {
        return err
    }
    return model.GeneratesImage(req.Prompt, req.NegativePrompt, req.Width, req.Height, req.Step, req.Seed, req.OutputFile)
}
```

## Config Options

Relevant `BackendConfig` fields for image generation:

```yaml
name: my-sd-model
backend: stablediffusion
parameters:
  model: v1-5-pruned-emaonly.safetensors
  size: 512x512   # default size
  step: 20        # default steps
seed: -1          # -1 for random
negative_prompt: "blurry, low quality"
```

## Serving Generated Images

Generated images are saved to `appConfig.ImageDir` and served statically:

```go
router.Static("/generated-images", appConfig.ImageDir)
```

The directory is configurable via `--image-path` CLI flag or `IMAGE_PATH` env var. Defaults to a temp directory if not set.

## Adding a New Image Backend

1. Implement the `GeneratesImage` gRPC method in your backend proto
2. Register the backend name in `core/backend/image.go`
3. Map config options to backend parameters in the loader
4. Test with a small model (e.g., `tinydream` is lightweight for CI)

## Testing

```go
func TestImageGenerationEndpoint(t *testing.T) {
    // Use tinydream or a mock backend for unit tests
    // Integration tests use a real stablediffusion model
    app, cl, ml := setupTestApp(t)

    req := httptest.NewRequest("POST", "/v1/images/generations", strings.NewReader(`{
        "model": "tinydream",
        "prompt": "a cat",
        "size": "64x64"
    }`))
    req.Header.Set("Content-Type", "application/json")

    resp, err := app.Test(req)
    require.NoError(t, err)
    require.Equal(t, 200, resp.StatusCode)

    var result schema.ImageGenerationResponse
    json.NewDecoder(resp.Body).Decode(&result)
    assert.Len(t, result.Data, 1)
    assert.NotEmpty(t, result.Data[0].URL)
}
```

## Common Issues

- **OOM errors**: Reduce image size or steps; stablediffusion is memory-hungry
- **Slow generation**: Expected on CPU; use `--threads` to tune
- **Format errors**: Ensure the model file matches the backend (`.ckpt` vs `.safetensors`)
- **Missing output file**: Check `ImageDir` is writable and has sufficient disk space

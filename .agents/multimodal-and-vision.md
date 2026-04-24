# Multimodal and Vision Support in LocalAI

This guide covers how LocalAI handles multimodal inputs (images, audio) alongside text, particularly for vision-capable models like LLaVA, Bakllava, and OpenAI-compatible vision endpoints.

## Overview

Multimodal requests follow the OpenAI Chat Completions API format where `content` can be an array of objects rather than a plain string. Each object has a `type` field (`text` or `image_url`) and the corresponding payload.

```json
{
  "model": "llava",
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "What is in this image?" },
        { "type": "image_url", "image_url": { "url": "data:image/jpeg;base64,..." } }
      ]
    }
  ]
}
```

## Request Parsing

The `schema.OpenAIRequest` struct handles polymorphic `content` fields via a custom unmarshaller. When content is an array, it's stored as `[]schema.ContentPart`.

```go
// schema/openai.go (simplified)
type ContentPart struct {
    Type     string     `json:"type"`
    Text     string     `json:"text,omitempty"`
    ImageURL *ImageURL  `json:"image_url,omitempty"`
}

type ImageURL struct {
    URL    string `json:"url"`
    Detail string `json:"detail,omitempty"` // "low", "high", "auto"
}
```

## Image Preprocessing Pipeline

Before passing images to a backend, LocalAI:

1. **Decodes** base64 data URIs or downloads remote URLs
2. **Validates** MIME type (jpeg, png, webp, gif)
3. **Writes** the image to a temp file
4. **Passes** the temp file path to the backend via `ImagePath` in the prediction options

```go
// core/images/preprocess.go
func PreprocessImage(imageURL string, tempDir string) (string, error) {
    if strings.HasPrefix(imageURL, "data:") {
        return decodeBase64Image(imageURL, tempDir)
    }
    return downloadImage(imageURL, tempDir)
}

func decodeBase64Image(dataURI string, tempDir string) (string, error) {
    // data:image/jpeg;base64,<data>
    parts := strings.SplitN(dataURI, ",", 2)
    if len(parts) != 2 {
        return "", fmt.Errorf("invalid data URI format")
    }
    decoded, err := base64.StdEncoding.DecodeString(parts[1])
    if err != nil {
        return "", fmt.Errorf("base64 decode failed: %w", err)
    }
    ext := extensionFromDataURI(parts[0]) // e.g. ".jpg"
    tmpFile, err := os.CreateTemp(tempDir, "img-*"+ext)
    if err != nil {
        return "", err
    }
    defer tmpFile.Close()
    if _, err := tmpFile.Write(decoded); err != nil {
        return "", err
    }
    return tmpFile.Name(), nil
}
```

## Backend Integration

Backends that support vision (e.g., llama.cpp with LLaVA clip) receive image paths through `pb.PredictOptions`:

```go
opts := &pb.PredictOptions{
    Prompt:    textPrompt,
    ImageData: [][]byte{imageBytes}, // raw bytes for gRPC backends
    // OR
    // ImagePath is set for backends that read from disk
}
```

The llama.cpp backend uses the `--mmproj` flag to load the multimodal projector alongside the base model:

```yaml
# models/llava.yaml
name: llava
backend: llama-cpp
parameters:
  model: llava-v1.6-mistral-7b.Q4_K_M.gguf
mmproj: mmproj-mistral7b-f16.gguf
```

## Content Extraction Helper

When building prompts, use the helper to flatten multimodal content:

```go
// Returns (textContent, imagePaths, error)
func ExtractContentParts(parts []schema.ContentPart, tempDir string) (string, []string, error) {
    var texts []string
    var images []string

    for _, part := range parts {
        switch part.Type {
        case "text":
            texts = append(texts, part.Text)
        case "image_url":
            if part.ImageURL == nil {
                continue
            }
            imgPath, err := PreprocessImage(part.ImageURL.URL, tempDir)
            if err != nil {
                return "", nil, fmt.Errorf("image preprocessing failed: %w", err)
            }
            images = append(images, imgPath)
        default:
            log.Warnf("unknown content part type: %s", part.Type)
        }
    }

    return strings.Join(texts, "\n"), images, nil
}
```

## Model Configuration

Key config fields for vision models:

| Field | Description |
|---|---|
| `mmproj` | Path to the multimodal projector (CLIP model) |
| `vision_application_regex` | Regex to determine if a request should use vision |
| `image_path_prefix` | Optional prefix added to resolved image paths |

## Cleanup

Temp files created during preprocessing are cleaned up after the request completes. The handler is responsible for deferring cleanup:

```go
tempDir, err := os.MkdirTemp("", "localai-vision-*")
if err != nil {
    return err
}
defer os.RemoveAll(tempDir)
```

## Supported Backends

| Backend | Vision Support | Notes |
|---|---|---|
| llama-cpp | ✅ | Requires `mmproj` config |
| ollama | ✅ | Handled natively |
| openai | ✅ | Proxied as-is |
| whisper | ❌ | Audio only |
| diffusers | N/A | Image generation |

## Testing Vision Endpoints

```bash
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "llava",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "Describe this image"},
        {"type": "image_url", "image_url": {"url": "https://example.com/cat.jpg"}}
      ]
    }]
  }'
```

See also: `core/backend/llm.go`, `api/openai/chat.go`, `pkg/model/loader.go`

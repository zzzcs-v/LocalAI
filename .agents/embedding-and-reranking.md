# Embedding and Reranking in LocalAI

This guide covers how embedding generation and reranking work in LocalAI, including backend selection, request handling, and response formatting.

## Overview

LocalAI supports two related but distinct inference tasks:

- **Embeddings**: Convert text into dense vector representations (used for semantic search, RAG pipelines, etc.)
- **Reranking**: Score and reorder a list of documents relative to a query (used to improve retrieval quality)

Both tasks share similar lifecycle patterns but differ in input/output shapes and backend requirements.

---

## Embedding Request Lifecycle

### 1. Incoming Request

The `/v1/embeddings` endpoint accepts an `EmbeddingRequest`:

```go
type EmbeddingRequest struct {
    Model string          `json:"model"`
    Input interface{}     `json:"input"` // string or []string
    EncodingFormat string `json:"encoding_format,omitempty"` // "float" or "base64"
    User   string         `json:"user,omitempty"`
}
```

The `Input` field can be a single string or a slice of strings. Normalize it early:

```go
func normalizeEmbeddingInput(input interface{}) ([]string, error) {
    switch v := input.(type) {
    case string:
        return []string{v}, nil
    case []interface{}:
        result := make([]string, 0, len(v))
        for _, item := range v {
            s, ok := item.(string)
            if !ok {
                return nil, fmt.Errorf("embedding input must be string or []string")
            }
            result = append(result, s)
        }
        return result, nil
    default:
        return nil, fmt.Errorf("unsupported embedding input type: %T", input)
    }
}
```

### 2. Backend Selection

Embedding backends are selected via `selectBackend` (see `model-loading-lifecycle.md`). Backends that support embeddings typically set `SupportsEmbeddings: true` in their capability flags.

Common embedding backends:
- `llama-cpp` (with embedding mode enabled)
- `bert-embeddings`
- `sentencetransformers`

In the model config, ensure:

```yaml
backend: bert-embeddings
embeddings: true
```

### 3. Generating Embeddings

Call the backend's `Embeddings` method through the model loader:

```go
embeddings, err := ml.Embeddings(opts, text)
if err != nil {
    return schema.ErrorResponse{
        Error: &schema.APIError{
            Code:    500,
            Message: fmt.Sprintf("failed to generate embeddings: %v", err),
            Type:    "embedding_error",
        },
    }
}
```

### 4. Response Format

Return an `EmbeddingResponse` compatible with the OpenAI spec:

```go
type EmbeddingResponse struct {
    Object string              `json:"object"` // always "list"
    Data   []EmbeddingData     `json:"data"`
    Model  string              `json:"model"`
    Usage  EmbeddingUsage      `json:"usage"`
}

type EmbeddingData struct {
    Object    string    `json:"object"` // always "embedding"
    Embedding []float32 `json:"embedding"`
    Index     int       `json:"index"`
}

type EmbeddingUsage struct {
    PromptTokens int `json:"prompt_tokens"`
    TotalTokens  int `json:"total_tokens"`
}
```

---

## Reranking Request Lifecycle

### 1. Incoming Request

The `/rerank` endpoint (non-OpenAI, Cohere-compatible) accepts:

```go
type RerankRequest struct {
    Model     string   `json:"model"`
    Query     string   `json:"query"`
    Documents []string `json:"documents"`
    TopN      int      `json:"top_n,omitempty"`
}
```

### 2. Response Format

```go
type RerankResponse struct {
    Results []RerankResult `json:"results"`
    Model   string         `json:"model"`
    Usage   RerankUsage    `json:"usage"`
}

type RerankResult struct {
    Index          int     `json:"index"`
    RelevanceScore float32 `json:"relevance_score"`
    Document       *RerankDocument `json:"document,omitempty"`
}

type RerankDocument struct {
    Text string `json:"text"`
}
```

### 3. TopN Filtering

If `TopN > 0`, sort results by `RelevanceScore` descending and return only the top N:

```go
if req.TopN > 0 && req.TopN < len(results) {
    sort.Slice(results, func(i, j int) bool {
        return results[i].RelevanceScore > results[j].RelevanceScore
    })
    results = results[:req.TopN]
}
```

---

## Configuration

### Model Config for Embeddings

```yaml
name: my-embedder
backend: bert-embeddings
parameters:
  model: /models/all-minilm-l6-v2.bin
embeddings: true
context_size: 512
```

### Model Config for Reranking

```yaml
name: my-reranker
backend: rerankers
parameters:
  model: /models/bge-reranker-base.bin
context_size: 512
```

---

## Common Pitfalls

- **Batch size limits**: Some backends can't handle large batches. Split inputs if needed and aggregate results.
- **Dimension mismatch**: If the downstream vector DB expects a fixed dimension, validate it against the model's output before returning.
- **Float vs base64**: The `encoding_format` field is often ignored by backends — handle the conversion at the API layer if needed.
- **Empty input**: Always validate that the input slice is non-empty before calling the backend.

---

## Testing

See `testing-and-mocking.md` for general patterns. For embeddings specifically:

```go
func TestEmbeddingNormalization(t *testing.T) {
    // single string
    out, err := normalizeEmbeddingInput("hello world")
    assert.NoError(t, err)
    assert.Equal(t, []string{"hello world"}, out)

    // slice of strings
    out, err = normalizeEmbeddingInput([]interface{}{"foo", "bar"})
    assert.NoError(t, err)
    assert.Equal(t, []string{"foo", "bar"}, out)
}
```

For integration tests, use a small fast embedding model (e.g., `all-minilm`) with a mock backend that returns fixed vectors.

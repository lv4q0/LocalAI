# Embeddings and Reranking in LocalAI

This guide covers how embeddings and reranking backends work in LocalAI, how to add support for new embedding models, and how the API endpoints are structured.

## Overview

LocalAI supports generating text embeddings and reranking documents via dedicated backends. Embeddings are vector representations of text used for semantic search, clustering, and retrieval-augmented generation (RAG). Reranking scores a list of documents against a query to reorder them by relevance.

## Embeddings API

LocalAI exposes an OpenAI-compatible embeddings endpoint:

```
POST /v1/embeddings
```

### Request

```json
{
  "model": "text-embedding-ada-002",
  "input": "The quick brown fox jumps over the lazy dog"
}
```

`input` can be a string or an array of strings.

### Response

```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "embedding": [0.0023064255, -0.009327292, ...],
      "index": 0
    }
  ],
  "model": "text-embedding-ada-002",
  "usage": {
    "prompt_tokens": 8,
    "total_tokens": 8
  }
}
```

## Reranking API

LocalAI exposes a reranking endpoint compatible with Cohere's rerank API:

```
POST /v1/rerank
```

### Request

```json
{
  "model": "rerank-english-v2.0",
  "query": "What is the capital of France?",
  "documents": [
    "Paris is the capital of France.",
    "Berlin is the capital of Germany.",
    "Madrid is the capital of Spain."
  ],
  "top_n": 2
}
```

### Response

```json
{
  "model": "rerank-english-v2.0",
  "results": [
    {
      "index": 0,
      "document": { "text": "Paris is the capital of France." },
      "relevance_score": 0.9876
    },
    {
      "index": 2,
      "document": { "text": "Madrid is the capital of Spain." },
      "relevance_score": 0.1234
    }
  ],
  "usage": {
    "total_tokens": 42
  }
}
```

## Model Configuration for Embeddings

To configure a model for embeddings, set the `embeddings` flag in the model YAML:

```yaml
name: my-embedding-model
backend: llama-cpp          # or bert-embeddings, transformers, etc.
parameters:
  model: models/my-embedding-model.gguf
embeddings: true
```

For reranking models:

```yaml
name: my-reranker
backend: rerankers
parameters:
  model: models/my-reranker
type: reranker
```

## Supported Backends for Embeddings

| Backend | Notes |
|---|---|
| `llama-cpp` | Supports embedding extraction from GGUF models |
| `bert-embeddings` | Dedicated BERT-based embedding backend |
| `transformers` | HuggingFace transformer models via Python gRPC backend |
| `sentencetransformers` | Optimized for sentence-level embeddings |

## Adding a New Embedding Backend

1. Implement the `EmbeddingBackend` interface in `pkg/backend/`:

```go
type EmbeddingBackend interface {
    Embeddings(ctx context.Context, text string) ([]float32, error)
}
```

2. Register the backend in `pkg/model/initializers.go` with the appropriate backend name.

3. Add a model config YAML to the gallery or `models/` directory with `embeddings: true`.

4. Test using:

```bash
curl http://localhost:8080/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"model": "my-embedding-model", "input": "hello world"}'
```

## Batch Embeddings

When `input` is an array, LocalAI processes each string and returns a list of embedding objects. Backends that support native batching (e.g., `sentencetransformers`) will process the batch in a single forward pass for efficiency.

## Normalization

Some backends return raw embeddings; others return L2-normalized vectors. Check the model card for the specific model. You can enable normalization in the model config:

```yaml
embedding_normalize: true
```

## Troubleshooting

- **Empty or zero embeddings**: Ensure `embeddings: true` is set in the model config.
- **Dimension mismatch**: Different models produce embeddings of different sizes (e.g., 384, 768, 1536). Ensure your downstream application matches the expected dimension.
- **Slow batch processing**: Consider using a dedicated embedding backend rather than a general-purpose LLM backend for large-scale embedding workloads.
- **CUDA out of memory**: Reduce batch size or use a quantized model variant.

# Model Gallery API

This document describes how to interact with the LocalAI model gallery API endpoints for browsing, installing, and managing gallery models.

## Overview

The gallery system allows users to discover and install pre-configured models from curated galleries. Each gallery entry includes model metadata, download URLs, and configuration templates.

## Gallery Endpoints

### List Available Models

```
GET /models/available
```

Returns a list of all models available across all configured galleries.

**Query Parameters:**
- `tag` (optional): Filter models by tag (e.g., `llm`, `image`, `audio`)
- `name` (optional): Search models by name substring

**Response:**
```json
[
  {
    "name": "mistral-7b-instruct",
    "description": "Mistral 7B Instruct model",
    "tags": ["llm", "instruct"],
    "license": "Apache-2.0",
    "urls": ["https://huggingface.co/..."],
    "installed": false
  }
]
```

### Install a Model

```
POST /models/apply
```

**Request Body:**
```json
{
  "id": "TheBloke/Mistral-7B-Instruct-v0.2-GGUF/mistral-7b-instruct-v0.2.Q4_K_M.gguf",
  "name": "mistral-7b-instruct",
  "overrides": {
    "parameters": {
      "model": "mistral-7b-instruct-v0.2.Q4_K_M.gguf"
    }
  }
}
```

**Response:**
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing"
}
```

The returned `uuid` can be used to poll job status.

### Check Installation Job Status

```
GET /models/jobs/{uuid}
```

**Response:**
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "status": "downloading",
  "progress": 45,
  "error": ""
}
```

Status values: `waiting`, `downloading`, `processing`, `done`, `error`

### Delete a Model

```
POST /models/delete
```

**Request Body:**
```json
{
  "name": "mistral-7b-instruct"
}
```

## Gallery Configuration

Galleries are configured in the LocalAI startup configuration:

```yaml
galleries:
  - name: localai
    url: https://raw.githubusercontent.com/mudler/LocalAI/master/gallery/index.yaml
  - name: community
    url: https://raw.githubusercontent.com/your-org/gallery/main/index.yaml
```

Or via environment variable:
```bash
GALLERIES='[{"name":"localai","url":"https://raw.githubusercontent.com/mudler/LocalAI/master/gallery/index.yaml"}]'
```

## Gallery Index Format

A gallery index YAML file lists available model definitions:

```yaml
- name: mistral-7b-instruct
  urls:
    - https://raw.githubusercontent.com/mudler/LocalAI/master/gallery/mistral-7b-instruct.yaml
  description: "Mistral 7B Instruct"
  tags:
    - llm
    - instruct
  license: Apache-2.0
```

## Model Definition Format

Each model definition YAML specifies files to download and configuration:

```yaml
name: mistral-7b-instruct
description: Mistral 7B Instruct v0.2
license: Apache-2.0
urls:
  - https://huggingface.co/TheBloke/Mistral-7B-Instruct-v0.2-GGUF
files:
  - filename: mistral-7b-instruct-v0.2.Q4_K_M.gguf
    sha256: "abc123..."
    uri: https://huggingface.co/TheBloke/Mistral-7B-Instruct-v0.2-GGUF/resolve/main/mistral-7b-instruct-v0.2.Q4_K_M.gguf
config_file: |
  name: mistral-7b-instruct
  parameters:
    model: mistral-7b-instruct-v0.2.Q4_K_M.gguf
    temperature: 0.7
    top_p: 0.95
  template:
    chat: mistral-instruct
  context_size: 4096
  f16: true
  gpu_layers: 35
```

## Handler Registration

Gallery routes are registered in the API layer. See `.agents/api-endpoints-and-auth.md` for the general pattern. Gallery-specific handlers live in `pkg/api/gallery.go` and are wired via `RegisterGalleryRoutes`.

## Related Files

- `pkg/gallery/` — core gallery logic (download, apply, index parsing)
- `pkg/model/` — model loader and configuration
- `.agents/adding-gallery-models.md` — how to contribute new gallery models
- `.agents/model-configuration.md` — full model config reference

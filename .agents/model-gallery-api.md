# Model Gallery API

This document describes how to interact with the LocalAI model gallery API endpoints for browsing, installing, and managing gallery models.

## Overview

The gallery system allows users to discover and install pre-configured models from a central registry. Models are defined as YAML manifests that describe download URLs, configuration, and metadata.

## Endpoints

### List Available Models

```
GET /models/available
```

Returns all models from configured gallery sources.

**Query Parameters:**
- `tag` - filter by tag (e.g. `llm`, `image`, `audio`)
- `name` - search by name substring

**Response:**
```json
[
  {
    "name": "mistral-7b-openorca",
    "description": "Mistral 7B fine-tuned on OpenOrca dataset",
    "tags": ["llm", "chat"],
    "license": "apache-2.0",
    "urls": ["https://..."],
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
  "id": "TheBloke/Mistral-7B-OpenOrca-GGUF/mistral-7b-openorca.Q4_K_M.gguf",
  "name": "mistral-openorca"
}
```

The `id` field follows the format `gallery_name@model_name` or a direct gallery entry ID.

**Response:**
```json
{
  "uuid": "abc-123-def",
  "status": "processing"
}
```

### Check Installation Status

```
GET /models/jobs/:uuid
```

**Response:**
```json
{
  "uuid": "abc-123-def",
  "status": "downloading",
  "progress": 42,
  "file_name": "mistral-7b-openorca.Q4_K_M.gguf",
  "downloaded_size": "1.2GB",
  "total_size": "4.1GB",
  "error": null
}
```

Status values: `waiting`, `downloading`, `processing`, `done`, `error`

### Delete a Model

```
POST /models/delete/:name
```

Removes the model config and optionally the model weights.

## Gallery Sources

Gallery sources are configured in the startup flags or environment:

```bash
--galleries '[{"name":"localai","url":"github:mudler/LocalAI/gallery/index.yaml"}]'
```

Or via environment variable:
```bash
GALLERIES='[{"name":"localai","url":"github:mudler/LocalAI/gallery/index.yaml"}]'
```

Multiple galleries can be configured. Models from all sources are merged in the `/models/available` response.

## Gallery Model YAML Format

Each gallery entry references a model YAML that looks like:

```yaml
name: mistral-7b-openorca
description: Mistral 7B OpenOrca
license: apache-2.0
urls:
  - https://huggingface.co/TheBloke/Mistral-7B-OpenOrca-GGUF/resolve/main/mistral-7b-openorca.Q4_K_M.gguf
files:
  - filename: mistral-7b-openorca.Q4_K_M.gguf
    sha256: abc123...
    uri: https://huggingface.co/TheBloke/...
config_file: |
  name: mistral-openorca
  parameters:
    model: mistral-7b-openorca.Q4_K_M.gguf
  template:
    chat: mistral-chat
  context_size: 4096
tags:
  - llm
  - chat
  - mistral
```

## Authentication

Gallery endpoints follow the same auth rules as other API endpoints. See `api-endpoints-and-auth.md` for details on setting up API key authentication.

## Adding Custom Gallery Sources

See `.agents/adding-gallery-models.md` for a full walkthrough on creating and publishing your own gallery index.

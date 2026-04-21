# Model Configuration Guide

This guide covers how to configure models in LocalAI, including YAML configuration files, model parameters, and backend-specific settings.

## Overview

LocalAI uses YAML configuration files to define how models are loaded and used. Each model can have its own configuration file that specifies the backend, parameters, and other settings.

## Configuration File Structure

Model configuration files are placed in the models directory (default: `models/`) and follow this naming convention:
- `<model-name>.yaml` — configuration file
- `<model-name>.bin` or `<model-name>.gguf` — model weights

### Basic Configuration

```yaml
# models/my-model.yaml
name: my-model
backend: llama-cpp
parameters:
  model: my-model.gguf
  context_size: 4096
  threads: 4
  f16: true
```

## Configuration Fields

### Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Model name used in API requests |
| `backend` | string | Backend to use (e.g., `llama-cpp`, `whisper`, `stablediffusion`) |
| `parameters` | object | Backend-specific parameters |
| `context_size` | int | Maximum context window size |
| `f16` | bool | Use 16-bit floating point |
| `threads` | int | Number of CPU threads |
| `gpu_layers` | int | Number of layers to offload to GPU |

### Parameters Block

The `parameters` block contains model-specific settings:

```yaml
parameters:
  model: path/to/model.gguf   # relative to models directory
  temperature: 0.7
  top_k: 40
  top_p: 0.95
  max_tokens: 512
  repeat_penalty: 1.1
```

### Template Configuration

Define prompt templates to format inputs for chat and completion endpoints:

```yaml
template:
  chat: |
    {{.Input}}
  completion: |
    {{.Input}}
  chat_message: |
    {{if eq .RoleName "system"}}### System:\n{{.Content}}\n{{end}}
    {{if eq .RoleName "user"}}### Human:\n{{.Content}}\n{{end}}
    {{if eq .RoleName "assistant"}}### Assistant:\n{{.Content}}\n{{end}}
```

### Function Calling

Enable function/tool calling support:

```yaml
function:
  disable_no_action: true
  grammar_role_name: "assistant"
  no_action_function_name: "none"
  response_regex:
    - '(?P<function>[\w.]+)\((?P<arguments>.*)\)'
```

## Backend-Specific Examples

### llama-cpp (LLM)

```yaml
name: llama3
backend: llama-cpp
parameters:
  model: Meta-Llama-3-8B-Instruct.Q4_K_M.gguf
  context_size: 8192
  gpu_layers: 35
  f16: true
  threads: 8
mmap: true
mmlock: false
```

### Whisper (Speech-to-Text)

```yaml
name: whisper-1
backend: whisper
parameters:
  model: whisper-base.en.bin
  language: en
```

### Stable Diffusion (Image Generation)

```yaml
name: stablediffusion
backend: stablediffusion
parameters:
  model: sd-v1-5.ggml.bin
step: 25
```

## Environment Variables

The following environment variables affect model configuration loading:

| Variable | Default | Description |
|----------|---------|-------------|
| `MODELS_PATH` | `./models` | Directory containing model files and configs |
| `THREADS` | CPU count | Default number of threads for all models |
| `CONTEXT_SIZE` | `512` | Default context size if not specified in config |
| `F16` | `false` | Enable f16 globally for all models |
| `DEBUG` | `false` | Enable verbose model loading output |

## Validation

When LocalAI starts, it validates all configuration files in the models directory. Common errors:

- **Missing model file**: The `.gguf`/`.bin` file referenced in `parameters.model` does not exist
- **Unknown backend**: The specified backend is not compiled into the binary
- **Invalid template**: Go template syntax errors in the `template` block

Run with `DEBUG=true` to see detailed loading information:

```bash
DEBUG=true ./local-ai --models-path ./models
```

## Multiple Configurations for One Model

You can create multiple configuration files pointing to the same model file with different parameters:

```yaml
# models/llama3-creative.yaml
name: llama3-creative
backend: llama-cpp
parameters:
  model: Meta-Llama-3-8B-Instruct.Q4_K_M.gguf
  temperature: 1.2
  top_p: 0.98

# models/llama3-precise.yaml  
name: llama3-precise
backend: llama-cpp
parameters:
  model: Meta-Llama-3-8B-Instruct.Q4_K_M.gguf
  temperature: 0.1
  top_p: 0.5
```

## See Also

- [Adding Backends](.agents/adding-backends.md)
- [Adding Gallery Models](.agents/adding-gallery-models.md)
- [API Endpoints and Auth](.agents/api-endpoints-and-auth.md)

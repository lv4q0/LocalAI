# Multimodal and Vision Support in LocalAI

This guide covers how to work with multimodal models (vision, audio, image generation) in LocalAI, including backend configuration, API usage, and adding new multimodal capabilities.

## Overview

LocalAI supports several multimodal modalities:
- **Vision**: Image understanding via LLaVA, BakLLaVA, MobileVLM, and similar models
- **Image Generation**: Stable Diffusion, DALL-E compatible endpoints
- **Audio Transcription**: Whisper-based speech-to-text
- **Text-to-Speech**: Various TTS backends

## Vision Models

### How Vision Works

Vision models in LocalAI use the OpenAI-compatible `/v1/chat/completions` endpoint with image content parts:

```json
{
  "model": "llava",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "image_url",
          "image_url": { "url": "data:image/jpeg;base64,<base64data>" }
        },
        {
          "type": "text",
          "text": "What is in this image?"
        }
      ]
    }
  ]
}
```

You can also pass a URL directly:
```json
"image_url": { "url": "https://example.com/image.jpg" }
```

### Model Configuration for Vision

In your model YAML config, enable vision support:

```yaml
name: llava-1.6
backend: llama-cpp
parameters:
  model: llava-v1.6-mistral-7b.Q4_K_M.gguf

# Vision-specific settings
mmproj: mmproj-model-f16.gguf   # multimodal projector file

# Optional: set context size large enough for image tokens
context_size: 4096
```

The `mmproj` field points to the multimodal projector weights, which must be downloaded alongside the main model.

### Supported Vision Backends

| Backend | Models | Notes |
|---------|--------|-------|
| `llama-cpp` | LLaVA, BakLLaVA, MobileVLM, Obsidian | Requires mmproj file |
| `ollama` | Any Ollama vision model | Managed externally |
| `openai` | GPT-4V (via proxy) | Requires API key |

## Image Generation

### Stable Diffusion Backend

LocalAI supports image generation via the `stablediffusion` backend (using stable-diffusion.cpp):

```yaml
name: stablediffusion
backend: stablediffusion
parameters:
  model: v1-5-pruned-emaonly.safetensors

# Image generation defaults
image_generation:
  steps: 20
  cfg_scale: 7.0
  width: 512
  height: 512
  sampler: euler_a
```

### Image Generation API

Use the OpenAI-compatible `/v1/images/generations` endpoint:

```bash
curl http://localhost:8080/v1/images/generations \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "stablediffusion",
    "prompt": "a photo of a cat sitting on a desk",
    "n": 1,
    "size": "512x512",
    "response_format": "b64_json"
  }'
```

The response follows the OpenAI format with `b64_json` or `url` response formats.

### Diffusers Backend

For HuggingFace Diffusers models:

```yaml
name: sdxl
backend: diffusers
parameters:
  model: stabilityai/stable-diffusion-xl-base-1.0

diffusers:
  pipeline_type: StableDiffusionXLPipeline
  cuda: true
  scheduler_type: euler_a
```

## Audio: Whisper (Speech-to-Text)

### Configuration

```yaml
name: whisper-1
backend: whisper
parameters:
  model: ggml-large-v3.bin
```

### Transcription API

```bash
curl http://localhost:8080/v1/audio/transcriptions \
  -F model=whisper-1 \
  -F file=@audio.mp3
```

Translation (to English) is also supported:
```bash
curl http://localhost:8080/v1/audio/translations \
  -F model=whisper-1 \
  -F file=@audio_french.mp3
```

## Text-to-Speech

### Piper TTS Backend

```yaml
name: tts-1
backend: piper
parameters:
  model: en_US-lessac-medium.onnx
```

### TTS API

```bash
curl http://localhost:8080/v1/audio/speech \
  -H 'Content-Type: application/json' \
  -d '{"model": "tts-1", "input": "Hello world", "voice": "alloy"}' \
  --output speech.mp3
```

## Adding a New Multimodal Backend

1. Implement the relevant interface methods in your backend:
   - For vision: handle image content parts in the `Predict`/`PredictStream` methods
   - For image gen: implement `GeneratesImage() bool` returning `true` and the `GenerateImage` method
   - For audio: implement `AudioTranscription` or `TTS` methods

2. Register image preprocessing in the request pipeline to download/decode images before passing to the backend.

3. Update `pkg/model/initializers.go` to handle your backend's multimodal configuration fields.

## Troubleshooting Vision Models

- **Images not recognized**: Ensure `mmproj` path is correct and the file exists
- **Out of memory**: Reduce `context_size` or use a smaller quantization
- **Slow inference**: Vision models are slower due to image encoding; this is expected
- **URL images failing**: LocalAI downloads URL images to a temp file; check network access and the `external_grpc_backends` logs

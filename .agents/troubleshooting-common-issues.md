# Troubleshooting Common Issues in LocalAI

This guide covers the most frequent problems encountered when running, developing, or extending LocalAI, along with their solutions.

---

## 1. Model Fails to Load

### Symptoms
- API returns `500 Internal Server Error` on inference requests
- Logs show `failed to load model` or `model not found`

### Causes & Fixes

**Wrong model path or filename**
```
# Check your models directory
ls -la /path/to/models/
# Ensure the model file matches the name in your YAML config
```

**Missing backend binary**
Some backends (e.g., llama.cpp) require compiled binaries. Run:
```bash
make build
# or for a specific backend:
make llama-cpp
```

**Insufficient memory**
Reduce context size or use a quantized model (e.g., Q4_K_M instead of F16).
```yaml
# In your model YAML config:
parameters:
  model: my-model.gguf
context_size: 512
```

See also: [Model Configuration](.agents/model-configuration.md), [Debugging Backends](.agents/debugging-backends.md)

---

## 2. API Returns 401 Unauthorized

### Symptoms
- All API requests return `{"error": "Unauthorized"}`

### Fix
If `API_KEY` is set in your environment or `.env` file, you must pass it in requests:
```bash
curl http://localhost:8080/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY"
```
To disable auth for local development, unset `API_KEY`:
```bash
unset API_KEY
```

See also: [API Endpoints and Auth](.agents/api-endpoints-and-auth.md)

---

## 3. Backend Process Crashes Immediately

### Symptoms
- Logs show `exit status 1` or `signal: killed` shortly after startup
- Model loads but inference always fails

### Causes & Fixes

**Binary built for wrong architecture**
```bash
file ./local-ai  # Check architecture matches your system
```

**Missing shared libraries (Linux)**
```bash
ldd ./backends/llama-cpp/llama-cpp-avx2 | grep "not found"
# Install missing libraries, e.g.:
sudo apt-get install libgomp1
```

**GPU out of memory**
Set `gpu_layers: 0` in your model YAML to run fully on CPU, then increase gradually.

---

## 4. Slow Inference / High Latency

### Causes & Fixes

**No GPU offloading**
```yaml
# model.yaml
backend: llama-cpp
gpu_layers: 35  # Increase to offload more layers to GPU
```

**Single-threaded CPU inference**
```yaml
threads: 8  # Match your CPU core count
```

**Model too large for available RAM/VRAM**
Switch to a smaller quantization. Q4_K_M is a good balance of speed and quality.

---

## 5. Gallery Model Download Fails

### Symptoms
- `POST /models/apply` returns an error or hangs
- Model never appears after install

### Fixes

**Check network access from the container/host**
```bash
curl -I https://huggingface.co
```

**Manually download and place the model**
```bash
wget https://example.com/model.gguf -O /models/my-model.gguf
```
Then create a YAML config manually. See [Model Configuration](.agents/model-configuration.md).

**Verify gallery YAML is valid**
```bash
python3 -c "import yaml; yaml.safe_load(open('gallery-entry.yaml'))"
```

See also: [Adding Gallery Models](.agents/adding-gallery-models.md)

---

## 6. New Backend Not Recognized

### Symptoms
- Setting `backend: my-backend` in a model YAML results in `unknown backend`

### Fix
Ensure your backend is registered. Check [Adding Backends](.agents/adding-backends.md) for the registration steps. Confirm the backend binary is present and executable:
```bash
ls -la ./backends/my-backend/
chmod +x ./backends/my-backend/my-backend
```

---

## 7. Logs & Debugging Tips

```bash
# Run with verbose/debug logging
DEBUG=true ./local-ai --log-level debug

# Follow logs in Docker
docker logs -f localai-container

# Check which models are loaded
curl http://localhost:8080/v1/models | jq
```

For backend-specific debugging, see [Debugging Backends](.agents/debugging-backends.md).

---

## 8. Getting Help

- Open an issue: https://github.com/mudler/LocalAI/issues
- Community Discord: linked in the main README
- Check existing issues before opening a new one — your problem may already be solved.

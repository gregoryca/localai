# LocalAI VS Code Chat Setup (8GB VRAM / 16GB System RAM)

> Optimized local AI inference for **VS Code Chat / GitHub Copilot Custom Endpoint** on **Linux Mint** with an **8GB NVIDIA GPU**.

---

## 📋 Quick Summary

| Component | Value |
|-----------|-------|
| **GPU** | NVIDIA RTX 3060 / 4060 / similar (8GB VRAM) |
| **System RAM** | 16GB |
| **OS** | Linux Mint (Docker) |
| **Docker Image** | `localai/localai:latest-gpu-nvidia-cuda-12` |
| **API Endpoint** | `http://localhost:8080/v1` |
| **Max Context** | 8,192 tokens (hard-capped for VRAM safety) |

---

## 🚀 Quick Start

```bash
# 1. Start the LocalAI container
docker compose up -d

# 2. Wait for the health check (~1-2 min for first model load)
#    Then verify:
curl http://localhost:8080/readyz

# 3. Place your GGUF model in ./models/
#    e.g., qwen2.5-coder-7b-instruct-q4_k_m.gguf

# 4. Create a model definition file:
#    ./models/qwen-2.5-coder-7b.yaml
#    (see "Model Definition Files" section below)

# 5. Connect VS Code Chat:
#    - Open VS Code Chat (Ctrl+Shift+G)
#    - Model Picker → "Add Custom Endpoint"
#    - Endpoint: http://localhost:8080/v1
#    - API Key: any-string-here
#    - Model Name: qwen-2.5-coder-7b
```

---

## 📦 Docker Compose

```yaml
version: '3.8'

services:
  api:
    container_name: localai
    image: localai/localai:latest-gpu-nvidia-cuda-12
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/readyz"]
      interval: 1m
      timeout: 20m
      retries: 5
    shm_size: 2gb
    volumes:
      - ./models:/models
    env_file:
      - .env
    network_mode: host
    restart: always
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              capabilities: [gpu]
              count: 1
```

**Key notes:**
- `network_mode: host` — avoids port mapping issues on Linux Mint with NVIDIA drivers.
- `shm_size: 2gb` — required for long-context inference.
- `count: 1` — reserves one full GPU for the container.

---

## 🧠 Model Definition Files

Place these YAML files in `./models/` alongside your GGUF model files.

### `qwen-2.5-coder-7b.yaml`

```yaml
name: qwen-2.5-coder-7b
backend: llama-cpp
parameters:
  model: qwen2.5-coder-7b-instruct-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 24
```

### `qwen3.8-9b-q4.yaml`

```yaml
name: qwen3.8-9b-q4
backend: llama-cpp
parameters:
  model: qwen3.8-9b-q4.gguf
  context_size: 8192
  gpu_layers: 24
```

### `deepseek-r1-distill-qwen-7b.yaml`

```yaml
name: deepseek-r1-distill-qwen-7b
backend: llama-cpp
parameters:
  model: deepseek-r1-distill-qwen-7b-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 24
```

### `gemma-4-12b-it-qat-mtp.yaml`

```yaml
name: gemma-4-12b-it-qat-mtp
backend: llama-cpp
parameters:
  model: gemma-4-12b-it-qat-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 20
```

### `gemma-4-e4b-it.yaml`

```yaml
name: gemma-4-e4b-it
backend: llama-cpp
parameters:
  model: gemma-4-e4b-it-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 32
```

### `granite-4.2-8b-q4.yaml`

```yaml
name: granite-4.2-8b-q4
backend: llama-cpp
parameters:
  model: granite-4.2-8b-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 28
```

---

## 📊 Model Memory Breakdown (8GB VRAM)

| Model | Quant | Model Size | GPU Layers | Buffer |
|-------|-------|------------|-------------|--------|
| Gemma 4 E4B-IT | Q4_K_M | ~3.0 GB | 32/32 | ~5 GB |
| Mistral Small 3 | Q4_K_M | ~4.2 GB | 28/28 | ~3.8 GB |
| Qwen 2.5 Coder 7B | Q4_K_M | ~4.7 GB | 24/28 | ~3 GB |
| Granite 4.2 8B | Q4_K_M | ~5.0 GB | 28/28 | ~3 GB |
| Qwen 3.8 9B | Q4_K_M | ~5.6 GB | 24/32 | ~2 GB |
| DeepSeek R1 Qwen 7B | Q4_K_M | ~4.7 GB | 24/28 | ~3.3 GB |
| Gemma 4 12B QAT-MTP | Q4_K_M | ~7.0 GB | 20/28 | ~1 GB |

> **Buffer** = 8GB − (model + KV cache + system UI). KV cache grows with context length.

---

## ⚙️ VS Code Chat Configuration

1. Open **VS Code Chat** (`Ctrl + Shift + G`).
2. Click the **Model Picker** (bottom of chat UI).
3. Choose **"Manage Models"** → **"Add Custom Endpoint"**.
4. Fill in:
   - **Endpoint:** `http://localhost:8080/v1`
   - **API Key:** `any-string-here`
   - **Model Name:** `qwen-2.5-coder-7b` (or any defined model)
5. Click **"Save"** and select the model as active.

---

## ⚠️ Critical Performance Rules

1. **Context Cap:** Never exceed `context_size: 8192`. Larger contexts spill into system RAM, killing generation speed.
2. **GPU Layers:** Keep the model fully on GPU. If Cinnamon or other processes consume VRAM, reduce `gpu_layers` incrementally.
3. **Session Hygiene:** Click **"New Session"** (trash can icon) frequently to flush the KV cache and restore speed.
4. **Threads:** `THREADS=1` is optimal for generation speed on 4–6 core CPUs.

---

## 🔧 Troubleshooting

| Symptom | Fix |
|----------|------|
| `CUDA out of memory` | Reduce `gpu_layers` by 4–8 and restart the container. |
| Slow generation (< 5 t/s) | Check `nvidia-smi` — if VRAM is > 7.5GB, reduce `gpu_layers`. |
| Container won't start | Verify `nvidia-container-toolkit` is installed: `nvidia-ctk --version`. |
| Port 8080 blocked | Ensure no other service is using port 8080. |
| Model not found | Confirm the GGUF file exists in `./models/` and the YAML `model` path matches. |

---

## 📁 Project Structure

```
localai/
├── docker-compose.yml
├── .env
├── models/
│   ├── qwen2.5-coder-7b-instruct-q4_k_m.gguf
│   ├── qwen-2.5-coder-7b.yaml
│   ├── qwen3.8-9b-q4.gguf
│   ├── qwen3.8-9b-q4.yaml
│   ├── deepseek-r1-distill-qwen-7b-q4_k_m.gguf
│   ├── deepseek-r1-distill-qwen-7b.yaml
│   ├── gemma-4-12b-it-qat-q4_k_m.gguf
│   ├── gemma-4-12b-it-qat-mtp.yaml
│   ├── gemma-4-e4b-it-q4_k_m.gguf
│   ├── gemma-4-e4b-it.yaml
│   ├── granite-4.2-8b-q4_k_m.gguf
│   └── granite-4.2-8b-q4.yaml
└── documentation/
    └── local_ai_vscode_setup_guide.md
```

---

## 📄 License

This setup is for personal use. Model files are subject to their respective licenses.

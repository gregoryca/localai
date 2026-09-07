# VS Code Local AI Chat Guide (8GB VRAM / 16GB System RAM)

This guide summarizes the optimized local AI setup for **VS Code Chat / GitHub Copilot Custom Endpoint** on **Linux Mint (Docker)** using an **8GB VRAM GPU**.

---

## 🚀 Recommended Models

1. **Qwen 2.5 Coder 7B (Instruct)** – *The Gold Standard*
   - **Quantization:** `Q4_K_M` GGUF (~4.7GB to 5.0GB).
   - **Why:** Excellent code generation, fits comfortably inside 8GB VRAM while leaving a 3GB buffer for system UI and an 8k context window. It also has strong tool-calling features required by GitHub Copilot.
2. **DeepSeek R1 Distill Qwen 7B** – *The Logic & Debug Specialist*
   - **Why:** Chain-of-thought reasoner ideal for complex architectural tracking and deep bug hunting, though slower for boilerplate.
3. **Mistral Small 3 (7B)** – *The Latency Champion*
   - **Why:** Extremely fast token-per-second generation on mid-range cards.

---

## 📦 Docker & LocalAI Stack Configuration

To pass your NVIDIA GPU cleanly into the Docker container on Linux Mint, ensure `nvidia-container-toolkit` is installed on your host system.

### 1. `docker-compose.yml`

```yaml
version: '3.8'

services:
  localai:
    image: localai/localai:latest-gpu-nvidia
    container_name: localai-copilot-backend
    restart: always
    environment:
      - MODELS_PATH=/models
      - THREADS=4               # Optimal for standard 4-6 core consumer CPUs
      - CONTEXT_SIZE=8192       # Hard cap to preserve 8GB VRAM context room
    ports:
      - "8080:8080"
    volumes:
      - ./models:/models
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### 2. Model Definition File (`/models/qwen-7b.yaml`)

Save this file into your local `./models` directory mapped into the container:

```yaml
name: qwen-2.5-coder-7b
backend: llama-cpp
parameters:
  model: qwen2.5-coder-7b-instruct-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 28                              # Offloads all layers entirely to VRAM
```

---

## ⚙️ Connecting to GitHub Copilot Chat Custom Endpoint

1. Open **VS Code Chat** (`Ctrl + Shift + G` or via sidebar).
2. Click the **Model Picker drop-down menu** at the bottom of the chat interface.
3. Select **"Manage Models"** or **"Add Custom Endpoint"**.
4. Configure an **OpenAI-Compatible** provider configuration:
   - **Endpoint URL:** `http://localhost:8080/v1`
   - **API Key:** `any-string-here` *(LocalAI skips verification, but Copilot requires a non-empty string)*
   - **Model Name:** `qwen-2.5-coder-7b`
5. Pin or set the model as your active assistant.

---

## ⚠️ Critical 8GB VRAM + 16GB RAM Performance Rules

- **Enforce Context Caps:** Always hard-cap context size to **8,192 tokens** maximum (`num_ctx` / `CONTEXT_SIZE`). Pushing to native 32k limits will cause context memory to overflow into your 16GB system RAM, stalling generation down to single-digit tokens per second.
- **Keep `gpu_layers` High:** Ensure the entire model fits on the GPU card. If Linux Mint UI (e.g., Cinnamon) uses too much background VRAM, step down the `gpu_layers` incrementally to prevent CUDA Out Of Memory (OOM) loops.
- **Wipe Session History Frequently:** Copilot heavily pads queries with workspace state and chat history. Frequently click the **"New Session" / Trash Can** icon in the VS Code chat interface to flush the VRAM KV-cache and recover speed.

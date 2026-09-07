# VS Code Local AI Chat Guide (8GB VRAM / 16GB System RAM)

This guide summarizes the optimized local AI setup for **VS Code Chat / GitHub Copilot Custom Endpoint** on **Linux Mint (Docker)** using an **8GB VRAM GPU**.

---

## 🚀 Recommended Models

1. **Qwen 2.5 Coder 7B (Instruct)** – *The Gold Standard*
   - **Quantization:** `Q4_K_M` GGUF (~4.7GB to 5.0GB).
   - **Why:** Excellent code generation, fits inside 8GB VRAM with a 3GB buffer for system UI and an 8k context window. It also has strong tool-calling features required by GitHub Copilot.
   - **Note:** On 8GB VRAM, the margin is tight. Consider `Q3_K_M` (~3.5GB) if you need extra headroom, or `Q8_0` (~6.2GB) if you want higher quality and have headroom for KV cache.
2. **DeepSeek R1 Distill Qwen 7B** – *The Logic & Debug Specialist*
   - **Why:** Chain-of-thought reasoner ideal for complex architectural tracking and deep bug hunting, though slower for boilerplate.
3. **Mistral Small 3 (7B)** – *The Latency Champion*
   - **Why:** Extremely fast token-per-second generation on mid-range cards.
   - **Quantization:** `Q4_K_M` GGUF (~4.2GB).
4. **Qwen 3.8 9B (Instruct)** – *The Heavyweight Performer*
   - **Quantization:** `Q4_K_M` GGUF (~5.6GB to 5.9GB).
   - **Why:** Strong reasoning and code capabilities, but on 8GB VRAM the margin is very tight (~2GB buffer). Consider `Q3_K_M` (~4.4GB) for safety.
5. **DeepSeek R1 Distill Qwen 7B** – *The Logic & Debug Specialist*
   - **Quantization:** `Q4_K_M` GGUF (~4.7GB).
   - **Why:** Chain-of-thought reasoner ideal for complex architectural tracking and deep bug hunting, though slower for boilerplate.
6. **Gemma 4 12B-IT QAT-MTP** – *The Reasoning Powerhouse*
   - **Quantization:** `Q4_K_M` GGUF (~7.0GB to 7.3GB).
   - **Why:** Massive reasoning capabilities, but on 8GB VRAM you have only ~1GB buffer — very tight. Use only if you need its specific MTP architecture.
7. **Gemma 4 E4B-IT** – *The 4B Reasoner*
   - **Quantization:** `Q4_K_M` GGUF (~2.9GB to 3.1GB).
   - **Why:** Surprisingly strong reasoning for its size, leaves ~5GB headroom for KV cache and system UI. Great fallback option.
8. **Granite 4.2 8B (Q4)** – *The Balanced Workhorse*
   - **Quantization:** `Q4_K_M` GGUF (~5.0GB).
   - **Why:** Solid all-rounder with good code and reasoning balance. Leaves ~3GB buffer on 8GB VRAM.

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
      - THREADS=1              # Minimal threads for inference (generation speed)
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

### 2. Model Definition Files

Save these files into your local `./models` directory mapped into the container.

#### `./models/qwen-2.5-coder-7b.yaml`

```yaml
name: qwen-2.5-coder-7b
backend: llama-cpp
parameters:
  model: qwen2.5-coder-7b-instruct-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 24                          # Leave 4 layers on CPU to stay within 8GB
                                          # Q4_K_M (~4.7GB) + KV cache (~1.5GB) + system UI buffer
```

#### `./models/qwen3.8-9b-q4.yaml`

```yaml
name: qwen3.8-9b-q4
backend: llama-cpp
parameters:
  model: qwen3.8-9b-q4.gguf
  context_size: 8192
  gpu_layers: 24                          # Leave 4 layers on CPU for 8GB safety
                                          # Q4_K_M (~5.6GB) + KV cache (~1.5GB) + system UI buffer
```

#### `./models/deepseek-r1-distill-qwen-7b.yaml`

```yaml
name: deepseek-r1-distill-qwen-7b
backend: llama-cpp
parameters:
  model: deepseek-r1-distill-qwen-7b-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 24                          # Leave 4 layers on CPU for 8GB safety
                                          # Q4_K_M (~4.7GB) + KV cache (~1.5GB) + system UI buffer
```

#### `./models/gemma-4-12b-it-qat-mtp.yaml`

```yaml
name: gemma-4-12b-it-qat-mtp
backend: llama-cpp
parameters:
  model: gemma-4-12b-it-qat-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 20                         # Reduced from 28 to stay within 8GB
                                          # Q4_K_M (~7.0GB) + KV cache (~1GB) + system UI buffer
```

#### `./models/gemma-4-e4b-it.yaml`

```yaml
name: gemma-4-e4b-it
backend: llama-cpp
parameters:
  model: gemma-4-e4b-it-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 32                         # All 32 layers fit comfortably on 8GB
                                          # Q4_K_M (~3.0GB) + KV cache (~1.5GB) + system UI buffer (~3.5GB)
```

#### `./models/granite-4.2-8b-q4.yaml`

```yaml
name: granite-4.2-8b-q4
backend: llama-cpp
parameters:
  model: granite-4.2-8b-q4_k_m.gguf
  context_size: 8192
  gpu_layers: 28                         # All 28 layers fit on 8GB
                                          # Q4_K_M (~5.0GB) + KV cache (~1.5GB) + system UI buffer (~1.5GB)
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

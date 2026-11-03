# VS Code Local AI Chat Guide (8GB VRAM / 16GB System RAM)
# Last verified: 2026-11-03

This guide summarizes the optimized local AI setup for **VS Code Chat / GitHub Copilot Custom Endpoint** on **Linux Mint (Docker)** using an **8GB VRAM GPU**.

---

## ✅ Key Findings (Verified 2026-11-03)

| Model | Actual Context | Actual GPU Layers | Actual VRAM |
|-------|---------------|-------------------|-------------|
| `gemma-4-e4b-it` | 128,000 | 99 (full GPU) | ~3.0 GB |
| `deepseek-r1-distill-qwen-7b` | 32,768 | 28 (full GPU) | ~4.7 GB |
| `qwen3.8-9b-q4` | 46,384 | 28 (full GPU) | ~5.6 GB |
| `granite-4.2-8b-q4` | 32,768 | 28 (full GPU) | ~5.0 GB |
| `gemma-4-12b-it-qat-mtp` | 32,000 | 28 (full GPU) | ~7.0 GB |

**Critical insight:** The guide's advice to "hard-cap context to 8192" is **incorrect**. The actual configs use 32k–128k contexts and run fine. The real constraint is **VRAM**, not context size.

---

## 🚀 Recommended Models

1. **Gemma 4 E4B-IT** – *The 4B Reasoner*
   - **Quantization:** `Q4_K_M` GGUF (~2.9GB to 3.1GB).
   - **Why:** Surprisingly strong reasoning for its size, leaves ~5GB headroom for KV cache and system UI. Great fallback option.
   - **Context:** 128,000 tokens (native).
   - **GPU Layers:** 99 (full offload).

2. **DeepSeek R1 Distill Qwen 7B** – *The Logic & Debug Specialist*
   - **Quantization:** `Q4_K_M` GGUF (~4.7GB).
   - **Why:** Chain-of-thought reasoner ideal for complex architectural tracking and deep bug hunting, though slower for boilerplate.
   - **Context:** 32,768 tokens.
   - **GPU Layers:** 28 (full GPU).

3. **Qwen 3.8 9B (Instruct)** – *The Heavyweight Performer*
   - **Quantization:** `Q4_K_M` GGUF (~5.6GB to 5.9GB).
   - **Why:** Strong reasoning and code capabilities. On 8GB VRAM the margin is tight (~2GB buffer). Consider `Q3_K_M` (~4.4GB) for safety.
   - **Context:** 46,384 tokens.
   - **GPU Layers:** 28 (full GPU).

4. **Granite 4.2 8B (Q4)** – *The Balanced Workhorse*
   - **Quantization:** `Q4_K_M` GGUF (~5.0GB).
   - **Why:** Solid all-rounder with good code and reasoning balance. Leaves ~3GB buffer on 8GB VRAM.
   - **Context:** 32,768 tokens.
   - **GPU Layers:** 28 (full GPU).

5. **Gemma 4 12B-IT QAT-MTP** – *The Reasoning Powerhouse*
   - **Quantization:** `Q4_K_M` GGUF (~7.0GB to 7.3GB).
   - **Why:** Massive reasoning capabilities, but on 8GB VRAM you have only ~1GB buffer — very tight. Use only if you need its specific MTP architecture.
   - **Context:** 32,000 tokens.
   - **GPU Layers:** 28 (full GPU).

6. **Mistral Small 3 (7B)** – *The Latency Champion*
   - **Quantization:** `Q4_K_M` GGUF (~4.2GB).
   - **Why:** Extremely fast token-per-second generation on mid-range cards.
   - **Context:** 32,768 tokens.

7. **Qwen 2.5 Coder 7B (Instruct)** – *The Gold Standard*
   - **Quantization:** `Q4_K_M` GGUF (~4.7GB to 5.0GB).
   - **Why:** Excellent code generation, fits inside 8GB VRAM with a 3GB buffer for system UI and an 8k context window. It also has strong tool-calling features required by GitHub Copilot.
   - **Context:** 8,192 tokens (conservative).
   - **GPU Layers:** 28 (full GPU).

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
      - CONTEXT_SIZE=8192       # Optional: conservative cap for safety
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

#### `./models/gemma-4-e4b-it.yaml`

```yaml
agent:
    max_iterations: 10
backend: llama-cpp
context_size: 128000
f16_kv: true
function:
    automatic_tool_parsing_fallback: true
    grammar:
        disable: true
gpu_layers: 99
known_usecases:
    - chat
    - vision
    - completion
    - edit
low_vram: true
min_p: 0
mmap: true
mmproj: mmproj-gemma-4-E4B-it-bf16.gguf
name: gemma-4-e4b-it
options:
    - use_jinja:true
parameters:
    model: gemma-4-E4B-it-Q4_K_M.gguf
repeat_penalty: 1
temperature: 0.7
template:
    use_tokenizer_template: true
top_k: 64
top_p: 0.95
```

#### `./models/deepseek-r1-distill-qwen-7b.yaml`

```yaml
backend: llama-cpp
context_size: 32768
f16_kv: true
flash_attention: true
gpu_layers: 28  # Increased to fully offload the 28 layers of Qwen2.5-7B
known_usecases:
    - chat
low_vram: false
mmap: true
name: deepseek-r1-distill-qwen-7b
parameters:
    model: DeepSeek-R1-Distill-Qwen-7B-Q4_K_M.gguf
stopwords:
    - <｜begin▁of▁sentence｜>
    - <｜end▁of▁sentence｜>
    - <｜User｜>
    - <｜Assistant｜>
template:
    chat: |
        {{if .System}}<｜begin of sentence｜>{{.System}}{{end}}
        {{range .Messages}}
        {{if eq .Role "user"}}<｜User｜>{{.Content}}<｜Assistant｜>{{else}}<｜Assistant｜>{{.Content}}<｜end of sentence｜>{{end}}
        {{end}}
    chat_message: |
        {{if eq .RoleName "system" -}}{{.Content }}
        {{ end -}}
        {{if eq .RoleName "user" -}}<｜User｜>{{.Content}}
        {{end -}}
        {{if eq .RoleName "assistant" -}}<｜Assistant｜>{{.Content}}<｜end▁of▁sentence｜>{{end}}
    completion: |
        {{.Input}}
threads: 6
```

#### `./models/qwen3.8-9b-q4.yaml`

```yaml
backend: llama-cpp
context_size: 46384
function:
    automatic_tool_parsing_fallback: true
    grammar:
        disable: true
gpu_layers: 28
mmap: true
known_usecases:
    - chat
name: qwen3.8-9b-q4
options:
    - use_jinja:true
parameters:
    min_p: 0
    model: llama-cpp/models/qwen3.8-9b/Qwen3.8-9B-Q4_K_M.gguf
    presence_penalty: 0
    repeat_penalty: 1
    temperature: 0.6
    top_k: 20
    top_p: 0.95
template:
    use_tokenizer_template: true
```

#### `./models/granite-4.2-8b-q4.yaml`

```yaml
backend: llama-cpp
context_size: 32768
function:
    automatic_tool_parsing_fallback: true
    grammar:
        disable: true
known_usecases:
    - chat
name: granite-4.2-8b-q4
options:
    - use_jinja:true
parameters:
    model: llama-cpp/models/granite-4.2-8b/granite-4.2-8b-Q4_K_M.gguf
template:
    use_tokenizer_template: true
```

#### `./models/gemma-4-12b-it-qat-mtp.yaml`

```yaml
backend: llama-cpp
# 1. Reduced to fit safely into 8GB VRAM alongside the vision model
context_size: 32000
gpu_layers: 28
no_mmap: true
mlock: true
low_vram: false
batch_size: 512
f16: true
# 2. Disabled draft model to save heavy VRAM overhead
# draft_model: llama-cpp/models/gemma-4-12B-it-qat-GGUF/mtp-gemma-4-12B-it.gguf
function:
    automatic_tool_parsing_fallback: true
    grammar:
        disable: true
# 3. Splits layers effectively between GPU (8GB) and System RAM (16GB)
flash_attention: true
# 4. Set to false to avoid unoptimized memory allocation
parallel: 1
known_usecases:
    - chat
mmproj: llama-cpp/mmproj/gemma-4-12B-it-qat-GGUF/mmproj-BF16.gguf
name: gemma-4-12b-it-qat-mtp
options:
    - use_jinja:true
# 5. Cleaned out speculative options because draft_model is off
parameters:
    min_p: 0
    model: llama-cpp/models/gemma-4-12B-it-qat-GGUF/gemma-4-12B-it-qat-UD-Q4_K_XL.gguf
    repeat_penalty: 1
    temperature: 1
    top_k: 64
    top_p: 0.95
template:
    use_tokenizer_template: true
```

---

## ⚙️ Connecting to GitHub Copilot Chat Custom Endpoint

1. Open **VS Code Chat** (`Ctrl + Shift + G` or via sidebar).
2. Click the **Model Picker drop-down menu** at the bottom of the chat interface.
3. Select **"Manage Models"** or **"Add Custom Endpoint"**.
4. Configure an **OpenAI-Compatible** provider configuration:
   - **Endpoint URL:** `http://localhost:8080/v1`
   - **API Key:** `any-string-here` *(LocalAI skips verification, but Copilot requires a non-empty string)*
   - **Model Name:** `gemma-4-e4b-it` (or whichever model you want to use)
5. Pin or set the model as your active assistant.

---

## ⚠️ Critical 8GB VRAM + 16GB RAM Performance Rules

- **Enforce Context Caps:** Always hard-cap context size to **8,192 tokens** maximum (`num_ctx` / `CONTEXT_SIZE`) **unless** you've verified the model fits comfortably in VRAM with larger contexts. The guide's original advice was overly conservative — many models (Gemma 4 E4B, DeepSeek R1 Qwen, Granite 4.2) handle 32k–128k contexts fine on 8GB VRAM.
- **Keep `gpu_layers` High:** Ensure the entire model fits on the GPU card. If Linux Mint UI (e.g., Cinnamon) uses too much background VRAM, step down the `gpu_layers` incrementally to prevent CUDA Out Of Memory (OOM) loops.
- **Wipe Session History Frequently:** Copilot heavily pads queries with workspace state and chat history. Frequently click the **"New Session" / Trash Can** icon in the VS Code chat interface to flush the VRAM KV-cache and recover speed.
- **Monitor VRAM Usage:** Use `nvidia-smi` in a background terminal to watch VRAM usage during generation. If you see the GPU approaching 8GB, reduce `gpu_layers` or switch to a smaller quantization.
- **Consider `low_vram: true`** in LocalAI config when using smaller models — this enables CPU offloading of less-used layers dynamically.

<h1 align="center">
  <br>
  <img height="300" src="https://user-images.githubusercontent.com/2420543/233147843-88697415-6dbf-4368-a862-ab217f9f7342.jpeg"> <br>
    LocalAI
<br>
</h1>

[![tests](https://github.com/go-skynet/LocalAI/actions/workflows/test.yml/badge.svg)](https://github.com/go-skynet/LocalAI/actions/workflows/test.yml) [![build container images](https://github.com/go-skynet/LocalAI/actions/workflows/image.yml/badge.svg)](https://github.com/go-skynet/LocalAI/actions/workflows/image.yml)

[![](https://dcbadge.vercel.app/api/server/uJAeKSAGDy?style=flat-square&theme=default-inverted)](https://discord.gg/uJAeKSAGDy) 

[Documentation website](https://localai.io/)

**LocalAI** is a drop-in replacement REST API that's compatible with OpenAI API specifications for local inferencing. It allows you to run LLMs (and not only) locally or on-prem with consumer grade hardware, supporting multiple model families that are compatible with the ggml format. Does not require GPU.

For a list of the supported model families, please see [the model compatibility table](https://localai.io/model-compatibility/index.html#model-compatibility-table).

In a nutshell:

- Local, OpenAI drop-in alternative REST API. You own your data.
- NO GPU required. NO Internet access is required either
  - Optional, GPU Acceleration is available in `llama.cpp`-compatible LLMs. See also the [build section](https://localai.io/basics/build/index.html). 
- Supports multiple models:
  - 📖 Text generation with GPTs (`llama.cpp`, `gpt4all.cpp`, ... and more)
  - 🗣 Text to Audio 🎺🆕
  - 🔈 Audio to Text (Audio transcription with `whisper.cpp`)
  - 🎨 Image generation with stable diffusion
- 🏃 Once loaded the first time, it keep models loaded in memory for faster inference
- ⚡ Doesn't shell-out, but uses C++ bindings for a faster inference and better performance. 

LocalAI was created by [Ettore Di Giacinto](https://github.com/mudler/) and is a community-driven project, focused on making the AI accessible to anyone. Any contribution, feedback and PR is welcome! 

See the [Getting started](https://localai.io/basics/getting_started/index.html) and [examples](https://github.com/go-skynet/LocalAI/tree/master/examples/) sections to learn how to use LocalAI. For a list of curated models check out the [model gallery](https://localai.io/models/).


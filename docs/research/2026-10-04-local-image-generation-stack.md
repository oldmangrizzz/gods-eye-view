# Local Image Generation Stack Research — 2026-10-04

## Scope

Evaluate current open image-generation models in roughly the 0.6B–8B class for the GMRI lab's available local hardware:

- 2017 iMac / Proxmox 9 / Radeon Pro 570 4 GB / 32 GB RAM.
- 2022 MacBook Air M2 / 8 GB unified memory.
- RunPod and cloud retained as overflow, not the default path.

The goal is not to pick the newest model. It is to identify the strongest practical local-first stack under tight VRAM/RAM and operating-cost constraints.

## Findings

### 1. Z-Image-Turbo is the strongest first target for the Radeon/Proxmox path

The official Tongyi-MAI model is Apache-2.0 and uses a Qwen3 text encoder. The reference BF16 repository is very large, but stable-diffusion.cpp explicitly documents Z-Image operation on GPUs with 4 GB of VRAM or less using quantized GGUF weights and a Qwen3-4B text encoder.

Primary sources:
- https://huggingface.co/Tongyi-MAI/Z-Image-Turbo
- https://github.com/leejet/stable-diffusion.cpp/blob/master/docs/z_image.md

Why it fits:
- Quantized GGUF path.
- Vulkan backend available in stable-diffusion.cpp.
- Explicit 4 GB VRAM guidance.
- Apache-2.0 upstream model.
- Small enough text encoder family to remain inside GMRI's normal local model budget.

### 2. FLUX.2 [klein] 4B is the best second model to validate

Black Forest Labs publishes FLUX.2 [klein] 4B under Apache-2.0. It supports both text-to-image and image editing. stable-diffusion.cpp added FLUX.2-klein support in January 2026.

Primary sources:
- https://huggingface.co/black-forest-labs/FLUX.2-klein-4B
- https://github.com/leejet/stable-diffusion.cpp

Important constraint:
- The official repository is 23.7 GB total and the BF16 single-file transformer is 7.75 GB, so the stock weights are not a 4 GB VRAM fit. The practical question is quantized Vulkan behavior on the Pro 570, which must be benchmarked locally before making it the default.

### 3. SANA-Sprint 1.6B is the efficiency candidate, but the runtime path is less convenient for the Pro 570

SANA-Sprint 1.6B is Apache-2.0, generates at 1024 px, supports 1–4-step inference and ControlNet, and uses Gemma2-2B-IT as its text encoder. NVIDIA reports very strong speed/quality efficiency on modern NVIDIA hardware.

Primary sources:
- https://huggingface.co/Efficient-Large-Model/Sana_Sprint_1.6B_1024px
- https://huggingface.co/docs/diffusers/api/pipelines/sana_sprint

Why it is not first:
- It is not currently listed among stable-diffusion.cpp's supported image architectures.
- Its best-documented path is Diffusers/PyTorch, which is less attractive on an older Polaris AMD GPU under Linux.
- It remains worth testing on Apple MPS and/or CPU as an alternate low-parameter renderer.

### 4. Lumina-Image 2.0 is a credible 2B-class fallback

Lumina-Image 2.0 is a 2B, Apache-2.0 flow-based image model with Diffusers support and documented CPU offload.

Primary source:
- https://huggingface.co/Alpha-VLLM/Lumina-Image-2.0

It is interesting for low-parameter experimentation, but it lacks the same 4 GB Vulkan deployment story Z-Image currently has.

### 5. Qwen-Image 2.1 is within the transformer's nominal 7B class, but the complete runtime is heavier

Hugging Face currently lists Qwen-Image 2.1 as a 7B text-to-image model. stable-diffusion.cpp has day-0 support, but its documented pipeline uses Qwen3-VL-8B as the text encoder plus the Qwen image model and VAE.

Primary sources:
- https://huggingface.co/Qwen/Qwen-Image-2.1
- https://github.com/leejet/stable-diffusion.cpp/blob/master/docs/qwen_image_2.1.md

Conclusion:
- It stays in the research pool, but should not be the first 4 GB Radeon target.
- Quantized CPU/RAM offload may make it usable, but that must be measured rather than assumed.

### 6. Older FLUX.1 is not a good fit for the stated 2B–8B local budget

FLUX.1 Schnell is a 12B transformer. stable-diffusion.cpp can make it run on 4–6 GB VRAM with quantization, but it violates the lab's preferred local parameter ceiling and offers less reason to accept that cost now that FLUX.2-klein 4B and Z-Image exist.

Primary sources:
- https://huggingface.co/black-forest-labs/FLUX.1-schnell
- https://github.com/leejet/stable-diffusion.cpp/blob/master/docs/flux.md

## Recommended local stack

### Proxmox / Radeon Pro 570
1. stable-diffusion.cpp
2. Vulkan backend
3. Z-Image-Turbo quantized GGUF as the default model
4. FLUX.2-klein 4B quantized as the second model after validation
5. CPU/RAM offload where needed
6. Embedded web UI/API as the local access surface

### M2 MacBook Air
- Keep image generation secondary because it is an 8 GB fanless field/development machine.
- Test SANA-Sprint 1.6B through MPS and/or lightweight Apple-native runtimes.
- Do not make sustained image generation a required background workload.

### Overflow
- RunPod for high-resolution, batch, or models that are inefficient locally.
- Cloud generation only as a reserve path.

## Acceptance benchmark before choosing defaults

Run the same fixed prompt suite across candidate models and record:
- cold-start time
- first-image time
- steady-state 512 and 1024 generation time
- peak VRAM
- peak host RAM
- thermal stability over 10 consecutive generations
- text rendering quality
- photorealism
- diagram/technical illustration quality
- prompt adherence
- image-edit fidelity
- crash/recovery behavior

Do not pick the default solely from leaderboard scores. Pick the model that produces useful GMRI output reliably on the actual hardware.

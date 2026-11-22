<div align="center">

# ⚡ ComfyUI-QwenImage21-PixelHead

**Drop-in Zero-VAE Direct RGBA Pixel Readout for Qwen-Image 2.1 (`5.55 ms` · `67.1×` Faster · Zero 2K VAE OOMs)**

[**English**](README.md) | [**简体中文**](README_zh.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Hugging Face Weights](https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-pocis%2FQwen--Image--2.1--PixelHead-yellow)](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead)
[![Weight Size](https://img.shields.io/badge/Weight-48.8%20MiB%20(BF16)-blue)](#2-download-pixel-head-weights)
[![Speedup](https://img.shields.io/badge/Decode%20Speed-5.55%20ms%20(67.1%C3%97)-brightgreen)](#benchmark)
[![Output](https://img.shields.io/badge/Output-Native%204--Ch%20RGBA-purple)](#features)

</div>

---

**`ComfyUI-QwenImage21-PixelHead`** replaces the heavy `644.2 MiB` 64-channel Qwen-Image 2.1 WanVAE decoder with a lightweight **`25.4M`-parameter (`48.8 MiB` BF16)** Direct Pixel Readout Head inspired by Apple's [*FastFrames* (`arXiv:2611.23226`)](https://arxiv.org/abs/2611.23226).

Instead of compressing rich DiT states down to a 64-channel bottleneck only to run a multi-gigabyte 3D-WanVAE upsampler at the end of sampling, this node taps the **`4096`-dim pre-`proj_out` contextualized DiT features** alongside the **denoised `64`-ch latent `x0_hat`** at the final sampling step—decoding straight to **4-channel RGBA pixels in `5.55 ms`** (`67.1×` faster than the official VAE) with **~0.2 GB peak VRAM**.

![1:1 Native Resolution Held-Out Comparison](assets/hero_1to1_comparison.png)

---

## ✨ Key Highlights

- **67.1× Faster Decode (`372.3 ms` → `5.55 ms`)**: Instant pixel readout at the end of `KSampler` with zero VAE load/offload stalls.
- **Eliminates High-Res `VAEDecode` OOMs**: Official 64-ch Qwen-Image 2.1 VAE spikes by `2–7.5 GB` VRAM on 1MP–2K images; PixelHead uses **`< 0.2 GB`** and returns to CPU immediately after decode.
- **65× Wider Information Bandwidth than Tiny Latent VAEs**: Reads `4096`-dim DiT hidden states + `64`-dim denoised latents (`38.57 dB` PSNR on sharp typography, `34.69 dB` on 2D anime cel art, zero FP32 output banding).
- **One Checkpoint, Universal Compatibility**:
  - Works seamlessly with **8-Step Turbo LoRA (`CFG 1.0`)** AND **20–35 Step Base (`CFG > 1.0`)**;
  - Works out-of-the-box for both **Text-to-Image (T2I)** and **`<image1>` Multi-Image Reference Editing (Omni Edit)** (`norm_out` automatically strips reference prefix tokens);
  - Compatible with ComfyUI's native **`QwenImage21Cache`** KV prefix cache.
- **Native 4-Channel RGBA Output**: Simultaneously emits `image_rgb`, `alpha_mask`, and `image_rgba` in a single `5.5 ms` pass.

---

## 📊 Benchmark (`768×768` Held-Out Split on NVIDIA L4)

Measured against the official `qwen_image_2.1_vae_bf16.safetensors` (`Comfy-Org/Qwen-Image-2.1`) on the exact same held-out seeds and denoised latents (alternating 8-step Turbo LoRA and 20–30 step Base):

| Decoder | File Size | Median Decode Latency | Speedup vs. Stock VAE | Held-Out PSNR (↑) | LPIPS-VGG (↓) | Native 4-Ch RGBA |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Official Qwen-Image 2.1 VAE** | `644.2 MiB` | `372.30 ms` | `1.0×` | *(Reference)* | `0.0000` | ✅ |
| **Ours: `QwenImage21 PixelHead`** | **`48.8 MiB` (BF16)** / `97.0 MiB` (FP32) | **`5.55 ms`** | **`67.1×`** | **`33.91 dB`** | **`0.1524`** | **✅ Full 4-Ch** |

**Per-Domain Held-Out Highlights:**
- **Sharp Typography & Posters**: **`38.57 dB` PSNR** (`0.0653` LPIPS)
- **2D Anime & Cel Shading**: **`34.69 dB` PSNR** (`0.1363` LPIPS)
- **Complex Fine Art, Oil Paintings & Engravings**: **`33.76 dB` PSNR** (up to `39.3–41.3 dB` on dark chiaroscuro & velvet portraits with zero color banding)

---

## 🌐 FastFrames Zero-VAE Direct PixelHead Ecosystem

Explore our open-source MIT Direct PixelHead family across major diffusion backbones:

| Target Model | ComfyUI Custom Node (GitHub & Registry) | Hugging Face Weights | Key Specs & Speedup |
| :--- | :--- | :--- | :--- |
| **Qwen-Image 2.1** (`1×` & `2×` SR) *(This Repo)* | **[Wither033/ComfyUI-QwenImage21-PixelHead](https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead)** | **[pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead)** | `48.8 MiB` (`1×`) / `52.1 MiB` (`1×`+`2×` `2048px`) · Native **4-Ch RGBA** · **`67.1×` faster** (`5.55 ms`, `33.91 dB` PSNR) |
| **Krea-2** (`Turbo` & `Raw`) | **[Wither033/ComfyUI-Krea2-PixelHead](https://github.com/Wither033/ComfyUI-Krea2-PixelHead)** | **[pocis/Krea-2-PixelHead](https://huggingface.co/pocis/Krea-2-PixelHead)** | `52.6 MiB` BF16 · `6144d` Bridged RGB · **`24.7×` faster** (`33.33 dB` PSNR, up to `41.6 dB` on portraits) |
| **Training Toolkit & Playbook** *(All 2D/3D DiTs)* | **[Wither033/FastFrames-PixelHead-Trainer](https://github.com/Wither033/FastFrames-PixelHead-Trainer)** | **[HF Unified Collection](https://huggingface.co/collections/pocis/fastframes-zero-vae-direct-pixelheads-comfyui-6b031df8e9212f8bae294154)** | Full DiT surgery methodology, `pixelhead_trainer` PyTorch library & 5 one-click Colab L4 training notebooks |

---

## 🚀 Installation & Model Weights

### 1. Install Custom Node
- **Via ComfyUI Manager**: Search for `ComfyUI-QwenImage21-PixelHead` and click **Install**.
- **Manual Git Clone**:
  ```bash
  cd ComfyUI/custom_nodes
  git clone https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead.git
  ```
  *(Zero extra dependencies beyond stock ComfyUI!)*

### 2. Download Pixel Head Weights
Download from Hugging Face (**[pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead)**) and place into **`ComfyUI/models/pixel_heads/`** *(or `ComfyUI/models/vae/`)*:
- **`qwen_image_2.1_pixel_head_x2_bf16.safetensors`** (`52.1 MiB` — **NEW! All-in-One `1×` + `2×` Super-Res**, decodes `1024 → 2048` 4-Ch RGBA in `~12 ms`, or switch `output_scale` to `1x` for exact `5.55 ms` native readout!)
- **`qwen_image_2.1_pixel_head_bf16.safetensors`** (`48.8 MiB` — **Recommended for `1×`-only**, BF16 trunk + FP32 norms & output layer, `0.000000` diff vs. FP32)
- **`qwen_image_2.1_pixel_head.safetensors`** (`97.0 MiB`) / **`qwen_image_2.1_pixel_head_x2.safetensors`** (`103.3 MiB` — Full FP32 master weights)

---

## 🧩 Usage in ComfyUI (3 Nodes)

Ready-to-run workflows are included in [`example_workflows/`](example_workflows/):
- English Workflow: [`01_QwenImage21_PixelHead_ZeroVAE_EN.json`](example_workflows/01_QwenImage21_PixelHead_ZeroVAE_EN.json)
- 中文对比工作流: [`01_QwenImage21_PixelHead_免VAE极速直出与对比_ZH.json`](example_workflows/01_QwenImage21_PixelHead_免VAE极速直出与对比_ZH.json)

```text
[Load Diffusion Model] ──► (Optional 8-Step LoRA / QwenImage21Cache) ──► [Qwen-Image 2.1 Pixel Head Tap]
                                                                               │              │
                                                                          MODEL│              │pixel_tap
                                                                               ▼              │
                                                                          [KSampler]          │
                                                                               │LATENT        │
                                                                               ▼              ▼
                                   [Load Qwen-Image 2.1 Pixel Head] ──► [Qwen-Image 2.1 Pixel Decode (Zero VAE)]
                                                                               │
                                                                               ├──► image_rgb
                                                                               ├──► alpha_mask
                                                                               └──► image_rgba
```

> **Note on Multi-Image Editing (`<image1>`)**: If you feed reference images into `TextEncodeQwenImage21`, you still connect a VAE to `TextEncodeQwenImage21` to encode the input reference image once, while **skipping `VAEDecode` at the end** using `Qwen-Image 2.1 Pixel Decode`. For pure Text-to-Image, **no VAE needs to be loaded at all**.

---

## ⚠️ Compatibility & Known Limits

- **Supported Samplers & Steps**: Trained across `ModelSamplingFlux(shift=0.69)` final-step $\sigma \in [0.038, 0.300]$, covering both **8-Step Turbo LoRA (`5–12` steps, `CFG 1.0`)** and **Base Qwen-Image 2.1 (`18–35` steps, `CFG 1.0–5.0`)**.
- **Standalone `.latent` Files**: Because the head reads live `4096`-dim DiT features from the final denoising step, it requires sampling through `Qwen-Image 2.1 Pixel Head Tap` and cannot blindly decode an external `.latent` file without running the DiT.
- **Ultra-Dense Micro-Screentones**: On extreme monochrome woodcut engravings or 2px manga screentones, 1-pixel phase differences lower pixelwise PSNR while preserving crisp visual line separation.

---

## 📜 License & Acknowledgements

- **Code & Weights License**: **MIT License** (100% clean-room PyTorch implementation).
- **Method Reference**: Built upon the Direct Pixel Decoder formulation introduced in [*FastFrames: Video Generation at Interactive Latency*](https://arxiv.org/abs/2611.23226) (Apple Research, 2026) and inspired by community 2D feature readout explorations (`sorryhyun/anima-pixel-decoder`).

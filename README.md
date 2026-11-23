# ComfyUI-QwenImage21-PixelHead

Direct RGBA pixel readout head (`v0.2` High-Fidelity & `v0.1` Ultra-Fast `1x` / `2x`) for **Qwen-Image 2.1** in ComfyUI, replacing the `644.2 MiB` 64-channel WanVAE decoder at the end of sampling.

[English](README.md) | [简体中文](README_zh.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Hugging Face](https://img.shields.io/badge/Hugging%20Face-pocis%2FQwen--Image--2.1--PixelHead-yellow)](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead)
[![Comfy Registry](https://img.shields.io/badge/Comfy%20Registry-comfyui--qwenimage21--pixelhead-blueviolet)](https://registry.comfy.org/publishers/wither033/nodes/comfyui-qwenimage21-pixelhead)

---

## What's New in `v0.2` (2026-11-24 — High-Fidelity Upgrade for Small Text & Portraits)

Addressing community feedback ([Issue #1](https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead/issues/1)) that `v0.1` was softer than the stock `644 MiB` VAE on tiny ticket text and close-up portrait skin pores/eyelashes:

- **Full-resolution `16px → 8px → 4px → 2px → 1px` progressive refinement pyramid (`QwenImage21PixelHeadV2`, `78.5 MiB` BF16)**: Replaces `v0.1`'s `8×8` linear block bottleneck with native pixel-level `3×3` residual refinement and an `FP32` output stage.
- **Explicit `SigmaAdaLN` step conditioning + real `1024×1024` `CFG=3.5` / `8-Step Turbo` trajectory distillation**: Eliminates feature compromise between Turbo and multi-step CFG sampling.
- **Perceptual + frequency-domain training (`Charbonnier L1 + Edge Gradient + LPIPS-VGG + 2D FFT`)**: Cuts held-out `LPIPS-VGG` perceptual blur by **`42.4%`** (`0.1456 → 0.0838`) while raising `PSNR` by **`+0.82 dB`** (`+1.41 dB` on close-up portraits).
- **Important**: Please update this custom node (`git pull` or ComfyUI Manager update to `v1.2.0+`) before loading `qwen_image_2.1_pixel_head_v02_bf16.safetensors`. Both `v0.1` and `v0.2` checkpoints are auto-detected by the same loader node.

![1:1 held-out comparison: Official 644MB VAE vs v0.1 vs v0.2](assets/v02_portrait_text_1to1.png)

*(Full 6-row `1:1` native-pixel comparison board: [`assets/v02_vs_v01_1to1_comparison.png`](assets/v02_vs_v01_1to1_comparison.png))*

---

## Features

- **Zero VAE VRAM spike**: Decodes without loading the `644.2 MiB` WanVAE (whose intermediate `VAEDecode` activations peak at `1.9–7.5 GB` at high resolutions), offloading weights back to CPU immediately after execution (`2K/4K` seamless cosine-blended tiling is also built in via `tile_mode`).
- **Two checkpoint tiers for different workflows**:
  - **`v0.2` High-Fidelity (`78.5 MiB` BF16 — Recommended)**: Crisp close-up portraits (`37.76 dB` PSNR), eyelashes, paper grain, and small text (`61.0 ms` on A100 / `~120 ms` on RTX 4060 Ti).
  - **`v0.1` Ultra-Fast (`48.8 MiB` `1x` / `52.1 MiB` `1x+2x`)**: `3.4–5.5 ms` instant preview for rapid seed/composition exploration + single-pass `2x` (`1024 → 2048`) branch.
- **Single checkpoint across sampling modes**:
  - Supports both 8-step Turbo LoRA (`CFG = 1.0`) and 20–35 step Base sampling (`CFG > 1.0`).
  - Supports Text-to-Image and `<image1>` multi-image reference editing (`norm_out` strips reference prefix tokens before `proj_out`).
  - Compatible with ComfyUI's `QwenImage21Cache` and outputs 4-channel RGBA (`image_rgb`, `alpha_mask`, `image_rgba`).

---

## Benchmark (`1024×1024` Real Multi-Step Held-Out Split: 50% `8s Turbo`, 50% `20s CFG=3.5`)

| Decoder | BF16 Size | `1024²` Latency (A100) | Speedup | Overall PSNR (↑) | Overall LPIPS-VGG (↓) | Portrait PSNR / LPIPS | Small Text PSNR / LPIPS |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Official Qwen-Image 2.1 VAE** | `644.2 MiB` | `201.33 ms` | `1.0×` | *(ref)* | `0.0000` | *(ref)* | *(ref)* |
| **`PixelHead v0.2` (High-Fidelity)** | **`78.5 MiB`** | **`61.02 ms`** | **`3.3×`** | **`33.86 dB`** | **`0.0838` (`-42.4%`)** | **`37.76 dB` / `0.0808`** | **`32.70 dB` / `0.0778`** |
| `PixelHead v0.1` (Ultra-Fast) | `48.8 MiB` | `3.39 ms` *(5.55 ms @ 768² L4)* | `59.4×` | `33.04 dB` | `0.1456` | `36.35 dB` / `0.1540` | `32.20 dB` / `0.1357` |

---

## Installation

### 1. Install or Update Custom Node (`v1.2.0+` required for `v0.2` weights)
Install/update via ComfyUI Manager (`ComfyUI-QwenImage21-PixelHead`), `comfy node install comfyui-qwenimage21-pixelhead`, or `git`:

```bash
cd ComfyUI/custom_nodes/ComfyUI-QwenImage21-PixelHead
git pull
```

*(Fresh install: `git clone https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead.git` inside `ComfyUI/custom_nodes`.)*

### 2. Download Weights
Download from [pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead) into `ComfyUI/models/pixel_heads/` (or `ComfyUI/models/vae/`):

- **`qwen_image_2.1_pixel_head_v02_bf16.safetensors` (`78.5 MiB` — Recommended)**: `v0.2` High-Fidelity `1x` head (sharp portraits, skin micro-texture, and small text).
- `qwen_image_2.1_pixel_head_bf16.safetensors` (`48.8 MiB`): `v0.1` Ultra-Fast `1x` head (`3.4–5.5 ms` rapid batch drafting).
- `qwen_image_2.1_pixel_head_x2_bf16.safetensors` (`52.1 MiB`): `v0.1` combined `1x` + `2x` (`1024 → 2048` quick upscale via `output_scale`).
- Full `FP32` variants (`qwen_image_2.1_pixel_head_v02.safetensors`, etc.) are also available in the Hugging Face repo.

---

## Usage

Example workflows are in [`example_workflows/`](example_workflows/):
- [`01_QwenImage21_PixelHead_ZeroVAE_EN.json`](example_workflows/01_QwenImage21_PixelHead_ZeroVAE_EN.json)
- [`01_QwenImage21_PixelHead_免VAE极速直出与对比_ZH.json`](example_workflows/01_QwenImage21_PixelHead_免VAE极速直出与对比_ZH.json)

```text
[Load Diffusion Model] ──► (Optional 8-Step LoRA / QwenImage21Cache) ──► [Qwen-Image 2.1 Pixel Head Tap]
                                                                               │              │
                                                                          MODEL│              │pixel_tap
                                                                               ▼              │
                                                                          [KSampler]          │
                                                                               │LATENT        │
                                                                               ▼              ▼
                                   [Load Qwen-Image 2.1 Pixel Head] ──► [Qwen-Image 2.1 Pixel Decode (Zero VAE)]
```

---

## Related Repositories

| Model | ComfyUI Node | Weights | Notes |
| :--- | :--- | :--- | :--- |
| **Qwen-Image 2.1** *(this repo)* | [Wither033/ComfyUI-QwenImage21-PixelHead](https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead) | [pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead) | **`v0.2` (`78.5 MiB`, `LPIPS 0.0838`)** + `v0.1` (`48.8 MiB`, `3.4–5.5 ms`), 4-ch RGBA |
| **Krea-2** | [Wither033/ComfyUI-Krea2-PixelHead](https://github.com/Wither033/ComfyUI-Krea2-PixelHead) | [pocis/Krea-2-PixelHead](https://huggingface.co/pocis/Krea-2-PixelHead) | `52.6 MiB` BF16, `6144d` bridged RGB, `59.55 ms` (`24.7×`), `33.33 dB` |
| **MiniMax-H3** *(preview)* | [Wither033/ComfyUI-MiniMaxH3-PixelHead](https://github.com/Wither033/ComfyUI-MiniMaxH3-PixelHead) | [pocis/MiniMax-H3-Video-PixelHead](https://huggingface.co/pocis/MiniMax-H3-Video-PixelHead) | Experimental preview (high-frequency video textures are currently soft) |
| **Training Code & Notes** | [Wither033/FastFrames-PixelHead-Trainer](https://github.com/Wither033/FastFrames-PixelHead-Trainer) | [HF Collection](https://huggingface.co/collections/pocis/fastframes-zero-vae-direct-pixelheads-comfyui-6b031df8e9212f8bae294154) | Training toolkit, DiT hook notes, and training notebooks |

## Limitations

- **Ultra-tiny `2px` slanted pseudo-text & dense copperplate cross-hatching**: `v0.2` closes ~85–90% of the gap on portraits and standard small print, while ultra-compressed `2px` slanted micro-script and `1px` dense engravings remain slightly softer than the `644 MiB` stock VAE (a faster wavelet-based `v0.3` is in progress).
- **Offline `.latent` files**: Decoding requires the final-step `4096`-dim DiT features captured by the Tap node during sampling.

## License & Credits

- **License**: MIT (clean-room PyTorch implementation).
- **References**: Direct pixel readout formulation from [*FastFrames: Video Generation at Interactive Latency*](https://arxiv.org/abs/2611.23226) (Apple Research, 2026); single-step replay distillation and ComfyUI hook design informed by `sorryhyun/anima-pixel-decoder`.

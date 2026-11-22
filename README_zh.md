<div align="center">

# ⚡ ComfyUI-QwenImage21-PixelHead

**Qwen-Image 2.1 免 VAE 毫秒级直出像素头（`5.55 ms` 解码 · 提速 `67.1×` · 彻底告别 2K 解码爆显存）**

[**English**](README.md) | [**简体中文**](README_zh.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Hugging Face Weights](https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-pocis%2FQwen--Image--2.1--PixelHead-yellow)](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead)
[![Weight Size](https://img.shields.io/badge/%E6%9D%83%E9%87%8D%E4%BD%93%E7%A7%AF-48.8%20MiB%20(BF16)-blue)](#2-%E4%B8%8B%E8%BD%BD%E5%83%8F%E7%B4%A0%E5%A4%B4%E6%9D%83%E9%87%8D)
[![Speedup](https://img.shields.io/badge/%E8%A7%A3%E7%A0%81%E8%80%97%E6%97%B6-5.55%20ms%20(67.1%C3%97%20%E6%8F%90%E9%80%9F)-brightgreen)](#%E5%AE%9E%E6%B5%8B%E6%80%A7%E8%83%BD%E5%9F%BA%E5%87%86)
[![Output](https://img.shields.io/badge/%E5%8E%9F%E7%94%9F%E8%BE%93%E5%87%BA-4%E9%80%9A%E9%81%93%20RGBA-purple)](#%E6%A0%B8%E5%BF%83%E4%BC%98%E5%8A%BF)

</div>

---

**`ComfyUI-QwenImage21-PixelHead`** 基于 Apple 2026 年 11 月最新论文 [*FastFrames* (`arXiv:2611.23226`)](https://arxiv.org/abs/2611.23226) 的直接像素读出（Direct Pixel Readout）架构，用仅 **`25.4M` 参数（`48.8 MiB` BF16）** 的轻量 ConvNeXt + PixelShuffle 像素头，彻底替代 `Qwen-Image 2.1` 重达 **`644.2 MiB`**、高分辨率极易爆显存（OOM）的 64 通道 WanVAE 解码器。

不同于只能读取 `64` 维压缩潜空间的传统 Tiny VAE（小解码器），本插件在采样最后一步无侵入截获 DiT `proj_out` 前的 **`4096` 维全量上下文特征** 与 **去噪潜变量 `x0_hat`**，在 **`5.55 ms` 内一步直出高清 RGB 与 Alpha 透明通道**（比官方 VAE 快 **67.1 倍**，额外峰值显存不到 `0.2 GB`）。

![1:1 原生像素放大对比图](assets/hero_1to1_comparison.png)

---

## ✨ 核心优势

- **解码提速 `67.1×`（`372.3 ms` → `5.55 ms`）**：采样结束瞬间出图，省去重型 VAE 加载与搬运卡顿。
- **彻底根治 8GB / 12GB 显卡“跑完 99% 死在 VAEDecode”痛点**：官方 64 通道 VAE 在 1MP～2K 分辨率下瞬时显存暴涨 `2～7.5 GB`；PixelHead 解码峰值仅需 **`< 0.2 GB`**，解完即刻归还显存。
- **信息带宽比潜空间小解码器高出 65 倍**：直接读取 `4096` 维 DiT 隐藏态 + `64` 维去噪潜变量，末层强制 `FP32` 投影杜绝平涂色阶断层（文字排版实测高达 **`38.57 dB`**，暗调丝绒油画单张高达 `39.3～41.3 dB`）。
- **单权重全模式通吃**：
  - 同时兼容 **8 步加速 LoRA（`CFG 1.0`）** 与 **20～35 步原版精修（`CFG > 1.0`）**；
  - 同时支持 **文生图（T2I）** 与 **`<image1>` 多参考图指令编辑（Omni Edit）**；
  - 完美兼容官方 **`QwenImage21Cache`** KV 前缀加速缓存，同时适用官方原版与社区微调版（如 `Qwen-Image-2.1-Uncensored-Omni`）。
- **全网首个原生 4 通道 RGBA 像素直出头**：单次前向同时输出 `image_rgb`、`alpha_mask` 与 `image_rgba`。

---

## 📊 实测性能基准（NVIDIA L4 · `768×768` 独立留出测试集）

以官方原版 `qwen_image_2.1_vae_bf16.safetensors` 为教师与参照基准，在同一随机种子与去噪潜变量下实测（50% 8步加速 LoRA + 50% 20～30步原版交替测试）：

| 解码器方案 | 权重体积 | 中位数解码延迟 | 相比官方 VAE 提速比 | 独立测试集 PSNR (↑) | LPIPS-VGG (↓) | 原生 4 通道 RGBA |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **官方原版 Qwen-Image 2.1 VAE** | `644.2 MiB` | `372.30 ms` | `1.0×` | *(基准 Reference)* | `0.0000` | ✅ |
| **Ours: `QwenImage21 PixelHead`** | **`48.8 MiB` (BF16)** / `97.0 MiB` (FP32) | **`5.55 ms`** | **`67.1×`** | **`33.91 dB`** | **`0.1524`** | **✅ 完整 4 通道** |

**细分场景表现：**
- **海报小字与锐利排版（Typography）**：**`38.57 dB` PSNR**（`LPIPS = 0.0653`，字迹刀锋锐利）
- **2D 二次元赛璐璐平涂与线稿（Anime Cel）**：**`34.69 dB` PSNR**（`LPIPS = 0.1363`）
- **古典肖像、暗调丝绒油画与素描版画（Fine Art）**：均值 **`33.76 dB`**（单张肖像与暗调油画达 `37.4～41.3 dB`，零色阶断层）

---

## 🌐 FastFrames 免 VAE 极速像素头开源矩阵（同系列联动）

欢迎体验同系列其他主流开源底座的 MIT 免 VAE 像素直出头：

| 适配底座模型 | ComfyUI 插件仓（GitHub & 官方注册表） | Hugging Face 权重仓 | 核心规格与实测提速 |
| :--- | :--- | :--- | :--- |
| **Qwen-Image 2.1**（`1×` 与 `2×` 超分）*(当前仓库)* | **[Wither033/ComfyUI-QwenImage21-PixelHead](https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead)** | **[pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead)** | `48.8 MiB` (`1×`) / `52.1 MiB` (`1×`+`2×` 直出 `2048`) · **原生 4 通道 RGBA** · **提速 `67.1×`**（`5.55 ms`，均值 `33.91 dB`） |
| **Krea-2**（`Turbo` 与 `Raw`） | **[Wither033/ComfyUI-Krea2-PixelHead](https://github.com/Wither033/ComfyUI-Krea2-PixelHead)** | **[pocis/Krea-2-PixelHead](https://huggingface.co/pocis/Krea-2-PixelHead)** | `52.6 MiB` BF16 · `6144d` 降维桥直出 RGB · **提速 `24.7×`**（高难艺术集均值 `33.33 dB`，肖像达 `37.1~41.6 dB`） |
| **MiniMax-H3** *(3D视频 · ⚠️ 预览版)* | **[Wither033/ComfyUI-MiniMaxH3-PixelHead](https://github.com/Wither033/ComfyUI-MiniMaxH3-PixelHead)** | **[pocis/MiniMax-H3-Video-PixelHead](https://huggingface.co/pocis/MiniMax-H3-Video-PixelHead)** | ⚠️ *早期实验预览版（暂不推荐正式使用）* · `45.7 MiB` BF16 · 提速 `8.5×`（等待 `v3 Sharp` 高频锐化更新） |
| **通用训练框架与实战方法论** *(2D图像 & 3D视频)* | **[Wither033/FastFrames-PixelHead-Trainer](https://github.com/Wither033/FastFrames-PixelHead-Trainer)** | **[HF 官方统一合集](https://huggingface.co/collections/pocis/fastframes-zero-vae-direct-pixelheads-comfyui-6b031df8e9212f8bae294154)** | 完整公开 5 大底座源码解剖法、踩坑复盘、`pixelhead_trainer` 库与单卡 Colab L4 一键训练台 |

---

## 🚀 安装与权重下载

### 1. 安装 ComfyUI 插件
- **方式一（ComfyUI Manager）**：打开插件管理器搜索 `ComfyUI-QwenImage21-PixelHead` 一键安装。
- **方式二（手动克隆）**：
  ```bash
  cd ComfyUI/custom_nodes
  git clone https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead.git
  ```
  *（基于原生 PyTorch + safetensors，无需额外安装任何第三方依赖！）*

### 2. 下载像素头权重
前往 Hugging Face（**[pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead)**）下载权重，放入 **`ComfyUI/models/pixel_heads/`**（或直接放入 `ComfyUI/models/vae/`）：
- **`qwen_image_2.1_pixel_head_x2_bf16.safetensors`**（`52.1 MiB` —— **新增全能二合一推荐！** 同时内置 `1×` 与 `2×` 超分双输出头，`1024` 潜空间 `~12 ms` 一步直出 `2048×2048` 4通道 RGBA，也可随时切回 `1x` 原生尺寸）
- **`qwen_image_2.1_pixel_head_bf16.safetensors`**（`48.8 MiB` —— **纯 `1×` 极速推荐**，BF16 主干 + FP32 归一化与输出层，与 FP32 版输出 `0.000000` 零误差）
- **`qwen_image_2.1_pixel_head.safetensors`**（`97.0 MiB`） / **`qwen_image_2.1_pixel_head_x2.safetensors`**（`103.3 MiB` —— 完整 FP32 科研母版权重）

---

## 🧩 ComfyUI 节点接法（开箱即用工作流）

[`example_workflows/`](example_workflows/) 目录内已附带双语对比工作流，直接拖入 ComfyUI 即可运行：
- 中文版：[`01_QwenImage21_PixelHead_免VAE极速直出与对比_ZH.json`](example_workflows/01_QwenImage21_PixelHead_免VAE极速直出与对比_ZH.json)
- 英文版：[`01_QwenImage21_PixelHead_ZeroVAE_EN.json`](example_workflows/01_QwenImage21_PixelHead_ZeroVAE_EN.json)

1. 将 **`Qwen-Image 2.1 Pixel Head Tap`** 串联在进入 `KSampler` 前的 `MODEL` 连线上；
2. 用 **`Qwen-Image 2.1 Pixel Decode (Zero VAE)`** 替代原本的 `VAEDecode`，接入 `KSampler` 的 `LATENT`、`Load Qwen-Image 2.1 Pixel Head` 与 `pixel_tap` 即可。
3. **文生图（T2I）全程无需加载任何 VAE**；若做 `<image1>` 多参考图编辑，仅在输入端编码参考图时使用一次 VAE，末尾出图同样走零显存极速直出。

---

## ⚠️ 适用范围与已知边界

- **采样步数区间**：已针对 `ModelSamplingFlux(shift=0.69)` 末步 $\sigma \in [0.038, 0.300]$ 双模式交替校准，覆盖 **8步加速 LoRA（5～12步，CFG 1.0）** 与 **原版精修（18～35步，CFG 1.0～5.0）**。
- **离线纯 `.latent` 文件说明**：本像素头依赖采样最后一步实时截获的 `4096` 维 DiT 隐藏态特征，不能脱离 DiT 单独盲解外部保存的纯 `.latent` 文件。
- **超密集微米级网点**：在极端密集的黑白木刻排线或 2px 漫画网点区域，结构线条依然清晰分离，但受 1px 相位微差影响逐像素 PSNR 读数会低于平滑画面。

---

## 📜 开源协议与致谢

- **开源协议**：**MIT License**（100% Clean-Room 独立实现，允许自由商用与平台集成）。
- **方法致谢**：理论架构源自 Apple Research 论文 [*FastFrames: Video Generation at Interactive Latency*](https://arxiv.org/abs/2611.23226)（2026），并致谢社区在 2D 扩散特征直出方向的早期探索（`sorryhyun/anima-pixel-decoder`）。

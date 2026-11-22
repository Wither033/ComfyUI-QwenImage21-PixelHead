# ComfyUI-QwenImage21-PixelHead

用于 **Qwen-Image 2.1** 的 ComfyUI 免 VAE 像素直出节点（支持原生 `1x` 与一步 `2x` 超分，输出 4 通道 RGBA），用来替代采样末尾 `644.2 MiB` 的 64 通道 WanVAE 解码器。

[English](README.md) | [简体中文](README_zh.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Hugging Face](https://img.shields.io/badge/Hugging%20Face-pocis%2FQwen--Image--2.1--PixelHead-yellow)](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead)
[![Comfy Registry](https://img.shields.io/badge/Comfy%20Registry-comfyui--qwenimage21--pixelhead-blueviolet)](https://registry.comfy.org/publishers/wither033/nodes/comfyui-qwenimage21-pixelhead)

参考 [*FastFrames* (`arXiv:2611.23226`)](https://arxiv.org/abs/2611.23226) 的直接像素读出方案，插件会在采样最后一步读取 DiT `proj_out` 之前的 `4096` 维隐藏特征与去噪潜变量 `x0_hat`，经过一个 `25.4M` 参数（BF16 下 `48.8 MiB`）的轻量 ConvNeXt + PixelShuffle 网络直接还原为像素。

在 NVIDIA L4 上测试 `768×768` 图像，中位数解码耗时从官方 VAE 的 `372.3 ms` 降到 `5.55 ms`（约 `67.1×`），解码阶段额外显存占用低于 `0.2 GB`。

![1:1 像素对比（左：官方 VAE，右：PixelHead）](assets/hero_1to1_comparison.png)

## 主要特性

- **低延迟、省显存**：原生 `1x` 解码约 `5.55 ms`，`2x`（`1024 → 2048`）约 `12 ms`，算完即退回内存，避免高分辨率下最后一步 `VAEDecode` 爆显存。
- **读取高维特征（`4096 + 64` 维）**：普通小 VAE 只能从压扁后的 `64` 维潜变量里猜细节；直接读取 `proj_out` 前的 DiT 特征能更好保留小字和细线（文字排版测试集 `38.57 dB`）。末层线性投影固定用 `FP32` 计算，避免 `BF16` 在平滑渐变处出现色阶断层。
- **单权重兼容常用模式**：
  - 同时支持 8 步加速 LoRA（`CFG = 1.0`）和 20～35 步常规采样（`CFG > 1.0`）；
  - 同时支持文生图（T2I）和 `<image1>` 多参考图编辑（`norm_out` 已自动切掉参考图前缀）；
  - 兼容官方 `QwenImage21Cache` 缓存节点及常见社区量化/微调权重。
- **原生 4 通道 RGBA**：一次前向同时输出 `image_rgb`、`alpha_mask` 和 `image_rgba`。

## 实测数据（NVIDIA L4 · `768×768` 留出测试集）

以官方 `qwen_image_2.1_vae_bf16.safetensors` 的解码结果为参考，在相同随机种子与潜变量下测试（8 步 LoRA 与 20～30 步原版各占一半）：

| 解码器 | 体积 | 中位数延迟 | 相对速度 | PSNR (↑) | LPIPS-VGG (↓) | 4 通道 RGBA |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| 官方 Qwen-Image 2.1 VAE | `644.2 MiB` | `372.30 ms` | `1.0×` | *(基准)* | `0.0000` | 支持 |
| `QwenImage21 PixelHead` (`1x`) | `48.8 MiB` (BF16) / `97.0 MiB` (FP32) | `5.55 ms` | `67.1×` | `33.91 dB` | `0.1524` | 支持 |

分类别均值：
- 文字与海报排版：`38.57 dB`（`LPIPS = 0.0653`）
- 2D 赛璐璐与线稿：`34.69 dB`（`LPIPS = 0.1363`）
- 油画、肖像与版画：`33.76 dB`（暗调肖像单张 `37.4～41.3 dB`）

## 同系列仓库

不同底座模型的内部特征空间互不通用，需要分别训练对应的像素头：

| 适配模型 | ComfyUI 节点仓库 | Hugging Face 权重 | 说明 |
| :--- | :--- | :--- | :--- |
| **Qwen-Image 2.1** *(本仓库)* | [Wither033/ComfyUI-QwenImage21-PixelHead](https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead) | [pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead) | `48.8 MiB` (`1x`) / `52.1 MiB` (`1x`+`2x`)，4 通道 RGBA，`5.55 ms` (`67.1×`)，均值 `33.91 dB` |
| **Krea-2** | [Wither033/ComfyUI-Krea2-PixelHead](https://github.com/Wither033/ComfyUI-Krea2-PixelHead) | [pocis/Krea-2-PixelHead](https://huggingface.co/pocis/Krea-2-PixelHead) | `52.6 MiB` BF16，`6144d` 降维桥 RGB，`59.55 ms` (`24.7×`)，均值 `33.33 dB` |
| **MiniMax-H3** *(预览版)* | [Wither033/ComfyUI-MiniMaxH3-PixelHead](https://github.com/Wither033/ComfyUI-MiniMaxH3-PixelHead) | [pocis/MiniMax-H3-Video-PixelHead](https://huggingface.co/pocis/MiniMax-H3-Video-PixelHead) | 早期实验预览版（复杂纹理偏糊，暂不推荐正式使用，待后续优化） |
| **训练代码与方法整理** | [Wither033/FastFrames-PixelHead-Trainer](https://github.com/Wither033/FastFrames-PixelHead-Trainer) | [HF 合集页](https://huggingface.co/collections/pocis/fastframes-zero-vae-direct-pixelheads-comfyui-6b031df8e9212f8bae294154) | 通用 `pixelhead_trainer` 库、各底座挂载点说明与 Colab L4 训练脚本 |

## 安装与权重

### 1. 安装插件
可在 ComfyUI Manager 搜索 `ComfyUI-QwenImage21-PixelHead` 安装，或执行 `comfy node install comfyui-qwenimage21-pixelhead`，也可手动克隆：

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead.git
```

### 2. 下载权重
从 [pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead) 下载模型，放入 `ComfyUI/models/pixel_heads/`（放 `ComfyUI/models/vae/` 也可以识别）：

- `qwen_image_2.1_pixel_head_bf16.safetensors`（`48.8 MiB`）：仅 `1x` 原生分辨率，体积最小。
- `qwen_image_2.1_pixel_head_x2_bf16.safetensors`（`52.1 MiB`）：同时包含 `1x` 和 `2x` 分支，可通过节点的 `output_scale` 切换 `1x` 或 `2x`（`1024 → 2048`）。
- `qwen_image_2.1_pixel_head.safetensors`（`97.0 MiB`）/ `qwen_image_2.1_pixel_head_x2.safetensors`（`103.3 MiB`）：FP32 完整权重。

## 节点连接方式

[`example_workflows/`](example_workflows/) 目录下提供了中英文示例工作流，拖入 ComfyUI 即可查看接法：
- 中文：[`01_QwenImage21_PixelHead_免VAE极速直出与对比_ZH.json`](example_workflows/01_QwenImage21_PixelHead_免VAE极速直出与对比_ZH.json)
- 英文：[`01_QwenImage21_PixelHead_ZeroVAE_EN.json`](example_workflows/01_QwenImage21_PixelHead_ZeroVAE_EN.json)

1. 把 `Qwen-Image 2.1 Pixel Head Tap` 串在进入 `KSampler` 之前的 `MODEL` 连线上；
2. 用 `Qwen-Image 2.1 Pixel Decode (Zero VAE)` 替换 `VAEDecode`，连上 `LATENT`、`pixel_head` 和 `pixel_tap`。
3. 纯文生图不需要加载任何 VAE；如果用 `<image1>` 做参考图编辑，输入端仍需用原版 VAE 编码参考图，采样结束后的出图改走 Pixel Decode 即可。

## 局限说明

- **步数范围**：训练时覆盖 `ModelSamplingFlux(shift=0.69)` 末步 $\sigma \in [0.038, 0.300]$（对应 5～12 步加速 LoRA 与 18～35 步常规采样）。
- **离线 `.latent` 文件**：解码需要 Tap 节点在采样最后一步截获的 `4096` 维实时特征，无法脱离采样过程单独解码外部保存的 `.latent` 文件。
- **极密排线与网点**：在极细密的黑白版画排线或 2px 漫画网点上，1 像素以内的相位偏移会拉低 PSNR 数值，但肉眼线条本身不会糊成一团。

## 许可与致谢

- **开源协议**：MIT License（独立编写的 PyTorch 实现）。
- **参考来源**：架构思路来自 Apple Research 的 [*FastFrames* (`arXiv:2611.23226`)](https://arxiv.org/abs/2611.23226)，单步加噪 Replay Buffer 训练方式与 ComfyUI 挂载钩子参考了 `sorryhyun/anima-pixel-decoder`。

---
type: entity
category: 研究
tags: [AI, 世界模型, 机器学习, 论文, 研究]
created: 2026-07-03
source: raw/research/2026-07-03-ai-world-models.md
---

# AI 世界模型研究

> 2026-07-03 整理。AI World Models 领域全景。

## 什么是世界模型

**世界模型（World Model）** 是能够学习环境内在动态规律并预测未来状态的生成模型。给定当前观测和动作，预测下一时刻的观测/状态。

## 技术路线

| 路线 | 代表 | 方法 |
|------|------|------|
| **自回归** | Sora、Genie | 时空离散化 → token → Transformer 逐 token 预测 |
| **扩散** | DIAMOND | 从噪声中逐步去噪生成未来帧 |
| **JEPA** | I-JEPA、V-JEPA | 在抽象表征空间预测，避免像素级不确定性 |
| **隐空间想象** | Dreamer 系列 | 在隐空间中规划未来轨迹，用于 RL 决策 |

## 关键人物

| 人物 | 机构 | 核心贡献 |
|------|------|----------|
| **Jürgen Schmidhuber** | KAUST | 提出 World Models 概念框架 (2018) |
| **David Ha** | Sakana AI | 合著开创性 *World Models* |
| **Yann LeCun** | Meta | JEPA 架构体系 |
| **Danijar Hafner** | DeepMind | Dreamer 系列 |
| **Fei-Fei Li** | Stanford/World Labs | 3D 空间智能世界模型 |
| **Jim Fan** | Nvidia | Voyager、Cosmos |

## 主要公司

| 公司 | 代表工作 | 方向 |
|------|----------|------|
| **OpenAI** | Sora (2024) | 视频生成世界模型 |
| **DeepMind** | Genie、Dreamer | 可交互世界模型 |
| **Meta** | I-JEPA、V-JEPA | 自监督预测 |
| **World Labs** | 3D 世界生成 | 空间智能 |
| **Wayve** | GAIA-1 | 自动驾驶 |
| **Nvidia** | Cosmos | 开放平台 |

## 关键论文

| 论文 | 年份 | 核心贡献 |
|------|------|----------|
| *World Models* | 2018 | VAE+MDN-RNN，在"梦"中训练 RL agent |
| *DreamerV3* | 2023 | 固定超参数、150+ 任务 |
| *Sora* | 2024 | DiT + 时空 patch，涌现物理模拟 |
| *Genie* | 2024 | 从互联网视频学习可交互世界 |

## 核心争论

1. **视频生成是否等于世界模型？** Sora 经常违背物理规律
2. **像素 vs 表征？** LeCun 坚持 JEPA vs Sora 像素级
3. **能替代物理模拟器吗？** 安全性问题
4. **缩放律是否有效？** 数据 vs 模型规模关系不明

## 相关概念

- [[世界模型（AI）]]
- [[自回归世界模型]]
- [[JEPA架构]]

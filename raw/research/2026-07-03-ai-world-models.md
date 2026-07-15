---
type: raw
tags: [来源, AI, 世界模型, 研究]
created: 2026-07-03
---

# AI 世界模型研究：热门人物与论文

> 2026-07-03 整理。涵盖 AI World Models 领域的关键人物、论文、公司与核心概念。

## 核心概念

| 概念                                                 | 定义                                                             |
| -------------------------------------------------- | -------------------------------------------------------------- |
| **World Model (世界模型)**                             | 能够学习环境内在动态规律并预测未来状态的生成模型。给定当前观测和动作，预测下一时刻的观测/状态。               |
| **AutoRegressive World Model**                     | 将世界状态离散化为 token，用自回归 Transformer 逐 token 预测未来，如 Sora、Genie。    |
| **Diffusion-based World Model**                    | 使用扩散模型对世界状态分布建模，从噪声中逐步去噪生成未来帧，如 DIAMOND、Diffusion World Model。 |
| **JEPA (Joint Embedding Predictive Architecture)** | Yann LeCun 提出的预测架构，在抽象表征空间而非像素空间进行预测，避免像素级生成的不确定性。             |
| **Learned World Simulator**                        | 通过海量数据训练得到的可交互环境模拟器，区别于硬编码的物理引擎。                               |
| **Latent Imagination**                             | 在学到的隐空间中"想象"未来轨迹，用于无模型强化学习的规划（Dreamer 系列的核心机制）。                |
| **Video Tokenization**                             | 将视频压缩为离散 token 序列，使视频生成可转化为语言模型式的 next-token prediction。       |

## 主要公司/实验室

| 组织 | 代表工作 | 方向 |
|------|----------|------|
| **OpenAI** | Sora (2024) | 视频生成世界模型，DiT 架构 + 时空 patch |
| **Google DeepMind** | Genie (2024), Genie 2 (2024), Dreamer 系列 | 可交互世界模型、强化学习世界模型 |
| **Meta (FAIR)** | I-JEPA, V-JEPA (2023-2024) | 自监督预测世界模型 |
| **World Labs** (Fei-Fei Li) | 3D 世界生成 (2024-) | 空间智能，从图像生成 3D 世界 |
| **Wayve** | GAIA-1 (2023), LINGO-1 | 自动驾驶世界模型 |
| **Nvidia** | Cosmos (2025) | 自动驾驶世界模型平台 |
| **UC Berkeley / Google** | UniSim (2023) | 通用世界模拟器 |
| **Runway** | Gen-1, Gen-2, Gen-3 Alpha | 视频生成 |
| **Decart** | Oasis (2024) | 实时可玩 Minecraft 世界模型 |

## 关键人物

### 学术先驱

| 姓名 | 机构 | 核心贡献 | 代表作 |
|------|------|----------|--------|
| **Jürgen Schmidhuber** | KAUST / NNAISENSE | 提出 World Models 概念框架；"预测世界"作为 AI 核心范式 | *World Models* (2018) |
| **David Ha** | Sakana AI (前 Google Brain) | 与 Schmidhuber 合著开创性 *World Models* 论文 | *World Models* (2018) |
| **Yann LeCun** | Meta / NYU | JEPA 架构体系；在表征空间而非像素空间预测 | *I-JEPA (2023)*, *V-JEPA (2024)*, *A Path Towards Autonomous Machine Intelligence* (2022) |
| **Danijar Hafner** | UC Berkeley → DeepMind | Dreamer 系列作者，从像素学习世界模型用于规划 | *DreamerV1-V3*, *DayDreamer* |
| **Pieter Abbeel** | UC Berkeley | 机器人世界模型；视频预测用于机器人操控 | *VideoGPT*, *DayDreamer* |
| **Sergey Levine** | UC Berkeley | 基于模型的强化学习；世界模型用于机器人操控 | 多篇 Model-based RL 论文 |
| **Fei-Fei Li** | Stanford / World Labs | World Labs 创立者，推动 3D "空间智能"世界模型 | World Labs (2024-) |
| **Chelsea Finn** | Stanford | 元学习与世界模型交叉 | 多篇 Meta-Learning 论文 |

### 工业界领军人物

| 姓名 | 机构 | 核心贡献 | 代表作 |
|------|------|----------|--------|
| **Tim Brooks / Bill Peebles** | OpenAI (Sora 团队) | Sora 视频生成世界模型核心成员 | *Sora (2024)*, DiT 架构 |
| **Jake Bruce / Tim Rocktäschel** | Google DeepMind | Genie 可交互世界模型 | *Genie (2024)* |
| **Alex Kendall** | Wayve | GAIA-1 自动驾驶世界模型 | *GAIA-1 (2023)* |
| **Jim Fan** | Nvidia (GEAR Lab) | Foundation Agent / Voyager / Eureka / Cosmos | *Voyager (2023)* |
| **Cristóbal Valenzuela** | Runway | Gen 系列视频世界模型 | *Gen-1 (2023)*, *Gen-2 (2023)* |
| **Timothy Lillicrap** | DeepMind | Dreamer 系列核心作者 | *DreamerV1-V3* |
| **Ashley Edwards** | DeepMind | 可交互世界模型 | Genie 2 团队 |

## 关键论文

### 基础性论文

| 论文 | 年份 | 作者 | 核心贡献 |
|------|------|------|----------|
| *World Models* | 2018 | David Ha, Jürgen Schmidhuber | VAE+MDN-RNN 三步框架，在"梦"中训练 RL agent |
| *DreamerV1* | 2020 | Danijar Hafner et al. | Latent imagination 进行 actor-critic 规划 |
| *DreamerV2* | 2021 | Danijar Hafner et al. | 离散隐变量 + KL balancing，Atari 人类水平 |
| *DreamerV3* | 2023 | Danijar Hafner et al. | 固定超参数在 150+ 任务上工作 |
| *A Path Towards Autonomous Machine Intelligence* | 2022 | Yann LeCun | 六模块认知架构，JEPA 作为世界模型路径 |

### JEPA 系列

| 论文 | 年份 | 核心 | 会议 |
|------|------|------|------|
| *I-JEPA* | 2023 | 表征空间预测图像 mask 区域 | CVPR 2023 |
| *V-JEPA* | 2024 | 从视频中学习时空表征 | 2024 |

### 视频生成 / 可交互世界模型

| 论文 | 年份 | 机构 | 核心贡献 |
|------|------|------|----------|
| *VideoGPT* | 2021 | UC Berkeley | 视频→token→自回归 范式 |
| *Sora* | 2024 | OpenAI | DiT + 时空 patch，涌现物理模拟特性 |
| *Genie* | 2024 | DeepMind | 从互联网视频学习可交互世界 |
| *Genie 2* | 2024 | DeepMind | 单张图片生成可交互 3D 世界 |
| *DIAMOND* | 2024 | Uni of Geneva | 扩散模型世界模型，Atari 100k SOTA |
| *Oasis* | 2024 | Decart | 实时可玩 Minecraft 世界模型 |
| *Veo* | 2024-2025 | DeepMind | 大规模视频生成 |

### 自动驾驶 / 机器人世界模型

| 论文 | 年份 | 机构 | 核心贡献 |
|------|------|------|----------|
| *GAIA-1* | 2023 | Wayve | 9B 参数 Transformer 自动驾驶世界模型 |
| *UniSim* | 2023 | DeepMind / UCB | 通用交互世界模拟器 |
| *DayDreamer* | 2022 | UCB | 真实机器人上的世界模型学习 |
| *Nvidia Cosmos* | 2025 | Nvidia | 开放的世界模型平台 |

## 关键争论

1. **视频生成是否等于世界模型？** Sora 虽有物理直觉，但经常违背基本物理规律
2. **像素级 vs 表征级预测？** LeCun 坚持 JEPA vs Sora/Genie 的像素级方法
3. **世界模型能否替代模拟器？** 安全关键应用（自动驾驶）的可靠性问题
4. **缩放律是否适用？** 数据规模 vs 模型规模的效用关系不明确
5. **世界模型需要多少先验？** Dreamer 系列（强结构先验）vs Sora（弱先验、纯 scale）

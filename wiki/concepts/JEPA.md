---
type: concept
category: AI技术
tags: [AI, 概念, JEPA, 世界模型, Meta]
created: 2026-07-03
source: raw/research/2026-07-03-ai-world-models.md
---

# JEPA

## 定义

JEPA（Joint Embedding Predictive Architecture，联合嵌入预测架构）是 Yann LeCun 提出的预测式世界模型架构。核心思想是：**在抽象表征空间中做预测，而非在像素空间中做生成**。

## 提出背景

LeCun 在 *A Path Towards Autonomous Machine Intelligence* (2022) 中指出：像素级预测（如视频生成）是不确定且低效的——同一个场景的未来有无数种可能，很多细节（如树叶晃动、云朵形状）对理解世界无关紧要。JEPA 通过在表征空间预测，抛开不相关的像素细节，专注学习世界的抽象规律。

## 核心机制

1. **Context Encoder** — 将当前观测映射到表征空间
2. **Target Encoder** — 将目标（未来）观测映射到表征空间
3. **Predictor** — 在表征空间中从 context 预测 target embedding
4. 使用 **contrastive / distillation loss** 而非像素级重建 loss

## 系列工作

| 工作 | 年份 | 模态 | 关键创新 |
|------|------|------|----------|
| **I-JEPA** | 2023 | 图像 | 在表征空间预测被 mask 的图像区域；ImageNet 线性探针超越 MAE |
| **V-JEPA** | 2024 | 视频 | mask 大多数帧，从少量可见帧预测被遮帧的表征；高效视频理解 |
| **MC-JEPA** | 2024 | 多模态 | 跨模态预测：从文本/动作预测视觉表征 |

## 与像素级方法的对比

| 维度 | JEPA | 像素级世界模型（Sora 等） |
|------|------|---------------------------|
| 预测空间 | 抽象表征 | 原始像素/视频帧 |
| 计算效率 | 高（不需要解码到像素） | 低（需要高质量生成） |
| 泛化能力 | 好（抽象表征更可迁移） | 依赖生成质量 |
| 可解释性 | 低（隐表征难以可视化） | 高（所见即所得） |
| 当前成熟度 | 论文阶段 | 已有产品级应用 |

## 相关人物

- Yann LeCun（Meta / NYU）— JEPA 提出者
- Mahmoud Assran — I-JEPA 第一作者

## 相关概念

- [[世界模型]]——JEPA 是构建世界模型的一种具体架构
- [[I-JEPA]]
- [[V-JEPA]]

## 相关综合

- [[AI世界模型综述]]——全景综述

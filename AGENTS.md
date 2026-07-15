# CLAUDE.md — AIA 知识库约定

## 项目概述

友邦人寿 (AIA China) 保险产品知识库。采用 LLM Wiki 方法论持续维护。

## 目录结构

```
/
├── raw/                     # 不可变原始数据（LLM 只读，不修改）
│   ├── aia-products/        # 产品 API 响应 / 产品页面截图
│   └── aia-news/            # 新闻稿 / 公告
├── wiki/                    # LLM 生成/维护的页面（LLM 完全拥有）
│   ├── entities/            # 实体页：每个产品一篇 [[产品名]]
│   ├── concepts/            # 概念页：保险术语、保障类型解释
│   ├── syntheses/           # 综合页：对比分析、体系总结
│   ├── sources/             # 来源摘要页：每个 raw 文件对应一篇
│   ├── index.md             # 全部页面目录 + 摘要
│   ├── log.md               # 追加式操作日志
│   └── overview.md          # 顶层综合概述
├── AGENTS.md                # 本文件
└── AIA产品/                 # 旧版（逐步迁移到 wiki/）
```

## 核心操作

### Ingest（摄入）
1. 将来源放入 `raw/` 目录，文件名 `YYYY-MM-DD-slug.md`
2. LLM 读取来源，写入摘要页 `wiki/sources/`
3. 如有新产品/概念，创建对应 `wiki/entities/` 或 `wiki/concepts/` 页面
4. 更新 `wiki/index.md` 和交叉引用
5. 在 `wiki/log.md` 追加 `## [日期] ingest | 标题`
6. 触及相关页面更新（通常 10-15 页）

### Query（查询）
- 优先引用 wiki 中的摘要/综合页，再 fallback 到 raw 原文
- 回答必须标注来源（`→ raw/xx:line`）
- 高质量回答可合并回 wiki 页面

### Lint（健康检查）
- 检查矛盾声明、过时信息、孤立页面、缺失交叉引用

## 文件命名

| 位置 | 格式 | 示例 |
|------|------|------|
| `raw/` | `YYYY-MM-DD-slug.md` | `2026-07-02-aia-product-model-json.md` |
| `wiki/entities/` | `产品名.md` | `友邦智选逸生医疗保险.md` |
| `wiki/concepts/` | `概念名.md` | `保证续保.md` |
| `wiki/syntheses/` | `主题-slug.md` | `友邦医疗险对比.md` |
| `wiki/sources/` | `来源-slug.md` | `aia-product-api-2026-07-02.md` |

## 页面格式规范

### Entity 页（产品）
```markdown
---
type: entity
category: 产品
tags: [保险, AIA, 医疗险]
created: 2026-07-02
updated: 2026-07-02
source: raw/aia-products/2026-07-02-aia-product-model-json.md
---

# 产品名

## 基本信息
- **分类**：[[意外医疗]]
- **官网**：[链接](url)

## 产品简介
...

## 相关概念
- [[保证续保]]

## 相关产品
- [[友邦智选逸生医疗保险]]
```

### Concept 页（概念）
```markdown
---
type: concept
category: 保险术语
tags: [保险, 概念]
created: 2026-07-02
---

# 概念名

## 定义
...

## 相关产品
- [[产品A]]
- [[产品B]]
```

## 标签体系

| 标签 | 含义 |
|------|------|
| `保险` | 社保/商保相关内容 |
| `AIA` | 友邦专属 |
| `医疗险` / `重疾险` / `意外险` / `寿险` / `年金险` | 产品类型 |
| `概念` | 术语定义 |
| `来源` | 原始数据文件 |

## 注意事项

- 每个实体页面必须标注 `source` 指向原始数据
- wikilink 用中文名 `[[产品名]]` 而非路径
- 日志固定前缀 `## [日期] 操作类型 | 描述`
- 新增信息触及相关页面交叉引用更新

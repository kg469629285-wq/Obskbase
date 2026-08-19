# RAG over COBOL：用 LLM Agent 对大型遗留代码库做语义检索、代码理解与变更影响分析——方法学与最佳实践调研报告

> **调研日期**：2026-08-19
> **调研范围**：方法学（methodology）而非具体产品评测
> **目标场景**：金融系统核心系统 COBOL/CL 代码库，用 agent 实现「口述需求 → 变更影响分析」与「口述需求 → 查清现行处理逻辑」
> **来源说明**：本报告所有引用均可溯源（官方文档 / arXiv 论文 / 厂商公告）。凡标注「前沿实验」的内容为研究阶段成果，不视为生产级结论；凡标注「成熟实践」的内容有厂商或工程实践背书。

---

## 摘要（TL;DR）

1. **代码 RAG 是可行的，但「纯向量检索」对代码库不够**。成熟做法是「语法感知 chunking + 混合检索（BM25 + 向量）+ 重排（rerank）」；对 COBOL，**PARAGRAPH/SECTION 是天然 chunk 边界**，IBM 官方明确用「code-aware segmentation」规避通用 LLM chunking 的缺陷。
2. **影响分析不能只靠 LLM 记忆，必须叠加「静态分析/代码知识图谱」**。IBM watsonx Code Assistant for Z 的 agentic 工作流正是「Z Understand 静态元数据 → 影响分析 → 代码生成 → 编译验证」；开源侧 Codebase-Memory（Tree-Sitter 知识图谱 + MCP）已证明图谱检索比「反复读文件」省 10 倍 token。
3. **LLM 直接做影响分析的能力被高估**：2025 年底的实证研究显示 GPT-5 在代码变更影响分析上表现不佳（见 §3.3）。正确姿势是「LLM 做规划与语义判断 + 确定性工具（调用图/数据流/编译）做事实核查」。
4. **防幻觉的成熟组合**：强制引用文件/行号（grounding）+ 黄金测试集（eval 体系）+ 编译/符号执行等确定性验证 + human-in-the-loop 确认点。IBM 用「数据字典/业务术语表」作为权威上下文，是 COBOL 场景防幻觉的关键一招。
5. **技术栈建议**：金融核心系统场景推荐「**自建语义索引 + 现成 agent 框架 + 确定性静态分析**」的混合架构（方案 B），而非纯自研或纯商用产品（详见 §6）。

---

## 0. 背景与问题定义

大型机（mainframe）上的 COBOL/CL 代码库具有以下特征，直接决定技术选型：

- **代码量大、文件长**：单个 COBOL 程序动辄数千行，常**超出 LLM 上下文窗口**（多智能体 COBOL 解释论文明确把「代码超过 token 窗口」列为首要挑战）。
- **文档缺失**：IBM 官方指出 Z 应用「跨数十年开发，文档碎片化、过时或缺失」。
- **强耦合**：程序 ↔ 数据文件（VSAM/DB2 表）↔ 作业（JCL/CL）↔ 字段之间存在复杂依赖，变更影响面大。
- **正确性要求极高**：金融核心系统不允许「大概对」。

因此，本报告围绕六个问题展开：① 代码 RAG 怎么做；② 代码知识图谱怎么做；③ agent 影响分析的模式；④ 口述需求如何结构化；⑤ 如何防幻觉；⑥ 2025-2026 务实技术栈。

---

## 1. 代码 RAG / 语义检索 over 代码库

### 1.1 通用代码 RAG 的成熟做法

**成熟实践**：Anthropic 官方《Introducing Contextual Retrieval》（2024-09）给出了 RAG 的标准流水线，并量化了各环节收益：

1. 把语料切成小块（通常几百 token）；
2. 用 embedding 模型向量化；
3. 检索时**同时用 BM25（词法精确匹配）与向量（语义相似）**，做 rank fusion 合并去重；
4. 用 reranker 精排，取 top-K 进 prompt。

关键量化结论（Anthropic 实验，跨代码库/论文/小说等域）：
- 纯向量检索 top-20 失败率 5.7%；
- **Contextual Embeddings（给每个 chunk 前置一段「该 chunk 在全文中的语境说明」再向量化）** 把失败率降到 3.7%（-35%）；
- **Contextual Embeddings + Contextual BM25** 降到 2.9%（-49%）；
- **再加 rerank** 降到 1.9%（-67%）。

> 来源：https://www.anthropic.com/engineering/contextual-retrieval

**对代码库的启示**：代码 chunk 的「语境」就是「它属于哪个程序/段落/数据段、被谁调用、操作哪些字段」。给每个 chunk 前置这类上下文，能显著提升检索命中率——这与 COBOL 场景高度契合（见 1.2）。

**成熟实践（代码专用）**：CelloAI（arXiv 2508.16713，2025-08，高能物理 HPC 代码库）验证了**语法感知 chunking（syntax-aware chunking）**：在 embedding 时**保留语法边界**，比随意切分检索更准；同时把**调用图（callgraph）知识**纳入系统，保持依赖感知。这是「代码 RAG 应尊重语法结构」的直接证据。

> 来源：https://arxiv.org/abs/2508.16713

**成熟实践（检索→定位）**：FaR-Loc（arXiv 2509.20552，2025-09）把 RAG 用于故障定位：① LLM 把失败测试/堆栈转成自然语言「功能描述」；② 用代码理解 embedding（UniXcoder 等，**含代码结构信息**）把描述与代码映射到同一语义空间做稠密检索；③ LLM 重排。在 Defects4J 上 Top-1 提升 14.6%~19.2%。结论：**「含结构信息的代码 embedding」显著优于纯文本 embedding**（Top-1 最高 +49%）。

> 来源：https://arxiv.org/abs/2509.20552

### 1.2 COBOL 特有的 chunking：PARAGRAPH/SECTION 边界

**核心判断：COBOL 的 PARAGRAPH / SECTION 是天然、高质量的 chunk 边界。**

理由（结合 COBOL 语言结构与上述证据）：
- COBOL 程序结构高度规整：`IDENTIFICATION DIVISION` → `ENVIRONMENT DIVISION` → `DATA DIVISION`（含 `WORKING-STORAGE`、`FILE SECTION`、`LINKAGE SECTION`）→ `PROCEDURE DIVISION`（由 PARAGRAPH/SECTION 组成）。每个 PARAGRAPH 通常对应一个业务步骤，语义内聚。
- 这与「语法感知 chunking 优于随意切分」（CelloAI）的结论一致：PARAGRAPH 就是 COBOL 的「语法边界」。
- 数据段（DATA DIVISION 的字段定义、`COPYBOOK`）与过程段（PROCEDURE DIVISION）应**分开索引**，因为「字段定义」与「字段使用」是两类不同查询。

**成熟实践（厂商背书）**：IBM watsonx Code Assistant for Z 的 Business Rule Discovery 明确采用 **「code-aware segmentation that is designed to reduce common pitfalls of generic LLM chunking」**（代码感知分段，规避通用 LLM chunking 的常见缺陷），用于大型 Z 程序。这是「COBOL 不能套用通用 chunking」的官方背书。

> 来源：https://www.ibm.com/new/announcements/agentic-ai-for-smarter-mainframe-modernization-with-ibm-watsonx-code-assistant-for-z

**前沿实验（COBOL 超长文件）**：多智能体 COBOL 代码解释（arXiv 2507.02182，2025-07）指出 COBOL 文件常**超出 LLM token 窗口**，其方案是「两个 agent 协作 + 把代码库上下文注入解释 prompt」，在 14 个开源 COBOL 项目上函数级解释优于基线（METEOR +12.67%）。这印证：**COBOL 必须分层（函数/段落 → 文件 → 项目）处理，不能一把梭**。

> 来源：https://arxiv.org/abs/2507.02182

**建议的 COBOL chunking 策略（综合上述证据）**：
| 层 | chunk 单元 | 用途 |
|---|---|---|
| 段落级 | 单个 PARAGRAPH/SECTION（含其调用的子段落引用） | 语义检索、逻辑解释 |
| 数据级 | 单个字段/记录/COPYBOOK 定义 | 「某字段在哪定义、什么类型」 |
| 程序级 | 程序头 + 依赖清单（调用的程序、使用的文件/表） | 影响分析入口 |
| 文件级 | 文件摘要（Contextual Retrieval 式前置语境） | 全局检索 |

### 1.3 检索策略：BM25 混合、GraphRAG、代码知识图谱

- **BM25 混合是必须的**：代码里大量「精确标识符」（程序名、字段名、表名、错误码）是语义 embedding 的弱项、BM25 的强项（Anthropic 原文以 "Error code TS-999" 为例）。COBOL 场景尤其如此——业务人员口述的「保单表」「保费字段」需要精确映射到 `POLICY-FILE`、`PREMIUM-AMOUNT` 这类标识符。
- **GraphRAG 用于「跨点连线」类问题**：Microsoft GraphRAG（arXiv 2404.16130）指出基线 RAG 在「需要跨多个片段连线综合」与「整体理解大语料」两类问题上表现差，GraphRAG 用「知识图谱 + 社区层级 + 社区摘要」解决。对 COBOL 影响分析，「改 A 会影响哪些 B」本质是图遍历问题，GraphRAG 思路（图谱 + 社区摘要）比纯向量更合适。
  - 来源：https://microsoft.github.io/graphrag/ ；https://arxiv.org/pdf/2404.16130
- **代码知识图谱是代码域的特化**：见 §2。

### 1.4 有没有专门讨论 COBOL/结构化代码 RAG 的资料？

**有，但数量少、且多为「厂商 + 论文」两条线**：
- 厂商线：IBM watsonx Code Assistant for Z（Z Understand 元数据 + 影响分析 + 代码感知分段）——最贴近「COBOL 结构化 RAG」的工业实现。
- 论文线：COBOL 专项 LLM（XMainframe、COBOL-Coder、COBOLAssist，见 §6）与 COBOL 解释/翻译（2507.02182、2504.10548 等）——但**专门写「COBOL 向量化检索」的公开论文极少**，多数是「COBOL 生成/翻译/解释」。这说明该细分领域仍偏前沿，落地主要靠工程组合（语法解析 + 通用 RAG 技术）。

---

## 2. 代码知识图谱（Code Knowledge Graph / GraphRAG over code）

### 2.1 构建「程序→调用→数据文件→字段」图谱

**成熟实践（通用代码域）**：Codebase-Memory（arXiv 2603.27277，2026-03）是开源系统，用 **Tree-Sitter 解析 66 种语言**构建持久化知识图谱，通过 **MCP（Model Context Protocol）** 暴露给 LLM，内置**调用图遍历、影响分析、社区发现**。在 31 个真实仓库上：答案质量 83%（对照「反复读文件」的探索型 agent 为 92%），但 **token 消耗少 10 倍、工具调用少 2.1 倍**；在图原生查询（hub 检测、调用者排序）上 19/31 语言持平或超越。

> 来源：https://arxiv.org/abs/2603.27277

**成熟实践（图谱+LLM 协同）**：CKGFuzzer（arXiv 2411.11532，2024-11）用**过程间程序分析**构建代码知识图谱（节点=函数/文件等代码实体），LLM agent 通过查询图谱理解 API 用途、生成 fuzz driver，覆盖率 +8.73%、人工审查工作量 -84.4%。证明「图谱给 LLM 提供结构化上下文」能显著提升下游任务。

> 来源：https://arxiv.org/abs/2411.11532

**对 COBOL 的映射**：把上述「函数/文件」节点替换为 COBOL 世界的实体：
- 节点：程序（PROGRAM）、段落（PARAGRAPH）、数据文件（VSAM/DB2 表）、字段（FIELD）、作业（JCL/CL）、COPYBOOK；
- 边：`CALL`（程序调用）、`READ/WRITE`（程序↔文件）、`REFERENCE`（段落↔字段）、`INCLUDE`（程序↔COPYBOOK）、`STEP`（作业↔程序）。

### 2.2 工具盘点

| 工具 | 类型 | 是否支持 COBOL | 说明 |
|---|---|---|---|
| **CodeQL**（GitHub） | 静态分析/查询语言 | **否**（支持 C/C++、C#、Java、JS/TS、Python、Ruby、Go、Swift、Kotlin） | 把代码抽成关系数据库（AST、数据流图、控制流图），用 QL 查询。是「确定性影响分析」的标杆，但**不支持 COBOL**，只能用于配套的 Java/现代层。 |
| **Sourcegraph code graph** | 代码智能平台 | 部分（需确认语言支持） | 预计算符号/引用/定义图，供 Cody 等做上下文。文档站为 JS 渲染，本调研未能直接抓取正文，仅作概念性提及。 |
| **Tree-Sitter** | 解析器生成器 | **支持 COBOL**（社区 grammar） | Codebase-Memory 的底层；可自建 COBOL 语法树 → 图谱。 |
| **Neo4j / 图数据库** | 存储/查询 | 通用 | 自定义方案：解析 COBOL 后灌入图库，用 Cypher 做影响分析（「谁调用了 X」「X 用了哪些字段」）。 |
| **IBM Z Understand** | 大型机静态分析 | **是** | 商业级 COBOL/PL-I 依赖分析，IBM agentic 工作流的事实层。 |
| **Microsoft GraphRAG** | 通用 GraphRAG 框架 | 通用（文本级） | 对代码需先做「代码→文本单元」预处理；社区摘要适合「整体理解」。 |

> CodeQL 来源：https://codeql.github.com/docs/codeql-overview/about-codeql/ （数据库含 AST、数据流图、控制流图；每种语言一个 extractor）

### 2.3 对 COBOL 调用链分析的适用性

- **COBOL 调用链是「显式且静态」的**：`CALL 'PROG-NAME'`、`PERFORM PARAGRAPH`、`COPY` 都是编译期可确定的，**非常适合确定性静态分析**——不需要 LLM 猜。这是 COBOL 相比动态语言做影响分析更「可靠」的一面。
- **成熟实践**：IBM 的 Impact Analysis 工具「leverages the static analysis from the Z Understand Metadata」——即**影响分析建立在静态分析之上**，LLM 只负责「理解需求、生成代码、解释结果」，事实（依赖关系）由确定性工具给出。
- **前沿实验（诚实警示）**：纯 LLM 做调用链/影响分析并不可靠（见 §3.3 GPT-5 研究）。因此**图谱/静态分析是「事实层」，LLM 是「语义层」**，二者分工，不要指望 LLM 替代静态分析。

---

## 3. LLM agent 的代码影响分析（impact analysis）

### 3.1 已知模式：planning + 工具调用 + 迭代确认

**成熟实践（Anthropic 官方 agent 模式）**：《Building effective agents》（2024-12）给出可组合模式谱系：
- **Workflow**（预定义路径）：prompt chaining、routing、parallelization、**orchestrator-workers**（中心 LLM 动态拆任务、派发给 worker、汇总）、evaluator-optimizer（生成-评估循环）；
- **Agent**（LLM 自主用工具循环）：开始于用户指令/对话 → 规划 → 独立执行 → **在 checkpoint 或遇到阻塞时暂停征求人工反馈** → 终止条件（如最大迭代次数）控制。

对「口述需求 → 影响分析」，最贴合的形态是 **orchestrator-workers + evaluator-optimizer 的组合**：主 agent 把需求拆成「找程序」「找数据文件」「找作业」「评估影响面」等子任务，worker 用检索/图谱工具执行，评估 agent 校验完整性，最后**在输出变更清单前设置人工确认点**。

> 来源：https://www.anthropic.com/engineering/building-effective-agents

**成熟实践（上下文管理）**：《Effective context engineering for AI agents》（2025-09）补充了长任务三件套：
- **Compaction**（上下文压缩续跑）；
- **结构化笔记/agentic memory**（把中间结论写进 NOTES 类文件，跨轮次保留）；
- **子 agent 架构**（每个子 agent 独立探索，只把 1-2k token 的浓缩结论返回主 agent）。

并强调**混合策略**：前置少量高价值上下文（如 CLAUDE.md/程序清单）+ **just-in-time 检索**（用 glob/grep/图谱工具按需取文件），避免「预索引过期」与「一次性塞满上下文」。对 COBOL 大代码库，**just-in-time + 图谱工具**是控制 token 成本的关键。

> 来源：https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

**成熟实践（工具设计 ACI）**：同一篇文章与《Building effective agents》都强调**工具即 agent 的接口**：工具要自包含、描述清晰、参数无歧义、返回 token 高效；「如果人类工程师都说不清该用哪个工具，agent 更做不到」。对 COBOL 场景，工具集应最小化且正交，例如：`search_program`、`get_callers(program)`、`get_fields(file)`、`get_jobs(program)`、`read_paragraph(program, para)`。

### 3.2 已知金融/大型机场景案例

**成熟实践（最直接对标）**：IBM watsonx Code Assistant for Z 的 **agentic 工作流（2.8 版，2025-12 发布）** 官方示例正是用户要的场景：

> 开发者输入：「I need to add a column to the Motor Policy Table that captures if the vehicle is an electric car. Can you help me update all of the programs with this field?」（我要给 Motor Policy 表加一列记录是否电动车，请帮我更新所有用到该字段的程序）

系统自动：**识别依赖 → 执行影响分析 → 生成代码（遵守编码规范）→ 编译构建验证**。其工具集基于 **MCP**：Z Understand 元数据检索、影响分析（基于静态分析）、代码生成与编码规范。

> 来源：https://www.ibm.com/new/announcements/agentic-ai-for-smarter-mainframe-modernization-with-ibm-watsonx-code-assistant-for-z

**成熟实践（金融/社保案例）**：IBM 官方案例 NOSI（National Organization for Social Insurance，社保机构）用 watsonx Code Assistant for Z 现代化核心应用：**分析冗余 COBOL 代码耗时 -94%（8 小时→30 分钟）**，**理解复杂应用耗时 -79%（24 小时→5 小时）**。

> 来源：https://www.ibm.com/products/watsonx-code-assistant-z （Case studies 部分）

**前沿实验（需求级影响分析）**：ProReFiCIA（arXiv 2511.00262，2025-10）用 LLM 做**需求变更影响分析**（改一条需求 → 找出受影响的其他需求）：工业数据集上 recall 85.7%；**加入 RAG（领域知识）后 recall 提升到 95.8%**；工程师只需复核全部需求的 3.0%~3.4%。这直接支持「口述需求 → 影响清单」的可行性，且量化了「RAG 注入领域知识」的增益。

> 来源：https://arxiv.org/abs/2511.00262

**前沿实验（发布/爆炸半径）**：LLM-Augmented Release Intelligence（arXiv 2603.14619，2026-03）把「变更摘要 + 静态任务-流水线依赖分析」结合，量化每次变更的 blast radius（爆炸半径），集成进 CI/CD。思路可迁移到「改一个程序 → 列出受影响的作业/批次」。

> 来源：https://arxiv.org/abs/2603.14619

### 3.3 诚实的数据：纯 LLM 影响分析能力被高估

**前沿实验（必须警惕）**：《A Dataset and Preliminary Study of Using GPT-5 for Code-change Impact Analysis》（arXiv 2512.19481，2025-12）：
- 构造了含 seed-change、change-pair、change-type 的数据集；
- 两种配置（① seed-change + 父提交树；② 再加 diff hunk）下，**GPT-5 与 GPT-5-mini 表现均不佳**；
- 提供 diff hunk 只带来轻微提升。

**结论**：**不要指望 LLM 凭上下文直接输出可靠的影响清单**。影响分析必须由确定性工具（调用图/数据流/编译）给出事实，LLM 负责需求理解、检索编排与结果解释。

> 来源：https://arxiv.org/abs/2512.19481

**前沿实验（编排方式）**：《Deterministic vs. LLM-Controlled Orchestration for COBOL-to-Python Modernization》（arXiv 2605.09894，2026-05）在 COBOL→Python 现代化上对比「确定性编排」与「LLM 自主编排」：**确定性编排准确率相当、最差情况鲁棒性更好、token 消耗最多省 3.5 倍**。对「结构化、有明确验证阶段」的工作流（影响分析正是如此），**固定执行策略优于完全 agentic**。

> 来源：https://arxiv.org/abs/2605.09894

---

## 4. 口述/自然语言需求 → 结构化变更清单

### 4.1 需求规格化与澄清

**成熟实践（Anthropic）**：《Building effective agents》指出 agent 开始于「与用户的指令或交互式讨论」，任务清晰后才独立执行，**执行中可回到人类处获取信息或判断**。对模糊口述需求，正确做法是**先澄清再执行**：agent 反问「涉及哪个表/哪个字段/哪个批次？」，把口述转成结构化需求卡（需求 ID、目标对象、变更类型、验收标准、影响范围）。

**成熟实践（IBM）**：IBM agentic 工作流把「数据字典 + 业务术语表」作为权威上下文注入——「每个企业都以独特方式定义变量」，数据字典让 LLM 用**组织真实语义**而非猜测缩写含义。这是「口述业务语言 → 系统标识符」映射的关键基础设施。

> 来源：https://www.ibm.com/new/announcements/agentic-ai-for-smarter-mainframe-modernization-with-ibm-watsonx-code-assistant-for-z

### 4.2 生成 AC 验收标准

**前沿实验（但证据扎实）**：《Epic-Organized vs. Requirement-Aligned Gherkin: An Empirical Evaluation of LLM-Based Acceptance Criteria Generation》（arXiv 2607.01980，2026-07）：
- 用 LLM 从需求文档自动生成 **Gherkin/BDD 验收标准**（PURE 数据集，107 条需求）；
- JSON 约束的 pipeline 生成场景**结构 100% 合法**；语义需求覆盖率 94.3%（基线 92.9%）；
- 专家盲评：epic 组织式在正确性（4.61 vs 4.14）、可执行性（4.61 vs 4.07）、完整性（4.31 vs 3.50）上更优。

**启示**：用**结构化输出约束（JSON schema）+ 按业务主题（epic）组织**生成 AC，比零样本自由生成更可靠。AC 可进一步作为影响分析的「验收锚点」与 eval 的「黄金答案」。

> 来源：https://arxiv.org/abs/2607.01980

### 4.3 建议的「口述需求 → 变更清单」流水线（综合 §3、§4 证据）

```
口述需求
  → [澄清 agent] 反问缺失信息（对象/字段/批次/生效时点），产出结构化需求卡
  → [规格化 agent] 需求卡 → 结构化变更描述（JSON：变更类型、目标程序/表/字段、AC 列表）
  → [影响分析 agent] 用图谱/静态分析工具遍历：调用者、数据文件、作业、COPYBOOK
  → [评估 agent] 对照 AC 校验清单完整性（evaluator-optimizer 循环）
  → [人工确认点] 输出「变更清单 + 证据（文件/行号）+ 置信度」，人工批准后进入实施
```

---

## 5. 验证与防幻觉

### 5.1 强制引用 / grounding

- **成熟实践**：RAG 的本质就是 grounding——把检索到的真实代码片段注入 prompt，让 LLM 基于证据回答（Anthropic Contextual Retrieval）。对代码场景，**强制输出「文件:行号」引用**是防幻觉的第一道闸：任何影响分析结论必须能回溯到具体程序/段落/字段。
- **成熟实践（权威上下文）**：IBM 用数据字典/业务术语表作为「权威事实源」，避免 LLM 猜测变量/缩写含义（见 §4.1）。
- **成熟实践（LLM-as-judge 防幻觉）**：Anthropic《Demystifying evals for AI agents》（2026-01）明确建议：给 LLM judge **「Unknown」逃生通道**（信息不足时允许返回 Unknown），用**结构化 rubric 分维度独立评分**，并**用人类专家校准 LLM judge**。

> 来源：https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

### 5.2 黄金测试集 / eval 体系

**成熟实践（Anthropic 2026-01 官方方法论）**：
- **Eval 结构**：task（任务）→ trial（多次尝试）→ grader（评分器）→ transcript（轨迹）→ outcome（环境终态）。对 agent，**outcome 比 transcript 更重要**（「说订好了」≠「数据库里真有订单」）。
- **三类 grader**：代码型（字符串/正则/静态分析/状态检查，快且客观）、模型型（rubric/LLM judge，灵活但需人工校准）、人工型（SME 评审，用于校准模型 grader）。
- **capability eval vs regression eval**：能力评估从低通过率起步「爬坡」；回归评估应接近 100% 通过率防回退。
- **pass@k vs pass^k**：前者「k 次尝试至少一次成功」，后者「k 次全部成功」。**面向客户/金融场景用 pass^k**（一致性优先）。
- **起步规模**：**20-50 个来自真实失败案例的任务就够起步**（80/20 原则），不必等几百个。
- **任务质量**：两个领域专家应能独立得出相同通过/失败结论；每个任务要有**参考解（reference solution）**证明可解。

**对 COBOL 影响分析场景的落地**：建一个「黄金影响分析集」——取 20-50 个真实历史变更（含已知受影响程序/文件/作业清单），作为回归 eval；每次改 prompt/索引/模型后跑一遍，防止「感觉变好了实际变差了」。

> 来源：https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

**成熟实践（确定性验证）**：COBOL 场景最有力的验证是**编译 + 符号执行**：
- IBM WCA4Z 团队论文（arXiv 2504.10548 / 2506.10999，ASE 2024）用**符号执行为 COBOL 生成单元测试**（mock 外部调用），转成 JUnit 验证 COBOL→Java 的**语义等价**；
- COBOLAssist（arXiv 2604.03978，2026-04）用**编译反馈迭代修复**：GPT-4o 编译成功率 41.8%→95.89%，GPT-4o-mini 29.5%→64.38%；
- SEDCoT（arXiv 2607.04092，2026-07）用**符号执行 + delta debugging** 把失败测试最小化为反例，加速修复。

> 来源：https://arxiv.org/abs/2504.10548 ；https://arxiv.org/abs/2604.03978 ；https://arxiv.org/abs/2607.04092

### 5.3 置信度标注

**前沿实验（有实证）**：SimpleDevQA（arXiv 2512.08867，2025-12）发现：**LLM 在开发知识问答上系统性过度自信（overconfidence）**，且「回答准确率与自报置信度正相关」。含义：置信度标注**不能直接信**，但可作为**排序/分流信号**（低置信度 → 强制人工复核），并需用黄金集校准。

> 来源：https://arxiv.org/abs/2512.08867

### 5.4 human-in-the-loop 确认点设计

**成熟实践**：
- Anthropic：agent 在 **checkpoint / 阻塞时暂停征求人工反馈**；编码 agent 即使测试通过，**人工评审仍必要**（对齐系统级需求）。
- IBM agentic 工作流：影响分析 → 生成代码 → **编译构建验证**，验证通过才交付——验证点是内置的 HITL 闸门。
- ProReFiCIA：工程师只需**复核被预测为受影响的 3.0%~3.4% 需求**——「LLM 先筛、人只审命中项」是控制人力成本的设计范式。

**建议的确认点**：① 需求卡确认（动手前）；② 影响清单确认（实施前，附证据）；③ 变更后验证（编译/测试/符号执行通过后）；④ 上线前回归（黄金集）。

---

## 6. 技术栈建议（2025-2026）

### 6.1 三种架构方案对比

**方案 A：纯商用产品（IBM watsonx Code Assistant for Z 等）**
- 组成：Z Understand 静态分析 + 领域微调 LLM + agentic 工作流 + MCP 工具集。
- 优点：COBOL 专项、开箱即用、有金融/社保案例背书（NOSI：分析耗时 -94%）；影响分析基于成熟静态分析，可靠性高。
- 缺点：**闭源、成本高、定制受限**；「口述需求→影响分析」的 prompt/流程难以深度定制；数据出域合规需评估。
- 适用：预算充足、要快速见效、接受厂商锁定。

**方案 B：自建语义索引 + 现成 agent 框架 + 确定性静态分析（推荐）**
- 组成：
  - **索引层**：Tree-Sitter（COBOL grammar）解析 → 段落/字段/程序级 chunk → 向量库（如 pgvector/Qdrant）+ BM25（如 Elasticsearch/Meilisearch）混合检索 + reranker；可选 Contextual Retrieval 式前置语境。
  - **图谱层**：自建「程序→调用→数据文件→字段→作业」知识图谱（Neo4j 或 SQLite 图），或复用 Codebase-Memory 式 Tree-Sitter 图谱 + MCP 暴露给 agent。
  - **Agent 层**：Claude Code / Anthropic Agent SDK（或 LangGraph 等）做 orchestrator-workers + evaluator-optimizer；工具集最小化（search/get_callers/get_fields/get_jobs/read_paragraph）。
  - **验证层**：编译（GnuCOBOL 等）+ 符号执行/单元测试 + 黄金影响分析集（20-50 例）做回归 eval。
- 优点：**可控、可定制、成本弹性**；事实层（图谱/编译）与语义层（LLM）分离，符合 §3.3 的实证结论；数据不出域。
- 缺点：工程量大（解析、图谱、eval 都要自建）；COBOL 向量检索无成熟现成件，需自己调。
- 适用：**金融核心系统（数据敏感、要深度定制、长期维护）**——即本报告目标场景。

**方案 C：纯自研（领域微调模型 + 全自建）**
- 组成：用 COBOL-Coder / XMainframe 类领域模型（arXiv 2604.03986 / 2408.04660）微调 + 全自建索引/图谱/agent。
- 优点：完全自主、可离线、无 API 依赖。
- 缺点：**成本与门槛最高**；领域模型在「理解/解释」上强，但**生成正确性仍有限**（COBOL-Coder 在 COBOLEval 上 pass@1 仅 49.33，GPT-4o 仅 16.4；COBOLAssist 显示通用模型编译成功率低需迭代修复）；维护微调管线是长期负担。
- 适用：强合规/离线约束、且已有模型训练能力的大厂。

> 领域模型数据来源：https://arxiv.org/abs/2604.03986 （COBOL-Coder：GPT-4o 编译成功率 41.8%、pass@1 16.4；COBOL-Coder 73.95%/49.33）；https://arxiv.org/abs/2408.04660 （XMainframe/MainframeBench）

### 6.2 推荐路线（务实落地）

1. **先做「确定性事实层」**：用 Tree-Sitter/静态分析把 COBOL 代码库解析成「程序/段落/字段/文件/作业」清单与调用图——这是影响分析的骨架，**不依赖 LLM，可靠且便宜**。
2. **再做「语义检索层」**：段落级 + 字段级 chunk，BM25+向量混合 + rerank；给 chunk 前置「所属程序/数据段」语境（Contextual Retrieval 思路）。
3. **然后接 agent**：Claude Code / Agent SDK 做 orchestrator-workers；工具集 = 图谱查询 + 代码读取 + 编译验证；**固定编排优于完全 agentic**（§3.3 实证）。
4. **最后建 eval 与 HITL**：20-50 例黄金影响分析集做回归；输出强制带「文件:行号」证据 + 置信度；在需求卡、影响清单、变更后验证三处设人工确认点。
5. **数据字典/业务术语表**尽早接入（IBM 经验），作为「口述业务语言 ↔ 系统标识符」的权威映射。

---

## 7. 结论

- **可行**：用 LLM agent 对 COBOL 大型机代码库做「口述需求 → 影响分析 / 现行逻辑查询」在 2025-2026 已具备成熟技术组合，IBM 已将其产品化并有金融/社保案例。
- **关键原则**：**LLM 做语义、确定性工具做事实**。影响分析的事实（调用链、数据依赖、编译结果）必须来自静态分析/图谱/编译，LLM 负责需求理解、检索编排、结果解释与文档生成。
- **COBOL 的天然优势**：调用链/数据依赖静态可解析，比动态语言更适合「确定性影响分析」；PARAGRAPH/SECTION 是天然 chunk 边界，比自由格式代码更适合 RAG。
- **主要风险**：① 纯 LLM 影响分析不可靠（GPT-5 实证）；② COBOL 向量检索无成熟现成件；③ LLM 对 COBOL 生成正确性有限（需编译/符号执行兜底）；④ 置信度不可直接信（需黄金集校准）。
- **落地顺序**：事实层（静态分析/图谱）→ 检索层（RAG）→ agent 层（编排）→ 验证层（eval + HITL），每层独立可验收。

---

## 附录：来源清单（全部可溯源）

**官方文档 / 厂商公告**
1. Anthropic — Introducing Contextual Retrieval（2024-09）：https://www.anthropic.com/engineering/contextual-retrieval
2. Anthropic — Building effective agents（2024-12）：https://www.anthropic.com/engineering/building-effective-agents
3. Anthropic — Effective context engineering for AI agents（2025-09）：https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
4. Anthropic — Demystifying evals for AI agents（2026-01）：https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
5. Microsoft — GraphRAG 文档：https://microsoft.github.io/graphrag/ ；论文：https://arxiv.org/pdf/2404.16130
6. IBM — Agentic AI for smarter mainframe modernization with watsonx Code Assistant for Z（2025-12/2026-03）：https://www.ibm.com/new/announcements/agentic-ai-for-smarter-mainframe-modernization-with-ibm-watsonx-code-assistant-for-z
7. IBM — watsonx Code Assistant for Z 产品页（含 NOSI 案例）：https://www.ibm.com/products/watsonx-code-assistant-z
8. GitHub — CodeQL 文档（About CodeQL）：https://codeql.github.com/docs/codeql-overview/about-codeql/

**arXiv 论文**
9. CelloAI（代码 RAG/语法感知 chunking）：https://arxiv.org/abs/2508.16713
10. FaR-Loc（RAG 故障定位/代码 embedding）：https://arxiv.org/abs/2509.20552
11. 多智能体 COBOL 代码解释：https://arxiv.org/abs/2507.02182
12. Codebase-Memory（Tree-Sitter 知识图谱 + MCP）：https://arxiv.org/abs/2603.27277
13. CKGFuzzer（代码知识图谱 + LLM）：https://arxiv.org/abs/2411.11532
14. ProReFiCIA（需求变更影响分析）：https://arxiv.org/abs/2511.00262
15. GPT-5 代码变更影响分析：https://arxiv.org/abs/2512.19481
16. LLM-Augmented Release Intelligence：https://arxiv.org/abs/2603.14619
17. Deterministic vs LLM-Controlled Orchestration（COBOL→Python）：https://arxiv.org/abs/2605.09894
18. LLM-Based Acceptance Criteria Generation（Gherkin）：https://arxiv.org/abs/2607.01980
19. SimpleDevQA（置信度/过度自信）：https://arxiv.org/abs/2512.08867
20. Automated Testing of COBOL to Java Transformation（WCA4Z 验证）：https://arxiv.org/abs/2504.10548 ；https://arxiv.org/abs/2506.10999
21. COBOLAssist（编译反馈修复）：https://arxiv.org/abs/2604.03978
22. SEDCoT（符号执行 + delta debugging）：https://arxiv.org/abs/2607.04092
23. COBOL-Coder（领域模型）：https://arxiv.org/abs/2604.03986
24. XMainframe / MainframeBench：https://arxiv.org/abs/2408.04660
25. AGRAG（图 RAG，统计式构图防幻觉）：https://arxiv.org/abs/2511.05549
26. SubgraphRAG（子图检索）：https://arxiv.org/abs/2410.20724
27. LIDL（知识图谱 + 多智能体缺陷定位）：https://arxiv.org/abs/2601.05539

**说明**：Google Cloud 官方博客《From COBOL to Java》（2023-04，https://cloud.google.com/blog/products/application-development/from-cobol-to-java）是业界公认的「LLM 做 COBOL→Java 转换 + 人工在环」早期公开案例，本调研网络环境无法直接抓取正文，故仅作背景提及、未引用其具体数据。

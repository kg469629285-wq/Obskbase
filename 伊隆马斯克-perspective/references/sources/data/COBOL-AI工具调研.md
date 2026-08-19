# COBOL/CL 大型机核心系统 AI 辅助代码分析工具调研报告

> 调研日期：2026-08-19
> 场景：金融核心系统（COBOL + CL + DDS/DB2）代码库，目标是用 AI 实现「口述需求 → agent 分析出所有需要做的变更」与「口述需求 → 查清现行处理逻辑」
> 说明：本报告所有工具均可在文末「信息来源」找到可验证的官方/权威链接。成熟度标注：🟢 生产可用 / 🟡 早期或受限 / 🔴 概念演示或已停运。

---

## 0. 执行摘要（TL;DR）

2026 年 8 月，针对「COBOL/CL 大型机核心系统」的 AI 辅助代码分析，市场已从 2024 年的「演示阶段」进入「**生产可用 + Agentic（智能体）化**」阶段。核心结论：

1. **最贴合你两个目标（变更影响分析 + 现行逻辑理解）的现成产品**是 **IBM watsonx Code Assistant for Z（WCA4Z）2.8**——它内置了「Z Understand 元数据 + 影响分析 + 业务规则发现 + 文档生成」的 MCP 工具链，官方演示场景就是「口述需求（给 Db2 表加一列）→ agent 自动找出所有受影响程序并改代码」。
2. **AWS Transform for mainframe** 是 2025-2026 年最激进的竞品，主打「agentic AI + 全链路可追溯（每行生成代码可回溯到原始 COBOL 规则）」，已 GA。
3. **IBM i / AS/400 侧**（CL/DDS 生态）没有 IBM Z 那么强的第一方 AI 工具，但 **ARCAD V26（DISCOVER + MCP Server）** 和 **Profound Logic（CoderFlow）** 是两条务实路径。
4. **通用 LLM 工具（Copilot/Cursor/Cody/Continue）对 COBOL 是「文本级可用、语义级不可靠」**：能解释、能生成，但官方内联补全不支持 COBOL、CodeQL 完全不支持 COBOL、跨程序/跨 copybook 的精确影响分析必须靠专门的静态分析元数据（Z Understand / ARCAD / SonarQube AST）兜底。
5. **学术证据一致表明**：通用 LLM 直接翻译/理解 COBOL 的正确率偏低（GPT-4o 编译通过率约 41.8%），需要「领域微调模型（COBOL-Coder/XMainframe）+ 符号执行验证 + 静态分析元数据」组合才能达到生产可用。

---

## 1. IBM watsonx Code Assistant for Z（WCA4Z）及 IBM 相关 COBOL AI 工具

### 1.1 watsonx Code Assistant for Z（WCA4Z）

- **厂商**：IBM
- **官网**：https://www.ibm.com/products/watsonx-code-assistant-z
- **定位**：面向 IBM Z 大型机的端到端应用现代化开发助手（生成式 AI + 自动化）。
- **能力**（官方产品页逐条列出）：
  - **应用发现与分析**：AI agent 自动梳理大型机应用、关系与依赖。
  - **自然语言解释代码**：AI agent 生成代码的自然语言解释与技术文档。
  - **生成 COBOL 代码**：基于在 IBM Z 数据上微调的 LLM，支持聊天式生成 + IDE 内联补全。
  - **自动重构为模块化服务**：把 COBOL/PL/I 应用自动重构为业务服务。
  - **性能优化**：对 COBOL 模块做源码级性能分析（AI Optimizer for Z）。
  - **COBOL → Java 转换**：生成式 AI 把 COBOL 转成高质量 Java，并**自动生成单元测试做语义等价性验证**（对比新 Java 服务与原始 COBOL）。
- **2025-12 发布的 2.8 版（Agentic AI）**（IBM 官方公告，2025-12-16 发布、2026-03-02 更新）：
  - 官方示例正是你要的场景：*「I need to add a column to the Motor Policy Table that captures if the vehicle is an electric car. Can you help me update all of the programs with this field?」*（给保单表加一列，帮我更新所有用到该字段的程序）→ agent 自动**识别依赖、做影响分析、生成代码、套用编码规范、编译构建验证**。
  - **MCP 工具集**：Z Understand 元数据检索（全企业应用捕获）、**影响分析**（基于 Z Understand 静态分析元数据识别应用依赖）、代码生成与编码规范。
  - **文档生成 + 业务规则发现（Business Rule Discovery）**：面向大型复杂 Z 程序，做「代码感知的分段」以规避通用 LLM 分块（chunking）的常见缺陷；可注入企业数据字典/业务词汇表，让 LLM 用企业真实语义而非猜测变量含义。
- **定价/商业模式**：企业订阅制（需联系 IBM 销售，无公开标价）；配套 IBM Developer for z/OS 生态。
- **成熟度**：🟢 生产可用（有真实客户案例，见下）。
- **案例**（官方）：葡萄牙国家社会保险机构 NOSI——分析冗余 COBOL 代码耗时从 8 小时降到 30 分钟（-94%），理解复杂应用从 24 小时降到 5 小时（-79%）。
- **适用场景**：IBM Z（z/OS）上的 COBOL/PL/I 核心系统。**这是与你的目标最匹配的现成产品**。

### 1.2 IBM 相关配套工具

- **IBM Z Understand**：WCA4Z 背后的静态分析/元数据引擎（应用依赖、调用图），是影响分析的地基。
- **IBM Z Open Editor**：免费 VS Code 扩展，提供 COBOL/PL/I/HLASM/REXX/JCL 语言服务器（详见第 3 节），并内置 **MCP Server（Agent Mode）** 供 AI agent 读取 z/OS 数据集/文件/作业上下文。
- **IBM Bob（AI Coding Agent）**：IBM 2025-2026 推出的通用 AI 编码 agent（https://www.ibm.com/products/ai-coding-agent），与 WCA4Z 互补，但 COBOL 专项能力在 WCA4Z。

---

## 2. 商业迁移/现代化工具

### 2.1 AWS Mainframe Modernization + AWS Transform for mainframe

- **厂商**：Amazon Web Services
- **官网**：https://aws.amazon.com/mainframe-modernization/ ；https://aws.amazon.com/transform/mainframe/
- **AWS Mainframe Modernization（经典服务）**：
  - **Replatform（换平台不换语言）**：集成 **Micro Focus**、**NTT DATA**、**Rocket Software** 工具链，保留 COBOL/PL/I 语言、现代化基础设施与 DevOps。
  - **数据复制**：与 **Precisely** 合作做近实时数据复制。
  - **代码转换**：与 **mLogica** 合作把 Assembler 自动转 COBOL。
  - ⚠️ 注意：官方公告「AWS Mainframe Modernization Self-Managed Experience 不再对新客户开放」。
- **AWS Transform for mainframe（2025-2026 新旗舰，agentic AI）**：
  - 定位：用 agentic AI 把大型机应用「重新想象」为云原生系统，把多年迁移压缩到数月。
  - 全生命周期：**评估（依赖/复杂度/工作负载边界）→ 业务规则抽取（每条规则可追溯到源码）→ 需求生成 → 代码生成（可追溯）→ 自动化测试（功能等价性验证）→ 部署产物（IaC）**。
  - **可追溯性**是核心卖点：每行生成代码都能回溯到需求再到原始源码，形成审计链，证明「没有丢失或发明任何东西」。
  - 与 **Kiro**（AWS 编码 agent）端到端集成；通过 **MCP** 等开放协议接入现有 IDE/CI/CD。
  - 客户案例：Toyota Motor North America；合作伙伴：DXC、HCL、Infosys、Kyndryl、NRI、Pega、TCS、IBM、Deloitte。
  - 获 ISG Provider Lens 2025/2026 认可（AWS 在 Mainframe Application Modernization Software 象限被列为 Leader）。
- **定价**：AWS 按服务计费（Transform 有独立定价页 https://aws.amazon.com/transform/pricing/）。
- **成熟度**：🟢 生产可用（GA）。
- **适用场景**：想**离开**大型机、迁到 AWS 云原生（Java/微服务）的客户。注意：它面向「迁移」，不是「留在原地做日常变更影响分析」。

### 2.2 BMC（BMC AMI + AMI Assistant）

- **厂商**：BMC Software
- **官网**：https://www.bmc.com/it-solutions/mainframe.html
- **产品**：BMC AMI（Automated Mainframe Intelligence）组合——DevOps、AIOps、DataOps、SecOps、混合云数据保护。
- **AI 能力**：**BMC AMI Assistant**——内置对话式 AI，帮助大型机团队「troubleshoot、**解释代码**、用生成式 AI 和自然语言指导加速工作」（获 2025 AI Breakthrough 奖）。
- **行业认可**：BMC 被 Gartner 评为「AI-Augmented Code Modernization Tools」魔力象限 **Challenger**。
- **定价**：企业订阅制（需联系销售）。
- **成熟度**：🟢 生产可用（运维/DevOps 侧成熟；代码解释为辅助能力）。
- **适用场景**：大型机**运维、DevOps、变更管理**场景；代码解释是加分项，但**不是**以「口述需求→影响分析」为核心的产品。

### 2.3 Micro Focus / OpenText（COBOL 工具链）

- **厂商**：Micro Focus（2023 年 1 月被 **OpenText** 收购，COBOL 产品线现归 OpenText）。
- **官网**：https://www.microfocus.com/en-us/cobol（抓取被反爬拦截，以下基于 AWS 官方页面确认的合作伙伴关系 + 公开产品知识，标注为「知识性」）。
- **产品**：Micro Focus COBOL / Enterprise Developer / COBOL for Visual Studio / AMC（Application Modernization and Connectivity）——业界最成熟的 COBOL 编译器与开发工具链之一，支持在 Windows/Linux 上编译运行大型机 COBOL（replatform 主力）。
- **AI 角度**：OpenText 主推 Aviator（通用 AI 平台）；COBOL 侧更多是「工具链 + 迁移服务」，**没有**像 IBM WCA4Z 那样的一站式 COBOL AI agent 产品（截至 2026-08，未检索到官方 COBOL AI 专项产品页）。
- **成熟度**：🟢 工具链成熟；AI 能力 🟡。
- **适用场景**：COBOL 应用的**跨平台 replatform**、本地编译/测试、与 AWS/Azure 集成。

### 2.4 Rocket Software

- **厂商**：Rocket Software
- **官网**：https://www.rocketsoftware.com/products/rocket-software-for-mainframe（抓取被反爬拦截，403；以下基于 AWS 官方页面确认的合作伙伴关系 + 公开产品知识）。
- **产品**：Rocket Software for Mainframe（大型机软件组合：数据库、现代化、运维）；是 **AWS Mainframe Modernization 的 replatform 合作伙伴**（AWS 官方页面明确列出）。
- **AI 角度**：Rocket 近年宣传「Rocket AI / 大型机 AI 就绪」，但**未检索到**与 WCA4Z 对等的 COBOL 代码分析 AI 产品（截至 2026-08）。
- **成熟度**：🟢 工具链成熟；AI 能力 🟡。
- **适用场景**：大型机 replatform、数据库（如 Rocket DB2/IMS 工具）、现代化咨询。

### 2.5 Microsoft Azure（大型机迁移）

- **官网**：https://azure.microsoft.com/en-us/solutions/mainframe-migration/（重定向到通用迁移中心）。
- **现状**：**微软没有第一方 COBOL AI 分析产品**。Azure 的大型机迁移走**合作伙伴路线**（Micro Focus/OpenText、Heirloom Computing、TmaxSoft 等）+ **Azure Logic Apps**（大型机与云集成）。
- **2025-2026 通用 AI 能力**（通用迁移中心页面确认）：**Azure Copilot migration agent**（迁移各阶段 AI 辅助）、**GitHub Copilot modernization**（批量评估、应用规划、自动升级、部署的 agent）。这些是**通用**能力，不针对 COBOL 语义。
- **成熟度**：🟡（大型机专项 AI 依赖合作伙伴）。
- **适用场景**：把大型机工作负载迁到 Azure 的客户；COBOL 语义分析需叠加第三方工具。

---

## 3. 开源 / IDE 级 COBOL 支持（语义理解 / 代码导航 / 检索）

### 3.1 SonarQube COBOL（SonarSource）

- **官网**：https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/languages/cobol.md
- **能力**（官方文档确认）：
  - COBOL 分析在 **Enterprise Edition（商业版）** 提供；支持 SonarQube for VS Code / Eclipse 连接模式。
  - **方言**：bull-gcos、hp-tandem、**ibm-os/vs-cobol、ibm-ile-cobol、ibm-cobol/ii、ibm-cobol/400、ibm-enterprise-cobol**、microfocus-cobol、microfocus-acucobol-gt、opencobol/cobol-it。
  - 固定/自由/可变格式；**copybook 支持**（`sonar.cobol.copy.directories`）；**DB2 目录支持**（通过 CSV 导入 SYSIBM 目录，做嵌入式 SQL 规则）。
  - **自定义规则**：基于 AST（`CobolCheck`），可订阅语法节点写规则。
- **定位**：**静态代码质量/安全分析**（bug、坏味道、漏洞），**不是** AI 语义理解或影响分析。但它的 AST 解析能力可作为「精确解析 COBOL + copybook + 嵌入式 SQL」的地基。
- **成熟度**：🟢 生产可用（商业插件，多年成熟）。
- **适用场景**：CI 质量门禁、代码规范、安全扫描；可作为自建 AI 分析管线的「解析层」。

### 3.2 Che4z LSP for COBOL / Code4z（Broadcom）

- **厂商**：Eclipse Che4z 开源项目（现归 **Broadcom Code4z** 生态）。
- **仓库**：https://github.com/eclipse-che4z/che-che4z-lsp-for-cobol（117 stars，活跃维护，2026-08 仍在更新）。
- **能力**（README 确认）：
  - 自动补全（关键字/变量/段落/CICS/DB2/Datacom SQL/copybook 变量/子程序名）。
  - **语法 + 语义检查**（语义分析能标出错误的变量/copybook/段落名）。
  - **Go To Definition / Find All References**（含 copybook 内、子程序间）。
  - **copybook 解析**：本地目录、远程 z/OS 数据集（Zowe）、TAR、Endevor 处理器组。
  - 支持 IBM Enterprise COBOL + CICS + DB2 SQL + Datacom；IDMS 方言有独立 add-on。
- **定位**：IDE 级「语义理解/代码导航/检索」——**这是开源侧对 COBOL 语义支持最强的 LSP**。但它**不做** AI 生成或跨应用影响分析。
- **成熟度**：🟢 生产可用（开源，广泛用于 VS Code）。
- **适用场景**：给开发者提供 COBOL 代码导航/检索；可作为自建 AI 管线的「语义索引层」。

### 3.3 IBM Z Open Editor

- **官网**：https://marketplace.visualstudio.com/items?itemName=IBM.zopeneditor（199,256 次安装，免费）。
- **能力**（Marketplace 页确认）：
  - 免费核心：COBOL v6.5 / PL/I / HLASM / REXX / JCL 语言服务器；实时语法检查、Problems 视图、大纲视图、**Go to definition / Find all references / Peek definition**、copybook 预览、重命名重构、200+ 代码片段。
  - **高级能力（需 IBM Developer for z/OS Enterprise Edition / Select / ADFz 许可，90 天试用）**：
    - **Program Control Flow Browser**（COBOL/PL/I 程序控制流交互图）
    - **Data flow graph browser for COBOL**（数据元素如何被填充/修改/写入的数据流图）
    - **ZCodeScan**（COBOL 最佳实践违规 + 安全漏洞扫描）
    - **Agent Mode：MCP Server**——把 z/OS 数据集/文件/作业上下文通过 Zowe API 提供给 AI agent 聊天面板。
- **成熟度**：🟢 生产可用（免费核心 + 商业高级版）。
- **适用场景**：开发者日常 COBOL 编辑/导航；**控制流/数据流图**对「理解现行处理逻辑」极有价值；MCP Server 可喂给自建 agent。

---

## 4. 通用代码库检索/理解工具用于 COBOL 的可行性

### 4.1 GitHub Copilot / Copilot Chat

- **官方文档**：https://docs.github.com/en/copilot/using-github-copilot/getting-started-with-github-copilot
- **现状**：官方表述为「为众多语言提供建议……在 Python、JavaScript、TypeScript、Ruby、Go、C#、C++ 上尤其出色」。**COBOL 不在官方内联补全（inline suggestions）支持语言列表**。
- **但**：Copilot Chat 是 LLM 驱动的，**可以**对 COBOL 文本做解释、问答、生成（把 COBOL 当普通文本）。质量取决于模型对 COBOL 的熟悉度（见第 5 节论文证据：通用模型 COBOL 正确率偏低）。
- **成熟度**：🟡（Chat 可用，内联补全不支持 COBOL）。
- **适用场景**：单文件解释、小段代码问答；**不适合**跨程序影响分析。

### 4.2 Sourcegraph / Cody / Deep Search

- **官网**：https://sourcegraph.com/docs/
- **能力**：Code Search（全仓库/全分支文本检索）、**Deep Search**（自然语言问答的 AI agent）、**Cody**（AI 编码助手）、Code Intelligence（跳转定义/找引用，依赖语言服务器）。
- **COBOL 现状**：Sourcegraph 的**文本检索**对 COBOL 完全可用（按字符串/正则搜全库）；但**精确代码导航（go-to-definition）依赖语言服务器**，COBOL 不在其主流语言服务器支持内（需自配 LSP）。Cody/Deep Search 是 LLM 驱动，可分析 COBOL 文本。
- **成熟度**：🟡（检索可用；语义导航受限）。
- **适用场景**：全库检索「某个字段/程序被谁引用」的**粗粒度**答案；配合 Che4z/Z Open Editor 的 LSP 可补精确导航。

### 4.3 Cursor

- **官网**：https://cursor.com/docs
- **能力**：语言无关的 agent 型编码工具；支持 MCP、Rules、Skills、Subagents；模型上下文最高 1M token（Claude/GPT/Gemini 等）。
- **COBOL 现状**：**无原生 COBOL LSP**，但作为 agent 可读 COBOL 文本、调用 MCP 工具（如 Z Open Editor 的 MCP、ARCAD 的 MCP）做分析。1M 上下文对「把整个程序/多个程序塞进上下文」有帮助。
- **成熟度**：🟡（agent 框架成熟，COBOL 语义靠模型 + 外部 MCP）。
- **适用场景**：自建「口述需求 → agent 分析」工作流的**宿主 IDE/agent 框架**。

### 4.4 Continue（开源）

- **官网**：https://continue.dev/（抓取超时；以下为公开产品知识）
- **能力**：开源 AI 编码助手（VS Code/JetBrains），语言无关（LLM 驱动），支持自定义模型、MCP、本地部署。
- **COBOL 现状**：同 Cursor——无原生 COBOL 支持，靠 LLM 文本理解 + 外部工具。
- **成熟度**：🟡（框架成熟，COBOL 语义靠模型）。
- **适用场景**：需要**本地/私有化部署**、可定制 agent 管线的团队。

### 4.5 CodeQL（GitHub）

- **官网**：https://codeql.github.com/docs/codeql-overview/supported-languages-and-frameworks/
- **现状**：**官方支持语言列表不含 COBOL**（支持 C/C++、C#、Go、Java、Kotlin、JavaScript/TypeScript、Python、Ruby、Rust、Swift、GitHub Actions）。**COBOL 完全不支持**。
- **成熟度**：🔴（对 COBOL 不可用）。
- **适用场景**：不适用 COBOL；若未来迁移到 Java，CodeQL 可用于新代码安全分析。

---

## 5. LLM 直接分析 COBOL 的做法（论文与案例）

arXiv 检索（2024-2026）确认了大量真实研究，核心结论：**通用 LLM 对 COBOL 是「低资源语言」，直接使用正确率不足，需要领域微调 + 符号执行验证 + 静态分析元数据**。

| 论文 | 时间 | 核心发现 |
|---|---|---|
| **XMainframe**（arXiv:2408.04660） | 2024-08 | 面向大型机/COBOL 的专用 LLM；选择题准确率比 DeepSeek-Coder 高 30%，COBOL 摘要 BLEU 是 GPT-3.5 的 6 倍 |
| **Automated Testing of COBOL→Java**（arXiv:2504.10548） | 2025-04 | **IBM WCA4Z 的官方测试框架**：符号执行生成 COBOL 单测 → 转 JUnit 验证与 Java 的语义等价 |
| **Automated Validation of COBOL→Java**（arXiv:2506.10999） | 2025-06 | 同上框架的验证/修复闭环 |
| **Enhancing COBOL Code Explanations: Multi-Agents**（arXiv:2507.02182） | 2025-07 | **多 agent 协作解释 COBOL**（函数/文件/项目级）；指出 COBOL 常超 LLM token 窗口，需代码库上下文注入；函数级解释 METEOR 提升 12.67% |
| **REFINE**（arXiv:2508.02827） | 2025-08 | IBM 内部用 LLM-as-Judge 评估 COBOL 生成/翻译/摘要；对齐分数从 <0.7 提升到 >0.9 |
| **EvoGraph**（arXiv:2508.05199） | 2025-08 | 图演化框架，COBOL→Java 达 93% 功能等价（测试验证） |
| **Beyond Blind Spots**（arXiv:2512.16272） | 2025-12 | **LLM-as-Judge 只能发现 COBOL 代码中 45-63% 的错误**；需「分析性检查器 + 提示注入」混合，最高到 74% |
| **Vintage Code, Modern Judges**（arXiv:2510.27244） | 2025-10 | SparseAlign：在稀疏人工标注下验证 LLM 评估器（用于 COBOL 代码解释） |
| **Perturbation robustness**（arXiv:2511.18488） | 2025-11 | LLM 系统对 COBOL 输入的微小扰动敏感（非鲁棒） |
| **Long-context code QA**（arXiv:2602.17183） | 2026-02 | 长上下文 COBOL 问答在干扰项/格式变化下性能大幅下降 |
| **COBOLAssist**（arXiv:2604.03978） | 2026-04 | 用编译反馈迭代修复 LLM 生成的 COBOL；GPT-4o 编译通过率 41.8%→95.89% |
| **COBOL-Coder**（arXiv:2604.03986） | 2026-04 | **领域微调 COBOL LLM**：COBOLEval 编译通过率 73.95%、Pass@1 49.33（GPT-4o 仅 41.8%/16.4）；Java→COBOL Pass@1 34.93（通用 LLM 接近 0）；开发者调研认为其更符合企业实践 |
| **Deterministic vs LLM-Controlled Orchestration**（arXiv:2605.09894） | 2026-05 | COBOL→Python：**确定性编排**比完全 agentic 更稳、token 省 3.5 倍、质量不降 |
| **SEDCoT**（arXiv:2607.04092） | 2026-07 | COBOL→C：LLM 初译 + 符号执行生成测试 + delta debugging 修复，比 SOTA 高 12% |

**对「口述需求→查清现行处理逻辑」的直接启示**：
- 多 agent + 代码库上下文注入（arXiv:2507.02182）证明：**把整个代码库/相关程序上下文喂给 LLM 能显著提升解释质量**——这正是「RAG/上下文工程」路线。
- 但 COBOL 程序常超 token 窗口 → 需要**代码感知的分段**（IBM WCA4Z 的 Business Rule Discovery 正是为此设计）。
- 通用 LLM 的 COBOL 输出**必须验证**（编译 + 符号执行 + 等价性测试），不能直接信任。

---

## 6. IBM i / AS/400 特定工具（CL/DDS 生态）

> 注意：你的代码库是「COBOL + CL + DDS/DB2」。CL（Control Language）和 DDS（Data Description Specifications）是 **IBM i（AS/400）专属**语言。IBM Z 的 AI 工具（WCA4Z）**不覆盖 IBM i**。以下为 IBM i 生态的 AI 工具。

### 6.1 ARCAD Software（DISCOVER + V26 MCP Server）—— 最相关

- **官网**：https://www.arcadsoftware.com/
- **产品**：
  - **DISCOVER**：AI 驱动的应用分析工具（application intelligence）——自然语言宏观视图、**自动依赖映射**、AI 助手分析应用、重建应用文档（应对「知识流失/退休」）。
  - **ARCAD V26（2026 发布）**：**MCP Server**——「把 AI 助手连接到 ARCAD 元数据仓库，提供**可靠的影响分析**、加速现代化」。这是 IBM i 侧**最接近你「口述需求→影响分析」目标**的现成能力。
  - 其他：ARCAD CodeChecker（源码质量/安全）、ARCAD Transformer（RPG 现代化）、ARCAD Verifier/iUnit（回归测试/单元测试）、DROPS（发布管理）、DOT（数据脱敏）。
- **客户**：480+ 客户 / 75 国，含 **ING Bank、HSBC**、Orange、法国国防部。
- **成熟度**：🟢 生产可用（DevOps/现代化工具成熟；AI/MCP 为 2026 新增）。
- **适用场景**：IBM i 上的 RPG/COBOL/CL 应用——**影响分析、依赖映射、文档重建**。

### 6.2 Profound Logic（CoderFlow + Profound AI）

- **官网**：https://www.profoundlogic.com/
- **产品**：Profound AppDev（含 Profound.js、Profound UI）、**CoderFlow**（agentic coding）、Profound API；服务含 **Agentic Coding Implementation**、**Code Conversion**、**Profound Discovery & Analysis**。
- **AI**：**Profound AI**（25 周年免费赠品，面向 IBM i 社区）；博客明确主张「通用 AI（ChatGPT）对 IBM i 企业系统不够用」。
- **定位**：IBM i「futurization」——把 RPG/COBOL 应用现代化为 Node.js/Python/COBOL，保留业务逻辑；AI 增强。
- **成熟度**：🟢（现代化服务成熟；AI 为 2025-2026 新增）。
- **适用场景**：IBM i 应用现代化/转换、agentic coding 实施、AI 使能。

### 6.3 Razza

- **现状**：**razza.com 域名已挂出售页（for sale）**——该公司（曾做 IBM i 现代化）疑似已停运/出售。**不要作为选型对象**。
- **成熟度**：🔴（停运迹象）。

### 6.4 i-UG（IBM i 用户组社区）

- **官网**：https://i-ug.org/（抓取超时；为 IBM i 社区组织）。
- **定位**：社区/知识分享（含 AI 在 IBM i 上的实践讨论），非商业工具。可作为情报来源。

### 6.5 CL / DDS 语言在主流 LLM 分析工具里的支持度

- **现状**：CL 和 DDS 在主流工具中**基本无原生支持**——没有官方 LSP、没有语法高亮（多数 IDE 需自定义）、CodeQL 不支持、GitHub Copilot 内联补全不支持。
- **LLM 文本级**：Claude/GPT 等**能读懂** CL/DDS 文本（训练数据含 IBM i 内容），可做解释/生成，但**无静态分析兜底**，跨程序影响分析不可靠。
- **务实路径**：CL/DDS 的精确分析依赖 **IBM i 生态工具**（ARCAD 元数据仓库、IBM Rational Developer for i / RDi 的交叉引用）或**自建解析器**（把 CL/DDS 解析成 AST/调用图喂给 LLM）。

---

## 7. 局限性与风险汇总（每种工具的坑）

| 工具/路线 | 主要局限 |
|---|---|
| **WCA4Z** | 仅 IBM Z（z/OS），**不覆盖 IBM i/CL**；企业订阅贵、需 IBM 实施；生成代码仍需人工评审（官方也强调用自动测试验证等价性） |
| **AWS Transform** | 面向「迁移离场」，不是「留在原地做日常变更分析」；绑定 AWS 云；agentic 输出仍需人工验证 |
| **BMC AMI Assistant** | 代码解释是辅助，核心是运维/DevOps；不做深度影响分析 |
| **Micro Focus/OpenText、Rocket** | 工具链成熟但**无一站式 COBOL AI agent**；AI 能力需自行拼装 |
| **Azure** | 无第一方 COBOL AI；依赖合作伙伴 |
| **SonarQube COBOL** | 只做静态质量/安全，**不做**语义理解/影响分析；需 Enterprise 版付费 |
| **Che4z / Z Open Editor** | 只做 IDE 级导航/检索，**不做** AI 生成或跨应用影响分析；Z Open Editor 高级功能（控制流/数据流图）需商业许可 |
| **Copilot/Cursor/Cody/Continue** | COBOL 无官方内联补全/无 LSP；**幻觉风险高**（论文：LLM-as-Judge 漏检 45-63% 错误）；跨程序影响分析不可靠；需人工核对 |
| **CodeQL** | **完全不支持 COBOL** |
| **通用 LLM 直接分析** | 编译通过率低（GPT-4o ~41.8%）；对输入扰动敏感；长上下文易被干扰项带偏；**必须**用编译/符号执行/等价性测试验证 |
| **CL/DDS** | 主流工具无原生支持；精确分析依赖 IBM i 生态工具或自建解析器 |
| **Razza** | 疑似停运（域名出售） |

**通用风险提示**：
1. **幻觉风险**：LLM 会「编造」不存在的程序/字段/依赖。银行核心系统**任何 AI 输出都必须有静态分析元数据（调用图/数据流）交叉验证**。
2. **老代码适配度**：几十年的 COBOL 常含方言扩展、copybook 嵌套、嵌入式 SQL、CICS/IMS 宏、预处理器——通用 LLM 和通用工具对这类「非标准」代码解析率低；**方言感知的解析器（SonarQube/Che4z/Z Open Editor/ARCAD）是必要地基**。
3. **人工投入**：即便用 WCA4Z/AWS Transform，官方流程都保留「人工评审 + 自动等价性测试」环节。AI 是「加速器」不是「无人驾驶」。

---

## 8. 结论：2026 年银行核心 COBOL/CL 代码库最务实的技术栈组合

假设：代码库 = **COBOL + CL + DDS/DB2**，目标 = 「口述需求 → agent 分析出所有需要做的变更」+「口述需求 → 查清现行处理逻辑」，且**大概率是 IBM i（AS/400）或 IBM Z 之一**。分两种情况给结论：

### 情况 A：IBM Z（z/OS）上的 COBOL + DB2（最主流银行核心形态）

**首选组合（开箱即用，最贴合目标）**：
1. **IBM watsonx Code Assistant for Z（WCA4Z）2.8** —— 唯一把「口述需求 → 影响分析 → 改代码 → 验证」做成产品化 agent 流程的工具（官方示例就是加表字段场景）。用它做「变更影响分析 + 业务规则发现 + 文档生成」。
2. **IBM Z Open Editor（免费）+ 高级版控制流/数据流图** —— 开发者日常导航 + 可视化「现行处理逻辑」。
3. **SonarQube COBOL（Enterprise）** —— CI 质量/安全门禁，作为解析层兜底。
4. **自建 agent 增强**：用 **Z Open Editor 的 MCP Server / WCA4Z 的 MCP 工具**把静态分析元数据喂给通用 agent（Cursor/Claude），实现「口述需求 → 检索 → 影响清单 → 人工确认」。

### 情况 B：IBM i（AS/400）上的 COBOL + CL + DDS（你明确提到 CL/DDS，很可能在此）

**首选组合**：
1. **ARCAD DISCOVER + ARCAD V26 MCP Server** —— 目前 IBM i 侧**唯一**把「AI 助手 + 元数据仓库 + 可靠影响分析」产品化的组合；DISCOVER 做依赖映射/文档重建，MCP 喂给通用 agent。
2. **Profound Logic（CoderFlow / Profound Discovery & Analysis）** —— agentic coding 实施 + 应用分析服务，作为补充或替代。
3. **IBM Rational Developer for i（RDi）** 的交叉引用/调用图 —— 作为 CL/DDS 的精确静态分析地基（RDi 是 IBM i 官方 IDE，交叉引用成熟）。
4. **自建 CL/DDS 解析层**：CL/DDS 无现成 LSP，若影响分析精度要求高，需把 CL/DDS 解析成调用图/数据字典，再喂给 LLM（可参考 SonarQube 的 AST 思路）。

### 跨平台通用底座（无论 A/B）
- **检索层**：Sourcegraph（全库文本检索）或自建向量库（RAG），解决「这个字段/程序被谁引用」的粗粒度问题。
- **Agent 宿主**：Cursor / Continue（开源可私有化）/ Claude 等，接 MCP 工具。
- **验证层（必须）**：编译 + 符号执行 + 等价性测试（参考 IBM WCA4Z 测试框架论文 arXiv:2504.10548 的做法），**任何 AI 生成的变更都要过这一关**。
- **领域模型**：若预算允许，可评估 **COBOL-Coder / XMainframe** 这类领域微调模型（arXiv:2604.03986 / 2408.04660）替代通用模型做 COBOL 生成/翻译。

### 一句话结论
> **2026 年最务实组合 = 一个「静态分析元数据引擎」（IBM Z 用 WCA4Z/Z Understand，IBM i 用 ARCAD DISCOVER）+ 一个「MCP 化的 agent 宿主」（Cursor/Claude/自建）+ 一个「方言感知的解析/质量层」（SonarQube COBOL / Che4z / Z Open Editor）+ 一个「强制验证层」（编译 + 符号执行等价性测试）。** 通用 LLM（Copilot/Cursor 等）只做「解释与草稿」，**绝不做**未经验证的跨程序影响分析结论。

---

## 附录：信息来源（可验证链接）

**IBM**
- WCA4Z 产品页：https://www.ibm.com/products/watsonx-code-assistant-z
- WCA4Z 2.8 Agentic AI 公告（2025-12/2026-03）：https://www.ibm.com/new/announcements/agentic-ai-for-smarter-mainframe-modernization-with-ibm-watsonx-code-assistant-for-z
- IBM Z Open Editor（VS Code Marketplace）：https://marketplace.visualstudio.com/items?itemName=IBM.zopeneditor
- IBM Bob（AI Coding Agent）：https://www.ibm.com/products/ai-coding-agent

**AWS**
- AWS Mainframe Modernization：https://aws.amazon.com/mainframe-modernization/
- AWS Transform for mainframe：https://aws.amazon.com/transform/mainframe/

**BMC**
- BMC AMI：https://www.bmc.com/it-solutions/mainframe.html
- BMC AMI Assistant / AI Breakthrough：https://www.bmc.com/it-solutions/mainframe-ai.html

**SonarSource**
- SonarQube COBOL 文档：https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/languages/cobol.md

**开源/IDE**
- Che4z LSP for COBOL：https://github.com/eclipse-che4z/che-che4z-lsp-for-cobol
- Code4z（Broadcom）：https://techdocs.broadcom.com/code4z

**通用工具**
- GitHub Copilot 快速上手（语言支持表述）：https://docs.github.com/en/copilot/using-github-copilot/getting-started-with-github-copilot
- CodeQL 支持语言（无 COBOL）：https://codeql.github.com/docs/codeql-overview/supported-languages-and-frameworks/
- Sourcegraph 文档：https://sourcegraph.com/docs/
- Cursor 文档：https://cursor.com/docs/
- Continue：https://continue.dev/

**IBM i 生态**
- ARCAD Software：https://www.arcadsoftware.com/（DISCOVER：https://www.arcadsoftware.com/discover/ ；V26 MCP：https://www.arcadsoftware.com/arcad/arcad-mcp-server-ibm-i-application-context-for-ai-agents/）
- Profound Logic：https://www.profoundlogic.com/
- Razza：https://www.razza.com/（域名出售中）
- i-UG：https://i-ug.org/

**学术论文（arXiv）**
- XMainframe：https://arxiv.org/abs/2408.04660
- COBOL-Coder：https://arxiv.org/abs/2604.03986
- COBOLAssist：https://arxiv.org/abs/2604.03978
- 多 Agent COBOL 代码解释：https://arxiv.org/abs/2507.02182
- IBM WCA4Z COBOL→Java 测试框架：https://arxiv.org/abs/2504.10548
- COBOL→Java 验证：https://arxiv.org/abs/2506.10999
- Beyond Blind Spots（LLM-as-Judge 漏检）：https://arxiv.org/abs/2512.16272
- 确定性 vs LLM 编排：https://arxiv.org/abs/2605.09894
- SEDCoT（COBOL→C）：https://arxiv.org/abs/2607.04092
- 长上下文 COBOL QA 鲁棒性：https://arxiv.org/abs/2602.17183

> 调研说明：Exa 搜索 API 全程限流，本报告主要依赖 webfetch 直接抓取官方页面/文档/arXiv 验证；Micro Focus/OpenText 与 Rocket Software 官网反爬（444/403），其信息以 AWS 官方页面确认的合作伙伴关系 + 公开产品知识为准，已在正文标注。所有「成熟度」标注基于官方页面表述与论文证据，未将概念演示当作生产可用。
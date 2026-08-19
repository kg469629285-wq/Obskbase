# IBM i（AS/400）平台 COBOL/CL/RPG 代码库 AI 辅助分析工具与产品现状调研（2025–2026）

> 调研日期：2026-08-19
> 目标场景：银行核心系统运行于 IBM i（AS/400），代码为 COBOL + CL + DDS/DB2 for i，目标用 AI 实现「口述需求 → 变更影响分析」与「口述需求 → 查清现行处理逻辑」。
> 方法说明：本报告基于官方产品页、官方 GitHub 仓库、IBM 官方文档及 IBM i 权威媒体（IT Jungle、AI News/TechForge）的公开信息交叉核验。凡无法在线核验的条目均明确标注「未能在线核验」。厂商宣传不作为独立验证，仅作参考并注明来源。

---

## 0. 结论速览（TL;DR）

| 需求 | 最务实选择 | 理由 |
|---|---|---|
| 确定性影响分析事实层 | **ARCAD DISCOVER**（或 Fresche X-Analysis） | 40 年确定性交叉引用/依赖仓库，零幻觉，天然适合「改 A 会影响谁」 |
| AI 对话/口述需求入口 | **IBM Bob Premium Package for i** 或 **Claude/Cursor + MCP** | Bob 原生连 IBM i、内置 explain/document 技能；通用 agent 走 MCP 桥接 |
| 把 IBM i 系统数据暴露给 AI | **IBM 官方 ibmi-mcp-server**（开源）+ **ARCAD MCP Server**（商业，70+ 工具） | 官方开源 MCP 已生产可用，支持 Claude/Cursor/VSCode 等 10+ 客户端 |
| 开发者侧确定性交叉引用 | **RDi**（IBM Rational Developer for i） | 成熟 Eclipse IDE，含分析/重构能力，可作人工复核层 |

**一句话结论**：2025–2026 年 IBM i 的 AI 代码分析已从「概念验证」进入「生产可用」阶段，但**没有任何单一工具能同时做到「确定性影响分析」+「自然语言查逻辑」**。最务实的组合是：**确定性仓库（ARCAD DISCOVER / X-Analysis）+ 官方开源 MCP（ibmi-mcp-server）+ 通用 AI agent（Claude/Cursor 或 IBM Bob）**，形成「确定性事实层 → MCP 桥接层 → LLM 对话层」三层架构。

---

## 1. IBM 官方是否有 IBM i 专属 AI 代码工具？

### 1.1 结论：有，但产品名不是「watsonx Code Assistant for i」

**「watsonx Code Assistant for i」作为独立在售产品名：未检索到。** 实际情况是：

1. **watsonx Code Assistant for RPG**（IBM i 专属）：IBM 罗切斯特实验室基于自研 Granite 模型开发，2025 年 5 月左右进入公开预览（IT Jungle 2025-05-21《Public Preview For Watson Code Assistant for i Available Soon》）。
2. **2025 年 10 月 TechXchange 大会**：IBM 将 IBM i 的 RPG 助手与 System Z 的 COBOL 助手**合并为「Project Bob」**（IT Jungle 2025-10-13《Big Blue Converges IBM i RPG And System Z COBOL Code Assistants Into "Project Bob"》）。
3. **IBM Bob 1.0 于 2026 年 3 月正式 GA**（IT Jungle 2026-03-02《IBM Gets Bob 1.0 Off The Ground》）。
4. **当前 IBM i 专属产品 = IBM Bob Premium Package for i**（IBM Bob 的 IBM i 增值包）。

> 证据（IT Jungle，2025-10-13）：IBM 在 TechX25 展示统一代码助手「Project Bob」，将此前两个「largely uncommercialized」的 watsonx Code Assistant（RPG 版与 COBOL 版）收敛合并；RPG II/III/ILE→自由格式 RPG、COBOL→Java 的训练成果被保留。
> 证据（AI News/TechForge，2026-04-28）：IBM 正式发布 Bob，并明确「IBM guarantees that current watsonx Code Assistant customers will maintain full support while they map out their adoption path to the new system」——即 Bob 是 watsonx Code Assistant 的继任者。

### 1.2 IBM Bob Premium Package for i 的能力（官方页面）

来源：https://www.ibm.com/products/ai-coding-agent/ibm-i-modernization

- **原生连接 IBM i**：通过 SSH 安全连接，可读/写代码、编译程序、执行 SQL、支持测试工作流。
- **预置技能（Pre-built skills）与 agentic 工作流**：explain（解释）、document（文档化）、refactor（重构）、code（编码）、transform（转换）、test（测试）；覆盖固定格式→自由格式 RPG、RPG II/III→ILE RPG、单体→模块化重构、嵌入式 SQL 现代化、DDS→DDL 转换。
- **IBM i 数据库模式**：SQL Index Strategy Advisor（索引策略顾问）、生成实体关系图（ER diagrams）。
- **定价**：个人版约 **USD 40/授权用户/月**（作为 IBM Bob 的增值包 add-on）。
- **客户案例**：Jack Henry（银行/金融科技）用其理解复杂 RPG 程序、自动生成技术文档、对比程序版本差异（IBM 官方客户故事 PDF，2026-05-15）。
- **底层模型**：多模型编排——Anthropic Claude、Meta Llama、Mistral、IBM Granite（AI News 2026-04-28；IT Jungle 2026-04-06）。
- **规模证据**：8 万+ IBM 员工使用 Bob，内部报告平均生产力提升 45%；APIS IT 用其将 20 年 EGL/CIS 系统文档化提速 10 倍、JCL/PL/I 准确率 100%（AI News 2026-04-28）。

### 1.3 与 WCA4Z（IBM Z 版）的区别

- **WCA4Z（watsonx Code Assistant for Z）**：面向 System Z 主机的 COBOL→Java 转换，2023 年 8 月预览。
- **IBM i 版**：面向 IBM i 的 RPG/COBOL/CL，重点是 RPG 现代化（固定格式→自由格式、RPG II/III→ILE）与 IBM i 原生操作。
- **合并后**：两者都并入 IBM Bob，Bob 提供统一 IDE + 多模型编排 + 治理（审批门、审计、成本路由）。IBM i 版通过「Premium Package for i」提供平台专属技能；Z 版通过「Premium Package for Z」提供主框架专属能力（IBM 官方页面）。

**成熟度：🟢 生产可用（GA 于 2026-03，但产品较新，生态仍在快速迭代）**

---

## 2. ARCAD Software（DISCOVER + V26 MCP Server）

### 2.1 DISCOVER —— AI 应用分析助手

来源：https://www.arcadsoftware.com/discover/ 及 https://www.arcadsoftware.com/discover/features/

- **定位**：AI 驱动的应用分析工具，基于 ARCAD 40 年积累的**确定性元数据仓库**（组件、字段、过程、字面量、依赖关系）。
- **能力**：
  - 自然语言提问应用（Conversational AI），输出文本+图表；
  - 自动生成功能架构（functional architecture）；
  - 识别程序/文件/模块间链接（交叉引用）；
  - 检测过时组件、冗余、不一致（Health Check）；
  - 度量复杂度、体量、注释率；
  - **数据血缘（data lineage）**：追踪数据在应用内的完整流转路径；
  - **多技术**：整合 IBM i、Java 等系统的交叉引用，映射混合应用。
- **部署与安全**：内置开源 LLM、**纯本地部署（无云依赖）、不暴露源码、GDPR 合规**。
- **客户**：Orange、AXA、FirstRand Bank、Cynergy Bank、Belfius、ING、HSBC 等（官方客户墙）。
- **权威媒体佐证**：IT Jungle 2025-02-10《ARCAD Discover: Global Application Analysis With An AI Interface》。

### 2.2 V26 与 ARCAD MCP Server（2026 年春季发布）

来源：https://www.arcadsoftware.com/arcad/arcad-mcp-server-ibm-i-application-context-for-ai-agents/

- **V26.0 已发布**（2026），官方口号「AI becomes genuinely useful in your environments」。
- **ARCAD MCP Server**：通过 MCP 协议暴露 ARCAD 元数据仓库，**70+ 个 MCP 工具**（resources、actions、methodological prompts）。
- **确定性优先**：注入 AI 的数据来自确定性算法而非 LLM 推断，「Zero hallucination」；只传输当前任务所需信息，完整源码不出环境。
- **兼容**：VS Code + GitHub Copilot、Bob（IBM i）、Claude、Make/N8N、本地 on-premise LLM；多 IDE、多 agent。
- **平台覆盖**：IBM i 先行，z/OS 后续（「IBM i today, z/OS tomorrow」）。
- **更新节奏**：每季度新增一批 MCP 工具，逐步覆盖 ARCAD 全产品线（DROPS、DISCOVER、DOT…）。
- **与 IBM Bob 集成**：ARCAD 正在推出 IBM Bob 插件（IT Jungle 2026-04-06）。
- **CEO 观点**（Philippe Magne，IT Jungle 2026-02-09 赞助文）：「AI 不能替代一切，是强大助手，不能完全替代人类专业判断」——务实定位。

**成熟度：DISCOVER 🟢 生产可用（成熟）；ARCAD MCP Server 🟡 早期（2026 春季发布，季度迭代中）**

---

## 3. Profound Logic（CoderFlow / Profound AI / Discovery & Analysis）

### 3.1 CoderFlow —— Agentic Coding 平台

来源：https://www.profoundlogic.com/coderflow/ 及 /coderflow/how-it-works/

- **定位**：面向复杂企业系统的自主 AI 开发平台（agentic coding），非简单聊天式助手。
- **能力**：
  - 自主执行完整「构建→测试→修复→再测试」循环直至验证通过；
  - 多 agent 并行探索、自动评判（judges）对比方案、选择验证通过的方案；
  - **支持 IBM i、RPG、COBOL、5250、富显示文件**，也支持 Node.js/Java/Python/.NET；
  - 多仓库编排、Git/JIRA/CI-CD 集成；
  - 模板覆盖转换、重构、文档化。
- **部署与安全**：本地/私有云部署，全部执行留在内网；仅最小上下文发给云端 LLM（Claude、OpenAI、Gemini）；模型不直接接触 IBM i/仓库/数据库；最小权限、审计、审批门。
- **宣称收益**：5–10 倍生产力（IT Jungle 2026-01-19《Profound Says New Agentic AI Dev Tool Delivers Huge Productivity Boost》，CEO Alex Roytman 语）。
- **注意**：5–10x 为厂商宣称，未见独立第三方基准。

### 3.2 Profound AI

来源：https://www.profoundlogic.com/ai/

- 无代码 AI 助手构建平台，可选 OpenAI/Google 等 LLM，连接业务数据；**支持通过 MCP 把 IBM i 接到其他 AI 工具**。
- 2025 年获「Low Code AI Solution of the Year」奖项（厂商自述）。

### 3.3 Profound Discovery & Analysis

来源：https://www.profoundlogic.com/profound-discovery/

- 定位为**咨询服务**（技术评估 + 现代化路线图），不是独立 AI 代码分析产品；可识别复杂度、技术债、现代化就绪度。

**成熟度：CoderFlow 🟡 早期（2026 年初发布，agentic 新范式，宣称收益待独立验证）；Profound AI 🟢 生产可用（通用 AI 平台）；Discovery & Analysis 🟢（服务型）**

---

## 4. IBM Rational Developer for i（RDi）

来源：https://www.ibm.com/products/rational-developer-for-i

- **定位**：基于 Eclipse 的 IBM i 集成开发环境（IDE），用于创建、维护、现代化 IBM i 应用。
- **能力**：集成搜索、编辑、构建、**分析（analysis）**、**重构（refactoring）**、调试器；支持 RPG 与 COBOL 现代化；三种版本（RPG and COBOL Tools；+Modernization Tools Java Edition；+Modernization Tools EGL Edition）。
- **交叉引用/调用图**：RDi 提供交叉引用与影响分析类能力（产品页明确列出「analysis」与「refactoring capabilities」）。**具体交叉引用文档页（ibm.com/docs/rdfi）因访问限制未能在线核验**，建议以产品页 + 实际试用为准。
- **作为确定性事实层的定位**：RDi 是**开发者桌面工具**，交叉引用基于源码/编译对象，适合单开发者人工复核；**不适合**作为全库级、可被 AI 批量调用的影响分析事实层（那是 ARCAD DISCOVER / X-Analysis 的定位）。
- **生态佐证**：Remain Software 的 Ai Chat 即作为 RDi 插件提供 AI 代码分析（IT Jungle 2023-05-17）。

**成熟度：🟢 生产可用（成熟 IDE，数十年历史）**

---

## 5. IBM i 上的 MCP（Model Context Protocol）生态

### 5.1 IBM 官方开源 MCP Server —— IBM/ibmi-mcp-server ⭐ 重点

来源：https://github.com/IBM/ibmi-mcp-server （77 stars / 35 forks / 501 commits / Apache-2.0 / TypeScript，持续活跃）
官方文档：https://ibm-d95bab6e.mintlify.app/

- **定位**：IBM 官方开源的 IBM i MCP Server + `ibmi` CLI，让 AI agent 通过 MCP 与 IBM i 交互。
- **架构**：AI 客户端 → MCP Server → YAML 定义的 SQL 工具 → **Mapepire**（IBM i 上的 WebSocket 数据库服务，默认端口 8076）→ Db2 for i。
- **核心创新**：把 SQL 查询写成 YAML 即变成 AI 可发现/可调用的工具（零代码，无需 TypeScript/REST）。
- **客户端**：Claude Desktop、VSCode、**Cursor**、Gemini CLI、Cline、Bob 等 **10+ 客户端**；支持 stdio（开发）与 HTTP（生产）传输。
- **生产特性**：OpenTelemetry 可观测性、审计日志、连接池、限流、多认证模式（含 IBM i HTTP 认证 + RSA 加密、JWT/OAuth）；尊重 IBM i 对象权限。
- **兼容性**：测试于 IBM i 7.3+；依赖 DB2 for i、QSYS2 系统服务、Mapepire。
- **预置工具集**：性能监控、安全与审计、作业管理、存储与 IFS、数据库（表/索引/约束/统计）。
- **Agent 框架示例**：Agno、LangChain、Google ADK。
- **部署**：Docker/Podman/OpenShift 配置齐全。

> 这是把 IBM i 系统数据暴露给 Claude/Cursor 的**现成官方做法**，且是 Apache-2.0 开源、可自行审计。

### 5.2 其他开源 IBM i MCP 项目（GitHub 检索，2026-08）

| 仓库 | 语言 | Stars | 说明 |
|---|---|---|---|
| [IBM/ibmi-mcp-server](https://github.com/IBM/ibmi-mcp-server) | TypeScript | 77 | IBM 官方，SQL 工具 + CLI，生产级 |
| [Strom-Capital/mcp-server-db2i](https://github.com/Strom-Capital/mcp-server-db2i) | TypeScript | 20 | DB2 for i 数据库查询与元数据检查 |
| [abstracta/IBM-AS-400-ISeries-MCP-Server](https://github.com/abstracta/IBM-AS-400-ISeries-MCP-Server) | Java | 20 | AS/400/iSeries MCP 服务器 |
| [FanMnz/ibmi-mcp-server](https://github.com/FanMnz/ibmi-mcp-server) | TypeScript | 7 | IBM i 源成员（source member）管理 |

### 5.3 商业 MCP Server

- **ARCAD MCP Server**：70+ 工具，暴露确定性应用仓库（见第 2 节）。
- **Fresche MCP Server**：Fresche 提供 MCP 连接器（「From Questions to Dashboards in Minutes」，见第 6 节）。
- **Profound AI**：支持通过 MCP 把 IBM i 接到其他 AI 工具。

**成熟度：IBM 官方 ibmi-mcp-server 🟢 生产可用（开源、活跃、文档齐全）；社区项目 🟡 早期；商业 MCP Server 🟡（2026 年新发布）**

---

## 6. 其他 IBM i 现代化 / AI 工具

### 6.1 Fresche Solutions —— X-Analysis AI（🟢 生产可用）

来源：https://freschesolutions.com/products/x-analysis/

- **X-Analysis AI**：AI 驱动的 IBM i 应用智能分析。
  - 自动文档化**数百万行 RPG、COBOL、Synon**；
  - 提取内嵌业务规则、映射依赖、暴露复杂度；
  - **跨代码/数据/流程追踪影响（impact tracing）**；
  - 零中断扫描整个 IBM i 环境。
- **规模背书**：40+ 年 IBM i 软件智能经验、2200+ 家组织、10 亿+ 行代码分析文档化（厂商数据）。
- **Fresche MCP Server**：提供 MCP 连接器（官方页面导航可见）。
- **AI 现代化服务**：先提取业务规则再 AI 生成新代码，重点做「约束幻觉」（IT Jungle 2025-11-10《Fresche Taps AI For New RPG-To-Java Conversion Tool》，GM Marcel Sarrasin 语）。
- **与 IBM Bob 合作**：Fresche Practice Director Steve Cast 是 IBM Bob 客户证言人（IBM 官方页面）。

### 6.2 Lansa（🟡 未能核验 AI 代码分析能力）

- 官网 https://lansa.com/ 定位为**低代码开发平台**（Visual LANSA），加速 IBM i 应用开发。
- **未检索到 Lansa 面向「代码库 AI 分析/影响分析」的明确产品**；其 AI 能力更多在低代码应用构建侧。**未能在线核验其 AI 辅助分析能力。**

### 6.3 Dekati（🔴 未能核验 / 疑似同名混淆）

- 检索到的唯一 dekati.com 是**芬兰颗粒物测量仪器公司**（fine particle measurement），与 IBM i 无关。
- **未检索到名为「Dekati」的 IBM i 现代化/AI 工具**。若用户所指为其他拼写（如 Datakit、Dekati 变体），请提供官网以便复核。**未能在线核验。**

### 6.4 Sysbus（🔴 未能核验）

- sysbus.com 无法正常访问（多次抓取返回空内容）；多引擎检索未找到名为「Sysbus」的 IBM i AI 分析产品。
- **未能在线核验。**

### 6.5 其他值得关注的 IBM i AI 工具（IT Jungle 2026-04-06 综述）

- **Remain Software Ai Chat**（🟡）：IBM i 社区首个编码助手（2023-05 发布），作为 RDi/MiWorkplace 插件，基于 ChatGPT 分析 IBM i 代码。
- **Polverini & Partners AI Production Gate**（🟡）：IBM i 上规模化自主 AI 代码生成的「确定性控制层」；ReplicTest 端到端测试套件将支持 IBM Bob。
- **Programmers.io TimeBridge**（🟡）：AI 驱动的 IBM i 现代化框架（代码文档化/生成 + 咨询 + 人力服务）。
- **CM First Group**：COBOL→现代 RPG，结合静态分析、自动化与 AI 辅助（POWERUp 2026 议题）。

---

## 7. 成熟度与局限汇总表

| 工具/产品 | 厂商 | 类型 | 影响分析 | 查逻辑/文档化 | MCP | 成熟度 | 主要局限 |
|---|---|---|---|---|---|---|---|
| IBM Bob Premium Package for i | IBM | AI 编码 agent | 部分（依赖技能） | ✅ explain/document | 有（Bob 生态） | 🟢 生产（2026-03 GA，较新） | 新生态、定价按用户、需评估银行合规 |
| ARCAD DISCOVER | ARCAD | 确定性应用分析 + AI | ✅✅ 核心 | ✅ | ✅（V26 MCP 70+ 工具） | 🟢 生产 | 商业授权、需部署仓库 |
| ARCAD MCP Server | ARCAD | MCP 桥接 | ✅（经仓库） | ✅ | ✅ | 🟡 早期（2026 春） | 季度迭代中、覆盖逐步扩大 |
| Profound CoderFlow | Profound Logic | Agentic coding | 部分 | ✅ | 有（Profound AI） | 🟡 早期 | 5–10x 为厂商宣称、新范式 |
| Profound AI | Profound Logic | 通用 AI 平台 | — | — | ✅ | 🟢 生产 | 非专用代码分析 |
| IBM ibmi-mcp-server | IBM（开源） | MCP 桥接 | 部分（SQL 工具） | 部分 | ✅✅ | 🟢 生产（开源活跃） | 面向 DB2/系统数据，非源码级影响分析 |
| Fresche X-Analysis AI | Fresche | 确定性分析 + AI | ✅✅ 核心 | ✅ | ✅（Fresche MCP） | 🟢 生产 | 商业授权 |
| RDi | IBM | IDE | ✅（人工复核级） | 部分 | — | 🟢 生产 | 桌面工具、非全库 AI 事实层 |
| Remain Ai Chat | Remain | AI 助手（RDi 插件） | 部分 | ✅ | — | 🟡 早期 | 基于 ChatGPT、能力有限 |
| Lansa | Lansa | 低代码平台 | 未能核验 | 未能核验 | — | 🟡 | 未见 AI 代码分析产品 |
| Dekati | — | — | — | — | — | 🔴 未能核验 | 疑似同名混淆（芬兰颗粒物公司） |
| Sysbus | — | — | — | — | — | 🔴 未能核验 | 官网不可访问 |

---

## 8. 针对「口述需求 → 影响分析 / 查逻辑」的最务实工具组合建议

### 8.1 核心判断

- **「查清现行处理逻辑」**：LLM 擅长，但**必须喂确定性上下文**，否则幻觉。因此需要「确定性仓库」或「源码级检索」作为事实层。
- **「变更影响分析」**：本质是**交叉引用/依赖图**问题，LLM 推断不可靠，必须由**确定性分析引擎**（ARCAD DISCOVER / X-Analysis / RDi 交叉引用）给出，再由 LLM 用自然语言解释。
- **银行合规约束**：源码不出内网、可审计、可回滚 → 优先**本地/私有化部署**方案。

### 8.2 推荐三层架构

```
┌─────────────────────────────────────────────────────────┐
│ 第 3 层：LLM 对话层（口述需求入口）                        │
│   IBM Bob Premium Package for i（IBM 原生）              │
│   或 Claude / Cursor / 本地 LLM（经 MCP 接入）            │
├─────────────────────────────────────────────────────────┤
│ 第 2 层：MCP 桥接层（把 IBM i 数据/仓库暴露给 AI）         │
│   IBM 官方 ibmi-mcp-server（开源，DB2/系统数据）          │
│   + ARCAD MCP Server（70+ 工具，应用仓库）                │
├─────────────────────────────────────────────────────────┤
│ 第 1 层：确定性事实层（影响分析/交叉引用/数据血缘）          │
│   ARCAD DISCOVER（首选）或 Fresche X-Analysis            │
│   RDi 交叉引用（开发者人工复核兜底）                       │
└─────────────────────────────────────────────────────────┘
```

### 8.3 分场景建议

| 场景 | 推荐组合 | 说明 |
|---|---|---|
| 银行核心（COBOL+CL+DDS，合规严） | **ARCAD DISCOVER（事实层）+ ARCAD MCP Server（桥接）+ 本地 LLM 或 Claude（对话层）** | DISCOVER 本地部署、源码不出环境、确定性影响分析；MCP 让 Claude/Cursor 直接问「改这个字段会影响哪些程序」 |
| 想用 IBM 官方全家桶 | **IBM Bob Premium Package for i + ARCAD DISCOVER** | Bob 原生连 IBM i、explain/document 技能；DISCOVER 补确定性影响分析；ARCAD 已出 Bob 插件 |
| 预算敏感/开源优先 | **IBM ibmi-mcp-server（开源）+ Claude/Cursor + RDi** | 官方开源 MCP 免费、可审计；RDi 交叉引用人工复核；适合先 PoC |
| 快速 PoC（1–2 周） | **ibmi-mcp-server + Claude Desktop/Cursor** | 30–40 分钟即可跑通首个 AI 查询（官方文档），验证「口述→查逻辑」可行性 |

### 8.4 落地注意事项

1. **影响分析必须确定性**：不要用纯 LLM 做影响分析，用 DISCOVER/X-Analysis 的交叉引用仓库，LLM 只负责把结果翻译成自然语言。
2. **MCP 是当前性价比最高的桥**：IBM 官方开源 ibmi-mcp-server 已生产可用，支持 Cursor/Claude，是「把 IBM i 暴露给通用 AI agent」的现成做法。
3. **先 PoC 再选型**：建议先用开源 MCP + Claude/Cursor 跑通「口述需求→查逻辑」，再评估是否采购 DISCOVER/X-Analysis 做确定性影响分析层。
4. **合规红线**：确认源码/数据不出内网（DISCOVER 本地部署、CoderFlow 本地执行、ibmi-mcp-server 自托管均满足）；LLM 若用云端，只传最小上下文。
5. **Dekati/Sysbus 不可作为选型依据**：未能核验为 IBM i AI 工具，建议从候选清单剔除或要求厂商提供官网。

---

## 9. 参考链接清单（可验证）

**IBM 官方**
- IBM Bob（AI coding agent）产品页：https://www.ibm.com/products/ai-coding-agent
- IBM Bob Premium Package for i：https://www.ibm.com/products/ai-coding-agent/ibm-i-modernization
- IBM Rational Developer for i：https://www.ibm.com/products/rational-developer-for-i
- IBM 官方开源 MCP：https://github.com/IBM/ibmi-mcp-server
- IBM i MCP Server 官方文档：https://ibm-d95bab6e.mintlify.app/

**ARCAD Software**
- 官网（V26 / MCP Server 入口）：https://www.arcadsoftware.com/
- DISCOVER：https://www.arcadsoftware.com/discover/ ；功能页：https://www.arcadsoftware.com/discover/features/
- ARCAD MCP Server：https://www.arcadsoftware.com/arcad/arcad-mcp-server-ibm-i-application-context-for-ai-agents/

**Profound Logic**
- CoderFlow：https://www.profoundlogic.com/coderflow/ ；How it works：https://www.profoundlogic.com/coderflow/how-it-works/
- Profound AI：https://www.profoundlogic.com/ai/
- Profound Discovery & Analysis：https://www.profoundlogic.com/profound-discovery/

**Fresche Solutions**
- X-Analysis AI：https://freschesolutions.com/products/x-analysis/
- Fresche MCP Server 入口：https://go.freschesolutions.com/mcp-connector

**社区/开源 MCP**
- Strom-Capital/mcp-server-db2i：https://github.com/Strom-Capital/mcp-server-db2i
- abstracta/IBM-AS-400-ISeries-MCP-Server：https://github.com/abstracta/IBM-AS-400-ISeries-MCP-Server
- FanMnz/ibmi-mcp-server：https://github.com/FanMnz/ibmi-mcp-server

**权威媒体（独立核验）**
- IT Jungle《Big Blue Converges IBM i RPG And System Z COBOL Code Assistants Into "Project Bob"》（2025-10-13）：https://www.itjungle.com/2025/10/13/big-blue-converges-ibm-i-rpg-and-system-z-cobol-code-assistants-into-project-bob/
- IT Jungle《Here Come The AI-Based Code Modernization Offerings》（2026-04-06）：https://www.itjungle.com/2026/04/06/here-come-the-ai-based-code-modernization-offerings/
- IT Jungle《AI Will Be Front And Center At POWERUp 2026》（2026-04-20）：https://www.itjungle.com/2026/04/20/ai-will-be-front-and-center-at-powerup-2026-next-week/
- IT Jungle《Public Preview For Watson Code Assistant for i Available Soon》（2025-05-21）：https://www.itjungle.com/2025/05/21/public-preview-for-watson-code-assistant-for-i-available-soon/
- IT Jungle《Profound Says New Agentic AI Dev Tool Delivers Huge Productivity Boost》（2026-01-19）：https://www.itjungle.com/2026/01/19/profound-says-new-agentic-ai-dev-tool-delivers-huge-productivity-boost/
- IT Jungle《Fresche Taps AI For New RPG-To-Java Conversion Tool》（2025-11-10）：https://www.itjungle.com/2025/11/10/fresche-taps-ai-for-new-rpg-to-java-conversion-tool/
- IT Jungle《ARCAD Discover: Global Application Analysis With An AI Interface》（2025-02-10）：https://www.itjungle.com/2025/02/10/arcad-discover-global-application-analysis-with-an-ai-interface
- AI News/TechForge《IBM launches AI platform Bob to regulate SDLC costs》（2026-04-28）：https://www.artificialintelligence-news.com/news/ibm-launches-ai-platform-bob-to-regulate-sdlc-costs/

---

*本报告由 AI 调研生成，所有结论均附可验证来源；厂商宣称数据（如 5–10x、10 亿行）未做独立验证，仅作参考。*

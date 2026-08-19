# 金融行业大型机 COBOL 系统的维护、现代化与 AI 辅助改造：真实案例与教训调研报告

> **报告日期**：2026-08-19
> **调研目的**：为"金融核心系统 COBOL/CL 代码库 + AI 口述需求→变更分析/查现行逻辑"项目校准真实世界的成功率与风险。
> **调研方法**：网络检索 + 直接抓取官方/权威来源（IBM 官方、英国国家审计署 NAO、欧盟委员会、CommBank 官网等）。
> **来源性质说明**：本报告区分三类来源——①**已在线核验**（本次调研中直接抓取到原文）；②**广泛报道但本次未能在线核验**（网络受限，但属主流媒体/官方公告反复记载的真实事件，附来源线索）；③**厂商口径**（IBM 等供应商发布的数据，非独立验证，已明确标注）。

---

## 摘要（TL;DR）

1. **存量巨大且仍在增长**：COBOL 至今支撑全球 40%+ 在线银行系统、95% ATM 交易、80% 线下刷卡，日交易额超 30 亿美元（IBM 口径，2025-2026）。全球在运行的 COBOL 代码被广泛引用为约 2200 亿行（厂商口径）。
2. **迁移是高风险动作**：成功的核心迁移（如澳洲联邦银行 2016）耗时数年、投入巨大；失败的迁移（英国 NHS NPfIT、英国 TSB 银行 2018）造成数十亿英镑损失或大规模客户事故。**教训高度一致：低估规模与复杂度、数据迁移风险、测试与回滚不足。**
3. **"不迁移、持续维护"是主流且理性的选择**：稳定性、成本、风险决定了多数银行继续维护 COBOL。维护期真实痛点是**文档缺失、知识在老师傅脑中、变更风险、合规审计**——这正是 AI 最能发力的地方。
4. **AI 辅助 COBOL 维护已有真实部署案例（2023-2026）**：埃及社保机构 NOSI 用 IBM watsonx Code Assistant for Z 做代码分析/文档化，量化指标为**代码分析时间 -94%（8h→30min）、理解复杂应用时间 -79%（24h→5h）**；IBM 2025 年 12 月发布的 2.8 版本已实现"口述需求→影响分析→代码生成→编译构建验证"的 agentic 工作流，与用户目标场景高度吻合。
5. **合规约束明确**：欧盟 AI Act（2024 生效、2026 适用）对高风险 AI 要求**活动日志（可追溯）、详细文档、人工监督、鲁棒性/准确性**；金融监管普遍要求 AI 生成代码必须人工审查、必须有完整审计轨迹。
6. **核心结论**：AI 在"读代码、解释、查逻辑、生成文档、辅助测试"上已被证明有效且有量化收益；但在"直接改生产核心逻辑"上，真实世界仍以**人工审查 + 语义等价测试 + 完整审计轨迹**为铁律。AI 是放大器，不是替代者。

---

## 一、金融 COBOL 系统存量现状（2024-2026）

### 1.1 关键存量数据

**已在线核验（IBM 官方，2025-05 发布 / 2026-02-25 更新）**：
> "COBOL's imperative, procedural and (in its newer iterations) object-oriented configuration serves as the foundation for more than 40% of all online banking systems. It also supports 80% of in-person credit card transactions, handles 95% of all ATM transactions, and powers systems that generate more than USD 3 billion of commerce each day."
> —— IBM, *What is COBOL?*, https://www.ibm.com/think/topics/cobol （IBM 官方口径，非独立验证；其脚注引用 PC Mag 2023-12-01 报道）

- **40%+ 在线银行系统**基于 COBOL
- **95% ATM 交易**、**80% 线下刷卡**由 COBOL 支撑
- 相关系统**日交易额超 30 亿美元**

**广泛引用但本次未能在线核验（厂商/媒体口径）**：
- "全球约 **2200 亿行** COBOL 代码仍在生产环境运行"——该数字源自 IBM 2020 年 COBOL 60 周年宣传及 PC Mag 2023-12-01《The World Depends on 60-Year-Old Code No One Knows Anymore》（IBM 官方页面脚注引用）。属厂商口径，业界广泛引用但无独立第三方审计。

### 1.2 COBOL 程序员老龄化与技能缺口

**已在线核验（IBM 新闻稿，2024-03-06）**：IBM 联合 SHARE 等成立"大型机技能委员会"（Mainframe Skills Council），引用 The Futurum Group《2024 Global Mainframe Skills Report》（由 IBM、Broadcom、21CS 委托，**厂商委托研究，非独立**）：
- **91%** 的雇主受访者计划未来 1-2 年内为大型机新岗位招聘人才
- **79%** 在招聘中级大型机人才，**51%** 在招聘初级岗位
- 65% 的大学负责人认为大型机人才比 5 年前更多（技能池在增长，但**资深人才仍稀缺**）
- 来源：https://newsroom.ibm.com/IBM-Mainframe-Skills

**真实银行佐证（同一新闻稿内引用）**：
- **M&T Bank**（美国银行）工程总监 Gary Fusco："We had a business challenge of a **tenured mainframe workforce**... By about six months into the program, our apprentices were coding on COBOL"——即"资深大型机团队老龄化"是真实痛点。
- **DNB**（挪威银行）大型机架构师 Janette Skoga："many of our talented employees and partners are **approaching retirement age**"——直接印证"知识在老师傅脑中、即将退休"。

### 1.3 小结

存量数据口径全部来自 IBM 及其委托研究（厂商口径），但方向一致且被行业广泛引用：**COBOL/大型机仍是金融核心的绝对主力，且人才供给结构性短缺**。这既是维护风险，也是 AI 辅助的动机来源。

---

## 二、著名 COBOL 现代化/迁移案例

### 2.1 成功案例

#### 案例 A：澳洲联邦银行（Commonwealth Bank of Australia）核心银行迁移（约 2012-2016）

- **事实**：CBA 通过"核心银行现代化计划"（Core Banking Modernisation, CBM），将基于 COBOL 的旧核心迁移到新平台（与 Accenture 合作，采用 SAP 组件 + Java 服务化架构），2016 年前后宣布完成，是全球规模最大的核心银行迁移之一。
- **来源**：The Australian（2016）、Computer Weekly、CommBank 官方新闻室等均有报道。**本次调研因网络受限未能在线核验原文**，但该事件属主流财经/IT 媒体反复记载的真实事件。
- **教训（成功侧）**：分阶段、可回滚、业务连续性优先；迁移是多年工程而非一次性切换。

#### 案例 B：埃及国家社会保险组织 NOSI（2023-2024）——"不迁库、留在 Z 平台 + AI 现代化"

- **事实**（**已在线核验，IBM 官方案例研究**）：NOSI 原计划与第三方服务商合作**迁离 IBM Z 平台**，但因"应用环境复杂 + **缺乏文档** + 数据库迁移对数据完整性和业务连续性构成重大风险"而受阻。2023 年参加 IBM TechXchange 后改为**留在 Z 平台、用 AI 辅助现代化**：用 watsonx Code Assistant for Z 对两个单体 COBOL 应用做试点，自动化文档、可视化应用流、识别冗余代码；2024 年升级到 IBM z16、存储扩容 30%。
- **量化指标**：代码分析时间 **-94%**（约 8 小时→30 分钟）；理解复杂应用时间 **-79%**（约 24 小时→5 小时）。
- **来源**：https://www.ibm.com/case-studies/national-organization-for-social-insurance （**厂商发布案例，非独立验证**）
- **启示**：**"不迁移"也可以是成功路径**——避免数据库迁移风险、保留数据在 Z 平台，用 AI 解决"看不懂、没文档"的问题。

#### 案例 C：法国 Crédit Mutuel Alliance Fédérale（2024）——银行 gen AI 工业化

- **事实**（**已在线核验，IBM 新闻稿 2024-06-27**）：法国最大银行保险集团之一，自 2016 年起用 AI；2023 年 AI 为 25,000 名顾问释放**近 100 万小时行政工时**；2024 年与 IBM 合作在自有数据中心部署 watsonx，计划工业化落地 **35 个 AI 用例**（客户关系、风险管理、合规、文档理解等），并用 watsonx.governance 落实其"可信 AI 宪章"。
- **来源**：https://newsroom.ibm.com/2024-06-27-Credit-Mutuel-Alliance-Federale-accelerates-deployment-of-generative-AI-in-collaboration-with-IBM （**厂商发布，非独立验证**）
- **启示**：银行把 AI 用于**理解、合规、文档、辅助**等低风险场景先行，且强调**治理（governance）与自有数据主权**。

### 2.2 失败/翻车案例

#### 案例 D：英国 NHS 国家 IT 计划（NPfIT，2002-2011）——政府级大型系统迁移失败

- **事实**（**已在线核验，英国国家审计署 NAO 官方报告，2011-05-18**）：
  - 已花费 **£27 亿**用于电子病历系统，NAO 判定"**不物有所值**"（does not represent value for money）；剩余计划支出 **£43 亿**。
  - 原目标 2010 年实现"每位患者电子病历"，实际推迟到 2015-16 年；在 North/Midlands/East 区域，**7 年仅向急性医院交付 4/97 个系统**。
  - NAO 负责人 Amyas Morse 原话："**fundamentally underestimating the scale and complexity of a major IT-enabled change programme**"（根本性地低估了大型 IT 变革项目的规模与复杂度）。
- **来源**：https://www.nao.org.uk/report/the-national-programme-for-it-in-the-nhs/ （**官方审计机构，独立**）
- **教训**：低估规模与复杂度、供应商合同管理失败、目标过大、缺乏分阶段交付。

#### 案例 E：英国 TSB 银行 IT 迁移事故（2018）——金融核心迁移翻车

- **事实**：2018 年 4 月，TSB 将约 **500 万客户**从 Lloyds Banking Group 平台迁移到母公司 Sabadell 自研的新平台（Proteo4，由 Sabadell 旗下 TECNOCOM 开发）。迁移引发重大 IT 故障：客户无法登录、看到他人账户余额、重复扣款、账户被锁等；CEO Paul Pester 于 2018 年 9 月辞职；银行承担巨额补偿与监管调查。
- **来源**：BBC、The Guardian、Computer Weekly 等主流媒体大量报道。**本次调研因网络受限未能在线核验原文**，但该事件是金融 IT 迁移失败的标志性案例，被广泛引用。
- **教训**：**数据迁移与并行运行风险**、测试不足、回滚机制缺失、对"新平台是否真的等价"缺乏验证——与 NHS 案例教训高度一致。

### 2.3 迁移案例教训汇总

| 维度 | 成功侧（CBA/NOSI） | 失败侧（NHS/TSB） |
|---|---|---|
| 规模认知 | 分阶段、多年工程 | 低估规模与复杂度 |
| 数据迁移 | 避免/谨慎（NOSI 不迁库） | 数据迁移是事故高发点（TSB） |
| 验证 | 语义等价、业务连续性优先 | 测试与回滚不足 |
| 治理 | 监管/审计全程参与 | 供应商合同与监管失控（NHS） |

---

## 三、"不迁移、持续维护"路线：为什么银行选择继续维护 COBOL

### 3.1 为什么继续维护（而非重写）

**已在线核验（IBM 官方，2025-2026）**，IBM 明确列出 COBOL 的四大优势：
- **稳定性**：关键任务系统高可用、极少故障，适合金融/政府 7×24 运行
- **可扩展性**：无需重写即可承载增长的工作负载
- **数据/文件处理能力**：批量处理、事务处理、多种文件访问方式（顺序/索引/相对）极强
- **互操作性**：可通过现代化与微服务、云、DevOps 集成
- 来源：https://www.ibm.com/think/topics/cobol

**已在线核验（IBM NOSI 案例）**：NOSI 选择"不迁库"的直接理由是**数据库迁移对数据完整性和业务连续性构成重大风险**，且外部承包商依赖导致项目延期。→ 迁移的隐性成本与风险往往高于"继续维护 + 增量现代化"。

### 3.2 维护期的真实痛点（AI 的发力点）

**已在线核验（IBM 2.8 公告，2025-12-16 发布 / 2026-03-02 更新）**：
> "Since IBM Z applications often span decades of development, this leaves teams with **fragmented, outdated, or missing documentation** that slows onboarding, complicates cross-team collaboration, and **increases reliance on scarce experts**."
> —— https://www.ibm.com/new/announcements/agentic-ai-for-smarter-mainframe-modernization-with-ibm-watsonx-code-assistant-for-z

痛点归纳（均有上述来源支撑）：
1. **文档缺失/过时**：NOSI"absence of proper documentation"；IBM"decades of development → fragmented, outdated, or missing documentation"。
2. **知识在老师傅脑中**：M&T Bank"tenured mainframe workforce"、DNB"approaching retirement age"、IBM"reliance on scarce experts"。
3. **变更风险**：核心系统任何改动影响面大，缺乏影响分析工具时只能靠人肉排查。
4. **合规审计**：金融核心变更需可追溯、可审计，而旧系统往往缺乏现代审计轨迹。

---

## 四、AI 辅助 COBOL 维护的真实案例（2023-2026）

### 4.1 案例：NOSI（埃及社保，2023-2024）——AI 做代码解释/文档化/冗余识别

- **已在线核验（IBM 官方案例研究）**，量化指标：
  - 代码分析时间 **-94%**（约 8 小时 → 30 分钟）
  - 理解复杂应用时间 **-79%**（约 24 小时 → 5 小时）
  - 用途：自动化文档、可视化应用流、识别冗余 COBOL 代码、给出现代化建议
- 来源：https://www.ibm.com/case-studies/national-organization-for-social-insurance （**厂商发布，非独立验证**）

### 4.2 案例：IBM watsonx Code Assistant for Z 2.8（2025-12）——"口述需求→变更分析"已产品化

- **已在线核验（IBM 官方公告）**，与用户目标场景**高度吻合**。官方示例 prompt：
  > "I need to add a column to the Motor Policy Table that captures if the vehicle is an electric car. Can you help me update all of the programs with this field?"
  > —— 即"口述需求 → 自动识别依赖、做**影响分析**、生成代码（遵循编码规范）、**编译构建验证**"的 agentic 工作流。
- 技术底座：**MCP（Model Context Protocol）** 工具集，包括：
  - **Z Understand 元数据检索**（全企业应用图谱）
  - **影响分析**（基于静态分析的依赖识别）
  - **代码生成与编码规范**（对齐企业大型机编码标准）
- 新增能力：**文档自动生成**（Business Rule Discovery，面向大型复杂 Z 程序，代码感知分段避免通用 LLM 分块陷阱）；**数据字典/业务词汇表增强**（把企业自定义变量语义注入 LLM，避免猜测缩写含义）。
- 来源：https://www.ibm.com/new/announcements/agentic-ai-for-smarter-mainframe-modernization-with-ibm-watsonx-code-assistant-for-z （**厂商发布，非独立验证**）

### 4.3 案例：Crédit Mutuel（法国，2024）——银行 gen AI 工业化 + 治理

- **已在线核验（IBM 新闻稿 2024-06-27）**：35 个 AI 用例、2023 年释放近 100 万小时行政工时、用 watsonx.governance 落实"可信 AI 宪章"（透明、可文档化、尊重隐私、人工监督）。
- 来源：https://newsroom.ibm.com/2024-06-27-Credit-Mutuel-Alliance-Federale-accelerates-deployment-of-generative-AI-in-collaboration-with-IBM （**厂商发布，非独立验证**）

### 4.4 效果评估与局限

- **可量化的收益确实存在**（NOSI 的 94%/79%、Crédit Mutuel 的 100 万小时），但**全部来自厂商发布**，缺乏独立第三方审计；真实收益因代码库质量、数据字典完备度、团队配合而异。
- **公开案例集中在"理解/文档/分析/辅助测试"**，而非"AI 直接改生产逻辑"。IBM 2.8 的 agentic 变更流程仍以"生成 + 编译构建验证 + 人工审查"收尾，**没有"AI 一键上线"的公开先例**。
- 未发现公开的、可核验的"银行用 LLM 直接改 COBOL 生产逻辑并上线"的案例——这本身就是重要信号：**真实世界把 AI 定位为辅助，而非决策者**。

---

## 五、合规与审计角度

### 5.1 欧盟 AI Act（已在线核验，欧盟委员会官方页面，2026-08-03 更新）

- **时间线**：2024-08-01 生效，2026-08-02 适用；高风险规则（Annex III 敏感领域）**2027-12-02** 起适用；AI Omnibus 简化法案 2026-07-27 生效。
- **高风险 AI 范围**（与金融核心直接相关）：
  - "AI safety components in **critical infrastructures**"
  - "AI use-cases utilised to give access to **essential private and public services (e.g. credit scoring)**"
- **高风险 AI 的强制义务**（对"AI 辅助改核心系统"直接适用）：
  - 充分的风险评估与缓解
  - 高质量数据集（降低歧视性结果）
  - **活动日志（logging of activity）以确保结果可追溯**
  - **详细文档**（供监管评估合规性）
  - 向部署者提供清晰充分的信息
  - **适当的人工监督（human oversight）**
  - 高水平的**鲁棒性、网络安全与准确性**
- 来源：https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai （**官方，独立**）

### 5.2 其他监管框架（通用知识，本次未能在线核验原文，标注为"广泛确立的框架"）

- **美国 Fed SR 11-7（2011）模型风险管理**：银行对模型（含 AI/ML）须有独立验证、持续监控、文档化与治理——是金融 AI 治理的基准框架。
- **巴塞尔委员会（BCBS）**：2021 年发布《AI 与机器学习对金融部门的潜在影响》报告，后续持续关注 AI 在银行的应用与风险。
- **中国**：金融监管总局（原银保监）对金融 AI 的监管方向一贯强调**人工审核、可解释性、完整审计轨迹、模型风险管理**；金融核心系统变更须符合信息科技风险管理与外包管理相关监管要求（如《银行保险机构信息科技风险管理办法》等）。具体条文本次未能在线核验，建议以监管官网原文为准。

### 5.3 对"AI 辅助改代码/查逻辑"的合规含义

综合上述（EU AI Act 为已核验硬约束，其余为行业共识）：
1. **AI 生成代码必须人工审查**——EU AI Act 明确要求"人工监督"；金融行业实践亦如此。
2. **必须有完整审计轨迹**——EU AI Act 要求"活动日志可追溯"；即每次 AI 查询/生成/变更都要留痕（谁、何时、什么 prompt、什么输出、谁批准）。
3. **必须验证行为等价**——IBM 官方做法是"自动生成单元测试，比较新代码与原始 COBOL 的**语义等价性**"（见 4.2 产品页），这正对应监管对"变更不改变业务行为"的要求。
4. **数据字典/业务词汇是准确性的前提**——IBM 2.8 明确"用企业数据字典注入 LLM，避免猜测变量/缩写含义"，否则 AI 输出不可信、不可审计。

---

## 六、教训总结：金融核心系统引入 AI 辅助的 5 条铁律

基于上述案例（成功侧 CBA/NOSI/Crédit Mutuel，失败侧 NHS/TSB，以及 EU AI Act 合规要求），提炼如下：

**铁律 1：AI 先做"读"，后做"改"——把 AI 定位为理解与文档化工具，而不是生产变更的执行者。**
> 证据：所有公开成功案例（NOSI、Crédit Mutuel、IBM 2.8）的 AI 用途都是代码解释、文档生成、影响分析、辅助测试；**没有任何公开可核验的"AI 直接改生产 COBOL 逻辑并上线"案例**。先让 AI 把"查现行逻辑、口述需求→影响分析"跑通，再谈变更。

**铁律 2：任何 AI 辅助的变更必须走"人工审查 + 语义等价测试 + 完整审计轨迹"。**
> 证据：EU AI Act 对高风险 AI 强制要求活动日志、详细文档、人工监督、鲁棒性/准确性（已核验）；IBM 官方流程以"自动生成单元测试验证 COBOL→Java 语义等价 + 编译构建验证"收尾。TSB 事故的根因之一正是"新平台行为不等价且未充分验证"。

**铁律 3：用"行为等价"而非"代码相似"来验收 AI 输出。**
> 证据：IBM 2.8 明确用"auto-generated unit tests to compare semantic equivalence of new Java service to original COBOL code"；NOSI 案例中 AI 的价值恰恰是"识别冗余/理解行为"。对金融核心，**行为不变是底线**。

**铁律 4：数据字典/业务词汇表是 AI 准确性的前提——没有它，AI 就是在猜。**
> 证据：IBM 2.8 明确"data dictionaries and business glossaries... supply the LLM with authoritative context, helping the system align outputs to the organization's actual semantics, rather than inferring or guessing at variable or abbreviation meanings"。金融 COBOL 的变量/缩写语义高度私有，必须先建立企业级语义资产。

**铁律 5：别把"迁移"当目标——先量化维护成本与风险，AI 辅助的增量现代化往往比大爆炸迁移更安全。**
> 证据：NOSI 放弃迁库、留在 Z 平台 + AI 现代化，避免了数据迁移风险（已核验）；NHS（£27 亿打水漂）与 TSB（500 万客户事故）的失败都源于"低估规模与复杂度 + 数据迁移风险"。对存量 COBOL/CL 核心，**"继续维护 + AI 辅助理解/变更分析"是真实世界成功率最高的路线**。

---

## 附录：来源清单与可信度标注

| # | 来源 | 类型 | 性质 | 本次是否在线核验 |
|---|---|---|---|---|
| 1 | IBM《What is COBOL?》https://www.ibm.com/think/topics/cobol （2025-05 发布/2026-02-25 更新） | 厂商官方 | 厂商口径 | ✅ 已核验 |
| 2 | IBM watsonx Code Assistant for Z 产品页 https://www.ibm.com/products/watsonx-code-assistant-z | 厂商官方 | 厂商口径 | ✅ 已核验 |
| 3 | IBM NOSI 案例研究 https://www.ibm.com/case-studies/national-organization-for-social-insurance | 厂商官方 | 厂商口径 | ✅ 已核验 |
| 4 | IBM watsonx Code Assistant for Z 2.8 agentic 公告 https://www.ibm.com/new/announcements/agentic-ai-for-smarter-mainframe-modernization-with-ibm-watsonx-code-assistant-for-z （2025-12-16/2026-03-02） | 厂商官方 | 厂商口径 | ✅ 已核验 |
| 5 | IBM Crédit Mutuel 新闻稿 https://newsroom.ibm.com/2024-06-27-Credit-Mutuel-Alliance-Federale-accelerates-deployment-of-generative-AI-in-collaboration-with-IBM （2024-06-27） | 厂商官方 | 厂商口径 | ✅ 已核验 |
| 6 | IBM 大型机技能委员会新闻稿 https://newsroom.ibm.com/IBM-Mainframe-Skills （2024-03-06，含 Futurum 委托研究） | 厂商官方 | 厂商口径（委托研究） | ✅ 已核验 |
| 7 | 英国 NAO《NHS NPfIT 报告》https://www.nao.org.uk/report/the-national-programme-for-it-in-the-nhs/ （2011-05-18） | 官方审计机构 | 独立 | ✅ 已核验 |
| 8 | 欧盟委员会 AI Act 官方页面 https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai （2026-08-03 更新） | 官方 | 独立 | ✅ 已核验 |
| 9 | 澳洲联邦银行核心迁移（2016） | 主流媒体（The Australian / Computer Weekly / CommBank 新闻室） | 媒体/官方 | ⚠️ 本次网络受限未能在线核验，属广泛记载的真实事件 |
| 10 | TSB 银行 IT 迁移事故（2018） | 主流媒体（BBC / The Guardian / Computer Weekly） | 媒体 | ⚠️ 本次网络受限未能在线核验，属广泛记载的真实事件 |
| 11 | 美国 Fed SR 11-7 模型风险管理（2011） | 监管 | 独立 | ⚠️ 未能在线核验，属广泛确立的框架 |
| 12 | 巴塞尔委员会（BCBS）AI/ML 报告（2021 起） | 监管 | 独立 | ⚠️ 未能在线核验，属广泛确立的框架 |
| 13 | 中国金融监管总局（原银保监）金融 AI 监管要求 | 监管 | 独立 | ⚠️ 未能在线核验，建议以监管官网原文为准 |

> **重要提示**：本报告所有"厂商口径"数据（IBM 及其委托研究）均非独立第三方验证，引用时需注意其宣传性质；#9-#13 因本次调研网络受限未能抓取原文，但均为真实、可溯源的主流事件/框架，建议后续在可访问网络环境下补充原文核验。

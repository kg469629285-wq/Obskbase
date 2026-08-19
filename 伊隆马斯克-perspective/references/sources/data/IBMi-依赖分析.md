# IBM i（AS/400）系统目录/对象依赖分析能力调研报告

> 调研日期：2026-08-19
> 目标：验证一个关键架构假设——IBM i 上做"确定性影响分析"（程序调用链/数据依赖）能否**不解析源码**，直接用系统目录（命令/SQL 视图）提取，再喂给 AI。
> 方法：IBM 官方文档（ibm.com 全域名 403 反爬，未能直接抓取）+ IBM i 权威专家博客（RPGPGM.COM、IT Jungle）交叉核验。**每条标注验证状态：✅已核验 / ⚠️官方文档待核验 / 🔴未核验。**

---

## 摘要（TL;DR）

**部分可行，且方向明确：**

1. **IBM i 在编译时把程序对程序/文件/服务程序的引用固化为对象属性**，可通过系统 SQL 视图/表函数直接查询，**不需要解析源码**——这是 IBM i 相比开放平台做影响分析的巨大优势。
2. **已验证的核心资产**（RPGPGM.COM 专家博客 + IBM docs 链接）：`PROGRAM_RESOLVED_ACTIVATIONS`（完整服务程序调用链）、`PROGRAM_RESOLVED_IMPORTS`（程序导入）、`BOUND_SRVPGM_INFO`（直接调用的服务程序）、`BINDING_DIRECTORY_INFO`（绑定目录）、`SYSMEMBERSTAT`（含源码成员）、`SYSFILES`（数据/源码文件区分）——足以支撑"程序→服务程序/模块"调用链和"源码成员定位"的自动化提取。
3. **已知漏洞（已验证 1 条）**：条件编译（RPGPPOPT(*NONE)）会导致依赖数据**包含未实际使用的文件**（过度报告）。动态 CALL、间接引用、数据区/数据结构参数传递等遗漏场景**未在本调研中核验**，属已知风险。
4. **AI 用例**：未找到"把 IBM i 系统目录喂给 LLM 做影响分析"的直接公开案例——这是空白点，需要你自己趟路。

---

## 一、DSPPGMREF 命令

- **官方文档**：https://www.ibm.com/docs/en/i/7.5?topic=ssw_ibm_i_75/cl/dsppgmref.htm （⚠️ URL 已知，但 ibm.com 403 未能抓取正文）
- **功能**：列出程序引用的对象（文件、程序、SQL 对象等）——这是 IBM i 影响分析的传统确定性事实源。
- **已验证的局限**（✅ IT Jungle 2008-05-07《Accurate Program References》，作者 Ted Holt）：条件编译 `RPGPPOPT(*NONE)` 时，DSPPGMREF 会同时显示两个文件，即使实际只使用其中一个——**依赖数据会过度报告**。
- **批量/API 自动提取细节**：🔴 未核验（官方文档 403）。实际生产中通常结合 DSPPGM（交互式）与 API。

---

## 二、QSYS2 系统 SQL 视图/表函数（核心资产）

### A. ✅ 已验证（RPGPGM.COM 专家博客抓取成功 + IBM docs 链接）

| 名称 | 类型 | 查什么 | 适用版本 | 来源 |
|---|---|---|---|---|
| **PROGRAM_RESOLVED_ACTIVATIONS** | 表函数 | 追踪程序调用的**完整服务程序链**（含 LEVEL 列，可递归展开） | 7.6 TR1, 7.5 TR7 | 官方：https://www.ibm.com/docs/en/i/7.6.0?topic=services-program-resolved-activations-table-function ；博客：https://www.rpgpgm.com/2026/02/find-all-services-programs-used-by.html |
| **PROGRAM_RESOLVED_IMPORTS** | 表函数 | 列出 ILE 程序/服务程序的**所有导入（imports）** | 仅 7.6 | 官方：https://www.ibm.com/docs/en/i/7.6.0?topic=services-program-resolved-imports-table-function ；博客：https://www.rpgpgm.com/2025/07/sql-table-function-to-list-all-imports.html |
| **BOUND_SRVPGM_INFO** | 视图 | 列出程序**直接调用**的服务程序（第一层） | 7.4 TR1, 7.3 TR7 | 博客：https://www.rpgpgm.com/2019/12/equivalent-of-dsppgm-and-dspsrvpgm-2.html |
| **BINDING_DIRECTORY_INFO** | 视图 | 列出**绑定目录（binding directory）**内容 | 版本未给出 | 博客：https://www.rpgpgm.com/2022/09/sql-view-listing-binding-directory.html |
| **SYSMEMBERSTAT** | 视图 | 列出**所有文件成员**（含源码成员），含 SOURCE_TYPE、时间戳 | 7.5 TR4, 7.4 TR10 | 博客：https://www.rpgpgm.com/2024/09/new-view-lists-all-file-members.html |
| **SYSFILES** | 视图 | FILE_TYPE 列（'DATA'/'SOURCE'），提供 DSPFD 类文件信息 | 版本未给出 | 博客：https://www.rpgpgm.com/2021/10/new-sql-view-gives-dspfd-information.html |
| **BOUND_MODULE_INFO** | 视图 | 模块级绑定信息 | 7.4 TR1, 7.3 TR7 | 与 BOUND_SRVPGM_INFO 一并提及（IBM 链接未写入调研记录） |

### B. ⚠️ 官方文档待核验（URL 已知，因 403 未能抓取正文）

| 名称 | 官方 URL | 状态 |
|---|---|---|
| **OBJECT_DEPENDENCY** | https://www.ibm.com/docs/en/i/7.5?topic=ssw_ibm_i_75/rzasq/object_dependency.htm | ⚠️ 待核验（RPGPGM 有相关文章但 URL 404 未找到） |
| **SYSPGMREF** | https://www.ibm.com/docs/en/i/7.5?topic=ssw_ibm_i_75/rzasq/syspgmref.htm | ⚠️ 待核验 |
| **SYSFIELDS** | https://www.ibm.com/docs/en/i/7.5?topic=ssw_ibm_i_75/rzasq/sysfields.htm | ⚠️ 待核验 |

---

## 三、DSPFFD / 字段级数据字典

- **DSPFFD 命令细节**：🔴 未核验（官方文档 403）。
- **已验证相关资产**：SYSFILES 视图提供 DSPFD 类文件信息；`SYSMEMBERSTAT` 定位源码成员。
- **字段级线索**（⚠️ 未抓取细节）：RPGPGM《Retrieve file's Column Headings using API》（2014-08，https://www.rpgpgm.com/2014/08/retrieve-files-column-headings-using-api.html）——用 API 取文件列标题，与"字段级数据字典"高度相关。
- **结论**：字段级数据字典提取的完整方案需在可访问 IBM 文档的环境补充核验；但方向明确（SYSFIELDS / API / DSPFFD）。

---

## 四、对象依赖 API

- **QUSRMBRD / QBNLPGMR / Qp2Term 等**：🔴 未核验——本调研未能获取任何功能/用法细节。
- **建议**：在可访问 IBM 文档的环境下核验 `QBNLPGMR`（列出程序引用 API）与 `QUSRMBRD`（数据区相关），它们是程序化提取对象依赖的官方 API 路径。

---

## 五、源码成员管理

- **已验证**：`SYSMEMBERSTAT`（7.5 TR4 / 7.4 TR10）列出所有文件成员（含源码成员），带 SOURCE_TYPE 和时间戳——**这是定位 COBOL/CL 源码物理文件的直接 SQL 路径**。
- **已验证**：`SYSFILES.FILE_TYPE`（'DATA'/'SOURCE'）区分数据文件与源码文件。
- **线索**（⚠️ 未抓取细节）：RPGPGM《Finding source members with the same name》（2025-07）。
- **结论**：源码成员定位可走 SQL（SYSMEMBERSTAT + SYSFILES），不需要手工翻 QSYS/QGPL 库。

---

## 六、局限（影响分析会漏什么）

### ✅ 已验证
- **条件编译过度报告**（IT Jungle 2008《Accurate Program References》）：`RPGPPOPT(*NONE)` 时 DSPPGMREF 显示未实际使用的文件 → **依赖数据会包含不存在的路径（假阳性）**。

### 🔴 已知风险（未在本调研核验，需你实测）
- **动态 CALL**：用变量名调程序（`CALL #PGM`），编译期无法确定目标 → 系统目录不记录。
- **间接引用**：通过数据区（*LDA / 用户空间）、数据结构传程序名 → 依赖隐藏。
- **条件编译/COPY 展开**：COPY 不同成员、编译时条件 → 实际依赖集合可能不同于静态记录。
- **程序对象 vs 源码**：系统目录记录的是**编译后的对象引用**；若源码与对象不同步（老系统常见），对象引用反映的是"上次编译时"的依赖。

### 应对建议
1. 用系统目录做**骨架**（确定性、免费、快），用源码解析 + LLM 做**补充**（找动态 CALL、间接引用）。
2. 对"假阳性"（过度报告）设置人工确认点——宁可多审，不可漏审（金融核心）。

---

## 七、AI/LLM 用例

- **未找到**"把 IBM i 系统目录喂给 LLM 做影响分析"的直接公开案例（🔴 空白点，需你作为先行者）。
- **邻近参考**：Astera《Makes Extracting Legacy Report Data an AI Specialty》（IT Jungle 2026-08-10，https://www.itjungle.com/2026/08/10/astera-makes-extracting-legacy-report-data-an-ai-specialty/）——用 AI 提取旧系统报表数据（主题相近但非依赖分析）。
- **可行性判断**：把上面的 QSYS2 视图（PROGRAM_RESOLVED_ACTIVATIONS / BOUND_SRVPGM_INFO / SYSMEMBERSTAT / SYSFILES / OBJECT_DEPENDENCY）查询结果导出为结构化数据（JSON/CSV），喂给 LLM 做"口述需求→影响分析"，技术上完全可行——这正是 ARCAD DISCOVER / Fresche X-Analysis 这类确定性仓库做的事。

---

## 八、结论：能否不解析源码直接用系统目录做影响分析？

**✅ 可行（骨架层），但需补两层：**

1. **骨架（确定性，免费，直接可用）**：QSYS2 视图/表函数查询程序调用链、服务程序依赖、数据/源码文件、成员定位——**不需要解析源码**。这是 IBM i 相对 IBM Z / 开放平台的独特优势。
2. **补充（源码级，LLM 辅助）**：动态 CALL、间接引用、数据区传参等系统目录看不到的部分 → 需源码解析（PARAGRAPH/字段级 chunk + 检索）或 LLM 扫描补充。
3. **人工兜底**：条件编译造成的假阳性（已验证）+ 动态依赖（未核验风险）→ 影响清单输出前设人工确认点。

**一句话**：IBM i 的系统目录是现成的"确定性影响分析地基"（比 IBM Z 更便宜可靠），用它做骨架、用 LLM 做语义解释与动态依赖补充、用人做最终确认——三层缺一不可。

---

## 附录：来源与验证状态

| 来源 | 类型 | 验证状态 |
|---|---|---|
| IBM i 官方文档（DSPPGMREF/OBJECT_DEPENDENCY/SYSPGMREF/SYSFIELDS） | 官方 | ⚠️ URL 已知，ibm.com 403 未能抓取正文 |
| IBM docs（PROGRAM_RESOLVED_ACTIVATIONS/IMPORTS） | 官方 | ✅ 链接经专家博客确认 |
| RPGPGM.COM（Simon Hutchinson，IBM i SQL 权威专家） | 专家博客 | ✅ 已抓取 |
| IT Jungle《Accurate Program References》（2008-05-07, Ted Holt） | 专家媒体 | ✅ 已抓取 |
| Astera 文章（IT Jungle 2026-08-10） | 媒体 | ✅ 已抓取（邻近参考） |
| QUSRMBRD/QBNLPGMR/Qp2Term API | 官方 API | 🔴 未核验 |

> 调研说明：本报告因 ibm.com 全域名反爬（403）未能直接核验 IBM 官方命令/视图正文，核心事实来自 IBM i 权威专家博客（RPGPGM.COM、IT Jungle），并经 IBM docs 链接交叉确认。建议在可访问 IBM 文档的环境下补充核验第三、四节内容。

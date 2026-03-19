# Claude Certified Architect（基础）Domain 2 学习知识点笔记

## 文档用途
这份文档专门记录 `Domain 2: Tool Design & MCP Integration（工具设计与 MCP 集成）` 的核心知识点、考试陷阱、实战场景与易错项。

使用规则：
- 每完成一个 `Task Statement（任务陈述）`，立即补充该节要点
- 优先记录“先做什么修复”与“考试为什么不选其他项”
- 牢记本域偏好：
  - `low-effort, high-leverage fixes（低成本、高杠杆修复）`
  - `scoped access（范围受限访问）`
  - `community servers（社区服务器）` 优先于自建

---

## Domain 2 结构总览

### 2.1 Tool interface design（工具接口设计）
- `tool descriptions（工具描述）` 是 LLM 做 `tool selection（工具选择）` 的首要机制
- 描述过于简略会导致误路由
- 正确第一步修复通常是增强描述，而不是立刻上 `routing classifier（路由分类器）`

### 当前笔记

#### 2.1 Tool interface design（工具接口设计）

**核心原则**
- `tool descriptions（工具描述）` 不是补充信息，而是模型选工具的核心依据
- 相似工具若都只有泛描述，模型很容易误路由

**好描述必须包含**
- `primary purpose（主要用途）`
- `inputs（输入）`：格式、类型、约束
- `example queries（示例请求）`
- `limitations（限制）`
- `explicit boundaries（明确边界）`

**经典误路由题**
- `get_customer`: "Retrieves customer information"
- `lookup_order`: "Retrieves order information"
- 用户查询订单状态时误调客户工具
- 正确第一步修复：扩写两个工具描述

**为什么不是其他选项**
- `few-shot examples（少样本示例）`：增加 token 开销，但没修根因
- `routing classifier（路由分类器）`：过度工程化，不是第一步
- `tool consolidation（工具合并）`：改动过大，不是低成本首选

**tool splitting（工具拆分）**
- 过于泛化的工具应拆成目的明确的工具
- 例如：
  - `extract_data_points（提取数据点）`
  - `summarize_content（总结内容）`
  - `verify_claim_against_source（根据来源验证主张）`

**系统提示词干扰**
- `system prompt（系统提示）` 中的关键词导向可能覆盖良好描述
- 更新描述后要检查系统提示是否制造了错误联想

**一句话记忆**
- 先修工具描述，再考虑分类器

#### 2.2 Structured error responses（结构化错误响应）

**核心机制**
- 工具失败时应使用 `isError（是否错误）` 明确标记错误响应
- 不要只返回模糊报错文本

**四类错误**
- `transient error（瞬时错误）`
  - 典型：超时、服务暂时不可用
  - 处理：可重试，本地优先恢复
- `validation error（校验错误）`
  - 典型：输入格式错误、缺少字段
  - 处理：修正输入后再试，不应原样重试
- `business error（业务错误）`
  - 典型：退款超限、违反策略
  - 处理：不可重试，走替代流程或人工审批
- `permission error（权限错误）`
  - 典型：拒绝访问、凭证不足
  - 处理：更换权限、升级凭证或人工介入

**结构化错误字段**
- `isError（是否错误）`
- `errorCategory（错误类别）`
- `isRetryable（是否可重试）`
- `message（人类可读说明）`

**高频易错点**
- `valid empty result（有效空结果）` 不等于 `access failure（访问失败）`
- 查询成功但无匹配，不应重试
- 没查到，不等于没查成

**多代理错误传播**
- 子代理应先对 `transient error（瞬时错误）` 做本地恢复
- 无法恢复再向上游传播
- 传播时应带：
  - `partial results（部分结果）`
  - 已尝试动作
  - 剩余失败点

**一句话记忆**
- 空结果不是错误；错误必须可分类、可决策

#### 2.3 Tool distribution and tool_choice（工具分发与 tool_choice）

**tool overload（工具过载）**
- 给单个 `Agent（代理）` 太多工具会降低 `tool selection（工具选择）` 可靠性
- 考试偏好每个代理约 `4-5 tools（4 到 5 个工具）`
- 工具应按 `role（角色）` 受限分配

**角色边界**
- `synthesis agent（综合代理）` 不应默认拿：
  - `web search tools（网页搜索工具）`
  - `document analysis tools（文档分析工具）`
- `web search agent（网页搜索代理）` 也不应拿文档分析工具
- 工具集应与代理职责对齐

**scoped cross-role tools（受限跨角色工具）**
- 若某代理高频遇到简单且低风险的小操作，可直接给它一个受限工具
- 典型例子：
  - 给 `synthesis agent（综合代理）` 一个 `scoped verify_fact tool（受限事实核验工具）`
- 适用条件：
  - 大多数场景简单
  - 高频
  - 回 coordinator（协调代理）会显著增加延迟

**tool_choice（工具调用策略）**
- `auto（自动）`
  - 默认模式
  - 模型自己决定是否调用工具，以及调用哪个
- `any（必须调用某个工具，但模型自选）`
  - 强制必须调工具
  - 但由模型自己在候选工具中选择
  - 适合需要保证结构化输出的情况
- `{"type": "tool", "name": "extract_metadata"}`（强制调用指定工具）
  - 必须调用这个特定工具
  - 适合强制前置步骤

**用受限工具替代泛工具**
- 优先给 `scoped tools（范围受限工具）`
- 不要轻易给 `full-access generic tools（全权限泛工具）`
- 例如优先给：
  - `load_document（加载文档）`
- 而不是：
  - `fetch_url（抓任意 URL）`

**一句话记忆**
- 工具少而准，优于工具多而杂

#### 2.4 MCP server integration（MCP 服务器集成）

**配置层级**
- `project-level（项目级）`：
  - `.mcp.json`
  - 受版本控制
  - 团队共享
- `user-level（用户级）`：
  - `~/.claude.json`
  - 个人使用
  - 不共享

**环境变量展开**
- `.mcp.json` 中应使用 `${GITHUB_TOKEN}` 这类 `environment variable expansion（环境变量展开）`
- 凭证不应写死进版本控制文件

**工具发现**
- 所有已配置 `MCP servers（MCP 服务器）` 的工具会在连接时一起被发现并同时可用
- 这会放大 `tool descriptions（工具描述）` 的重要性

**MCP resources（MCP 资源）**
- 用于暴露内容目录、文档层级、数据库结构等
- 让代理在调用工具前先理解可用数据空间
- 可减少盲目探索式查询

**build vs use（自建还是使用现成）**
- 标准集成优先评估 `community MCP servers（社区 MCP 服务器）`
- 只有在团队专有工作流或社区方案无法满足时才考虑自建

**一句话记忆**
- 标准集成先用现成，凭证用环境变量

#### 2.5 Built-in tools（内建工具）

**Grep（内容搜索） vs Glob（路径匹配）**
- `Grep（内容搜索）`：
  - 搜文件内容
  - 适合找函数调用、错误信息、import（导入）
- `Glob（路径匹配）`：
  - 按模式匹配文件路径
  - 适合找测试文件、配置文件、指定扩展名文件

**Read / Write / Edit（读取 / 写入 / 编辑）**
- `Edit（编辑）`
  - 适合小而精确的局部修改
  - 需要唯一锚点
- 若锚点不唯一或上下文不足：
  - 回退到 `Read + Write（读取 + 写入）`

**incremental codebase understanding（增量式代码库理解）**
- 先 `Grep（内容搜索）` 找入口
- 再 `Read（读取）` 顺着导出与调用链追踪
- 不要一开始就读取整个代码库

**典型顺序题**
- 找调用某个废弃函数的文件：
  - 先用 `Grep（内容搜索）`
- 找这些调用者的测试文件：
  - 再用 `Glob（路径匹配）`

**一句话记忆**
- `Grep（内容搜索）` 搜内容，`Glob（路径匹配）` 找文件；`Edit（编辑）` 失败就 `Read + Write（读取 + 写入）`

---

## 待补充章节

---

## 模拟测验记录

### Domain 2 第一次 7 题模拟测验
- 得分：`7/7`
- 结果判断：达到考试准备标准（`6+/7`）
- 当前结论：Domain 2 已达到可上考场水平

**正确率说明**
- `2.1 Tool interface design（工具接口设计）`：掌握
- `2.2 Structured error responses（结构化错误响应）`：掌握
- `2.3 Tool distribution and tool_choice（工具分发与 tool_choice）`：掌握
- `2.4 MCP server integration（MCP 服务器集成）`：掌握
- `2.5 Built-in tools（内建工具）`：掌握

**一句话总结**
- 先修描述，再谈分类；先分清错误，再谈恢复；先做受限工具，再给泛权限；先用社区服务器，再考虑自建。

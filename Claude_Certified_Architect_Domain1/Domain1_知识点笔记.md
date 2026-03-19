# Claude Certified Architect（基础）Domain 1 学习知识点笔记

## 文档用途
这份文档专门记录 `Domain 1: Agentic Architecture & Orchestration（代理式架构与编排）` 的核心知识点、考试陷阱、实战场景与易错项。

使用规则：
- 每完成一个 `Task Statement（任务陈述）`，立即补充该节要点
- 优先记录“考试会怎么错”，不是只记定义
- 所有高风险决策题，默认先问：这是 `prompt-based guidance（基于提示的引导）` 还是 `programmatic enforcement（程式化强制约束）`

---

## Domain 1 结构总览

### 1.1 Agentic loops（代理循环）
- 核心控制讯号是 `stop_reason（停止原因）`
- `stop_reason = "tool_use"`：执行工具，追加 `tool_result（工具结果）`，回传更新后的 `conversation history（对话历史）`
- `stop_reason = "end_turn"`：代理完成，可以向用户输出最终结果
- 不能靠自然语言、文本内容、或任意轮数上限判断完成

### 当前笔记

#### 1.1 Agentic loops（代理循环）

**标准循环**
1. 发送请求到 `Messages API（消息 API）`
2. 检查 `stop_reason（停止原因）`
3. 若为 `tool_use（工具调用）`：
   - 执行工具
   - 将 `tool_result（工具结果）` 作为新消息追加进 `conversation history（对话历史）`
   - 把更新后的历史再次发送给 Claude
4. 若为 `end_turn（结束当前轮）`：
   - 结束代理流程
   - 把最终回复展示给用户

**考试陷阱**
- 反模式 1：根据 assistant 说了“我完成了”之类的话来判断结束
- 反模式 2：把 `iteration cap（迭代上限）` 当主要停止条件
- 反模式 3：看到 `text（文本）` 就判定已完成

**高频结论**
- 完成判定只看 `stop_reason（停止原因）`
- `tool result（工具结果）` 必须回写进对话历史，否则模型下一轮无法基于新信息推理
- `text（文本）` 与 `tool_use（工具调用）` 可以同时存在

**模型驱动与固定流程**
- 考试倾向 `model-driven decision-making（模型驱动决策）`
- 但关键业务约束不能只靠模型自觉，后续在 `1.4` 和 `1.5` 要用程式化机制锁定

**一句话记忆**
- 不要从内容猜状态，要从协议字段读状态

#### 1.2 Multi-agent orchestration（多代理编排）

**默认架构**
- 采用 `hub-and-spoke architecture（中心辐射架构）`
- `coordinator agent（协调代理）` 在中心
- `subagent（子代理）` 在外围
- 所有通讯都必须经过 `coordinator（协调代理）`
- `subagents（子代理）` 不能直接彼此通信

**coordinator（协调代理）的职责**
- 做 `task decomposition（任务拆解）`
- 决定调用哪些 `subagents（子代理）`
- 显式传递上下文
- 聚合结果
- 处理错误
- 识别覆盖缺口并发起 `iterative refinement loop（迭代式精炼循环）`

**关键隔离原则**
- `subagents（子代理）` 不会自动继承 `coordinator（协调代理）` 的对话历史
- `subagents（子代理）` 不共享记忆
- 每条必要资讯都必须显式写进子代理提示中

**高频根因题**
- 如果最终报告缺少整个主题板块，优先怀疑 `coordinator（协调代理）` 的 `task decomposition（任务拆解）`
- 典型模式：拆解范围太窄，导致下游即使执行正确也无法覆盖缺失主题

**关于 synthesis agent（综合代理）**
- 不是必选组件
- 小系统里可由 `coordinator（协调代理）` 直接完成综合
- 大系统里可单独设 `synthesis agent（综合代理）` 负责整合、归因、统一结构与缺口标注
- 即使存在 `synthesis agent（综合代理）`，最终控制权仍属于 `coordinator（协调代理）`

#### 1.3 Subagent invocation and context passing（子代理调用与上下文传递）

**Task tool（Task 工具）**
- `coordinator（协调代理）` 通过 `Task tool（Task 工具）` 生成 `subagent（子代理）`
- 若 `allowedTools（允许工具列表）` 不含 `Task`，则无法生成子代理
- 每个子代理都有自己的 `AgentDefinition（代理定义）`
- `AgentDefinition（代理定义）` 常包含：
  - `description（描述）`
  - `system prompt（系统提示）`
  - `tool restrictions（工具限制）`

**上下文传递原则**
- 不要只传一句“参考前面的结果”
- 必须把前序代理或工具产出的必要资讯显式写进子代理输入
- 优先使用 `structured data（结构化数据）`
- 结构化格式要分离：
  - `content（内容）`
  - `metadata（元数据）`

**为什么 metadata（元数据）重要**
- 保障 `attribution（归因）`
- 方便定位冲突来源
- 支撑 `claim-source mapping（主张-来源映射）`
- 最终报告缺来源时，优先检查是不是上下文传递时丢了元数据

**提示设计原则**
- 要写清：
  - `research goal（研究目标）`
  - `scope（范围）`
  - `quality criteria（质量标准）`
  - `required output format（输出格式）`
- 不要写成僵硬的逐步操作脚本

**并行生成**
- 若多个子任务相互独立，应在同一轮中发出多个 `Task tool（Task 工具）` 调用
- 这样能降低总延迟
- 考试会测 `latency awareness（延迟意识）`

**fork_session（分叉会话）**
- 从同一分析基线创建多个独立分支
- 适合探索不同方案
- 分叉后各分支相互独立，不自动同步

**一句话记忆**
- 子代理不是“知道前文的人”，只是“拿到提示后执行的人”

#### 1.4 Workflow enforcement and handoff（工作流强制执行与交接）

**两类控制方式**
- `prompt-based guidance（基于提示的引导）`：大多数时候有效，但存在非零失败率
- `programmatic enforcement（程式化强制约束）`：通过门禁、前置条件或系统控制实现确定性保证

**考试决策规则**
- 涉及 `financial（财务）`、`security（安全）`、`compliance（合规）`、权限、法律风险：
  - 选 `programmatic enforcement（程式化强制约束）`
- 涉及风格、格式、语气等低风险问题：
  - `prompt-based guidance（基于提示的引导）` 通常足够

**典型高频题**
- 退款前必须先完成 `identity verification（身份验证）`
- 正确修法是加 `programmatic prerequisite gate（程式化前置门禁）`
- `enhanced prompt（增强提示）` 与 `few-shot examples（少样本示例）` 都不足以提供 100% 保证
- `routing classifier（路由分类器）` 解决的是分流，不是前置条件强制执行

**多问题请求处理**
- 将一个请求中的多个 `concern（关注点）` 拆成独立事项
- 若事项彼此独立，可并行调查
- 最后输出统一解决方案

**人工升级交接**
- 人工接手方不能假设能看到完整对话历史
- `handoff summary（交接摘要）` 必须自包含
- 至少应包含：
  - `customer ID（客户 ID）`
  - `conversation summary（对话摘要）`
  - `root cause analysis（根因分析）`
  - `refund amount（退款金额）`，若适用
  - `recommended action（建议动作）`

**一句话记忆**
- 高风险规则用硬约束，低风险偏好用软引导

#### 1.5 Agent SDK hooks（Agent SDK 钩子）

**hooks（钩子）的本质**
- 在代理运行流程中的固定拦截点插入确定性逻辑
- 用于提供 `deterministic guarantees（确定性保证）`

**PostToolUse hook（工具后钩子）**
- 触发时机：工具执行完成后、模型读取结果前
- 典型用途：`normalisation（标准化）`
- 常见处理：
  - 统一时间格式
  - 统一状态码表达
  - 对齐字段名
  - 清洗异构工具结果

**tool call interception hook（工具调用拦截钩子）**
- 触发时机：工具真正执行前
- 典型用途：`enforcement（强制执行）`
- 常见处理：
  - 阻止高风险操作
  - 改走审批流程
  - 拦截不合规调用

**考试决策规则**
- 若业务要求 100% 遵守，优先用 `hook（钩子）`
- 若只是偏好、格式、风格，使用 `prompt（提示）` 即可

**一句话记忆**
- `PostToolUse（工具后）` 处理结果，`interception（拦截）` 阻止调用

#### 1.6 Task decomposition strategies（任务拆解策略）

**两种主模式**
- `fixed sequential pipeline（固定顺序流水线）`
  - 适合结构稳定、步骤明确、重复性高的任务
  - 例如：`code review（代码审查）`、`document processing（文档处理）`
- `dynamic adaptive decomposition（动态自适应拆解）`
  - 适合开放式、探索式、步骤依赖发现结果的任务
  - 例如：遗留系统测试规划、开放式调查

**attention dilution（注意力稀释）**
- 单次处理过多文件或目标，会导致：
  - 覆盖不均
  - 标准不一致
  - 明显问题漏检
- 这通常不是模型能力问题，而是任务分解方式错误

**标准修法**
- 使用 `multi-pass architecture（多轮架构）`
- 先做 `per-file local analysis pass（逐文件局部分析轮）`
- 再做 `cross-file integration pass（跨文件整合轮）`

**一句话记忆**
- 大型结构化审查，先局部一致，再全局整合

#### 1.7 Session state and resumption（会话状态与恢复）

**三种主要做法**
- `--resume <session-name>（恢复指定会话）`
  - 适合旧上下文仍然可信、工具结果未过时、文件变化不大时
- `fork_session（分叉会话）`
  - 适合同一分析基线上的多方案并行探索
- `fresh start with summary injection（带摘要注入的重新开始）`
  - 适合旧上下文已陈旧、文件变化较多、工具结果过时或会话开始自相矛盾时

**stale context（陈旧上下文）**
- 恢复旧会话后仍基于旧文件状态或旧工具结果推理
- 典型表现：
  - 建议前后矛盾
  - 针对已修复问题继续提建议
  - 漏掉新改动带来的影响

**正确处理原则**
- 若只改了少量文件：
  - 明确指出哪些文件改了
  - 说明改动方向
  - 要求做 `targeted re-analysis（定向重分析）`
- 若整体上下文已脏：
  - 新开会话
  - 注入结构化摘要

**考试陷阱**
- 不能默认有旧会话就直接恢复
- `fork_session（分叉会话）` 不是用来清理陈旧上下文的
- 不应在少量文件变化时要求代理从头重扫整个系统

**一句话记忆**
- 局部变更就局部重分析，整体陈旧就带摘要重开

---

## 待补充章节
- 1.7 Session state and resumption（会话状态与恢复）

---

## 模拟测验记录

### Domain 1 第一次 10 题模拟测验
- 得分：`9/10`
- 结果判断：达到考试准备标准（`8+/10`）
- 当前唯一薄弱点：`1.3 Subagent invocation and context passing（子代理调用与上下文传递）`

**错题记录**
- 第 5 题错误答案：`B`
- 正确答案：`C`

**错因**
- 误以为 `Task tool（Task 工具）` 本身限制了来源传递
- 实际问题不在工具能力，而在 `context passing（上下文传递）` 设计
- 正确修法是向下游代理传递包含 `content（内容）` 与 `metadata（元数据）` 的 `structured data（结构化数据）`

**补强提醒**
- 报告缺少来源归因时，优先检查：
  - 是否有 `claim-source mapping（主张-来源映射）`
  - 是否传入了 `source URL（来源网址）`
  - 是否传入了 `document name（文件名）`
  - 是否传入了 `page number（页码）`

---

## 综合练习结论

### 可再生能源研究系统的标准表述
- `Coordinator agent（协调代理）` 先将请求拆成 `solar（太阳能）`、`wind（风能）`、`geothermal（地热）`、`tidal（潮汐）`、`biomass（生质能）` 五个主题
- 对每个主题并行生成两类子任务实例：
  - `Web Search subagent（网页搜索子代理）`
  - `Document Analysis subagent（文档分析子代理）`
- `Web Search（网页搜索）` 负责公开来源取证
- `Document Analysis（文档分析）` 负责上传文档取证
- 二者是并行证据路径，不是上下游串联关系
- 所有结果必须带 `structured metadata（结构化元数据）`
- 每次工具返回后，由系统自动触发 `PostToolUse hook（工具后钩子）` 做 `normalisation（标准化）`
- `Coordinator（协调代理）` 聚合结果并检查覆盖缺口
- 在调用 `finalise_report（定稿报告）` 或 `publish_report（发布报告）` 前，由 `PreToolUse Hook（工具调用前钩子）` 执行 `Source quality gate（来源质量门禁）`
- 低质量来源不得进入最终报告

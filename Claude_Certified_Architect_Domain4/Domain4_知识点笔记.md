# Claude Certified Architect（基础）Domain 4 学习知识点笔记

## 文档用途
这份文档专门记录 `Domain 4: Prompt Engineering & Structured Output（提示工程与结构化输出）` 的核心知识点、考试陷阱、实战场景与易错项。

使用规则：
- 每完成一个 `Task Statement（任务陈述）`，立即补充该节要点
- 优先记录“哪种 technique（技术）适用于哪类问题”，而不是泛泛记概念
- 牢记本域偏好：
  - 明确标准优于模糊置信度描述
  - `few-shot（少样本）` 优于继续堆 instructions（指令）
  - `tool_use + schema（工具调用 + 模式）` 解决语法问题，不自动解决语义问题

---

## Domain 4 结构总览

### 4.1 Explicit criteria（显式标准）
- 明确的 categorical criteria（分类式标准）优于 “be conservative（保守一点）” 这类模糊指令
- 高误报会摧毁整体信任，不只是摧毁单一类别信任
- severity（严重级别）要用 code examples（代码示例）校准，而不是靠纯文字解释

### 当前笔记

#### 4.1 Explicit criteria（显式标准）

**核心原则**
- 具体、可分类、可判定的标准优于基于信心或语气的模糊要求
- 错误示例：
  - “Be conservative”
  - “Only report high-confidence findings”
- 正确示例：
  - “Only flag findings when claimed behaviour contradicts actual code behaviour”
  - “Report bugs and security vulnerabilities, skip minor style issues”

**false positive trust problem（误报信任问题）**
- 某一类问题误报过多，会让开发者不再信任其他类别的发现
- 正确修法：
  - 先临时关闭高误报类别
  - 再单独优化该类别的 prompt（提示）
  - 目的是先恢复整体信任

**severity calibration（严重级别校准）**
- 不能只写 “critical（严重）/ minor（轻微）” 的抽象定义
- 应用实际 code examples（代码示例）展示：
  - 哪类问题算 critical（严重）
  - 哪类问题算 major（主要）
  - 哪类问题算 minor（轻微）

**一句话记忆**
- 明确判定标准优于模糊保守措辞

**考试标准补充**
- 不要让模型自己定义 “保守” 或 “高置信度”
- 要直接给出：
  - 报什么
  - 不报什么
  - 哪些类别暂时禁用
  - 严重级别如何划分

---

## 待补充章节

---

## 模拟测验记录

### Domain 4 第一次 8 题模拟测验
- 得分：`8/8`
- 结果判断：达到考试准备标准（`7+/8`）
- 当前结论：Domain 4 已达到可上考场水平

**正确率说明**
- `4.1 Explicit criteria（显式标准）`：掌握
- `4.2 Few-shot prompting（少样本提示）`：掌握
- `4.3 Structured output with tool_use（基于 tool_use 的结构化输出）`：掌握
- `4.4 Validation-retry loops（验证-重试循环）`：掌握
- `4.5 Batch processing（批处理）`：掌握
- `4.6 Multi-instance review（多实例审查）`：掌握

**补充说明：temperature（温度参数）**
- `temperature（温度参数）` 控制采样随机性
- 温度越高：
  - 输出越发散
  - 创造性更强
  - 一致性通常更差
- 温度越低：
  - 输出越稳定
  - 更接近确定性
  - 但不能替代明确标准、few-shot（少样本）或 schema（模式）设计
- 在本 domain（领域） 的考试里，`temperature（温度参数）` 通常不是首选修复手段

**一句话总结**
- 明确标准定边界，few-shot（少样本）保一致，schema（模式）保结构，验证重试修错位，批处理看延迟，多实例保独立。

#### 4.2 Few-shot prompting（少样本提示）

**核心原则**
- 当问题是：
  - 格式不一致
  - 模糊判断摇摆
  - 抽取字段经常空置
  - 多种文档结构下表现不稳定
- 优先考虑 `few-shot prompting（少样本提示）`

**few-shot（少样本）该怎么构造**
- 使用 `2-4 examples（2 到 4 个示例）`
- 优先选择模糊、易误判、结构差异大的案例
- 每个示例应包含：
  - 输入
  - 正确输出
  - 为什么这样判断
  - 为什么不选其他看似合理方案

**考试高频结论**
- `few-shot（少样本）` 优于继续堆 instructions（指令）
- 好的 few-shot（少样本） 不只是示范格式，还示范判断边界
- varied-format examples（多格式示例） 能显著降低抽取幻觉与漏抽

#### 4.3 Structured output with tool_use（基于 tool_use 的结构化输出）

**可靠性层级**
- `tool_use + JSON schema（工具调用 + JSON 模式）` 比 `prompt-only JSON（纯提示 JSON）` 更可靠
- 核心优势：
  - 几乎消除 `syntax errors（语法错误）`
  - 让结构化输出可稳定解析

**它不能防住什么**
- `semantic errors（语义错误）`
- `field placement errors（字段放错）`
- `fabrication（编造）`

**tool_choice（工具调用策略）**
- `auto（自动）`
  - 默认
  - 模型可能直接回文本
- `any（必须调用某个工具，但模型自选）`
  - 适合要求必须得到结构化输出、但文档类型未知
- `{"type": "tool", "name": "..."}`（强制指定工具）
  - 适合强制前置步骤

**schema 设计原则**
- 对源文档可能缺失的信息：
  - 使用 `optional（可选）`
  - 使用 `nullable（可空）`
- 对模糊情况：
  - 加 `unclear（不明确）`
- 对可扩展分类：
  - 加 `other（其他）` + detail（详情）
- 在 prompt（提示） 中同时写格式标准化规则

**一句话记忆**
- schema 管结构，不管真伪与逻辑

#### 4.4 Validation-retry loops（验证-重试循环）

**标准模式**
- 将以下三者一起送回模型进行重试：
  - `original document（原始文档）`
  - `failed extraction（失败抽取结果）`
  - `specific validation error（具体验证错误）`

**重试有效边界**
- 有效：
  - `format mismatches（格式不匹配）`
  - `structural output errors（结构输出错误）`
  - `misplaced values（值放错字段）`
- 无效：
  - `source truly missing（源文档确实没有信息）`

**辅助字段**
- `detected_pattern（检测模式）`
  - 记录某条 finding（发现）是由哪类模式触发的
  - 便于后续分析哪些类型最容易被驳回
- `calculated_total（计算总额）`
  - 用于与 `stated_total（声明总额）` 比对
- `conflict_detected（冲突标记）`
  - 标记源文档内部信息冲突

**一句话记忆**
- 重试能修“抽错”，修不了“文档没写”

#### 4.5 Batch processing（批处理）

**硬事实**
- `Message Batches API（消息批处理 API）`
  - 约 `50% cost savings（50% 成本节省）`
  - `up to 24-hour processing window（最长 24 小时处理窗口）`
  - `no guaranteed latency SLA（无保证延迟 SLA）`
  - 单请求内不支持多轮 `tool calling（工具调用）`

**适用边界**
- `Synchronous API（同步 API）`
  - 适合 blocking workflows（阻塞型工作流）
  - 如：`pre-merge checks（合并前检查）`
- `Batch API（批处理 API）`
  - 适合 latency-tolerant workflows（可容忍延迟工作流）
  - 如：隔夜报告、每周审计、夜间批量抽取

**custom_id（自定义 ID）**
- 用于稳定关联 request（请求） 与 response（响应）
- 尤其适合失败项识别与重提

**失败处理**
- 只重提失败项
- 必要时先修改失败项处理方式：
  - chunking（分块）
  - prompt（提示）优化
  - schema（模式）调整

**考试高频结论**
- 先在 sample set（样本集） 调优，再大批量跑
- 不要因为便宜就把所有同步任务都改成批处理

**一句话记忆**
- 批处理省钱，不保低延迟；阻塞型任务继续用同步

#### 4.6 Multi-instance review（多实例审查）

**same-session self-review（同会话自审）的局限**
- 同一 session（会话） 保留先前 reasoning context（推理上下文）
- 更容易为原有决定辩护
- 更不容易真正质疑自己的输出

**正确做法**
- 使用 `independent instance（独立实例）` 进行 review（审查）
- 让审查者只看：
  - 当前输入
  - 产物
  - 审查标准

**multi-pass architecture（多轮架构）**
- `per-file local analysis pass（逐文件局部分析轮）`
  - 保证每个文件审查深度一致
- `cross-file integration pass（跨文件整合轮）`
  - 捕捉数据流和跨模块问题

**confidence-based routing（基于置信度的分流）**
- 可以用于将低置信度 findings（发现） 路由给人工复核
- 但阈值必须用 `labelled validation set（带标注验证集）` 校准

**一句话记忆**
- 生成和审查分实例；局部和整合分轮做

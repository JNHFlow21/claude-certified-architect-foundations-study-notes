# Claude Certified Architect（基础）Domain 5 学习知识点笔记

## 文档用途
这份文档专门记录 `Domain 5: Context Management & Reliability（上下文管理与可靠性）` 的核心知识点、考试陷阱、实战场景与易错项。

使用规则：
- 每完成一个 `Task Statement（任务陈述）`，立即补充该节要点
- 优先记录“什么一定不能做”的反模式
- 牢记本域偏好：
  - 交易性事实不能被摘要化
  - 错误不能静默吞掉
  - 冲突来源不能强行二选一

---

## Domain 5 结构总览

### 5.1 Context preservation（上下文保真）
- `progressive summarisation trap（渐进式摘要陷阱）` 会丢失金额、日期、百分比和客户预期
- 交易性事实应抽成 `persistent case facts block（持久案例事实块）`
- `lost in the middle（中间丢失效应）` 需要通过把关键结论放在前面、清晰分节来缓解
- 工具结果应先裁剪再进上下文

### 当前笔记

#### 5.1 Context preservation（上下文保真）

**progressive summarisation trap（渐进式摘要陷阱）**
- 把详细历史压缩成摘要时，最容易丢失：
  - 金额
  - 日期
  - 百分比
  - 客户明确期待
- 错误示例：
  - `Customer wants a refund of $247.83 for order #8891 placed on March 3rd`
  - 被压成：
  - `customer wants a refund for a recent order`

**正确修法**
- 把交易性事实抽成 `persistent case facts block（持久案例事实块）`
- 每轮 prompt（提示） 都带上
- 不要摘要化这个 facts block（事实块）

**lost in the middle（中间丢失效应）**
- 模型对开头和结尾更稳定
- 中间埋得太深的重要信息更容易被忽略
- 修法：
  - 关键结论放在开头
  - 使用明确 section headers（章节标题）

**tool result trimming（工具结果裁剪）**
- 工具返回 40 个字段，不等于都要进上下文
- 先裁成当前任务真正需要的字段，再追加进历史

**full history requirement（完整历史要求）**
- 后续 API 请求必须携带完整会话历史
- 缺早期消息会破坏会话连贯性

**upstream agent optimisation（上游代理优化）**
- 上游代理尽量返回：
  - key facts（关键事实）
  - citations（引用）
  - relevance scores（相关性分数）
- 避免长篇 reasoning chains（推理链）挤占下游上下文预算

**一句话记忆**
- 事实块永不摘要，冗余结果先进门裁剪

**考试标准补充**
- `case facts block（案例事实块）` 存的是精确交易性事实，不是普通摘要
- 工具结果只保留当前任务真正需要的字段

---

## 待补充章节

---

## 模拟测验记录

### Domain 5 第一次 6 题模拟测验
- 得分：`6/6`
- 结果判断：达到考试准备标准（`5+/6`）
- 当前结论：Domain 5 已达到可上考场水平

**正确率说明**
- `5.1 Context preservation（上下文保真）`：掌握
- `5.2 Escalation and ambiguity resolution（升级与歧义处理）`：掌握
- `5.3 Error propagation（错误传播）`：掌握
- `5.4 Codebase exploration（代码库探索）`：掌握
- `5.5 Human review and confidence calibration（人工复核与置信度校准）`：掌握
- `5.6 Information provenance（信息来源追踪）`：掌握

**一句话总结**
- 事实块别摘要，升级看边界不看情绪，错误别静默吞，探索要防退化，准确率要分层看，来源映射要跟主张绑定。

#### 5.2 Escalation and ambiguity resolution（升级与歧义处理）

**三类可靠升级触发**
- `Customer explicitly requests a human（客户明确要求人工）`
- `Policy exceptions or gaps（政策例外或空白）`
- `Inability to make meaningful progress（无法取得有意义进展）`

**两类不可靠触发**
- `sentiment-based escalation（基于情绪的升级）`
- `self-reported confidence（模型自报置信度）`

**frustration nuance（情绪细粒度处理）**
- 客户情绪强烈但问题简单：
  - 先承认不便并提供解决路径
- 客户明确说要人工：
  - 立即升级，不先调查

**ambiguous customer matching（模糊客户匹配）**
- 多个账户匹配时：
  - 不得按启发式猜
  - 应要求 `additional identifiers（补充标识）`

**一句话记忆**
- 先看明确偏好与政策边界，不看情绪和模型自信

#### 5.3 Error propagation（错误传播）

**结构化错误上下文**
- `failure type（失败类型）`
- `attempted action（已尝试动作）`
- `partial results（部分结果）`
- `alternative approaches（可替代路径）`

**两大反模式**
- `silent suppression（静默吞错）`
- `workflow termination（单点失败终止整条链）`

**access failure（访问失败） vs valid empty result（有效空结果）**
- 访问失败：
  - 未成功触达数据源
  - 可考虑重试
- 有效空结果：
  - 成功触达数据源但无匹配
  - 不应重试

**coverage annotations（覆盖标注）**
- synthesis（综合）阶段要显式标注：
  - 哪些部分证据充分
  - 哪些部分存在覆盖缺口
  - 缺口原因是什么

**一句话记忆**
- 错误要带恢复上下文，缺口要显式说明

#### 5.4 Codebase exploration（代码库探索）

**context degradation（上下文退化）**
- 长时间探索会话后，模型容易从引用具体文件/类/函数退化成引用“典型模式”
- 原因通常是：
  - 冗长探索输出堆积
  - 关键发现被上下文淹没
  - 早期事实不再容易被稳定调用

**缓解手段**
- `scratchpad files（草稿 / 发现文件）`
  - 把关键发现写到稳定文件中
- `subagent delegation（子代理委派）`
  - 分头探索，主代理只保留高层协调
- `summary injection（摘要注入）`
  - 阶段结束后注入结构化摘要进入下一阶段
- `/compact（压缩）`
  - 在会话膨胀时减少上下文占用

**crash recovery（崩溃恢复）**
- 用 `manifest（状态清单）` 或类似结构化状态文件保存：
  - 当前阶段
  - 已完成任务
  - 已确认关键发现
  - 待继续问题
- 恢复时由 coordinator（协调代理） 读取并重新注入上下文

**一句话记忆**
- 关键发现外部化，探索任务分散化，恢复状态结构化

#### 5.5 Human review and confidence calibration（人工复核与置信度校准）

**aggregate metrics trap（总体指标陷阱）**
- `overall accuracy（总体准确率）` 可能掩盖某些文档类型或字段上的高错误率
- 不能只看总体平均值

**正确验证方式**
- 按 `document type（文档类型）` 分层
- 按 `field segment（字段分段）` 分层
- 识别哪类文档、哪类字段最脆弱

**stratified random sampling（分层随机抽样）**
- 不能只抽低置信度项
- 也必须抽高置信度样本，以发现“高置信度地做错”的情况

**field-level confidence（字段级置信度）**
- 比整份文档一个总 `confidence（置信度）` 更有用
- 可支持更细粒度的人审路由

**confidence calibration（置信度校准）**
- 阈值必须用 `labelled validation set（带标注验证集）` 校准
- 不能拍脑袋设阈值

**一句话记忆**
- 总体分高不等于局部可靠；高置信度也要抽查

#### 5.6 Information provenance（信息来源追踪）

**structured claim-source mappings（结构化主张-来源映射）**
- 每条 finding（发现）应保留：
  - `claim（主张）`
  - `source URL（来源 URL）`
  - `document name（文档名）`
  - `relevant excerpt（相关摘录）`
  - `publication/data date（发布日期 / 数据日期）`
- 不能只保留一个“来源列表”

**为什么必须结构化保留**
- 摘要或综合时，如果 claim（主张） 与 source（来源） 不绑定：
  - attribution（归因）会消失
  - 无法验证具体结论来自哪里
  - 下游无法可靠合并来源

**冲突处理**
- 两个可信来源冲突时：
  - 不要任意选一个
  - 保留两者的值及其来源

**temporal awareness（时间维度意识）**
- `publication date（发布日期）` 或 `data collection date（数据采集日期）` 常能解释数字差异
- 很多差异不是矛盾，而是统计时点不同

**content-appropriate rendering（按内容类型选择呈现方式）**
- `financial（财务）` -> `tables（表格）`
- `news（新闻）` -> `prose（叙述）`
- `technical findings（技术发现）` -> `structured lists（结构化列表）`

**一句话记忆**
- 主张和来源绑在一起走完整条链；冲突来源并存，不擅自选边

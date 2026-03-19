# Domain 5 综合练习：上下文保真与可靠性

## 练习目标
构建一个符合 `Domain 5: Context Management & Reliability（上下文管理与可靠性）` 考试思路的小型多代理方案，覆盖以下能力：

- `coordinator（协调代理）` + `two subagents（两个子代理）`
- `persistent case facts block（持久案例事实块）`
- `structured error propagation（结构化错误传播）`
- `coverage annotations（覆盖标注）`
- `conflicting sources（冲突来源）` 并存处理
- `provenance-preserving synthesis（保留来源的综合）`

---

## 一、目标场景

场景选择：`Multi-Agent Research System（多代理研究系统）`

用户请求：

> 请研究 `geothermal energy（地热能源）` 与 `solar energy（太阳能）` 的近两年成本变化，并说明哪一种更适合未来三年的基础设施投资。请保留来源，并说明任何证据缺口。

这道练习故意同时制造：
- 长上下文累积
- 子代理超时
- 冲突来源
- 归因丢失风险

---

## 二、系统结构

### 1. `Coordinator agent（协调代理）`

职责：
- 解析用户问题
- 维护 `persistent case facts block（持久案例事实块）`
- 派发子任务
- 接收子代理结果与错误
- 汇总 `claim-source mapping（主张-来源映射）`
- 在最终综合里保留 `coverage gaps（覆盖缺口）`

### 2. `Web research subagent（网页研究子代理）`

职责：
- 检索公开网页来源
- 返回结构化主张、来源、摘录、日期

### 3. `Journal lookup subagent（期刊查询子代理）`

职责：
- 查询学术或期刊型来源
- 返回结构化主张、来源、摘录、日期
- 某些练习场景下模拟 timeout（超时）

### 关键架构原则
- 所有 communication（通信） 回到 `Coordinator（协调代理）`
- 子代理不直接互相通信
- 最终 synthesis（综合） 由 `Coordinator（协调代理）` 完成

---

## 三、Persistent case facts block（持久案例事实块）

### 为什么要有
如果只做 `progressive summarisation（渐进式摘要）`，下面这些信息会越来越模糊：
- 研究主题
- 时间窗口
- 比较对象
- 用户明确要求保留来源
- 用户要求指出证据缺口

### 正确做法
把不会随着会话推进而“应该被压缩”的关键事实抽成固定 facts block（事实块）。

### 示例

```text
Case Facts
- Topic A: geothermal energy
- Topic B: solar energy
- Comparison window: last 2 years
- Decision question: better fit for infrastructure investment over next 3 years
- Output requirement: preserve source attribution
- Output requirement: explicitly note evidence gaps
```

### 规则
- 每轮 prompt（提示） 都带
- 只允许显式更新，不允许模糊摘要化
- 不埋在长段落中间

---

## 四、结构化结果格式

每个子代理都返回紧凑、结构化数据，而不是长篇 reasoning（推理）。

### finding（发现）结构

```json
{
  "claim": "Solar module costs fell by 8% year-on-year.",
  "source_url": "https://example.org/solar-report",
  "document_name": "Global Solar Outlook 2026",
  "relevant_excerpt": "Solar module costs fell 8% year-on-year in 2025.",
  "publication_date": "2026-01-15",
  "relevance_score": 0.92,
  "topic": "solar"
}
```

### 为什么这样设计
- 保持 `claim-source mapping（主张-来源映射）`
- 控制上下文体积
- 方便后续合并与冲突处理

---

## 五、模拟 timeout（超时）与 structured error propagation（结构化错误传播）

### 故意制造的失败
`Journal lookup subagent（期刊查询子代理）` 在查询 `geothermal energy（地热能源）` 时发生 timeout（超时）。

### 错误做法 1：silent suppression（静默吞错）

```json
{
  "success": true,
  "results": []
}
```

问题：
- 上游会误以为“查到了但没有结果”
- 彻底破坏恢复逻辑

### 错误做法 2：workflow termination（单点失败终止整条链）
- 因为地热期刊超时，就不再输出任何研究结果

问题：
- 丢掉已成功拿到的太阳能研究结果

### 正确做法

```json
{
  "isError": true,
  "failure_type": "transient",
  "attempted_action": "journal_search(query='geothermal energy cost change last 2 years')",
  "partial_results": [],
  "alternative_approaches": [
    "retry journal search later",
    "fallback to public policy reports"
  ],
  "message": "Journal database timed out while searching geothermal sources."
}
```

### 核心原则
- 错误要把恢复所需上下文带回来
- 不吞错
- 不因为一个点失败就整条链停掉

---

## 六、Coverage annotations（覆盖标注）

### 正确综合输出必须显式说明缺口

如果 `geothermal（地热）` 的学术来源超时了，正确的综合输出不是静默略过，而是显式写：

```text
Coverage gap: academic journal coverage for geothermal energy is limited due to database timeout; conclusions for geothermal rely primarily on public policy and industry reports.
```

### 为什么
- 让消费者知道哪些结论证据充分
- 让消费者知道哪些地方证据较弱
- 不让“省略”伪装成“完整覆盖”

---

## 七、Conflicting sources（冲突来源）

### 练习中的冲突
两份可信来源对太阳能成本变化给出不同数字：

- 来源 A：`8%` decline（下降）
- 来源 B：`11%` decline（下降）

并且：
- A 发布日期：`2025-10-01`
- B 发布日期：`2026-01-15`

### 错误做法
- 直接选一个数字
- 或者只保留更新的一条，不说明另一条存在

### 正确做法
在综合里并存保留：

```text
Source A (2025-10-01) reports an 8% year-on-year decline in solar costs.
Source B (2026-01-15) reports an 11% decline. The difference may reflect different reporting windows and publication dates.
```

### 核心原则
- 不擅自选边
- 保留双方归因
- 用 `publication date（发布日期）` 帮助解释差异

---

## 八、最终 synthesis（综合）长什么样

### 正确输出结构

1. `Case facts summary（案例事实摘要）`
2. `Well-supported findings（证据充分的结论）`
3. `Conflicting findings with attribution（带归因的冲突结论）`
4. `Coverage gaps（覆盖缺口）`
5. `Investment interpretation（投资解读）`

### 示例骨架

```text
Case facts
- Compare geothermal vs solar over the last 2 years
- Preserve source attribution

Well-supported findings
- Solar costs declined according to multiple recent reports...

Conflicting findings
- Report A states 8% decline...
- Report B states 11% decline...

Coverage gaps
- Geothermal journal evidence is limited due to timeout...

Investment interpretation
- Solar currently has stronger accessible evidence base...
```

### 为什么这样组织
- 重要事实放前面，避免 `lost in the middle（中间丢失）`
- 结论、冲突、缺口分开展示
- attribution（归因） 不会在 prose（叙述）里丢失

---

## 九、这道题在考什么

### `5.1 Context preservation（上下文保真）`
- `case facts block（案例事实块）` 永不摘要
- 上下游输出要紧凑结构化

### `5.2 Escalation and ambiguity resolution（升级与歧义处理）`
- 本练习不以客户升级为主，但可类比“无法取得有意义进展”时的人工兜底逻辑

### `5.3 Error propagation（错误传播）`
- timeout（超时） 必须结构化传播
- 保留 `partial results（部分结果）` 与 `alternative approaches（替代路径）`

### `5.4 Codebase exploration（代码库探索）`
- 本练习不直接考代码探索，但同样体现“压缩冗余上下文，保留关键结构”

### `5.5 Human review and confidence calibration（人工复核与置信度校准）`
- 可扩展加入 field/topic confidence（字段/主题置信度） 作为人工复核优先级信号

### `5.6 Information provenance（信息来源追踪）`
- `claim-source mapping（主张-来源映射）`
- 冲突来源并存
- 时间维度解释差异

---

## 十、一句话标准答案

用 `Coordinator（协调代理）` 维护永不摘要的 `case facts block（案例事实块）`，让两个子代理返回带来源映射的紧凑结构化结果；当一个子代理 timeout（超时） 时，通过 `structured error propagation（结构化错误传播）` 返回失败类型、已尝试动作、部分结果和替代路径；最终综合里同时保留 `coverage gaps（覆盖缺口）` 与 `conflicting sources（冲突来源）` 的 attribution（归因），而不是静默省略或强行二选一。

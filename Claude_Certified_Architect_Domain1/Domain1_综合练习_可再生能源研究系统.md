# Domain 1 综合练习：Renewable Energy Technologies（可再生能源技术）研究系统

## 练习目标
构建一个符合 `Domain 1: Agentic Architecture & Orchestration（代理式架构与编排）` 考试思路的小型 `Multi-Agent Research System（多代理研究系统）`，覆盖以下核心能力：

- `Coordinator agent（协调代理）`
- `Web Search subagent（网页搜索子代理）`
- `Document Analysis subagent（文档分析子代理）`
- `context passing（上下文传递）` 与 `structured metadata（结构化元数据）`
- `programmatic prerequisite gate（程式化前置门禁）`
- `PostToolUse hook（工具后钩子）`
- `multi-concern request（多问题请求）`

---

## 一、目标场景

用户请求：

> 请研究 `renewable energy technologies（可再生能源技术）` 的现状与前景，覆盖 `solar（太阳能）`、`wind（风能）`、`geothermal（地热）`、`tidal（潮汐）`、`biomass（生质能）`。同时结合我上传的行业报告，给出结构化结论，并标明高可信度来源。

---

## 二、正确架构

### 1. Coordinator agent（协调代理）

职责：
- 解析用户请求
- 做 `task decomposition（任务拆解）`
- 决定调用哪些子代理
- 将上下文显式传递给子代理
- 聚合研究结果
- 检查覆盖缺口
- 在最终输出前执行 `source quality gate（来源质量门禁）`

### 2. Web Search subagent（网页搜索子代理）

职责：
- 搜集公开网页资料
- 按主题产出结构化研究结果
- 返回 `claim-source mapping（主张-来源映射）`

### 3. Document Analysis subagent（文档分析子代理）

职责：
- 读取上传的 PDF 或报告
- 抽取与指定主题相关的证据
- 返回带页码与文档名的结构化证据

### 架构要点
- 只有两种 `subagent definition（子代理定义）`
- 但可以按主题并行生成多个子任务实例
- 不单独设置 `Synthesis agent（综合代理）`
- 最终综合由 `Coordinator（协调代理）` 完成

---

## 三、任务拆解策略

这里同时存在两层拆解：

### 1. capability decomposition（能力拆解）
- `Web Search（网页搜索）`
- `Document Analysis（文档分析）`

### 2. topic decomposition（主题拆解）
- `solar（太阳能）`
- `wind（风能）`
- `geothermal（地热）`
- `tidal（潮汐）`
- `biomass（生质能）`

正确做法不是让一个子代理“一次性研究所有可再生能源”，而是由 `Coordinator（协调代理）` 组合这两层拆解，例如：

- `Web Search subagent` 查 `solar`
- `Web Search subagent` 查 `wind`
- `Web Search subagent` 查 `geothermal`
- `Document Analysis subagent` 抽取报告中关于 `solar`
- `Document Analysis subagent` 抽取报告中关于 `wind`

这样既避免 `narrow decomposition failure（狭窄拆解失败）`，也减少 `attention dilution（注意力稀释）`。

---

## 四、数据流与上下文传递

### 步骤 1：Coordinator（协调代理）解析请求

输入：
- 用户目标
- 研究范围
- 上传文档列表
- 输出质量要求

输出：
- 主题列表
- 子任务计划
- 统一输出格式

### 步骤 2：并行调用子代理

`Coordinator（协调代理）` 在同一轮里发出多个 `Task tool（Task 工具）` 调用。

示意：
- Web Search / solar
- Web Search / wind
- Web Search / geothermal
- Web Search / tidal
- Web Search / biomass
- Document Analysis / report-A / solar
- Document Analysis / report-A / wind
- Document Analysis / report-A / geothermal

### 步骤 3：子代理返回结构化数据

不允许只返回自然语言摘要。必须返回带 `metadata（元数据）` 的结构化对象。

示例：

```json
{
  "topic": "solar",
  "claim": "Utility-scale solar costs continued to decline in 2025.",
  "source_type": "web",
  "source_title": "IEA Solar Outlook 2025",
  "source_url": "https://example.org/solar-outlook",
  "document_name": null,
  "page_number": null,
  "evidence_excerpt": "Utility-scale solar costs fell a further 8% year-on-year.",
  "source_quality_score": 0.93,
  "published_at": "2025-11-03T00:00:00Z"
}
```

文档型示例：

```json
{
  "topic": "geothermal",
  "claim": "Geothermal deployment remains constrained by drilling cost.",
  "source_type": "document",
  "source_title": "Global Renewable Infrastructure Report",
  "source_url": null,
  "document_name": "global_renewables_2026.pdf",
  "page_number": 47,
  "evidence_excerpt": "High drilling cost remains the primary constraint on geothermal expansion.",
  "source_quality_score": 0.88,
  "published_at": "2026-01-15T00:00:00Z"
}
```

### 关键原则
- 子代理不会自动知道前文
- 所有必要资讯必须显式传递
- `content（内容）` 与 `metadata（元数据）` 必须同时传递
- 最终综合缺少引用时，优先检查 `context passing（上下文传递）`

---

## 五、Programmatic Prerequisite Gate（程式化前置门禁）

这次练习选择的是 `Source quality gate（来源质量门禁）`。

### 规则
只有当某条研究结论满足最低来源质量门槛时，才能进入最终报告。

例如：

```text
if source_quality_score < 0.80:
    block inclusion in final report
```

### 触发点
在 `Coordinator（协调代理）` 准备调用 `finalise_report（定稿报告）` 或 `publish_report（发布报告）` 前检查。

### 为什么这是 gate（门禁）
- 它是业务控制规则
- 不满足条件就直接阻断
- 不是让模型“尽量注意来源质量”

### 如何实现
可以通过 `PreToolUse Hook（工具调用前钩子）` 拦截 `finalise_report`：

```text
if any(claim.source_quality_score < 0.80 for claim in draft_report.claims):
    block tool call
    return "Report blocked: low-quality sources detected."
```

---

## 六、PostToolUse Hook（工具后钩子）

### 用途
标准化不同工具的返回格式，降低模型推理负担。

### 常见标准化项
- `Unix timestamp（Unix 时间戳）` 转 `ISO 8601（标准时间格式）`
- `status_code（状态码）` 转文本标签
- 字段名统一为：
  - `source_title`
  - `source_url`
  - `published_at`
  - `source_quality_score`

### 示例

工具 A 返回：

```json
{
  "url": "https://example.org/report",
  "ts": 1762128000,
  "score": 0.91
}
```

经过 `PostToolUse hook（工具后钩子）` 后变为：

```json
{
  "source_url": "https://example.org/report",
  "published_at": "2025-11-03T00:00:00Z",
  "source_quality_score": 0.91
}
```

---

## 七、Multi-concern Request（多问题请求）

测试请求不要只给单一任务，而要故意加入多个 concern（关注点）。

### 示例请求

> 请研究 `renewable energy technologies（可再生能源技术）` 的现状与前景，重点覆盖 `solar（太阳能）`、`wind（风能）`、`geothermal（地热）`、`tidal（潮汐）`、`biomass（生质能）`。另外，请结合我上传的报告，指出哪些技术最适合未来五年的基础设施投资，并且只保留高可信度来源。

这个请求包含三个 concern（关注点）：
- 覆盖多个技术主题
- 使用上传文档作为额外证据
- 最终结论必须经过来源质量过滤

---

## 八、标准执行流程

1. `Coordinator（协调代理）` 解析用户请求，识别主题与输出要求
2. 生成按主题拆分的并行 `Task tool（Task 工具）` 调用
3. `Web Search subagent（网页搜索子代理）` 与 `Document Analysis subagent（文档分析子代理）` 返回结构化结果
4. `PostToolUse hook（工具后钩子）` 统一标准化字段
5. `Coordinator（协调代理）` 聚合结果，检查覆盖缺口
6. 如缺少某主题，例如 `tidal（潮汐）`，则发起补查
7. 在定稿前执行 `Source quality gate（来源质量门禁）`
8. 仅保留达标证据进入最终报告
9. 输出带 `claim-source mapping（主张-来源映射）` 的最终结果

---

## 九、考试映射

这套练习对应的考点：

- `1.1 Agentic loops（代理循环）`
  - 多轮工具调用与结果回写
- `1.2 Multi-agent orchestration（多代理编排）`
  - `Coordinator（协调代理）` 居中调度
- `1.3 Subagent invocation and context passing（子代理调用与上下文传递）`
  - 显式传递 `structured metadata（结构化元数据）`
- `1.4 Workflow enforcement and handoff（工作流强制执行与交接）`
  - 使用 `programmatic prerequisite gate（程式化前置门禁）`
- `1.5 Agent SDK hooks（Agent SDK 钩子）`
  - 使用 `PreToolUse Hook（工具调用前钩子）` 与 `PostToolUse hook（工具后钩子）`
- `1.6 Task decomposition strategies（任务拆解策略）`
  - 同时使用能力拆解与主题拆解
- `1.7 Session state and resumption（会话状态与恢复）`
  - 若研究主题或文档变更，应对变更范围做 `targeted re-analysis（定向重分析）`

---

## 十、一句话总结构图

`Coordinator（协调代理）` 按主题并行调用 `Web Search（网页搜索）` 与 `Document Analysis（文档分析）`，所有结果以带 `metadata（元数据）` 的结构化格式回流，经 `PostToolUse hook（工具后钩子）` 标准化，再由 `Source quality gate（来源质量门禁）` 过滤后进入最终综合。

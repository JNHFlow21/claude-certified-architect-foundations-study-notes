# Claude Certified Architect（基础）Domain 3 学习知识点笔记

## 文档用途
这份文档专门记录 `Domain 3: Claude Code Configuration & Workflows（Claude Code 配置与工作流）` 的核心知识点、考试陷阱、实战场景与易错项。

使用规则：
- 每完成一个 `Task Statement（任务陈述）`，立即补充该节要点
- 这一个 domain（领域）高度依赖文件位置与配置项，优先记录“放在哪里”和“何时使用”
- 牢记本域偏好：
  - 配置层级必须放对
  - 能直接执行就别过度规划
  - CI 里必须非交互运行

---

## Domain 3 结构总览

### 3.1 CLAUDE.md hierarchy（CLAUDE.md 层级体系）
- `user-level（用户级）`：`~/.claude/CLAUDE.md`
- `project-level（项目级）`：`.claude/CLAUDE.md` 或仓库根 `CLAUDE.md`
- `directory-level（目录级）`：子目录里的 `CLAUDE.md`
- 新成员拿不到团队规则时，优先怀疑规则被放在 `user-level（用户级）`

### 当前笔记

#### 3.1 CLAUDE.md hierarchy（CLAUDE.md 层级体系）

**三层规则**
- `user-level（用户级）`
  - 路径：`~/.claude/CLAUDE.md`
  - 仅个人生效
  - 不共享，不进版本控制
- `project-level（项目级）`
  - 路径：`.claude/CLAUDE.md` 或根 `CLAUDE.md`
  - 团队共享，进版本控制
- `directory-level（目录级）`
  - 路径：某子目录下的 `CLAUDE.md`
  - 仅在该目录上下文生效

**考试高频陷阱**
- 新成员拿不到规则时，先查团队规范是否被错误地放在 `user-level（用户级）`

**模块化组织**
- 可用 `@import（导入）` 引用外部规则文件
- `.claude/rules/` 适合按主题拆分规则

**调试工具**
- `/memory（内存）` 用于查看实际加载了哪些记忆文件

#### 3.2 Custom slash commands and skills（自定义斜杠命令与技能）

**目录位置**
- 项目共享命令：`.claude/commands/`
- 个人命令：`~/.claude/commands/`
- 项目技能：`.claude/skills/`
- 个人技能：`~/.claude/skills/`

**边界**
- `CLAUDE.md（CLAUDE.md）`：总是加载，放通用标准
- `skills（技能）`：按需调用，放任务型工作流

**高频前置元数据**
- `context: fork（上下文：分叉）`
- `allowed-tools（允许工具）`
- `argument-hint（参数提示）`

#### 3.3 Path-specific rules（路径特定规则）

**基本形式**
- 放在 `.claude/rules/`
- 使用带 `paths（路径）` 的 `YAML frontmatter（YAML 前置元数据）`

**优势**
- 用 glob（通配模式）跨整个代码库命中特定文件类型
- 比 `directory-level（目录级）` `CLAUDE.md` 更适合分散在多目录的同类文件
- 只在命中文件时加载，更省 token（令牌）

#### 3.4 Plan mode vs direct execution（规划模式 vs 直接执行）

**何时用 `Plan mode（规划模式）`**
- 大规模改动
- 存在多个合理方案
- 需要架构决策
- 涉及大量文件且存在不确定性
- 需要先探索代码库再决定方案

**何时用 `Direct execution（直接执行）`**
- 范围小且清楚
- 单文件、单函数修复
- 正确做法已知

**Explore subagent（探索子代理）**
- 用来隔离冗长探索输出
- 返回摘要，避免主会话被噪音污染

**考试偏好**
- 常见最佳实践是：
  - `Plan mode（规划模式）` 做调查与设计
  - `Direct execution（直接执行）` 落地实施

#### 3.5 Iterative refinement（迭代式细化）

**技术优先级**
- `concrete input/output examples（具体输入输出示例）` 优于 `prose descriptions（纯文字描述）`
- 可测试任务优先考虑 `test-driven iteration（测试驱动迭代）`
- 不熟领域可使用 `interview pattern（访谈式提问）`

**反馈方式**
- `batch feedback（批量反馈）`
  - 适合相互影响的问题
- `sequential feedback（顺序反馈）`
  - 适合彼此独立的问题

**考试高频结论**
- 当模型反复误解文字描述时，第一步先换成具体前后示例
- 先做低成本、高杠杆修正，不要先复杂化工作流

#### 3.6 CI/CD integration（CI/CD 集成）

**必背参数**
- 在 CI 中运行 Claude Code 时，必须使用 `-p（打印模式）`
- 否则会进入 `non-interactive（非交互）` 以外的等待输入状态，导致 job（作业）挂住

**结构化输出**
- 可使用 `--output-format json（输出格式 JSON）`
- 配合 `--json-schema（JSON Schema）`
- 让下游自动化系统稳定消费结果

**会话隔离**
- 生成代码的同一 `session（会话）` 不适合审查自己刚生成的代码
- 应使用独立 `review instance（审查实例）`

**增量审查**
- 新 commit（提交） 到来后，重新审查时应带上旧 findings（发现）
- 并要求只报告：
  - 新问题
  - 仍未解决的问题

**CLAUDE.md 在 CI 中的作用**
- CI 中调用的 Claude Code 同样会受 `CLAUDE.md（CLAUDE.md）` 影响
- 应在其中写清：
  - `testing standards（测试标准）`
  - `valuable test criteria（高价值测试标准）`
  - `available fixtures（可用测试夹具）`

**一句话记忆**
- CI 里先记住 `-p`；生成和审查要分会话

---

## 待补充章节

---

## 模拟测验记录

### Domain 3 第一次 8 题模拟测验
- 得分：`8/8`
- 结果判断：达到考试准备标准（`7+/8`）
- 当前结论：Domain 3 已达到可上考场水平

**正确率说明**
- `3.1 CLAUDE.md hierarchy（CLAUDE.md 层级体系）`：掌握
- `3.2 Custom slash commands and skills（自定义斜杠命令与技能）`：掌握
- `3.3 Path-specific rules（路径特定规则）`：掌握
- `3.4 Plan mode vs direct execution（规划模式 vs 直接执行）`：掌握
- `3.5 Iterative refinement（迭代式细化）`：掌握
- `3.6 CI/CD integration（CI/CD 集成）`：掌握

**一句话总结**
- 规则放对层级，工作流放对载体；复杂任务先规划，CI 必须用 `-p（打印模式）`。

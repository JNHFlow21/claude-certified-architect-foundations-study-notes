# Domain 3 综合练习：Claude Code 配置与工作流

## 练习目标
构建一个符合 `Domain 3: Claude Code Configuration & Workflows（Claude Code 配置与工作流）` 考试思路的小型项目配置方案，覆盖以下能力：

- `CLAUDE.md hierarchy（CLAUDE.md 层级体系）`
- `project-level（项目级）` 与 `directory-level（目录级）` 规则分层
- `.claude/rules/` 中的 `path-specific rules（路径特定规则）`
- 自定义 `skill（技能）`，并配置 `context: fork（上下文：分叉）`
- CI 中使用小写 `-p（打印模式）`
- `JSON output（JSON 输出）` 供自动化系统消费

---

## 一、目标项目场景

项目包含以下结构：

```text
repo/
  CLAUDE.md
  .claude/
    rules/
      tests.md
      api.md
    skills/
      review-brainstorm/
        SKILL.md
  services/
    api/
      CLAUDE.md
      users.ts
      users.test.ts
  web/
    components/
      card.tsx
      card.test.tsx
  scripts/
    ci-review.sh
```

这个项目同时包含：
- 团队通用规范
- API 子目录特有规则
- 跨全仓的测试文件规范
- 一个高噪音分析型 skill（技能）
- 一条 CI 审查脚本

---

## 二、CLAUDE.md hierarchy（CLAUDE.md 层级体系）

### 1. project-level（项目级） `CLAUDE.md`

放在仓库根目录：

`repo/CLAUDE.md`

### 这里放什么
- 团队通用命名规范
- 通用测试质量要求
- 代码审查优先级
- 通用 PR（拉取请求）修改原则

### 示例内容

```md
# Project Standards

- Prefer explicit naming for public APIs.
- Add tests for behavioural changes, not boilerplate-only tests.
- Preserve existing fixture patterns before introducing new fixtures.
- For risky multi-file changes, investigate before editing.
```

### 为什么放这里
- 这是 `project-level（项目级）`
- 进入版本控制
- 新成员 clone（克隆）仓库即可获取

---

### 2. directory-level（目录级） `CLAUDE.md`

放在：

`repo/services/api/CLAUDE.md`

### 这里放什么
- API 子目录特有命名规范
- 错误返回格式
- 请求校验约定

### 示例内容

```md
# API Module Rules

- Use snake_case only in wire formats, never in internal TypeScript variables.
- Return structured error objects with code, message, and retryability.
- Validate request inputs before business logic execution.
```

### 为什么不用根 `CLAUDE.md`
- 因为这些规则只适用于 `services/api/`
- 不应污染 `web/` 或其他目录

---

## 三、.claude/rules/ 与 path-specific rules（路径特定规则）

这部分用来解决：
- 测试文件分散在整个仓库
- API 文件分散在特定路径模式

### 1. 测试规则

文件：

`repo/.claude/rules/tests.md`

内容：

```md
---
paths: ["**/*.test.ts", "**/*.test.tsx", "**/*.spec.ts", "**/*.spec.tsx"]
---

# Test File Rules

- Prefer behaviour-focused assertions over snapshot-only coverage.
- Reuse existing fixtures when possible.
- Avoid asserting internal implementation details unless required for regression coverage.
```

### 为什么这样做
- 测试文件可能散落在 50+ 个目录
- 用 glob（通配模式）一次命中全部
- 比在每个目录放 `CLAUDE.md` 更好维护
- 只在命中测试文件时加载，更省 token（令牌）

---

### 2. API 文件规则

文件：

`repo/.claude/rules/api.md`

内容：

```md
---
paths: ["services/api/**/*.ts"]
---

# API File Rules

- Preserve response schema compatibility unless the task explicitly changes it.
- Prefer shared validation helpers before introducing new ad hoc checks.
- Document any new error code in the module-level API docs.
```

### 为什么这样做
- 这是按文件类型和路径模式生效
- 比写死在根 `CLAUDE.md` 更精确

---

## 四、自定义 skill（技能）

### 目标
创建一个高噪音、探索型的 skill（技能），用于复杂代码审查前的脑暴与影响面梳理。

### 位置

`repo/.claude/skills/review-brainstorm/SKILL.md`

### 关键配置

```md
---
name: review-brainstorm
description: Explore risky code changes before implementation.
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: "<area-or-change-summary>"
---

# Review Brainstorm Skill

Use this skill to inspect a risky or broad change before editing.

Steps:
1. Identify relevant entry points.
2. Search for callers and tests.
3. Summarise likely risk areas.
4. Return only a concise action-oriented summary.
```

### 为什么这是标准答案
- `context: fork（上下文：分叉）`
  - 隔离冗长探索输出
- `allowed-tools（允许工具）`
  - 限制为只读工具，防止误改
- `argument-hint（参数提示）`
  - 调用 skill（技能）时若未给参数，提示开发者输入目标范围

---

## 五、CI script（CI 脚本）

### 目标
让 Claude Code 在 CI 中做非交互式 PR 审查，并输出机器可解析 JSON。

### 文件

`repo/scripts/ci-review.sh`

### 示例脚本

```bash
#!/usr/bin/env bash
set -euo pipefail

claude -p "Review this pull request for correctness, regressions, and missing tests." \
  --output-format json \
  --json-schema ./schemas/review-findings.schema.json
```

### 为什么必须这样写
- `-p（打印模式）`
  - 必须使用
  - 否则 CI 会等待交互输入并挂住
- `--output-format json（输出格式 JSON）`
  - 让结果可被自动化系统消费
- `--json-schema（JSON Schema）`
  - 保证结构稳定，方便下游发布 PR 评论

---

## 六、这道综合练习在考什么

### 3.1 CLAUDE.md hierarchy（CLAUDE.md 层级体系）
- 团队通用规范放 `project-level（项目级）`
- 局部模块规则放 `directory-level（目录级）`

### 3.2 Commands and skills（命令与技能）
- `skills（技能）` 是按需调用的任务工作流
- 不是全局规范容器

### 3.3 Path-specific rules（路径特定规则）
- 跨全仓文件类型规则应用 glob（通配模式）
- 不要为测试规则在每个目录都复制 `CLAUDE.md`

### 3.4 Plan mode vs direct execution（规划模式 vs 直接执行）
- 这个 skill（技能） 体现的是探索先行、主会话保持干净

### 3.5 Iterative refinement（迭代式细化）
- 虽然本练习不直接考 examples（示例） 技巧，但 `review-brainstorm` 用于在执行前先澄清影响面

### 3.6 CI/CD integration（CI/CD 集成）
- `-p（打印模式）` 必背
- 审查结果应使用结构化 JSON 输出

---

## 七、标准验收点

### 验收 1：新成员一致性
- 新成员 clone（克隆） 项目后，能自动获取项目级与目录级规则

### 验收 2：测试规则命中
- 编辑任意目录下的 `*.test.tsx` 文件时，都会加载测试规则

### 验收 3：API 规则命中
- 编辑 `services/api/**/*.ts` 时，自动加载 API 路径规则

### 验收 4：skill（技能） 行为
- 调用 `review-brainstorm` 时，冗长探索留在 forked context（分叉上下文）
- 主会话只收到摘要

### 验收 5：CI 行为
- CI 运行 Claude Code 不挂住
- 输出为结构化 JSON

---

## 八、一句话标准答案

用根 `CLAUDE.md` 承载团队通用规范，用目录级 `CLAUDE.md` 承载 API 模块特有规则；把测试与 API 文件规则写入 `.claude/rules/` 并用 glob（通配模式）精确命中；把高噪音分析流程做成带 `context: fork（上下文：分叉）` 的 skill（技能）；在 CI 里用小写 `-p（打印模式）` 加 JSON 输出，确保非交互执行与结构化结果。

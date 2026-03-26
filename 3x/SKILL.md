---
name: 3x
description: "三个臭皮匠，顶个诸葛亮 — 同时启动 Claude、Codex、Gemini 三个 subagent 并行分析同一个问题，汇总三方独立意见。适用于需要多角度意见的场景：分析问题、做决策、审代码、方案评估、翻译润色等。当用户想要第二/第三意见、想对比不同 AI 的看法、或者提到'三个臭皮匠'时使用此 skill。"
---

# 三个臭皮匠

同时派出 Claude、Codex、Gemini 三路并行分析同一个问题，最后汇总三方结果。
三路全部通过后台 Bash 调用各自的 CLI，实现真正并行。

## 前置条件

需要 `claude`、`codex` 和 `gemini` 三个 CLI 工具在 PATH 中可用。

## 工作流程

### Step 1: 构造任务描述

取用户的原始问题作为 `{task}`。如果用户提供了 `$ARGUMENTS`，将其作为 `{task}`。

判断两个维度：
- **是否代码相关**：涉及代码分析、审查、Bug 排查、架构设计等 → 需要指定项目路径和代码只读工具
- **是否需要联网**：需要查询最新信息、调研方案、搜索文档等 → 需要开放联网工具

获取当前工作目录 `{cwd}`。

默认启动全部三路（Claude + Codex + Gemini）。如果用户明确指定了参与者（如"只问 Claude 和 Gemini"），则只启动指定的。

### Step 2: 同一消息中启动三路后台任务

在**同一条回复**中发出 3 个工具调用，全部使用 `run_in_background: true`，实现真正并行。
后台任务的权限请求会正常弹出，用户可以正常批准。
三路统一用管道传入 prompt，避免 task 中的引号、`$` 等特殊字符被 shell 解析。

**任务 1 — Claude（后台 Bash）**

通过 `claude -p` CLI 运行。
Claude CLI 默认不开放任何工具，需通过 `--allowedTools` 显式授权。按需组合：
- 联网：`WebSearch,WebFetch`
- 代码只读：`Read,Grep,Glob,Bash(git diff:*),Bash(git log:*),Bash(git show:*)`

```
Bash tool:
  description: "Claude 分析"
  run_in_background: true
  timeout: 600000
  command: |
    {代码: cd "{cwd}" && }echo "{task}" | claude -p --allowedTools "{按需组合上述工具，逗号分隔}"
```

**任务 2 — Codex（后台 Bash）**

通过 `codex exec` CLI 运行。使用 `read-only` 沙盒确保安全。
默认具备联网能力。Codex 要求 git repo：代码任务用 `-C "{git repo 根目录}"`，非代码任务加 `--skip-git-repo-check` 跳过。

```
Bash tool:
  description: "Codex 分析"
  run_in_background: true
  timeout: 600000
  command: |
    echo "{task}" | codex exec {非代码: --skip-git-repo-check} {代码: -C "{cwd}"} -s read-only
```

**任务 3 — Gemini（后台 Bash）**

通过 `gemini` CLI 运行。使用 `--approval-mode yolo -s` 自动审批 + 沙盒。
Gemini 默认具备联网能力，无需额外参数。
**重要**：Gemini 没有原生的"只读但允许 shell"模式，因此在 prompt 前注入只读约束指令，防止擅自修改文件。

```
Bash tool:
  description: "Gemini 分析"
  run_in_background: true
  timeout: 600000
  command: |
    {代码: cd "{cwd}" && }printf '%s\n\n%s' '[STRICT RULE] You are in READ-ONLY analysis mode. You MUST NOT create, modify, or delete any files. Only use read operations (read_file, grep_search, list_dir, shell commands like git log/diff/show/blame, find, cat, ls, etc). If you feel the urge to write a file, output the suggestion as text instead.' '{task}' | gemini --approval-mode yolo -s -p ""
```

### Step 3: 汇总三方结果

等 3 个后台任务全部返回后，输出汇总报告：

```
## 三个臭皮匠 — 汇总报告

### Claude 的意见
{摘要}

### Codex 的意见
{摘要}

### Gemini 的意见
{摘要}

### 共识与分歧

**共识**：三方都同意的点...

**分歧**：各方不同的观点...

### 综合建议
基于三方意见的最终建议...
```

重点是找出共识和分歧：
- **三方一致** → 高可信度，直接采纳
- **两方一致** → 较高可信度，值得关注
- **仅一方提出** → 可能是独特视角，需进一步验证

## 规则

- **全部后台 Bash 并行**：3 个 Bash 工具调用都使用 `run_in_background: true`，在同一条消息中发出。不要使用 Agent 工具。
- **不要预嚼食物**：不要预先收集文件内容或执行命令塞进 prompt。三个 AI 都有自己获取上下文的能力。你只需描述清楚任务和工作目录。
- **独立性**：三路任务互不知道对方的存在和结果，确保意见独立。
- **汇总要有判断**：不是简单罗列三方结果，要分析共识和分歧，给出综合建议。
- **汇总用中文**：三路 AI 可以自由选择回复语言，但最终汇总报告使用中文。
- **错误容忍**：如果某个 AI 调用失败，用剩余两个的结果继续汇总，告知用户哪个失败了。

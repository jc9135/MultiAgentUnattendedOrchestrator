# MultiAgentUnattendedOrchestrator

一个面向复杂任务的无人值守多 Agent 协作系统。

本项目的推荐使用方式是：**用户直接在 Codex 对话中布置任务，由 Codex 负责统筹、监控、验收和审批，由 Claude Code worker 负责具体实现。** CLI/API 仍然保留，但主要作为 Codex 背后的内部执行接口和调试入口。

## 核心定位

```text
用户在 Codex 对话中布置任务
  ↓
Codex 作为 Supervisor 创建 Run、拆任务、调度和监控
  ↓
Claude Code workers 并行执行具体实现任务
  ↓
Codex Reviewer 验收 diff、测试结果和产物
  ↓
通过后提交 artifact，失败则重试、拆小任务或请求人工审批
```

Codex 负责“管活”：

- 理解用户目标
- 生成或补全 TaskContract
- 调度任务 DAG
- 控制权限、预算、并发和资源锁
- 监控 Claude Code worker
- 做 Reviewer 验收
- 请求人工审批
- 生成最终报告

Claude Code 负责“干活”：

- 在指定 worktree 或 attempt 目录中实现任务
- 运行指定验证命令
- 输出结构化结果、日志、diff 和产物

## 对话式使用方式

你不需要手动执行 `orchestrator run ...`。推荐直接在 Codex 中说：

```text
按照设计文档实现 MVP 的 M1 到 M3。简单文档任务用 fast Claude，核心代码和调度逻辑用 strong Claude。
```

Codex 内部会执行：

```text
1. 创建 Run
2. 读取设计文档和当前仓库状态
3. 生成或补全任务 DAG
4. 根据 WorkerProfile 选择 Claude Code worker
5. 为每个任务创建 ExecutionAttempt
6. 启动 Claude Code 执行具体任务
7. 收集 result.json、日志、测试输出和产物
8. 执行 Reviewer 验收
9. 在对话中汇报进度或请求审批
10. 生成最终 report.md
```

常用自然语言指令示例：

```text
开始实现 MVP 第一阶段。
继续上次的 Run，看看现在到哪了。
暂停当前 Run。
只重试失败的任务。
把 T-003 换成 strong Claude 重试。
只跑测试，不改代码。
总结失败原因。
生成本次执行报告。
批准这个 L2 操作。
拒绝这个高风险操作。
```

## 多 Claude Code worker 与模型切换

系统支持自动启动多个 Claude Code worker。每个 worker 可以通过 `ccswitch` 或等价 wrapper 使用不同模型。

示例配置：

```yaml
workers:
  - id: claude-fast-1
    type: claude-code
    command: claude
    model_switch_command: "ccswitch fast"
    max_concurrency: 1
    task_types: ["docs", "simple_code", "tests"]

  - id: claude-strong-1
    type: claude-code
    command: claude
    model_switch_command: "ccswitch strong"
    max_concurrency: 1
    task_types: ["core_code", "debugging", "architecture"]

  - id: claude-review-1
    type: claude-code
    command: claude
    model_switch_command: "ccswitch review"
    max_concurrency: 1
    task_types: ["code_review"]
```

如果你的本机命令不是 `ccswitch`，可以在配置中改成实际命令，例如：

```yaml
model_switch_command: "ccswtich strong"
```

或者：

```yaml
model_switch_command: "claude-model strong"
```

模型路由建议：

- 文档、简单测试、格式整理：`claude-fast`
- 状态机、存储、调度、恢复、权限：`claude-strong`
- 失败重试后的修复：升级到 `claude-strong`
- 最终验收：默认由 Codex Reviewer 执行，可选增加 `claude-review`

## 并发与隔离规则

多个 Claude Code worker 可以并行执行，但必须由 Codex Supervisor 控制：

- 一个 worker 同一时间只处理一个 ExecutionAttempt。
- 每个任务都有独立 attempt 目录。
- 涉及代码修改的任务必须使用独立 worktree 或等价隔离目录。
- Scheduler 必须先获取 ResourceLock，再启动 worker。
- Claude Code 只返回临时产物，不直接提交最终结果。
- Artifact Manager 在 Reviewer 通过后统一提交产物。

推荐目录结构：

```text
runs/run_001/
  attempts/
    attempt_T001_1/
      task.md
      result.json
      logs.txt
  worktrees/
    T001/
    T002/
  artifacts/
  report.md
```

## 人工审批

低风险任务可以自动执行。以下操作需要审批：

- 安装依赖
- 访问外网 API
- 调用付费 API
- 修改数据库
- 删除数据
- 发布生产环境
- 任何不可恢复操作

审批通过或拒绝都通过 Codex 对话完成：

```text
批准 approval_001。
拒绝 approval_001，原因是风险太高。
```

## CLI 的角色

CLI 不是主要用户入口，但会保留给内部执行、调试和自动化集成使用。

可能的内部命令形态：

```bash
orchestrator status run_001
orchestrator report run_001
orchestrator worker run --profile claude-strong-1 --task runs/run_001/attempts/attempt_T001_1/task.md
```

日常使用时，用户优先和 Codex 对话，不需要记忆这些命令。

## 当前阶段目标

MVP 第一阶段目标：

- Codex 对话中提交任务目标
- Planner 补全受控 DAG 的 TaskContract
- Supervisor 调度任务
- mock worker 和 Claude Code worker 可执行任务
- SQLite 持久化 Run、Task、Attempt、Review、Artifact、ToolInvocation
- ToolPolicy 控制 L0/L1 自动执行
- ResourceLock 防止并行冲突
- Reviewer 基于证据验收
- 最终生成 Markdown 报告

详细设计见：

[multi_agent_unattended_orchestrator_design.md](./multi_agent_unattended_orchestrator_design.md)

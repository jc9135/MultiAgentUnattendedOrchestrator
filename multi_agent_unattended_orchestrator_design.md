# 多 Agent 协作无人值守任务系统设计文档

版本：v1.1  
日期：2026-07-01  
目标：设计并实现一套可靠、高效、可恢复、可扩展的多 Agent 协作系统，用于自动完成复杂任务。

## 1. 背景与目标

复杂任务通常包含需求理解、资料检索、方案设计、代码实现、测试验证、质量审查、文档交付等多个环节。单个 Agent 在长流程中容易出现上下文漂移、遗漏要求、无法恢复、质量不可控等问题。

本系统采用“Supervisor 状态机 + 专家 Agent 池 + 验收 Agent + Checkpoint 持久化”的方式，将复杂任务拆成可调度、可验证、可恢复的子任务。

### 1.1 核心目标

- 支持复杂任务自动拆解、调度、执行、验收和交付。
- 支持多个 Agent 并行处理互不依赖的子任务。
- 支持失败重试、任务恢复、人工审批和执行日志追踪。
- 所有关键产出必须经过独立验收，避免 Agent 自说自话式完成。
- 先实现最小可用版，再逐步扩展为完整平台。

### 1.2 非目标

- 第一阶段不做完整低代码平台。
- 第一阶段不做复杂权限系统和多租户隔离。
- 第一阶段不追求所有任务完全无人干预，而是做到低风险任务自动执行，高风险任务人工审批。
- 第一阶段不允许 Agent 直接操作生产环境、真实用户数据或不可恢复资源。

### 1.3 关键设计原则

无人值守系统的核心不是让 Agent 更自由，而是让每一步都可约束、可审计、可恢复。

- **Supervisor 唯一决策**：所有任务拆解、状态变更、权限升级、重试和交付都必须由 Supervisor 执行。
- **任务契约先于执行**：每个任务在执行前必须具备输入、输出、验收标准、资源锁、权限等级和风险等级。
- **工具权限硬约束**：Agent 不能直接决定自己可用哪些工具；工具调用必须经过 ToolPolicy 校验。
- **产物先暂存后提交**：Agent 产出先写入 attempt 级临时目录，通过 Reviewer 后再提交为正式产物。
- **恢复必须幂等**：系统重启、超时重试或 Agent 崩溃后，不得重复执行不可逆副作用。
- **证据驱动验收**：Reviewer 必须基于任务契约、产物 diff、测试输出和执行日志给出结论，不能只输出主观判断。

## 2. 总体方案

系统采用单 Supervisor 多 Agent 的架构。Supervisor 是唯一总控，负责规划、调度、状态流转、异常处理和最终交付。各专家 Agent 只负责明确、窄范围的任务。

```text
用户在 Codex 对话中提交目标
  ↓
CodexConversationController 创建 Run
  ↓
Planner Agent 生成或补全任务 DAG
  ↓
Supervisor 调度执行
  ↓
Claude Code / 本地 Agent Worker 并行处理子任务
  ↓
Spec Reviewer / Quality Reviewer / Test Agent 验收
  ↓
Integrator Agent 汇总结果
  ↓
最终交付 + 日志 + 可追溯报告
```

### 2.1 推荐交互模式

系统的主要入口应优先设计为 **Codex 对话入口**，而不是要求用户手动执行 CLI 命令。用户在 Codex 中用自然语言布置任务、查看状态、审批高风险动作和接收最终报告。CLI/API 仍然保留，但作为 Codex 背后的内部执行接口和自动化集成接口。

```text
用户与 Codex 对话
  ↓
CodexConversationController
  ↓
Supervisor / Scheduler / ToolPolicy / Reviewer
  ↓
Claude Code Worker 池
  ↓
attempt 产物、测试结果、执行日志
  ↓
Codex 在对话中汇报进度、请求审批、交付报告
```

这种模式下，Codex 负责“管活”：理解目标、拆任务、调度、监控、验收、重试、审批和报告。Claude Code 负责“干活”：在受限任务契约和隔离工作区内完成具体实现。

## 3. 推荐技术栈

### 3.1 MVP 技术栈

适合先跑通核心逻辑。

```text
语言：Python 3.11+
Agent 框架：OpenAI Agents SDK
状态编排：LangGraph
存储：SQLite
配置：YAML / dotenv
日志：structlog / 标准 logging
命令行入口：Typer / Click
对话入口：Codex Conversation
外部实现 Worker：Claude Code
Worker 模型切换：ccswitch 或等价 wrapper
```

### 3.2 工程化技术栈

适合长期运行和多人使用。

```text
后端：FastAPI
编排：LangGraph 或 Temporal
Agent：OpenAI Agents SDK
数据库：Postgres
缓存与队列：Redis + Celery / RQ / BullMQ
对象存储：S3 / MinIO
前端控制台：Next.js
观测：OpenTelemetry + Grafana
```

### 3.3 推荐落地路线

第一阶段采用：

```text
Python + LangGraph + OpenAI Agents SDK + SQLite + Codex 对话入口 + CLI 内部接口
```

原因：

- 实现成本低。
- 容易调试。
- 足够验证核心流程。
- 用户可以直接通过 Codex 对话布置任务，不需要记忆 CLI 参数。
- 后续可以平滑迁移到 Postgres、FastAPI 和任务队列。

## 4. 系统模块设计

### 4.1 Supervisor

Supervisor 是系统总控，不直接完成业务任务，只负责编排和决策。

职责：

- 接收用户目标。
- 调用 Planner 生成任务 DAG。
- 按依赖关系调度子任务。
- 选择合适的 Agent 和模型。
- 控制并发、预算、超时、重试。
- 调用 Reviewer 验收结果。
- 处理失败、阻塞和人工审批。
- 生成最终交付报告。

关键原则：

- 系统只能有一个最终决策者。
- Agent 之间不直接自由对话，必须通过 Supervisor 传递结构化结果。
- 所有任务状态必须持久化。

### 4.1.1 CodexConversationController

CodexConversationController 是面向用户的对话控制层，负责把自然语言交互转换为可持久化、可调度、可恢复的 Run。

职责：

- 接收用户在 Codex 对话中的目标描述。
- 读取项目设计文档、当前 Run 状态和历史报告。
- 创建 Run，并调用 Planner 生成或补全受控 DAG。
- 将用户后续自然语言指令映射为操作，例如暂停、继续、重试、查看状态、审批、拒绝、只跑测试、生成报告。
- 在对话中展示任务进度、失败原因、审批请求和最终交付物。
- 当 Codex 会话中断后，可从 SQLite 和 runs 目录恢复上下文。

对话示例：

```text
用户：实现 MVP 的 M1 到 M3，简单任务用 fast Claude，核心模块用 strong Claude。
Codex：已创建 run_20260701_001，拆分 8 个任务，其中 3 个并行执行，2 个需要 strong worker。
Codex：T-003 状态机实现失败 1 次，已切换 strong worker 重试。
Codex：T-005 需要安装依赖，权限等级 L2，是否批准？
用户：批准。
Codex：所有任务已 verified，报告在 runs/run_20260701_001/report.md。
```

### 4.2 Planner Agent

职责：

- 理解用户目标。
- 拆解任务。
- 定义每个任务的输入、输出、验收标准和依赖关系。
- 生成任务 DAG。

第一阶段建议支持两种模式：

- **受控模式**：用户或模板提供初始任务 DAG，Planner 只补全任务契约。MVP 优先实现该模式。
- **自动模式**：Planner 从用户目标自动拆解任务 DAG。该模式需要更多 Reviewer 和人工确认机制，放在受控模式稳定之后。

输出示例：

```json
{
  "tasks": [
    {
      "id": "T-001",
      "title": "需求澄清与成功标准定义",
      "agent_type": "planner",
      "dependencies": [],
      "risk_level": "low",
      "permission_level": "L0",
      "timeout_seconds": 600,
      "inputs": {
        "user_goal": "实现一个多 Agent 自动任务系统"
      },
      "expected_output_schema": {
        "type": "object",
        "required": ["deliverables", "constraints", "acceptance_criteria"]
      },
      "allowed_files": ["docs/**"],
      "forbidden_actions": ["delete_file", "network_write", "production_deploy"],
      "resource_locks": [
        {
          "resource_type": "file_glob",
          "resource_id": "docs/**",
          "mode": "write"
        }
      ],
      "validation_commands": [],
      "acceptance_criteria": [
        "明确最终交付物",
        "列出限制条件",
        "定义验收标准"
      ]
    },
    {
      "id": "T-002",
      "title": "资料检索与事实核验",
      "agent_type": "researcher",
      "dependencies": ["T-001"],
      "risk_level": "medium",
      "permission_level": "L2",
      "timeout_seconds": 1200,
      "inputs": {
        "questions": ["确认所选框架的当前能力边界"]
      },
      "expected_output_schema": {
        "type": "object",
        "required": ["findings", "sources", "uncertainties"]
      },
      "allowed_files": ["runs/artifacts/**"],
      "forbidden_actions": ["write_workspace_code", "delete_file", "production_deploy"],
      "resource_locks": [
        {
          "resource_type": "network_domain",
          "resource_id": "official_docs",
          "mode": "read"
        }
      ],
      "validation_commands": [],
      "acceptance_criteria": [
        "引用可信来源",
        "标注不确定信息",
        "输出结构化摘要"
      ]
    }
  ]
}
```

### 4.3 Research Agent

职责：

- 检索外部资料。
- 阅读文档、网页、论文或代码仓库。
- 做事实核验。
- 输出带来源的结构化结论。

约束：

- 不得凭记忆回答易变信息。
- 必须标注来源和时间。
- 不确定信息必须显式标记。

### 4.4 Architect Agent

职责：

- 设计系统架构。
- 设计模块边界。
- 设计数据结构和接口。
- 识别技术风险。

输出：

- 架构说明。
- 模块清单。
- 数据流。
- 接口草案。
- 风险列表。

### 4.5 Implementer Agent

职责：

- 根据任务说明完成具体实现。
- 写代码、配置、脚本或文档。
- 运行局部测试。
- 输出修改摘要和验证结果。

约束：

- 只能修改任务允许范围内的文件。
- 不得擅自扩大需求。
- 遇到歧义必须返回 `NEEDS_CONTEXT`。

### 4.6 Test Agent

职责：

- 设计测试用例。
- 执行自动化测试。
- 复现失败。
- 生成测试报告。

测试类型：

- 单元测试。
- 集成测试。
- 端到端测试。
- 回归测试。
- 关键路径冒烟测试。

### 4.7 Spec Reviewer Agent

职责：

- 检查结果是否满足原始需求。
- 检查是否遗漏验收标准。
- 检查是否做了多余功能。
- 检查 Agent 输出是否符合 `expected_output_schema`。
- 引用具体证据说明每条验收标准是否满足。

输出：

```json
{
  "passed": false,
  "summary": "未通过。实现缺少 checkpoint 恢复逻辑。",
  "issues": [
    {
      "severity": "high",
      "criterion": "状态变更必须落库",
      "message": "任务要求支持失败恢复，但实现中没有 checkpoint 逻辑",
      "evidence": ["agent_output.summary", "diff:app/orchestration/state_machine.py"]
    }
  ],
  "required_fixes": ["补充 checkpoint 写入和恢复读取逻辑"]
}
```

### 4.8 Quality Reviewer Agent

职责：

- 检查代码质量、设计质量、安全风险和可维护性。
- 检查边界条件。
- 检查异常处理。

关注点：

- 复杂度是否过高。
- 是否存在隐藏副作用。
- 是否有权限风险。
- 是否缺少测试。
- 是否存在不可恢复操作。
- 是否存在重复执行风险。
- 是否违反任务的 `allowed_files`、`forbidden_actions` 或权限等级。

### 4.9 Integrator Agent

职责：

- 汇总所有 verified 子任务。
- 做最终一致性检查，确认已提交产物之间没有残留冲突。
- 生成最终交付物。
- 生成执行报告。

约束：

- Integrator 不负责解决执行期资源冲突；资源冲突必须在调度前由 Scheduler 和 ResourceLock 处理。
- Integrator 不得合并未通过 Reviewer 的临时产物。
- 如果发现已验证任务之间存在语义冲突，必须创建修复任务并交回 Supervisor 调度。

### 4.10 ToolPolicy

ToolPolicy 是 Agent 使用工具前的硬约束层，不由 Agent 自行绕过。

职责：

- 根据任务的 `permission_level`、`risk_level`、Agent 类型和工具类型判断是否允许执行。
- 对命令、文件路径、网络域名、环境变量和执行时间做限制。
- 对 L2 及以上操作生成审计记录。
- 对 L3/L4 操作创建 HumanApproval，而不是直接执行。

### 4.11 Artifact Manager

Artifact Manager 负责产物的暂存、校验和正式提交。

核心规则：

- 每次 ExecutionAttempt 都有独立的临时产物目录。
- Agent 只能写入本次 attempt 的临时目录或任务允许的工作区范围。
- Reviewer 通过前，临时产物不得进入最终交付。
- Reviewer 通过后，由 Supervisor 调用 Artifact Manager 原子提交产物。
- 如果任务重试，新的 attempt 不得覆盖旧 attempt 的产物。

### 4.12 WorkerAdapter

WorkerAdapter 把不同执行器统一封装为可调度 worker。Supervisor 不直接依赖某一个具体 Agent 产品，而是通过 WorkerAdapter 分配任务。

MVP 建议至少支持：

- `mock`：用于测试调度链路，不调用真实外部 Agent。
- `claude-code`：调用本机 Claude Code 完成具体实现。
- `local-script`：用于执行受限脚本、测试命令或简单文件生成。

统一输入：

- TaskContract。
- ExecutionAttempt。
- attempt 临时目录。
- 允许文件列表。
- 禁止动作列表。
- validation commands。

统一输出：

```json
{
  "status": "DONE",
  "summary": "完成状态机实现并通过测试",
  "artifacts": ["runs/run_001/attempts/attempt_001/output.patch"],
  "evidence": ["pytest tests/test_state_machine.py passed"],
  "files_changed": ["orchestrator/orchestration/state_machine.py"],
  "side_effect_status": "staged"
}
```

### 4.13 ClaudeCodeWorkerAdapter

ClaudeCodeWorkerAdapter 负责自动启动 Claude Code 子进程，让 Claude Code 作为 Implementer Worker 执行具体任务。

职责：

- 根据 WorkerProfile 选择 Claude Code 命令和模型切换命令。
- 为每个任务创建独立 worktree 或隔离工作目录。
- 将 TaskContract 渲染为 `task.md`。
- 调用 Claude Code 执行任务。
- 捕获 stdout、stderr、退出码、执行耗时和生成文件。
- 要求 Claude Code 输出结构化 `result.json`。
- 将执行日志写入 ToolInvocation 和 attempt 日志。

Claude Code 不得负责：

- 修改任务范围。
- 跳过 Reviewer。
- 决定是否提交最终产物。
- 决定是否执行高风险工具。
- 与其他 worker 自由协商任务。

### 4.14 WorkerProfile 与模型切换

WorkerProfile 描述可用 worker、模型能力、并发限制和适用任务类型。

示例：

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

如果本机使用的命令名不是 `ccswitch`，例如误拼或别名为 `ccswtich`，应通过配置显式声明，不应在代码中写死。

模型路由建议：

- 文档、简单测试、格式整理：`claude-fast`。
- 状态机、存储、调度、恢复、权限：`claude-strong`。
- 失败重试后的修复：升级到 `claude-strong`。
- 最终验收：默认由 Codex Reviewer 执行，可选增加 `claude-review` 交叉复核。

并发规则：

- 一个 Claude Code worker 同一时间只处理一个 ExecutionAttempt。
- 每个 ExecutionAttempt 必须有独立 attempt 目录。
- 涉及代码修改的任务必须使用独立 worktree 或等价隔离目录。
- Scheduler 必须先获取 ResourceLock，再启动 Claude Code worker。
- worker 完成后只返回临时产物，正式提交由 Artifact Manager 完成。

## 5. 任务状态机设计

每个子任务都必须通过状态机流转。

```text
pending
  ↓
ready
  ↓
running
  ↓
completed
  ↓
spec_reviewing
  ↓
quality_reviewing
  ↓
testing
  ↓
verified
```

异常状态：

```text
blocked
failed
needs_human_approval
cancelled
```

状态机由事件驱动，任何代码路径都不得直接跳过状态转移表修改状态。

### 5.1 状态说明

| 状态 | 含义 |
|---|---|
| pending | 任务已创建，但依赖未完成 |
| ready | 依赖已完成，可以执行 |
| leased | 任务已被某个执行器领取，但 Agent 尚未开始执行 |
| running | Agent 正在执行 |
| completed | 执行完成，等待验收 |
| spec_reviewing | 正在检查是否符合需求 |
| quality_reviewing | 正在检查质量和风险 |
| testing | 正在运行测试或验证 |
| committing | 产物正在从临时区提交到正式区 |
| verified | 任务通过所有验收 |
| blocked | 缺少信息或外部条件无法继续 |
| failed | 多次重试后失败 |
| needs_human_approval | 需要人工审批 |
| cancelled | 用户或系统取消 |

### 5.2 状态转移表

| 当前状态 | 事件 | 下一个状态 | 说明 |
|---|---|---|---|
| pending | dependencies_verified | ready | 所有依赖任务已 verified |
| ready | lease_acquired | leased | Scheduler 获取任务租约和资源锁 |
| leased | agent_started | running | Agent 开始执行 |
| leased | lease_expired | ready | Agent 未启动，释放租约 |
| running | agent_done | completed | Agent 返回 DONE 或 DONE_WITH_CONCERNS |
| running | needs_context | blocked | Agent 明确需要人工补充上下文 |
| running | agent_failed | failed 或 ready | 按 RetryPolicy 判断是否重试 |
| running | heartbeat_timeout | failed 或 ready | 需要基于 attempt 幂等判断 |
| completed | spec_review_started | spec_reviewing | 开始需求验收 |
| spec_reviewing | spec_passed | quality_reviewing | 需求验收通过 |
| spec_reviewing | spec_failed | ready 或 failed | 创建修复 attempt 或失败 |
| quality_reviewing | quality_passed | testing | 质量验收通过 |
| quality_reviewing | quality_failed | ready 或 failed | 创建修复 attempt 或失败 |
| testing | tests_passed | committing | 自动验证通过 |
| testing | tests_failed | ready 或 failed | 创建修复 attempt 或失败 |
| committing | commit_succeeded | verified | 产物正式提交 |
| committing | commit_failed | failed 或 needs_human_approval | 提交失败需要人工判断风险 |
| needs_human_approval | approved | ready | 审批通过后重新进入调度 |
| needs_human_approval | rejected | blocked 或 cancelled | 审批拒绝后阻塞或取消 |
| blocked | context_supplied | ready | 补充上下文后继续 |
| ready/running/blocked | user_cancelled | cancelled | 用户或系统取消 |

### 5.3 失败处理策略

```text
第 1 次失败：同 Agent 重试
第 2 次失败：换更强模型或换 Agent
第 3 次失败：拆小任务后重试
仍失败：进入人工审批或标记 blocked
```

重试必须满足以下规则：

- 每次重试都创建新的 `ExecutionAttempt`。
- 重试不得覆盖前一次 attempt 的输入、输出、日志和产物。
- 对包含外部写操作的任务，只有确认上一次 attempt 未提交副作用，才能自动重试。
- 如果无法判断副作用是否已发生，必须进入 `needs_human_approval`。

## 6. 数据模型设计

### 6.1 Run

一次用户目标执行对应一个 Run。

```json
{
  "id": "run_20260701_001",
  "user_goal": "实现一个多 Agent 自动任务系统",
  "status": "running",
  "created_at": "2026-07-01T10:00:00+08:00",
  "updated_at": "2026-07-01T10:15:00+08:00",
  "budget": {
    "max_tokens": 1000000,
    "max_cost_usd": 20,
    "max_duration_minutes": 120
  }
}
```

### 6.2 Task

```json
{
  "id": "T-003",
  "run_id": "run_20260701_001",
  "title": "实现任务状态机",
  "description": "实现 Task 状态流转与持久化",
  "agent_type": "implementer",
  "status": "running",
  "dependencies": ["T-001", "T-002"],
  "risk_level": "medium",
  "permission_level": "L1",
  "timeout_seconds": 900,
  "inputs": {
    "spec": "..."
  },
  "expected_output_schema": {
    "type": "object",
    "required": ["files_changed", "summary", "validation_results"]
  },
  "allowed_files": [
    "app/orchestration/state_machine.py",
    "tests/test_state_machine.py"
  ],
  "forbidden_actions": [
    "delete_file",
    "network_write",
    "production_deploy"
  ],
  "resource_locks": [
    {
      "resource_type": "file",
      "resource_id": "app/orchestration/state_machine.py",
      "mode": "write"
    }
  ],
  "validation_commands": [
    "pytest tests/test_state_machine.py"
  ],
  "acceptance_criteria": [
    "支持 pending 到 verified 的状态流转",
    "支持失败重试计数",
    "状态变更必须落库"
  ],
  "attempts": 1,
  "max_attempts": 3
}
```

### 6.3 ExecutionAttempt

一次 Task 可以有多次 ExecutionAttempt。Task 表达“要做什么”，ExecutionAttempt 表达“这一次怎么做、做到了哪里”。

```json
{
  "id": "attempt_001",
  "task_id": "T-003",
  "attempt_no": 1,
  "lease_id": "lease_20260701_001",
  "agent_name": "implementer",
  "model": "gpt-5",
  "status": "completed",
  "started_at": "2026-07-01T10:05:00+08:00",
  "ended_at": "2026-07-01T10:08:00+08:00",
  "last_heartbeat_at": "2026-07-01T10:07:30+08:00",
  "idempotency_key": "run_20260701_001:T-003:1",
  "temp_artifact_dir": "runs/run_20260701_001/attempts/attempt_001/",
  "side_effect_status": "none",
  "input_tokens": 12000,
  "output_tokens": 3000,
  "cost_usd": 0.18,
  "error": null
}
```

`side_effect_status` 用于恢复和重试判断：

- `none`：没有外部副作用，可以安全重试。
- `staged`：只产生了临时产物，可以清理或重试。
- `committed`：产物或外部操作已经提交，不得自动重试。
- `unknown`：无法判断副作用状态，必须进入人工审批。

### 6.4 AgentExecution

AgentExecution 记录一次模型或工具调用的细节，可作为 ExecutionAttempt 的子记录。

```json
{
  "id": "exec_001",
  "attempt_id": "attempt_001",
  "task_id": "T-003",
  "execution_type": "model_call",
  "agent_name": "implementer",
  "model": "gpt-5",
  "status": "completed",
  "started_at": "2026-07-01T10:05:00+08:00",
  "ended_at": "2026-07-01T10:08:00+08:00",
  "input_tokens": 12000,
  "output_tokens": 3000,
  "cost_usd": 0.18
}
```

### 6.5 ReviewResult

```json
{
  "id": "review_001",
  "task_id": "T-003",
  "attempt_id": "attempt_001",
  "review_type": "spec",
  "passed": true,
  "summary": "任务满足状态机验收标准。",
  "issues": [],
  "evidence": [
    "runs/run_20260701_001/attempts/attempt_001/output.json",
    "pytest tests/test_state_machine.py passed"
  ],
  "required_fixes": [],
  "reviewer_agent": "spec_reviewer",
  "created_at": "2026-07-01T10:09:00+08:00"
}
```

### 6.6 Artifact

```json
{
  "id": "artifact_001",
  "run_id": "run_20260701_001",
  "task_id": "T-003",
  "attempt_id": "attempt_001",
  "path": "runs/run_20260701_001/artifacts/state_machine.patch",
  "artifact_type": "patch",
  "status": "committed",
  "checksum": "sha256:...",
  "created_at": "2026-07-01T10:08:00+08:00"
}
```

### 6.7 ResourceLock

```json
{
  "id": "lock_001",
  "run_id": "run_20260701_001",
  "task_id": "T-003",
  "attempt_id": "attempt_001",
  "resource_type": "file",
  "resource_id": "app/orchestration/state_machine.py",
  "mode": "write",
  "status": "held",
  "expires_at": "2026-07-01T10:20:00+08:00"
}
```

### 6.8 ToolInvocation

```json
{
  "id": "tool_001",
  "attempt_id": "attempt_001",
  "tool_name": "shell",
  "permission_level": "L1",
  "approved": true,
  "command": "pytest tests/test_state_machine.py",
  "started_at": "2026-07-01T10:07:00+08:00",
  "ended_at": "2026-07-01T10:07:30+08:00",
  "exit_code": 0,
  "output_path": "runs/run_20260701_001/attempts/attempt_001/tool_001.log"
}
```

### 6.9 HumanApproval

```json
{
  "id": "approval_001",
  "task_id": "T-007",
  "attempt_id": "attempt_003",
  "reason": "任务需要删除远程环境中的旧数据",
  "risk_level": "high",
  "requested_action": "delete_remote_data",
  "status": "pending",
  "requested_at": "2026-07-01T10:20:00+08:00"
}
```

### 6.10 WorkerProfile

```json
{
  "id": "claude-strong-1",
  "type": "claude-code",
  "command": "claude",
  "model_switch_command": "ccswitch strong",
  "max_concurrency": 1,
  "task_types": ["core_code", "debugging", "architecture"],
  "enabled": true
}
```

### 6.11 WorkerExecution

```json
{
  "id": "worker_exec_001",
  "attempt_id": "attempt_001",
  "worker_profile_id": "claude-strong-1",
  "command": "ccswitch strong && claude",
  "worktree_path": "runs/run_20260701_001/worktrees/T-003",
  "task_prompt_path": "runs/run_20260701_001/attempts/attempt_001/task.md",
  "result_path": "runs/run_20260701_001/attempts/attempt_001/result.json",
  "status": "completed",
  "started_at": "2026-07-01T10:05:00+08:00",
  "ended_at": "2026-07-01T10:08:00+08:00",
  "exit_code": 0
}
```

## 7. 调度流程

### 7.1 主流程

```text
1. 用户在 Codex 对话中提交目标
2. CodexConversationController 创建 Run
3. Planner 生成或补全任务 DAG
4. Supervisor 校验任务契约
5. Supervisor 保存任务 DAG
6. Scheduler 找出所有 ready 任务
7. Scheduler 检查预算、权限等级和 ResourceLock
8. Scheduler 根据 WorkerProfile 选择 Claude Code 或本地 worker
9. Scheduler 为任务创建 ExecutionAttempt 并获取 lease
10. ToolPolicy 为 worker 分配受限工具和上下文
11. WorkerAdapter 执行任务并写入 attempt 临时产物目录
12. Supervisor 调用 Spec Reviewer
13. Spec Reviewer 通过后调用 Quality Reviewer
14. Quality Reviewer 通过后调用 Test Agent
15. 测试通过后进入 committing
16. Artifact Manager 原子提交产物
17. 提交成功后标记 verified
18. Codex 在对话中汇报进度或请求审批
19. 继续调度后续任务
20. 所有任务 verified 后调用 Integrator
21. 生成最终交付物和执行报告
```

### 7.2 并行策略

可以并行的任务：

- 不同资料源检索。
- 不同模块实现。
- 文档生成和测试设计。
- 多个独立失败的排查。

不能并行的任务：

- 修改同一文件的任务。
- 依赖同一未完成接口的任务。
- 需要共享外部状态的任务。
- 高风险生产操作。

资源锁规则：

- 任务声明 `resource_locks`，Scheduler 在创建 ExecutionAttempt 前尝试获取锁。
- 读锁之间可以并行，写锁与任何读写锁冲突。
- 文件锁支持精确路径和 glob，但 glob 锁必须规范化后比较。
- 外部资源也必须建模为锁，例如数据库表、远程 API、端口、测试环境、缓存 key。
- 租约过期后，锁不能立即无条件释放；必须结合心跳和 attempt 副作用状态判断。
- SQLite MVP 可用单进程事务实现锁；工程化版本应迁移到 Postgres advisory lock 或队列级锁。

### 7.3 调度前校验

任务进入 `ready` 后，Scheduler 必须先做以下校验：

- DAG 无环，依赖任务全部 `verified`。
- 任务包含必需字段：输入、输出 schema、验收标准、权限等级、风险等级、超时、资源锁。
- 任务预算不会超过 Run 剩余预算。
- 任务权限等级不高于当前自动执行上限。
- `allowed_files` 和 `forbidden_actions` 没有冲突。
- 所需工具都能被 ToolPolicy 明确允许或转入人工审批。

### 7.4 模型路由

| 任务类型 | 推荐模型策略 |
|---|---|
| 简单格式转换 | 快速低成本模型 |
| 单文件实现 | 标准模型 |
| 多模块设计 | 强推理模型 |
| 安全审查 | 强推理模型 |
| 最终集成 | 强推理模型 |

模型路由必须同时考虑成本、风险和失败次数：

- 首次低风险任务优先使用标准模型。
- Reviewer、安全审查、最终集成优先使用强推理模型。
- 第 2 次失败可升级模型。
- 第 3 次失败优先拆小任务，而不是继续堆模型。

当接入 Claude Code worker 时，模型路由由 WorkerProfile 执行：

- `claude-fast` 处理低风险文档、简单测试、单文件小改。
- `claude-strong` 处理核心代码、复杂调试、状态机、存储、调度和权限。
- `claude-review` 可作为可选交叉 Reviewer，但不能替代 Codex/Supervisor 的最终验收。
- 模型切换命令必须配置化，例如 `ccswitch fast`、`ccswitch strong`；如果本机命令名称不同，使用配置覆盖。

## 8. 权限与安全设计

### 8.1 工具权限分级

| 等级 | 权限 | 是否自动执行 |
|---|---|---|
| L0 | 读取本地文件、生成文档 | 是 |
| L1 | 写入工作区文件、运行测试 | 是 |
| L2 | 安装依赖、访问外网 API | 条件允许 |
| L3 | 修改数据库、调用付费 API | 需要审批 |
| L4 | 删除数据、生产发布、迁移数据 | 必须审批 |

MVP 默认自动执行上限为 L1。L2 只能在配置明确开启后执行，L3/L4 必须人工审批。

### 8.2 Guardrails

必须设置的规则：

- 禁止 Agent 自行扩大权限。
- 禁止未审批删除文件或数据。
- 禁止在日志中输出密钥。
- 禁止跳过 Reviewer。
- 禁止未验证任务进入最终交付。
- 禁止 Agent 直接读取未授权环境变量。
- 禁止 Agent 在 `allowed_files` 之外写入工作区。
- 禁止未记录 ToolInvocation 的工具调用。
- 禁止 Reviewer 审核自己同一 attempt 的产物。

### 8.3 ToolPolicy 细则

| 工具类型 | MVP 默认策略 | 说明 |
|---|---|---|
| file_read | L0 自动允许 | 限制在工作区和显式允许路径内 |
| file_write | L1 自动允许 | 只能写入 `allowed_files` 或 attempt 临时目录 |
| shell | L1 条件允许 | 命令 allowlist，限制超时和工作目录 |
| network_read | L2 条件允许 | 只允许配置中的域名或官方文档源 |
| network_write | L3 审批 | 包括调用付费 API、创建远程资源 |
| package_install | L2 条件允许 | 需要记录包名、版本、来源 |
| database_write | L3 审批 | MVP 不自动执行 |
| destructive_action | L4 审批 | 删除、迁移、发布、重置等操作 |

Shell 命令 MVP allowlist 建议：

- `pytest`
- `python -m pytest`
- `ruff`
- `mypy`
- `npm test`
- `npm run lint`

不建议 MVP 自动允许：

- `rm`
- `git reset`
- `git checkout`
- `curl | sh`
- 带 shell 重定向写入敏感路径的命令
- 任何生产环境 CLI

### 8.4 密钥与日志脱敏

- Agent 只能拿到任务需要的最小环境变量集合。
- 日志写入前必须执行密钥模式脱敏。
- ToolInvocation 只保存命令、退出码和截断输出；完整输出如可能含密钥，应存为受限 artifact。
- `.env`、私钥文件、云厂商凭据默认不可读，除非任务显式审批。

## 9. Checkpoint 与恢复

### 9.1 持久化内容

- Run 状态。
- Task 状态。
- ExecutionAttempt 状态。
- Agent 输入输出。
- Review 结果。
- 测试结果。
- 错误日志。
- 产物路径。
- ResourceLock。
- ToolInvocation。
- HumanApproval。
- 状态转移事件。

### 9.2 恢复策略

系统重启后：

```text
1. 读取所有未完成 Run
2. 找出 leased/running/committing 且心跳超时的 ExecutionAttempt
3. 检查 attempt 的 side_effect_status
4. side_effect_status 为 none/staged 时释放锁并按 RetryPolicy 重试
5. side_effect_status 为 committed 时校验产物并进入 Review 或 verified
6. side_effect_status 为 unknown 时进入 needs_human_approval
7. 恢复未完成的 HumanApproval
8. 继续调度 ready 任务
```

### 9.3 幂等与副作用控制

- 每个 attempt 都必须有 `idempotency_key`。
- 外部 API 调用如支持幂等键，必须传入 `idempotency_key`。
- 文件写入先进入临时目录，再由 Artifact Manager 原子提交。
- 数据库写入、远程资源创建、生产操作在 MVP 中默认不自动执行。
- 任务恢复时，不能只根据 Task 状态判断是否重试，必须结合 attempt、artifact、tool invocation 和 side_effect_status。

### 9.4 心跳与租约

- Scheduler 创建 attempt 时生成 `lease_id`。
- Agent 执行期间定期更新 `last_heartbeat_at`。
- 心跳超时不等于任务失败，只表示 Supervisor 需要进入恢复判断。
- 租约释放必须记录状态转移事件，避免同一任务被两个执行器同时领取。

## 10. 项目目录结构

MVP 推荐目录：

```text
agent-orchestrator/
  README.md
  pyproject.toml
  .env.example

  app/
    main.py
    config.py

    agents/
      base.py
      planner.py
      researcher.py
      architect.py
      implementer.py
      test_agent.py
      spec_reviewer.py
      quality_reviewer.py
      integrator.py

    conversation/
      controller.py
      commands.py
      status_renderer.py

    workers/
      base.py
      mock_worker.py
      claude_code.py
      profiles.py

    orchestration/
      supervisor.py
      graph.py
      scheduler.py
      state_machine.py
      retry_policy.py
      resource_lock.py
      artifact_manager.py

    policy/
      tool_policy.py
      permission.py
      redaction.py

    storage/
      db.py
      models.py
      repositories.py

    tools/
      file_tool.py
      shell_tool.py
      browser_tool.py
      search_tool.py

    schemas/
      run.py
      task.py
      task_contract.py
      review.py
      agent_result.py
      execution_attempt.py
      artifact.py
      tool_invocation.py
      worker_profile.py

    observability/
      logger.py
      tracing.py

  tests/
    test_state_machine.py
    test_scheduler.py
    test_retry_policy.py
    test_tool_policy.py
    test_artifact_manager.py
    test_recovery.py
    test_supervisor_flow.py

  runs/
    run_<id>/
      attempts/
      worktrees/
      artifacts/
      logs/
      report.md
```

## 11. 核心接口设计

### 11.1 Agent 基类

```python
from typing import Protocol

class Agent(Protocol):
    name: str
    agent_type: str

    async def run(self, task: "TaskContext") -> "AgentResult":
        ...
```

### 11.2 AgentResult

```python
from pydantic import BaseModel, Field
from typing import Literal

class AgentResult(BaseModel):
    status: Literal[
        "DONE",
        "DONE_WITH_CONCERNS",
        "NEEDS_CONTEXT",
        "BLOCKED",
        "FAILED"
    ]
    summary: str
    output: dict | None = None
    concerns: list[str] = Field(default_factory=list)
    evidence: list[str] = Field(default_factory=list)
    artifacts: list[str] = Field(default_factory=list)
    side_effect_status: Literal["none", "staged", "committed", "unknown"] = "none"
```

### 11.3 ReviewerResult

```python
from pydantic import BaseModel, Field

class ReviewerResult(BaseModel):
    passed: bool
    summary: str
    issues: list[dict] = Field(default_factory=list)
    evidence: list[str] = Field(default_factory=list)
    required_fixes: list[str] = Field(default_factory=list)
```

### 11.4 Supervisor 主循环伪代码

```python
async def run_supervisor(run_id: str):
    while True:
        run = load_run(run_id)
        recover_stale_attempts(run_id)

        if all_tasks_verified(run):
            await integrator.run(run)
            mark_run_completed(run_id)
            break

        ready_tasks = find_ready_tasks(run)

        for task in ready_tasks:
            contract_errors = validate_task_contract(task)
            if contract_errors:
                mark_blocked(task, contract_errors)
                continue

            if requires_human_approval(task) or not tool_policy.can_auto_run(task):
                mark_needs_approval(task)
                continue

            lease = try_acquire_lease_and_locks(task)
            if not lease:
                continue

            attempt = create_execution_attempt(task, lease)
            result = await dispatch_agent(task, attempt)

            if result.status in ["DONE", "DONE_WITH_CONCERNS"]:
                save_attempt_result(attempt, result)

                spec = await spec_reviewer.run(task, attempt, result)
                if not spec.passed:
                    await request_fix(task, attempt, spec.required_fixes)
                    continue

                quality = await quality_reviewer.run(task, attempt, result)
                if not quality.passed:
                    await request_fix(task, attempt, quality.required_fixes)
                    continue

                tests = await test_agent.run(task, attempt, result)
                if not tests.passed:
                    await request_fix(task, attempt, tests.required_fixes)
                    continue

                mark_committing(task)
                artifact_manager.commit(attempt)
                mark_verified(task)

            elif result.status == "NEEDS_CONTEXT":
                mark_blocked(task, result.summary)

            elif result.status in ["BLOCKED", "FAILED"]:
                apply_retry_policy(task, attempt, result)
```

实际实现中，ExecutionAttempt 结束、失败、阻塞或进入人工审批时，都必须通过统一的 finalizer 更新 attempt 状态、释放可释放的 ResourceLock、写入状态转移事件，并保存最后一次心跳。

## 12. 最小 MVP 范围

第一版只实现以下能力：

### 12.1 必须实现

- Codex 对话中提交任务目标，并由 CodexConversationController 创建 Run。
- CLI 保留为内部执行和调试入口，但不是主要用户入口。
- 支持用户提供或模板生成任务 DAG。
- Planner 在 MVP 中只负责补全和校验任务契约，不强制要求完全自动拆解复杂目标。
- Supervisor 调度任务。
- 至少 3 类 Agent：Planner、Implementer、Reviewer。
- 至少 2 类 Worker：mock worker、Claude Code worker。
- 支持 WorkerProfile 配置，包括 worker id、类型、命令、模型切换命令、适用任务类型和并发限制。
- 支持通过 `ccswitch` 或配置化 wrapper 为不同 Claude Code worker 切换不同模型。
- SQLite 保存 Run、Task、ExecutionAttempt、AgentExecution、Review、Artifact、ResourceLock、ToolInvocation。
- 支持任务状态流转。
- 支持失败重试。
- 支持 attempt 级临时产物目录。
- 支持 L0/L1 ToolPolicy。
- 支持 Reviewer 证据链。
- 支持 Codex 对话中的状态查询、暂停、继续、重试、审批和报告摘要。
- 支持最终报告生成。

### 12.2 暂不实现

- Web 控制台。
- 多用户权限系统。
- 分布式任务队列。
- 多租户隔离。
- 复杂计费系统。
- 生产环境发布能力。
- 完全自由的 Planner 自动拆解。
- 自动执行 L3/L4 高风险工具。
- 真实生产数据库写入。
- 跨机器分布式锁。
- Claude Code worker 之间自由通信。
- 由 Claude Code 直接决定最终合并或提交产物。

## 13. 里程碑计划

### 13.1 M1：骨架与状态机

交付：

- 项目结构。
- Task/Run 数据模型。
- ExecutionAttempt 数据模型。
- SQLite 持久化。
- 状态机。
- 状态转移事件记录。
- 基础测试。

验收标准：

- 可以创建 Run。
- 可以创建 Task DAG。
- Task 可以按状态转移表从 pending 流转到 verified。
- 非法状态转移会被拒绝。
- 状态变更可以落库。

### 13.2 M2：任务契约与受控 DAG

交付：

- TaskContract schema。
- Planner 契约补全能力。
- DAG 校验。
- 任务风险等级和权限等级校验。

验收标准：

- 输入一个受控 DAG 后，可以补全必需契约字段。
- 缺少关键字段的任务会进入 blocked。
- 有环 DAG 会被拒绝。

### 13.3 M3：Agent 执行链路

交付：

- Agent 基类。
- Implementer Agent。
- Reviewer Agent。
- Supervisor 调度主循环。
- Attempt 临时产物目录。
- Reviewer 证据链。

验收标准：

- 子任务可以被 Agent 执行。
- 执行结果写入 attempt。
- 执行结果可以被 Reviewer 基于证据验收。
- Reviewer 通过前产物不会进入正式 artifacts。

### 13.4 M4：失败恢复与重试

交付：

- RetryPolicy。
- failed / blocked / needs_human_approval 状态。
- checkpoint 恢复。
- 心跳与租约。
- side_effect_status 判断。

验收标准：

- Agent 失败后可以自动重试。
- 超过次数后进入 blocked。
- 程序重启后可以恢复未完成 Run。
- `side_effect_status=unknown` 的任务不会自动重试。

### 13.5 M5：权限、资源锁与并行调度

交付：

- DAG 依赖分析。
- 并发任务执行。
- ResourceLock。
- ToolPolicy。
- 文件冲突检测。
- L0/L1 自动执行策略。

验收标准：

- 无依赖任务可以并行执行。
- 有冲突的任务不会并行修改同一资源。
- 超出权限等级的工具调用会被拒绝或转入审批。
- 所有工具调用都有 ToolInvocation 记录。

### 13.6 M6：报告与可观测性

交付：

- 执行报告。
- 任务日志。
- token/cost 统计。
- 失败原因汇总。
- 状态转移时间线。
- 产物清单。
- Reviewer 证据摘要。

验收标准：

- 每次 Run 都有完整报告。
- 可以追踪每个任务由哪个 Agent 执行、失败原因是什么、验证结果是什么。
- 可以追踪每个正式产物来自哪个 task、attempt 和 review。

## 14. 测试策略

### 14.1 单元测试

- 状态机流转。
- DAG ready 任务计算。
- RetryPolicy。
- 权限判断。
- ReviewerResult 解析。
- TaskContract 校验。
- ResourceLock 冲突检测。
- ToolPolicy allow/deny。
- Artifact Manager 原子提交。

### 14.2 集成测试

- 从用户目标到最终报告的完整流程。
- Agent 返回失败后的重试流程。
- Reviewer 不通过后的修复流程。
- 系统重启后的恢复流程。
- 任务心跳超时后的恢复流程。
- 权限不足时进入 HumanApproval 的流程。
- 并行任务资源锁冲突流程。

### 14.3 冒烟测试

测试目标示例：

```text
请生成一个包含 README、设计说明和测试计划的小型项目骨架。
```

预期：

- Planner 补全或生成至少 3 个子任务契约。
- Implementer 完成文件生成。
- Reviewer 基于产物和测试证据验收通过。
- 最终输出报告。

### 14.4 故障注入测试

- 执行过程中杀掉进程，重启后应恢复未完成 Run。
- 模拟 Agent 无心跳，任务应根据 `side_effect_status` 正确重试或审批。
- 模拟 Reviewer 不通过，应创建修复 attempt。
- 模拟文件锁冲突，两个任务不得同时写同一文件。
- 模拟工具越权调用，应被 ToolPolicy 拒绝并记录。

## 15. 风险与应对

| 风险 | 影响 | 应对 |
|---|---|---|
| Agent 误解任务 | 产出偏离目标 | 强制 Spec Reviewer 验收 |
| Agent 自行扩大范围 | 成本和风险上升 | 任务范围约束 + Guardrails |
| 并行任务冲突 | 文件或状态覆盖 | 资源锁和文件冲突检测 |
| 长任务中断 | 进度丢失 | Checkpoint 持久化 |
| 成本失控 | API 费用超限 | Run 级预算和模型路由 |
| 高风险操作误执行 | 数据损坏 | L3/L4 权限人工审批 |
| 重试导致重复副作用 | 文件重复提交或远程资源重复创建 | ExecutionAttempt + idempotency_key + side_effect_status |
| Reviewer 误判 | 错误产物进入交付 | 证据驱动验收 + 测试输出 + 强模型复核 |
| Planner 拆解过度自由 | DAG 不可执行或范围膨胀 | MVP 采用受控 DAG，Planner 只补全契约 |
| 日志泄露密钥 | 安全事故 | 最小环境变量暴露 + 日志脱敏 |

## 16. 成功标准

MVP 成功标准：

- 能从 Codex 对话中接收一个受控 DAG 或模板化目标，并补全可执行任务契约。
- 能调度至少 3 个 Agent 完成任务。
- 能自动启动至少 1 个 Claude Code worker 执行具体实现任务。
- 能通过 WorkerProfile 将不同任务路由到不同 Claude Code 模型配置。
- 每个任务都经过 Reviewer 基于证据验收。
- 失败任务可以自动重试。
- 程序中断后可以从 checkpoint 恢复。
- 重试不会覆盖历史 attempt，也不会自动重复未知副作用。
- L0/L1 工具调用被 ToolPolicy 约束并完整记录。
- 产物通过 Reviewer 后才进入正式 artifacts。
- 用户可在 Codex 对话中查看状态、批准高风险动作、请求重试和获取报告摘要。
- 最终生成可读执行报告。

工程化成功标准：

- 支持多 Run 并发。
- 支持 Web 控制台查看状态。
- 支持人工审批。
- 支持成本统计和运行追踪。
- 支持工具权限分级。
- 支持 Planner 自动拆解复杂目标并由用户或 Reviewer 确认 DAG。
- 支持 Postgres/Redis/队列化调度。
- 支持更细粒度的组织级权限和审计。

## 17. 最终建议

不要一开始建设完整平台。建议先实现 Codex 对话入口驱动的本地 MVP，CLI 仅作为内部执行和调试接口：

```text
用户在 Codex 对话中输入目标
  ↓
CodexConversationController 创建 Run
  ↓
受控 DAG 或模板目标
  ↓
Planner 补全任务契约
  ↓
Supervisor 调度
  ↓
ToolPolicy + ResourceLock
  ↓
Claude Code / mock worker 执行
  ↓
Reviewer 基于证据验收
  ↓
Artifact Manager 提交产物
  ↓
SQLite 保存状态
  ↓
Markdown 报告输出
  ↓
Codex 对话中交付摘要、状态和审批请求
```

当 MVP 可以稳定跑通 5 到 10 个真实低风险任务，并且能通过故障注入测试后，再逐步引入：

- Planner 自动拆解复杂目标。
- L2 网络读取和依赖安装。
- FastAPI、Postgres、Redis 和 Web 控制台。
- 多 Run 并发和更完整的人工审批工作流。
- 更多 WorkerAdapter，例如 Codex worker、远程 runner、人类 worker。

这样既能快速验证核心价值，又不会过早陷入平台工程复杂度。第一阶段的重点不是让 Agent 最大程度自由，而是证明系统能在清晰边界内稳定、可恢复、可审计地完成任务。

## 18. 后续工程化 TODO

后续平台化建设必须以 MVP 的稳定性指标为前置条件，不应在核心执行链路尚未稳定时提前引入 Web 控制台、分布式队列、多租户或复杂权限。推荐按以下阶段推进。

### 18.1 阶段 P1：MVP 稳定化

进入条件：

- M1 到 M6 已完成。
- 至少 5 个真实低风险任务可以端到端跑通。
- 故障注入测试通过，包括进程中断、Reviewer 不通过、资源锁冲突和工具越权。

TODO：

- 补齐状态转移事件表和 Run 时间线报告。
- 完善 RetryPolicy，支持按错误类型区分是否重试。
- 完善 ToolPolicy 配置文件，支持项目级 allowlist。
- 增加 artifact checksum 校验和产物清理策略。
- 增加成本预算硬限制，超过预算时自动进入 blocked。
- 增加运行报告模板，输出任务 DAG、attempt、review、tool invocation 和 artifact 清单。
- 增加一组固定 benchmark 任务，用于回归测试系统行为。

验收标准：

- 连续跑 10 次 benchmark 任务无状态卡死。
- 任意失败任务都能在报告中定位到原因、attempt、日志和 Reviewer 结论。
- 任意正式产物都能追溯到 task、attempt、review 和提交时间。

禁止提前做：

- 不做多用户。
- 不做 Web 控制台。
- 不做分布式队列。
- 不开放 L3/L4 自动执行。

### 18.2 阶段 P2：Planner 自动拆解增强

进入条件：

- 受控 DAG 模式稳定。
- TaskContract 字段和 Reviewer 证据链已经在真实任务中验证可用。

TODO：

- 引入 Planner 自动拆解模式，但默认只生成草稿 DAG。
- 增加 DAG Reviewer，专门检查任务粒度、依赖、权限、风险和资源锁。
- 增加用户确认环节，高风险或大范围 DAG 必须人工确认后执行。
- 增加任务拆分策略：单任务文件变更过多、验收标准过多或权限过高时自动拆小。
- 增加任务模板库，例如文档任务、代码修改任务、测试补齐任务、调研任务。
- 增加 DAG 版本号和变更记录，避免执行中途无追踪地改计划。

验收标准：

- Planner 生成的 DAG 通过 DAG Reviewer 后才能执行。
- 自动拆解任务的失败率、重试率和人工介入率可统计。
- Planner 不得生成缺少 TaskContract 必填字段的任务。

禁止提前做：

- 不允许 Planner 直接执行任务。
- 不允许 Planner 绕过 DAG Reviewer。
- 不允许高风险 DAG 自动执行。

### 18.3 阶段 P3：服务化与 API

进入条件：

- CLI MVP 稳定。
- Run、Task、Attempt、Review、Artifact 的数据模型基本稳定。

TODO：

- 引入 FastAPI，提供 Run 创建、查询、取消、审批和报告下载接口。
- 将 SQLite 迁移到 Postgres。
- 增加数据库迁移工具，例如 Alembic。
- 增加 API 级鉴权，MVP 可先使用本地 token。
- 增加后台 worker 进程，API 只负责提交和查询，执行由 worker 负责。
- 增加 Run cancellation，确保取消时能释放 lease 和 ResourceLock。
- 增加 API 契约测试。

验收标准：

- CLI 和 API 共用同一套 domain/service 层。
- API 重启不影响 worker 中已持久化的 Run 恢复。
- 数据库迁移可重复执行，并能在测试库完整初始化。

禁止提前做：

- 不把业务逻辑写进 API handler。
- 不在 API 层直接调用 Agent。
- 不做复杂组织权限和计费。

### 18.4 阶段 P4：队列化与多 Run 并发

进入条件：

- 单机 worker 稳定。
- Postgres 版本的数据一致性测试通过。

TODO：

- 引入 Redis + RQ/Celery，或直接评估 Temporal。
- 将 Task 调度改为队列任务，但状态机仍由 Supervisor/domain 层控制。
- 增加全局并发限制、Run 级并发限制、Agent 类型并发限制。
- 增加跨 worker ResourceLock。
- 增加 worker heartbeat 和 worker crash recovery。
- 增加队列积压、任务耗时、失败率和成本指标。

验收标准：

- 多个 Run 并发时不会互相覆盖 artifact 或锁。
- worker 崩溃后，租约能被恢复流程正确接管。
- 同一资源的写锁在多 worker 下仍然互斥。

禁止提前做：

- 不为了队列化重写 Agent 逻辑。
- 不让队列系统直接决定任务状态。
- 不把锁只放在内存里。

### 18.5 阶段 P5：Web 控制台与人工审批

进入条件：

- API 和 worker 架构稳定。
- HumanApproval 流程在 CLI/API 中已可用。

TODO：

- 实现 Run 列表、Run 详情、任务 DAG 可视化。
- 展示 Task、Attempt、Review、ToolInvocation、Artifact 时间线。
- 实现人工审批页面，展示风险原因、请求动作、相关日志和建议选项。
- 实现 Run 暂停、恢复、取消。
- 实现 artifact 浏览和报告下载。
- 增加基础告警：预算接近上限、任务连续失败、人工审批超时。

验收标准：

- 用户可以通过控制台理解当前 Run 卡在哪里、为什么卡住、下一步需要谁处理。
- 审批操作有审计记录。
- 控制台只调用 API，不直接访问数据库或执行工具。

禁止提前做：

- 不做复杂低代码编排器。
- 不做花哨但不可运维的 DAG 编辑器。
- 不在前端实现权限判断的真实逻辑。

### 18.6 阶段 P6：权限、多用户与审计

进入条件：

- 单用户平台化流程稳定。
- L0 到 L2 权限策略已经稳定。

TODO：

- 引入用户、项目、角色和权限模型。
- 支持项目级 ToolPolicy、预算、模型路由和网络域名配置。
- 增加审批人角色和审批策略。
- 增加审计日志不可变记录。
- 支持密钥托管和最小权限注入。
- 支持按项目隔离 artifacts、logs 和数据库记录。

验收标准：

- 用户只能查看和操作自己有权限的 Run。
- 审批、取消、重试、工具调用都有审计记录。
- 密钥不会出现在普通日志、报告和 Reviewer 输出中。

禁止提前做：

- 不在权限模型稳定前开放多租户。
- 不允许 Agent 读取全局密钥。
- 不允许跨项目共享工作区目录。

### 18.7 阶段 P7：高级 Agent 能力

进入条件：

- 平台的状态、权限、恢复、审计能力稳定。

TODO：

- 增加 Research Agent 的来源可信度评分。
- 增加 Architect Agent 的设计审查模板。
- 增加 Test Agent 自动生成和运行测试的能力。
- 增加多 Reviewer 策略：需求验收、质量验收、安全验收、回归验收。
- 增加 Agent 评测集，统计不同模型和提示词的成功率、成本和耗时。
- 增加模型路由策略配置，按任务类型、风险、失败次数动态选择模型。

验收标准：

- Agent 能力增强不能绕过 TaskContract、ToolPolicy 和 Reviewer。
- 每次提示词或模型策略变更都有评测结果。
- 高风险任务至少经过两个不同类型 Reviewer。

禁止提前做：

- 不让 Agent 之间自由对话绕过 Supervisor。
- 不引入不可审计的长期记忆。
- 不让模型路由只按成本最低决策。

### 18.8 长期可选方向

- 引入 Temporal 替代自研恢复和长流程编排。
- 支持插件化工具市场，但每个工具必须声明权限等级、输入输出 schema 和审计策略。
- 支持多工作区隔离执行，例如容器、沙箱或远程 runner。
- 支持组织级成本看板和模型效果看板。
- 支持任务模板 marketplace。
- 支持人类和 Agent 混合协作的任务评论、复核和交接。

### 18.9 防跑偏检查清单

每进入一个新阶段前，必须回答以下问题：

- 当前阶段是否依赖上一个阶段的验收标准？
- 新增能力是否复用现有 TaskContract、ExecutionAttempt、ToolPolicy、ResourceLock 和 Artifact Manager？
- 新增能力是否引入了新的不可恢复副作用？
- 新增能力是否能被测试、审计和回放？
- 是否有功能只是“看起来像平台”，但不能提升稳定性、可恢复性、可审计性或任务成功率？
- 是否有更简单的 CLI 或配置方式可以先验证价值？

如果以上任一问题无法回答清楚，应暂停平台化扩展，先回到 MVP 稳定性建设。

# Herdr Workflow

一个面向 Herdr 共享工作区的顺序式多 Agent 编程工作流。它使用明确的角色分工、文件化 handoff、测试证据和有界修复循环，让规划、实现、调试与审查能够在不同 Agent 之间稳定传递。

该仓库提供的是一个 Codex Skill。核心执行规则位于 [`SKILL.md`](./SKILL.md)。

## 设计目标

这个工作流主要解决以下问题：

- 让一个 Agent 负责总体计划和流程决策；
- 让实现与审查由不同角色完成，减少自我确认；
- 使用共享文件传递目标、诊断和审查结果；
- 以实际 diff、测试命令和输出作为完成证据；
- 只在持续失败时启动 debugger，避免无意义的额外调用；
- 使用 Herdr 的生命周期等待能力，减少 planner 的轮询和模型消耗；
- 限制修复循环次数，避免 Agent 在同一问题上无限往返。

工作流不绑定模型或提供商。启动 Agent 时使用哪种 CLI、模型和推理强度，由用户根据任务、额度和可用环境决定。

## 前置条件

开始前需要满足：

1. 当前 Agent 位于 Herdr 管理的 pane 中；
2. 环境变量 `HERDR_ENV` 等于 `1`；
3. 已安装并可使用 `/herdr` skill；
4. 所有参与者位于同一 Herdr workspace，并共享同一个项目目录；
5. 项目中的现有用户修改必须得到保留。

可用以下命令检查运行环境：

```bash
test "${HERDR_ENV:-}" = 1
```

如果检查失败，应从 Herdr 内部启动工作流，而不是从普通终端跨会话控制 Herdr。

## 工作流结构

```mermaid
flowchart TD
    P["Planner 制定计划"] --> C["Coder 实现并测试"]
    C --> T{"相关检查通过？"}
    T -- "是" --> R["Reviewer 独立审查"]
    T -- "否，先直接修复" --> C
    T -- "同一失败持续存在" --> D["Debugger 诊断"]
    D --> F["Coder 按诊断修复"]
    F --> T
    R --> S{"审查状态"}
    S -- "PASS" --> E["Planner 汇总"]
    S -- "FIX_REQUIRED" --> C
```

正常路径是：

```text
planner → coder → reviewer → planner
```

debugger 不在正常路径中。它只在满足触发条件时介入：

```text
coder 持续失败 → debugger 诊断 → coder 修复 → reviewer
```

## 角色职责

| 角色 | 主要职责 | 是否修改业务代码 |
|---|---|---:|
| `planner` | 检查仓库、编写计划、分派任务、判断是否进入修复或调试分支、汇总结果 | 原则上不修改 |
| `coder` | 根据计划实现功能、修改测试、运行检查并报告证据 | 是 |
| `reviewer` | 检查计划、diff、测试证据和行为，输出结构化审查结论 | 否 |
| `debugger` | 对持续失败进行根因诊断，提出最小修复和验证方法 | 否 |

角色必须在 Agent 名称或启动提示中明确指定，不能仅根据模型名称推断职责。

## Handoff 文件

所有协调文件位于项目根目录的 `.herdr/`：

```text
.herdr/
├── PLAN.md
├── DEBUG.md
└── REVIEW.md
```

`DEBUG.md` 仅在条件式调试被触发时出现。

除非用户明确要求，否则不要修改项目的 `.gitignore` 或其他仓库配置来处理这些文件。

### PLAN.md

由 planner 在编码前创建，至少包含：

1. 用户目标；
2. 验收标准；
3. 仓库现状与约束；
4. 预计修改的文件或组件；
5. 实现步骤；
6. 验证步骤；
7. 尚未解决的假设或问题。

coder、debugger 和 reviewer 都应以该文件作为任务范围依据。

### DEBUG.md

由 debugger 创建，至少包含：

1. 失败命令和实际输出；
2. 最小复现方式；
3. 根因假设；
4. 支持该假设的证据；
5. 涉及的文件、符号、接口或假设；
6. 最小修复建议；
7. 用于确认或否定诊断的验证方法。

debugger 只提供诊断，不直接修改源代码或测试。coder 负责实施修复。

### REVIEW.md

由 reviewer 创建。第一个非空行必须严格为：

```text
PASS
```

或者：

```text
FIX_REQUIRED
```

后续内容应包含：

- 具体发现；
- 严重程度；
- 文件或符号位置；
- 问题为什么会影响正确性；
- 建议的验证方式。

`PASS` 只代表当前请求范围内没有剩余的可执行正确性问题，不代表仓库中所有既有失败都已消失。

## 正常执行流程

### 1. Planner 创建计划

planner 检查：

- `git status`；
- 相关源文件与测试；
- 仓库约束；
- 用户已有但尚未提交的修改。

然后写入 `.herdr/PLAN.md`。

### 2. Coder 实现和验证

planner 使用一次带 `--wait` 的提示提交任务：

```bash
herdr agent prompt coder \
  "Read .herdr/PLAN.md. Implement the requested change, run the relevant checks, and report exact evidence. Do not dispatch other agents." \
  --wait
```

coder 必须报告实际执行的命令及结果，不能只声明“已经完成”。

### 3. Reviewer 独立审查

相关检查通过后，planner 提交 reviewer：

```bash
herdr agent prompt reviewer \
  "Read .herdr/PLAN.md, inspect the current diff and test evidence, and write .herdr/REVIEW.md. Do not modify business code. The first non-empty line must be PASS or FIX_REQUIRED." \
  --wait
```

如果结果为 `FIX_REQUIRED`，planner 将具体发现交回 coder。默认最多执行两轮 fix-review。

## 条件式 Debugger

### 触发条件

满足以下任一条件时可以启动 debugger：

- 同一相关失败在 coder 初次实现和一次针对性修复后仍然存在；
- 失败跨越多个组件，现有证据不足以安全定位根因；
- review 要求的修复导致同一测试持续失败；
- 继续让 coder直接尝试会变成缺少证据的猜测。

普通、明确、可直接修复的错误不应启动 debugger。

### 执行方式

```bash
herdr agent prompt debugger \
  "Read .herdr/PLAN.md, inspect the current diff and failing test evidence, and write .herdr/DEBUG.md with a minimal reproduction, root-cause hypothesis, supporting evidence, affected locations, recommended repair, and verification steps. Diagnose only; do not modify source code or tests." \
  --wait
```

planner 检查 `.herdr/DEBUG.md` 是否提供了可验证的诊断，然后只让 coder 执行一次基于诊断的修复。

默认最多一轮 debugger-diagnosis/fix：

- 修复后检查通过：进入 reviewer；
- 同一失败仍然存在：停止循环并向用户报告诊断和证据。

## 等待与额度优化

Herdr 的 `agent prompt --wait` 会提交提示并等待目标进入 `idle`、`done` 或 `blocked`。正常情况下，planner 不需要围绕它建立轮询循环。

正常路径遵守以下规则：

- 每项任务只提交一次 `agent prompt --wait`；
- 不在提交后立即追加 `agent get`；
- 不执行仅用于确认提交的短 `agent read`；
- 不手动发送 Enter 作为常规兜底；
- 不每隔固定时间轮询；
- 不为了保持 planner 活跃而周期性生成进度消息；
- 使用 `agent prompt --wait` 返回的生命周期状态；
- 完成后只收集一次必要的语义结果。

优先读取 handoff 文件。如果结果只存在于终端 transcript 中，再执行一次大小合适的 `agent read`，不要同时追加冗余状态查询。

### 异常恢复

仅在真实异常发生后进行恢复检查：

- `agent_prompt_stalled`；
- timeout；
- `blocked`；
- 缺少预期 handoff 文件；
- 重复的生命周期或提交故障。

`agent_prompt_stalled` 或 timeout 不代表提示一定没有发送。重试前先读取目标一次：

- 如果 Agent 正在工作，使用一次 `agent wait` 等待 settled state；
- 如果任务确实不存在且 Agent 已准备好，最多重新提交一次；
- 不盲目补发 Enter；
- 重复失败时停止并报告现场状态。

## 状态与正确性

Herdr 生命周期状态只回答 Agent 是否仍在工作：

| 状态 | 含义 |
|---|---|
| `working` | Agent 正在执行 |
| `blocked` | Agent 等待批准、回答或交互 |
| `idle` / `done` | Agent 已准备接受新输入 |
| `unknown` | Herdr 无法可靠判断状态 |

这些状态不能证明任务在语义上正确。最终完成仍需检查：

- 当前 diff；
- 实际测试命令和输出；
- `.herdr/PLAN.md` 的验收标准；
- `.herdr/REVIEW.md`；
- 条件式调试发生时的 `.herdr/DEBUG.md`。

## 共享工作区安全

- coder、debugger 和 reviewer 顺序执行；
- 只有 coder 修改源代码和测试；
- 不并行修改同一工作树；
- 不重置或丢弃用户已有修改；
- 不关闭或移动不属于本流程的 pane；
- 不添加绕过批准或危险 sandbox 的参数；
- 遇到缺少授权、破坏性操作或关键用户决策时停止；
- 反复出现基础设施故障时停止，不无限重试。

## 完成报告

planner 最终应报告：

- 修改了哪些文件和行为；
- 执行了哪些验证命令；
- 每项验证的实际结果；
- 最终 review 状态；
- debugger 是否被触发及其结论；
- 尚未解决的失败；
- 与任务有关的假设或后续工作。

Agent 进入 `idle` 或 `done` 并不等于工作成功。

## 调用示例

在 Herdr 管理的项目工作区内，可以这样提出请求：

```text
Use $herdr-workflow to implement this change through planning, coding,
conditional debugging when needed, and independent review.
```

也可以直接用自然语言要求：

```text
使用 herdr-workflow 完成这个功能。先制定计划，再编码和测试；
只有持续失败时才启用 debugger，最后必须独立 review。
```

## 工作流边界

这个 Skill：

- 不替用户选择模型或提供商；
- 不替代项目自身的 `AGENTS.md`、测试规范或贡献指南；
- 不保证现有测试全部通过；
- 不把 Agent 的自述当作成功证明；
- 不授权推送、部署、删除或其他超出用户请求范围的操作；
- 不在普通单 Agent 请求中自动启动多 Agent 编排。

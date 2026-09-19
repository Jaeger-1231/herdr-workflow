# Coder implementation style

This is the single source of truth for the implementation style used by every
`coder` work period in this workflow. It applies to initial implementation,
review-requested fixes, and debugger-guided repairs.

Before editing source code or tests, the coder must read this file completely.
If the working context becomes incomplete, uncertain, or appears to have been
compressed during a long task, re-read this file and `.herdr/PLAN.md` before
continuing. Do not assume that context compression reloads files from disk.

## Linear imperative research scripting

请使用线性、命令式、研究脚本式的代码风格
（linear imperative research scripting）。

优先使用最基础、最直接的命令，减少函数封装、抽象层、循环和防御性代码。代码从上到下自然执行，让用户能够逐行理解数据处理过程。

同时保持代码紧凑：简单函数调用、变量列表、列选择、`merge`、`rename`、`read_excel` 等操作尽量写在一行，不要为了格式美观把一个简单命令拆成很多行。只有当单行明显过长、包含多层逻辑或影响理解时才换行。

整体风格更接近 Stata do-file 或传统研究代码，而不是工程化 Python 项目。优先保证：

1. 看得懂；
2. 改得动；
3. 步骤清楚；

而不是追求工程规范和抽象设计。

不要使用 Black/PEP 8 式的强制展开格式。允许较长的单行代码，只要逻辑简单。

## Boundaries

This style does not authorize changing the requested behavior, skipping relevant
tests, or ignoring explicit user, repository, or language constraints. When a
repository-specific instruction directly conflicts with this style, the planner
must resolve the conflict in `.herdr/PLAN.md`; the coder must follow the resolved
plan.

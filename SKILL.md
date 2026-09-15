---
name: herdr-workflow
description: "Coordinate a sequential planner-coder-reviewer coding workflow inside Herdr using shared handoff files, agent prompts, tests, and bounded review-fix loops. Use when the user explicitly asks to orchestrate multiple coding agents in one Herdr workspace."
---

# Herdr coding workflow

Use this skill only when the user asks for a multi-agent coding workflow in Herdr. It is an orchestration protocol, not a replacement for repository-specific engineering judgment.

## Preconditions

Before using any Herdr control command, verify that the current agent is running in a Herdr-managed pane:

```bash
test "${HERDR_ENV:-}" = 1
```

If the check fails, stop and tell the user that the workflow must be started inside Herdr. Do not inspect or control a Herdr session from an ordinary terminal.

Use the installed `/herdr` skill for Herdr command semantics. The workflow assumes one workspace whose panes share the same project directory. Do not create a new workspace, move panes, or change the project directory unless the user requests it.

## Roles

Every participating agent must be assigned one explicit role in its startup prompt or agent name:

- `planner`: owns orchestration, creates the plan, dispatches the other agents, decides whether another loop is needed, and reports the final result. The planner may be a Codex agent with access to the `/herdr` skill.
- `coder`: reads the plan, changes source and tests, runs the relevant checks, and reports evidence. It does not dispatch other agents or rewrite the plan unless the planner explicitly asks.
- `reviewer`: reviews the plan, diff, tests, and relevant behavior without changing business code. It writes a structured review result and identifies concrete fixes.

Do not infer a role from the model name alone. If the role is ambiguous, ask the user or establish it before dispatching work.

## Handoff protocol

Keep coordination artifacts in `.herdr/` at the project root. Create the directory only if it does not exist. Do not add it to `.gitignore` or alter repository configuration unless the user asks.

The planner writes `.herdr/PLAN.md` before implementation. It should contain:

1. the user goal and acceptance criteria;
2. repository findings and constraints;
3. files or components expected to change;
4. implementation and verification steps;
5. unresolved assumptions or questions.

The coder reads `.herdr/PLAN.md`, implements the requested change, runs the appropriate tests or checks, and leaves the working tree and test evidence available for review. It must not claim success without reporting the commands it ran and their outcomes.

The reviewer reads `.herdr/PLAN.md`, the current diff, and the available test evidence. It writes `.herdr/REVIEW.md` with exactly one status on the first non-empty line:

```text
PASS
```

or:

```text
FIX_REQUIRED
```

The remainder of `REVIEW.md` must contain concrete findings, severity, file or symbol references, and a verification suggestion. `PASS` means no actionable correctness issue remains within the requested scope; it does not mean unrelated pre-existing failures disappeared.

## Dispatch sequence

Use named live agents and Herdr's agent surface rather than guessing pane IDs. Create or split all required panes before starting agents, because `agent start` requires an available shell pane. Preserve the shared project cwd.

Start agents with the model and provider arguments requested by the user. Herdr passes arguments after `--` to the underlying CLI; it does not choose or validate model/provider names. Never add approval-bypass or dangerous sandbox flags just to make the loop unattended.

The normal sequence is:

1. Planner inspects the repository and writes `.herdr/PLAN.md`.
2. Planner prompts `coder` and waits for a settled result:

   ```bash
   herdr agent prompt coder \
     "Read .herdr/PLAN.md. Implement the requested change, run the relevant checks, and report exact evidence. Do not dispatch other agents." \
     --wait --timeout 120000
   ```

3. Planner prompts `reviewer` and waits:

   ```bash
   herdr agent prompt reviewer \
     "Read .herdr/PLAN.md, inspect the current diff and test evidence, and write .herdr/REVIEW.md. Do not modify business code. The first non-empty line must be PASS or FIX_REQUIRED." \
     --wait --timeout 120000
   ```

4. Planner reads `.herdr/REVIEW.md`. If it says `FIX_REQUIRED`, it gives the findings to `coder`, waits for the fix and tests, then asks `reviewer` to review again.
5. Use at most two fix-review iterations by default. If the reviewer still reports `FIX_REQUIRED`, stop and report the remaining findings rather than looping indefinitely.
6. If an agent is `blocked`, inspect its state and output before sending keys or answering. Do not blindly resubmit a timed-out prompt.

`agent prompt --wait` tracks Herdr lifecycle state, not the semantic correctness of the work. Always inspect the final diff and test evidence before reporting completion.

### Reliable submission and continuation

Treat prompt submission and workflow continuation as planner-owned responsibilities. Herdr does not automatically wake a planner after a later agent completion.

After `agent prompt`, immediately inspect `agent get` and a short `agent read`. If the target remains `idle` and the task text is visibly sitting in its input box, send logical key `enter` exactly once, then confirm that the state becomes `working` or `blocked`. Do not send another Enter when the agent is already working.

Keep the planner turn active while an agent works by waiting in bounded intervals no longer than 60 seconds and sharing concise progress updates. When a wait returns `blocked`, inspect the UI. If the user must answer an approval or question, record which agent and handoff stage are pending. On the next user turn, inspect that agent first and resume the same wait; do not assume a completion notification will restart the planner automatically.

When the agent becomes `idle` or `done`, read its final output and required handoff artifact immediately. A coder is complete only when its implementation and test evidence are available; then dispatch the reviewer in the same planner workflow. Apply the same submission check and bounded-wait loop to the reviewer.

## Shared-state safety

- Run coder and reviewer sequentially; do not let both modify the same source files concurrently.
- Keep the planner's plan and the reviewer's report separate from business code.
- Preserve unrelated user changes. Do not reset, discard, or close panes/workspaces that this workflow did not create.
- Before coding, record relevant `git status` information. Before review, inspect the diff from the task start when possible.
- Treat tests, logs, and agent claims as evidence to verify, not as proof by themselves.
- Stop for missing authorization, destructive operations, unresolved user decisions, or repeated infrastructure failures.

## Completion report

The planner's final response should state:

- the files and behavior changed;
- the verification commands and results;
- the final review status (`PASS` or unresolved `FIX_REQUIRED` findings);
- any pre-existing failures, assumptions, or follow-up work.

Do not report success merely because all agents reached `idle` or `done`; those states only mean the agents are ready for input.

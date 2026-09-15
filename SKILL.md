---
name: herdr-workflow
description: "Coordinate a sequential planner-coder-reviewer coding workflow with conditional debugging inside Herdr using shared handoff files, agent prompts, tests, and bounded fix loops. Use when the user explicitly asks to orchestrate multiple coding agents in one Herdr workspace."
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
- `debugger`: is started only when the conditional debugging branch is triggered. It diagnoses persistent failures and writes evidence-backed repair guidance without changing source code or tests.

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

When conditional debugging is triggered, the debugger reads `.herdr/PLAN.md`, the current diff, the failing command and output, and the coder's prior repair evidence. It writes `.herdr/DEBUG.md` containing:

1. the exact failure and a minimal reproduction when available;
2. the root-cause hypothesis and supporting evidence;
3. affected files, symbols, assumptions, or interfaces;
4. a minimal recommended repair;
5. commands or observations that would confirm or falsify the diagnosis.

The debugger must not modify source code or tests. The coder remains responsible for applying the diagnosis and running verification.

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

Use named live agents and Herdr's agent surface rather than guessing pane IDs. Create or split a required pane before starting its agent, because `agent start` requires an available shell pane. A debugger pane need not be created or started unless the conditional branch is triggered. Preserve the shared project cwd.

Start agents with the model and provider arguments requested by the user. Herdr passes arguments after `--` to the underlying CLI; it does not choose or validate model/provider names. Never add approval-bypass or dangerous sandbox flags just to make the loop unattended.

The normal sequence is:

1. Planner inspects the repository and writes `.herdr/PLAN.md`.
2. Planner prompts `coder` and waits for a settled result:

   ```bash
   herdr agent prompt coder \
     "Read .herdr/PLAN.md. Implement the requested change, run the relevant checks, and report exact evidence. Do not dispatch other agents." \
     --wait
   ```

3. Planner checks the coder's reported verification. If the relevant checks pass, it prompts `reviewer` and waits:

   ```bash
   herdr agent prompt reviewer \
     "Read .herdr/PLAN.md, inspect the current diff and test evidence, and write .herdr/REVIEW.md. Do not modify business code. The first non-empty line must be PASS or FIX_REQUIRED." \
     --wait
   ```

4. Planner reads `.herdr/REVIEW.md`. If it says `FIX_REQUIRED`, it gives the findings to `coder`, waits for the fix and tests, then asks `reviewer` to review again.
5. Use at most two fix-review iterations by default. If the reviewer still reports `FIX_REQUIRED`, stop and report the remaining findings rather than looping indefinitely.
6. If an agent is `blocked`, inspect its state and output before sending keys or answering. Do not blindly resubmit a timed-out prompt.

### Conditional debugging

Do not run the debugger on the normal path. Trigger it when the same relevant failure remains after the coder's initial attempt and one focused repair attempt, or when the available evidence cannot localize the root cause well enough for a safe repair. This branch may be entered from initial implementation or from a review-requested fix.

When triggered:

1. Start the debugger if it is not already available, then submit one diagnostic prompt with `--wait`:

   ```bash
   herdr agent prompt debugger \
     "Read .herdr/PLAN.md, inspect the current diff and failing test evidence, and write .herdr/DEBUG.md with a minimal reproduction, root-cause hypothesis, supporting evidence, affected locations, recommended repair, and verification steps. Diagnose only; do not modify source code or tests." \
     --wait
   ```

2. Validate that `.herdr/DEBUG.md` addresses the observed failure. If it lacks a reproducible hypothesis or actionable verification, stop and report the unresolved diagnosis rather than asking the coder to guess.
3. Prompt `coder` once to apply the diagnosis, run the specified verification, and report exact evidence.
4. If the relevant checks pass, continue to reviewer. If the same failure remains, stop and report the diagnosis and evidence.

Use at most one debugger-diagnosis/fix cycle by default. Do not alternate indefinitely between coder and debugger.

`agent prompt --wait` tracks Herdr lifecycle state, not the semantic correctness of the work. Always inspect the final diff and test evidence before reporting completion.

### Efficient waiting and recovery

Treat a successful `agent prompt --wait` call as the normal synchronization boundary. Herdr submits the prompt with Enter and waits for the target to reach a settled lifecycle state, so the planner should not build a polling loop around it.

On the normal path:

1. Submit the prompt once with `--wait`. Let the command remain pending until Herdr returns `idle`, `done`, or `blocked`. Add a timeout only when the user or execution environment requires a bounded wait.
2. Use the state returned by `agent prompt --wait` directly. Do not immediately follow submission with `agent get` or a short `agent read` merely to verify that the prompt was accepted.
3. Do not send Enter as a submission fallback, poll at fixed intervals, or produce periodic progress messages solely to keep the planner active.
4. When the target reaches `idle` or `done`, collect its semantic result once. Prefer the required handoff artifact; if the result exists only in the transcript, perform one appropriately sized `agent read`. Do not pair that read with a redundant status query.
5. When the target reaches `blocked`, inspect the blocking UI once and stop for any user decision or authorization that is required. Do not answer approvals or questions by guessing.

Use recovery checks only after an actual exceptional result:

- `agent_prompt_stalled` or a timeout does not prove that the prompt was not submitted. Read the target once before retrying. If it is working, use one `agent wait` call for the settled state. If the task is absent and the agent is ready, resubmit the prompt once.
- Never send Enter blindly after a stalled submission. If text is visibly left unsubmitted and Herdr cannot submit it reliably, report the infrastructure failure instead of adding repeated keystroke heuristics.
- After repeated submission or lifecycle failures, stop and report the observed state rather than looping.

Herdr lifecycle states are synchronization signals, not evidence of semantic correctness. Before reporting completion, verify the required handoff artifact, current diff, and test evidence.

## Shared-state safety

- Run coder, debugger, and reviewer sequentially. Only the coder modifies source files and tests.
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
- the debugger diagnosis and post-repair evidence, if that branch was triggered;
- any pre-existing failures, assumptions, or follow-up work.

Do not report success merely because all agents reached `idle` or `done`; those states only mean the agents are ready for input.

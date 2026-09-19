---
name: herdr-workflow
description: "Coordinate a sequential planner-coder-reviewer coding workflow with selectable quiet or eight-minute coder progress, conditional debugging, and an optional reviewed Git delivery step inside Herdr. Use when the user explicitly asks to orchestrate multiple coding agents in one Herdr workspace."
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

- `planner`: owns orchestration, creates the plan, asks the user to select the coder progress mode, dispatches the other agents, decides whether another loop is needed, presents requested coder progress checkpoints, performs any authorized Git delivery, and reports the final result. The planner may be a Codex agent with access to the `/herdr` skill.
- `coder`: reads the plan and the complete `references/coder-style.md` file from this skill before changing source code or tests, follows that style throughout the work period, runs the relevant checks, records structured progress checkpoints when `progress-8m` is selected, and reports evidence. It does not dispatch other agents or rewrite the plan unless the planner explicitly asks.
- `reviewer`: reads the plan, the complete `references/coder-style.md` file from this skill, the diff, tests, and relevant behavior without changing business code. It writes a structured review result and identifies concrete fixes, including material violations of the coder style contract when they affect readability or the requested research workflow.
- `debugger`: is started only when the conditional debugging branch is triggered. It diagnoses persistent failures and writes evidence-backed repair guidance without changing source code or tests.

Do not infer a role from the model name alone. If the role is ambiguous, ask the user or establish it before dispatching work.

## Coder implementation style

Every participating agent is expected to load this skill, and the planner must
ensure that condition before dispatching it. The implementation style is kept
in [`references/coder-style.md`](references/coder-style.md), which is the single
source of truth for coder-authored code. A coder must read that reference
completely before making any source-code or test changes and follow it
throughout every coder work period, including initial implementation, reviewer
fixes, and debugger-guided repairs. If a target agent cannot load this skill,
do not assume that it can access this reference; stop and report the handoff
problem instead.

If the coder's working context becomes incomplete, uncertain, or appears to
have been compressed, it must re-read `references/coder-style.md` and
`.herdr/PLAN.md` before continuing. Context compression does not itself
guarantee that files are read again. Do not duplicate this reference into
`.herdr/`; `.herdr/` is reserved for task-specific handoff state unless the user
explicitly requests another arrangement.

The reviewer must also read the reference before reviewing coder-authored
changes and should report material style violations when they harm readability,
maintainability, or the requested research-script workflow. The style contract
does not override explicit user, repository, or language constraints; resolve a
direct conflict in `.herdr/PLAN.md`.

## Handoff protocol

Keep coordination artifacts in `.herdr/` at the project root. Create the directory only if it does not exist. Do not add it to `.gitignore` or alter repository configuration unless the user asks.

The planner writes `.herdr/PLAN.md` before implementation. It should contain:

1. the user goal and acceptance criteria;
2. repository findings and constraints;
3. files or components expected to change;
4. implementation and verification steps;
5. unresolved assumptions or questions;
6. the selected coder progress mode (`quiet` or `progress-8m`);
7. the Git delivery mode, remote, and target branch when delivery is in scope.

In `progress-8m` mode, the coder maintains `.herdr/PROGRESS.md` during every coder work period. Each checkpoint replaces the previous checkpoint and includes a timestamp plus:

1. work completed since the previous checkpoint;
2. the problem currently being solved;
3. the next planned action;
4. blockers or decisions needed;
5. the most recent relevant command and result, when available.

The coder writes an initial checkpoint when it starts and refreshes it at material phase changes. While it remains active, no more than eight minutes should pass between checkpoints. A long foreground command may delay a checkpoint; in that case the next checkpoint must identify the command and its outcome. In `quiet` mode, do not require periodic checkpoints or create planner wake-ups solely for progress reporting.

The coder reads `.herdr/PLAN.md` and the complete `references/coder-style.md`
file from this skill before implementing the requested change, runs the
appropriate tests or checks, and leaves the working tree and test evidence
available for review. It must not claim success without reporting the commands
it ran and their outcomes.

When conditional debugging is triggered, the debugger reads `.herdr/PLAN.md`, the current diff, the failing command and output, and the coder's prior repair evidence. It writes `.herdr/DEBUG.md` containing:

1. the exact failure and a minimal reproduction when available;
2. the root-cause hypothesis and supporting evidence;
3. affected files, symbols, assumptions, or interfaces;
4. a minimal recommended repair;
5. commands or observations that would confirm or falsify the diagnosis.

The debugger must not modify source code or tests. The coder remains responsible for applying the diagnosis and running verification.

When Git delivery succeeds, the planner writes `.herdr/PUBLISH.md` containing the selected delivery mode, remote, branch, full commit SHA, push result, pull-request URL when applicable, and the verification and review state used to authorize delivery.

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
2. Immediately before the first coder dispatch, planner asks the user to choose one of these modes unless the user already selected one in the current request:

   - `quiet` (default and recommended): no periodic report; resume planner only when coder settles or becomes blocked.
   - `progress-8m`: while coder is active, present one structured progress report every eight minutes.

   Do not dispatch coder until the selection is known. Record it in `.herdr/PLAN.md` and reuse it for later coder fix tasks unless the user changes it.
3. Planner prompts `coder` according to the selected mode.

   For `quiet`:

   ```bash
   herdr agent prompt coder \
     "Read .herdr/PLAN.md and read the complete references/coder-style.md file from this skill before making any source-code or test changes. Follow that style throughout this coder work period. Implement the requested change, run the relevant checks, and report exact final evidence. Do not dispatch other agents." \
     --wait
   ```

   For `progress-8m`:

   ```bash
   herdr agent prompt coder \
     "Read .herdr/PLAN.md and read the complete references/coder-style.md file from this skill before making any source-code or test changes. Follow that style throughout this coder work period. Implement the requested change and run the relevant checks. Maintain .herdr/PROGRESS.md with a timestamp, completed work, current problem, next action, blockers, and the latest relevant command/result; refresh it at material phase changes and at least every eight minutes while active. Report exact final evidence. Do not dispatch other agents." \
     --wait --timeout 480000
   ```

   If the command times out while the coder is still working, the planner handles a progress checkpoint as described below and then continues with `agent wait coder --timeout 480000`. The timeout ends only the wait; it does not cancel the coder.

4. Planner checks the coder's reported verification. If the relevant checks pass, it prompts `reviewer` and waits:

   ```bash
   herdr agent prompt reviewer \
     "Read .herdr/PLAN.md and the complete references/coder-style.md file from this skill. Inspect the current diff and test evidence for correctness and material violations of the coder style contract, then write .herdr/REVIEW.md. Do not modify business code. The first non-empty line must be PASS or FIX_REQUIRED." \
     --wait
   ```

5. Planner reads `.herdr/REVIEW.md`. If it says `FIX_REQUIRED`, it gives the findings to `coder` with an explicit instruction to re-read `.herdr/PLAN.md` and the complete `references/coder-style.md` file before editing, waits for the fix and tests, then asks `reviewer` to review again.

   A review-fix prompt should have this shape:

   ```bash
   herdr agent prompt coder \
     "Re-read .herdr/PLAN.md and the complete references/coder-style.md file from this skill before editing. Apply the concrete findings in .herdr/REVIEW.md, run the relevant checks, and report exact final evidence. Do not dispatch other agents." \
     --wait
   ```
6. Use at most two fix-review iterations by default. If the reviewer still reports `FIX_REQUIRED`, stop and report the remaining findings rather than looping indefinitely.
7. If an agent is `blocked`, inspect its state and output before sending keys or answering. Do not blindly resubmit a timed-out prompt.

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
3. Prompt `coder` once to re-read `.herdr/PLAN.md` and the complete `references/coder-style.md` file from this skill, apply the diagnosis, run the specified verification, and report exact evidence:

   ```bash
   herdr agent prompt coder \
     "Re-read .herdr/PLAN.md and the complete references/coder-style.md file from this skill before editing. Apply the actionable diagnosis in .herdr/DEBUG.md, run the specified verification, and report exact evidence. Do not dispatch other agents." \
     --wait
   ```
4. If the relevant checks pass, continue to reviewer. If the same failure remains, stop and report the diagnosis and evidence.

Use at most one debugger-diagnosis/fix cycle by default. Do not alternate indefinitely between coder and debugger.

### Coder progress modes

`quiet` is the low-cost default. Use one foreground `agent prompt --wait` or `agent wait` call without a scheduled progress timeout. The planner remains suspended until coder settles; do not read `.herdr/PROGRESS.md` or generate periodic user updates.

`progress-8m` is an intentional exception to the no-polling normal path and applies only while the coder is actively working. Use it for the initial implementation and later coder repair tasks after the user selects it.

At each 480-second timeout:

1. Use the existing wait result to confirm that the coder has not settled; do not add an immediate `agent get`.
2. Read `.herdr/PROGRESS.md` once.
3. Present a concise user-facing update with the checkpoint timestamp and exactly these fields: completed, current problem, next action, and blockers.
4. Resume waiting with one `herdr agent wait coder --timeout 480000` call in the same orchestration turn. Do not end the workflow or require a user reply unless a real decision is blocked.
5. Repeat only while the coder remains active.

If `.herdr/PROGRESS.md` is missing or older than the previous eight-minute window, do not invent progress. Report that no fresh structured checkpoint is available, read the coder's visible output once if needed, and ask the coder once to refresh the file at the next safe boundary without resubmitting the main task.

A progress checkpoint is informational. It must not be treated as task completion, review evidence, or permission to perform Git delivery. This cadence deliberately adds model and orchestration work; do not apply it to reviewer or debugger waits unless the user explicitly asks.

`agent prompt --wait` tracks Herdr lifecycle state, not the semantic correctness of the work. Always inspect the final diff and test evidence before reporting completion.

### Planner continuation invariant

After dispatching another agent, the planner must keep the current orchestration turn open until that agent settles and the next handoff has been processed. A Herdr completion notification can update terminal or UI state, but it does not by itself create a new planner model turn.

1. Prefer a foreground `herdr agent prompt <name> ... --wait` call. If the prompt was already submitted, use a foreground `herdr agent wait <name>` call instead.
2. Do not detach the Herdr wait, launch it with shell backgrounding, or end the planner turn while expecting a later notification to resume reasoning.
3. If the planner's execution host yields a session or task handle for a long-running foreground command, use that host's supported wait/resume operation on the same handle. Tool names differ between hosts; do not assume that a command such as `WaitFor` exists.
4. If a host-level wait times out while the Herdr command is still running, resume waiting on the same command or agent. Do not resubmit the task.
5. When the target settles, read the required handoff once and continue immediately in the same planner turn: coder result to reviewer, reviewer findings to coder, or final `PASS` to the delivery gate.

The planner may end the turn before the workflow settles only when the target is genuinely blocked on user input or authorization, the user requested manual pacing, or the host cannot maintain a blocking wait. In that case, report the exact pending agent and handoff stage instead of claiming automatic continuation.

### Efficient waiting and recovery

For reviewer, debugger, and `quiet` coder tasks, treat a successful `agent prompt --wait` call as the normal synchronization boundary. Herdr submits the prompt with Enter and waits for a settled lifecycle state, so the planner should not build a polling loop around those roles. Only `progress-8m` coder tasks use explicit eight-minute progress windows.

Outside the coder progress protocol:

1. Submit the prompt once with `--wait`. Let the command remain pending until Herdr returns `idle`, `done`, or `blocked`. Add a timeout only when the user or execution environment requires a bounded wait.
2. Use the state returned by `agent prompt --wait` directly. Do not immediately follow submission with `agent get` or a short `agent read` merely to verify that the prompt was accepted.
3. Do not send Enter as a submission fallback, poll at fixed intervals, or produce periodic progress messages solely to keep the planner active.
4. When the target reaches `idle` or `done`, collect its semantic result once. Prefer the required handoff artifact; if the result exists only in the transcript, perform one appropriately sized `agent read`. Do not pair that read with a redundant status query.
5. When the target reaches `blocked`, inspect the blocking UI once and stop for any user decision or authorization that is required. Do not answer approvals or questions by guessing.

A 480-second timeout produced in `progress-8m` mode is a scheduled checkpoint, not an infrastructure error. Other timeouts use the recovery rules below.

Use recovery checks only after an actual exceptional result:

- `agent_prompt_stalled` or a timeout does not prove that the prompt was not submitted. Read the target once before retrying. If it is working, use one `agent wait` call for the settled state. If the task is absent and the agent is ready, resubmit the prompt once.
- Never send Enter blindly after a stalled submission. If text is visibly left unsubmitted and Herdr cannot submit it reliably, report the infrastructure failure instead of adding repeated keystroke heuristics.
- After repeated submission or lifecycle failures, stop and report the observed state rather than looping.

Herdr lifecycle states are synchronization signals, not evidence of semantic correctness. Before reporting completion, verify the required handoff artifact, current diff, and test evidence.

## Git delivery

Git delivery is a post-review gate owned by the planner, not a separate agent role. Determine the delivery mode before implementation and record it in `.herdr/PLAN.md`:

- `no-push`: do not create a commit or push;
- `commit-only`: create a local commit but do not push;
- `push-branch`: commit and push the current task branch;
- `push-and-pr`: commit, push the task branch, and create a pull request when the available tools support it.

If the user has not explicitly authorized a delivery mode, use `no-push`. A request to edit code does not by itself authorize a remote push.

For a push mode, resolve the remote and branch before coding. Prefer a task branch created from the intended base before implementation. Do not push directly to the default or protected branch unless the user explicitly requested that exact target.

Run the delivery gate only after the final reviewer status is `PASS`, all required checks have passed, and no debugger finding remains unresolved:

1. Inspect `git status --short`, the current branch, configured remote, staged changes, and the task diff.
2. Exclude unrelated user changes and coordination artifacts under `.herdr/` unless the user explicitly requested them in version control.
3. Stage only the files belonging to the approved task; do not use a broad add command when unrelated changes may exist.
4. Create a clear commit and verify its full SHA and contents.
5. For an authorized push mode, push the task branch without force.
6. For `push-and-pr`, create the pull request only after the push succeeds.
7. Write the verified outcome to `.herdr/PUBLISH.md`.

Do not deliver when review says `FIX_REQUIRED`, tests are failing, the branch or remote is ambiguous, unrelated changes cannot be separated, authorization is missing, or the operation would require force push or history rewriting. Stop and ask for direction instead.

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
- the Git delivery mode, branch, commit SHA, push result, and pull-request URL when applicable;
- any pre-existing failures, assumptions, or follow-up work.

Do not report success merely because all agents reached `idle` or `done`; those states only mean the agents are ready for input.

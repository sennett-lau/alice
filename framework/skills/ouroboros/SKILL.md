---
name: ouroboros
preamble-tier: 4
version: 1.0.0
description: |
  Continuous improve loop: read open `docs/todos/findings/`, run `diagnosis`
  when the backlog is empty, fan candidate resolutions through `hugh`/`diana`
  in parallel with `resolution-evaluator` feedback loops, merge passing branches sequentially, run
  scrutiny validation, and repeat. Use when asked for ouroboros, self-improve loop,
  or keep improving until findings are clean.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
---

## Preamble (run first)

```bash
eval "$(.alice/bin/alice-slug 2>/dev/null || true)"
ROOT="${ROOT:-$(git rev-parse --show-toplevel)}"
RUN_TS=$(date -u +%Y%m%dT%H%M%SZ)
mkdir -p "$ROOT/.alice/mem/ouroboros" "$ROOT/docs/todos/findings"
echo "BRANCH: ${BRANCH:-$(git branch --show-current 2>/dev/null || echo unknown)}"
echo "RUN_TS: $RUN_TS"
```

# Ouroboros

## Overview

Runs an iterative diagnose -> resolve -> evaluate -> merge -> re-diagnose loop. `diagnosis` discovers user-facing issues, `hugh` fans candidate resolutions out to isolated `diana` workers, each worker uses `resolution-evaluator` until pass or handback, and the main session merges passing branches only after scrutiny validation.

## When to Use

- Use when asked for `/ouroboros`, "ouroboros", "self-improve loop", "keep resolving until clean", or "iteratively improve the app".
- Use when findings in `docs/todos/findings/` should be resolved with parallel worktrees and repeated validation.
- Use when the user accepts heavy automation and local branch/merge activity.

When NOT to use:

- Do not use for a single known issue; use `diana` or implement normally.
- Do not use for report-only testing; use `diagnosis` or `qa --report-only`.
- Do not use if the working tree is dirty and the user has not agreed to preserve or isolate those changes.
- Do not use when pushing/deploying is required; Ouroboros is local-only.

## Process

### Arguments

```text
/ouroboros [MAX_ITER] [--dry-run]
```

- `MAX_ITER`: optional cap. Default is unbounded until clean.
- `0` or `--dry-run`: inventory state and report what iteration 1 would do.

### Required Components

- Skills: `diagnosis`, `hugh`, `diana`.
- Agents: `user-testing-validator`, `findings-triager`, `resolution-evaluator`.
- Project-local browser: `.alice/skills/browse/dist/browse`.
- Curated backlog: `docs/todos/findings/`.

If any required skill or agent is not registered, abort and tell the user to sync/restart the agent session. Do not substitute generic agents for this loop.

### State

Run state lives under `.alice/mem/ouroboros/<run-ts>/`:

```text
state.json
iter-01.md
iter-02.md
summary.md
```

State records iteration number, open findings selected, hugh run metadata, diana self-validation results, merge outcomes, resolved slugs, requeued slugs, and errors.

### Step 1 - Inventory Findings

List open findings:

```bash
find docs/todos/findings -maxdepth 1 -name '*.md' \
  ! -name overview.md ! -name _dropped.md -print
```

For each file, read the metadata block. Open means `Status: open`. Ignore `in-progress` unless the user explicitly asks to recover interrupted work. Sort high -> medium -> low, then oldest first.

If no open findings exist, run `diagnosis` as the User-Testing Validation pass. When it completes, inventory again. If still empty, terminate `clean`.

### Step 2 - Build the Hugh Resolution Batch

Create one Hugh feature per open finding. Each feature brief must be generic and include:

```markdown
- Title: Resolve: <slug>
  Source: docs/todos/findings/<slug>.md
  Description: |
    Resolve this finding in an isolated diana worktree.

    Procedure:
    1. Read the finding. Mark Status: in-progress while working.
    2. Implement the smallest correct resolution. Add regression coverage when the repo
       has an established test surface.
    3. Commit the change and capture the commit SHA.
    4. Start any required dev server using the port base allocated by Hugh for
       this worktree. Do not use ports owned by the main Ouroboros session.
    5. Spawn `resolution-evaluator` with:
       slug=<slug>
       branch=<your branch>
       commit_sha=<candidate commit>
       site_url=<your worktree's site URL>
       server_url=<your worktree's server URL if any>
       affected_urls=<finding URLs rebased to your dev server>
       run_dir=.alice/mem/diana/<slug>/resolution-evaluation/
    6. If rating >= 90, record PASS and stop.
    7. If rating < 90 and feedback is actionable, make another commit and
       re-run `resolution-evaluator`. Continue until PASS or you judge the finding
       cannot be resolved safely in this loop.
    8. Before handback, set the finding Status back to open. Ouroboros marks
       resolved only after merge and scrutiny validation.

    Final report must include:

    ## Self-validation
    final_status: PASS | GAVE_UP | ERRORED
    final_rating: <0-100>
    attempts: <N>
    branch: <branch>
    last_commit: <SHA>
    notes: <one paragraph>

  Effort tier: high
  Constraints:
    - Follow Alice rules and project CLAUDE.md.
    - Stay inside your worktree.
    - Do not run diagnosis.
    - Do not push, open a PR, or merge.
```

Dispatch `/hugh` with all finding features, `--effort=high`, and an explicit max parallel value suitable for the machine and number of findings. Hugh owns worktree isolation and port allocation.

### Step 3 - Collect Diana Results

Wait for Hugh to drain all Dianas. Parse each `## Self-validation` block:

- `PASS`: eligible for merge gate.
- `GAVE_UP`: skip merge and append feedback to the finding.
- `ERRORED`: skip merge and append error notes to the finding.

Do not run `resolution-evaluator` from the Ouroboros session. It runs inside each Diana's worktree.

### Step 4 - Merge Gate

Process PASS branches sequentially. Before each merge:

1. Ensure current branch is the intended integration branch.
2. Ensure working tree is clean except expected finding-doc updates.
3. Verify the source branch exists.

Merge:

```bash
git merge --ff-only <branch> || git merge --no-ff --no-edit <branch>
```

On conflict, abort the merge, append `## Merge feedback - iter <N>` to the finding, and requeue.

### Step 5 - Scrutiny Validation

After each successful merge, run cheap deterministic trunk checks. Infer commands from project `CLAUDE.md`; if not specified, use the repo's obvious package scripts or language-native checks.

Minimum gate:

1. Lint or equivalent static check if available.
2. Typecheck or compile if available.
3. Dev boot or smoke start for the app.
4. Browser sanity on the main URL and any route touched by the finding:
   - page responds
   - console has no critical runtime/hydration errors
   - network has no unexpected failures

If any gate fails, revert the merge, append feedback to the finding, and requeue. Do not weaken the gate to make progress.

### Step 6 - Resolve or Requeue Findings

For each merged and scrutiny-passed finding:

- Set `Status: resolved`.
- Append `## Resolution` with merge commit, source branch, final rating, attempts, and summary.
- Refresh `docs/todos/findings/overview.md`.

For GAVE_UP, ERRORED, conflict, or regression:

- Keep `Status: open`.
- Append `## Diana feedback - iter <N>`, `## Diana error - iter <N>`, or `## Merge feedback - iter <N>`.
- Keep branch refs for later work, but remove disposable worktrees after logging.

### Step 7 - Iteration Log and Loop

Write `.alice/mem/ouroboros/<run-ts>/iter-<NN>.md` with:

- Findings selected.
- Hugh/Diana outcomes.
- Merge outcomes.
- Scrutiny validation results.
- Resolved, requeued, and errored slugs.

Update `state.json`. If `MAX_ITER` is reached, terminate `capped`. Otherwise return to Step 1. The next time the backlog is empty, `diagnosis` is the behavioral re-evaluation pass.

### Step 8 - Termination Report

Write `summary.md` and report:

```text
/ouroboros <clean|capped|errored|interrupted> - iter <N> of <MAX or unbounded>

Resolved this run: <count>
- <slug> - rating <score> - merge <sha>

Still open: <count>
- <slug> - last status <PASS|GAVE_UP|ERRORED|REGRESSION> - rating <score>

Run dir: .alice/mem/ouroboros/<run-ts>/
```

Keep user-facing output under 300 words.

## Scrutiny Validation vs User-Testing Validation

- **User-Testing Validation** is `diagnosis`: independent agents use the UI end to end and create findings.
- **Resolution Evaluation** is `resolution-evaluator`: a Diana-local rating of one candidate resolution against one finding.
- **Scrutiny Validation** is the main session's post-merge gate: compile/static checks, app boot, and browser sanity.

Do not collapse these into one step. They catch different failure modes.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "There are no findings, so we are done." | An empty backlog may mean nobody looked. Run `diagnosis` once before declaring clean. |
| "Ouroboros can run the resolution evaluator itself." | Evaluator feedback belongs inside the Diana resolution loop, against the worker's branch and dev server. |
| "A PASS branch can be merged with the others in parallel." | Merges mutate trunk. Sequential merge plus scrutiny is the safety boundary. |
| "A compile pass is enough." | Boot and browser sanity catch runtime failures static checks miss. |
| "A conflict can be resolved quickly in main." | Conflict resolution is feature work. Requeue with feedback so Diana handles it in isolation. |
| "The loop should push or open PRs." | Ouroboros is local automation. Remote side effects stay human-controlled. |

## Red Flags

- `resolution-evaluator` is spawned by the main Ouroboros session.
- Hard-coded ports appear in prompts instead of passing URLs/port bases from Hugh/Diana context.
- Findings are marked resolved before merge and scrutiny validation.
- Multiple PASS branches merge concurrently.
- `diagnosis` and `ouroboros` run concurrently in different sessions.
- The loop weakens lint/typecheck/boot checks after a failure.

## Verification

- [ ] Required skills and agents are registered.
- [ ] Open findings were read from `docs/todos/findings/`, or `diagnosis` ran because none existed.
- [ ] Hugh received one feature per open finding.
- [ ] Each Diana result included a `## Self-validation` block.
- [ ] Only PASS branches entered merge gate.
- [ ] Every merged branch passed scrutiny validation before its finding was resolved.
- [ ] Requeued findings include actionable feedback.
- [ ] `.alice/mem/ouroboros/<run-ts>/` contains state, iter logs, and summary.

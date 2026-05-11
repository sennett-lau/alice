---
name: diagnosis
preamble-tier: 4
version: 1.0.0
description: |
  Fan out parallel `user-testing-validator` agents against a running application,
  then triage their raw end-to-end findings into `docs/todos/findings/`. Use
  when asked to run diagnosis, user-testing validation, multi-agent dogfood,
  parallel UI testing, or find gaps and bugs through agentic end-to-end testing.
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
mkdir -p "$ROOT/.alice/mem/user-testing-validator" "$ROOT/docs/todos/findings"
echo "BRANCH: ${BRANCH:-$(git branch --show-current 2>/dev/null || echo unknown)}"
```

# Diagnosis

## Overview

Runs a User-Testing Validation pass: multiple independent `user-testing-validator` agents use the application from the UI, write raw evidence, and a `findings-triager` consolidates real issues into `docs/todos/findings/`.

## When to Use

- Use when asked for `/diagnosis`, "diagnosis", "user-testing validation", "multi-agent testing", "dogfood with users", or "find gaps / bugs through end-to-end agents".
- Use when the goal is discovery and triage, not immediate fixing.
- Use before `ouroboros` when the findings backlog may be stale or empty.

When NOT to use:

- Do not use for a single targeted browser check; use `browse`.
- Do not use for test-and-fix in one session; use `qa` or `ouroboros`.
- Do not use for source-only review; use `review` or `security-audit`.

## Process

### Arguments

```text
/diagnosis [N] [STEP_BUDGET] [ENTRY_URL]
```

- `N`: number of parallel testers. Default `3`, cap `8`.
- `STEP_BUDGET`: soft action cap per tester. Default `25`, range `10..60`.
- `ENTRY_URL`: target app URL. If absent, infer from project docs or ask once.

### Step 1 - Setup Gate

1. Load `.alice/rules/sub-agent-orchestration.md`.
2. Confirm the `browse` binary exists:
   ```bash
   B="$ROOT/.alice/skills/browse/dist/browse"
   [ -x "$B" ] && echo READY || echo NEEDS_SETUP
   ```
   If missing, ask before running `.alice/skills/browse/setup`.
3. Determine `ENTRY_URL`:
   - Prefer explicit user argument.
   - Else read project `CLAUDE.md` dev-server notes.
   - Else probe common local URLs only long enough to identify a running app.
   - If still unknown, ask the user for the URL.
4. Sanity-browse once:
   ```bash
   "$B" goto "$ENTRY_URL"
   "$B" text | head
   "$B" console
   "$B" network
   ```
   If the page cannot render, stop and report the failure as setup-blocking.
5. Confirm `user-testing-validator` and `findings-triager` are registered agents. If this skill was just added mid-session and the harness cannot see them, tell the user to restart the agent session after syncing Alice.

### Step 2 - Prepare Personas

Pick `N` distinct personas. Prefer `.alice/mem/user-testing-validator/personas-seed.md` if present. Otherwise use generic archetypes:

- skeptical newcomer trying to understand whether the product is worth time.
- goal-driven evaluator trying to complete one obvious core workflow.
- privacy-cautious visitor avoiding unnecessary signup or data sharing.
- impatient mobile user watching for layout shifts, slow feedback, and dead ends.
- returning user checking whether a remembered workflow still makes sense.
- power user looking for keyboard, filtering, bulk, or advanced controls.
- accessibility-minded user relying on labels, focus order, and clear state.
- edge-case user trying back, reload, half-filled forms, and duplicate clicks.

If `N > 8`, stop and suggest multiple smaller runs.

### Step 3 - Fan Out Testers

In one Agent dispatch batch, spawn `N` `user-testing-validator` agents in parallel. Each prompt must include:

- Persona.
- `Entry URL: <ENTRY_URL>`.
- `Mode: fresh`.
- `Step budget: <STEP_BUDGET>`.
- `Auth: <caller-provided auth or anonymous>`.
- Instruction to read `user-testing-validator` spec before starting.
- Instruction to return the output contract exactly.

Follow `.alice/rules/sub-agent-orchestration.md`: capture agent IDs, poll progress at least once per minute, surface stuck agents instead of silently killing them, and drain all agents before triage.

### Step 4 - Triage

After testers return, invoke `findings-triager` once:

```text
Triage all active raw findings under .alice/mem/user-testing-validator/.
The following testers just completed in this diagnosis run: <names>.
Process older unarchived tester dirs too if present. Cluster, promote,
drop with reasons, archive processed dirs, and return your output contract.
```

The triager writes `docs/todos/findings/overview.md`, detail files, and `_dropped.md`.

### Step 5 - Report

Return under 300 words:

```text
/diagnosis complete

Target: <ENTRY_URL>
Agents: <N> (<names>)
- <name>: <status> - <high>/<medium>/<low> findings - <steps> steps

Raw findings: <total>
Triaged into: <clusters> clusters (<high>/<medium>/<low>)
Top issues:
- <slug> - <severity> - <summary>

Index: docs/todos/findings/overview.md
Raw workspaces: .alice/mem/user-testing-validator/
```

Flag convergent findings where at least 3 independent testers reported the same high-severity issue.

## State and Local Configuration

- Raw run memory: `.alice/mem/user-testing-validator/`.
- Editable flow hints: `.alice/mem/user-testing-validator/default-flows.md`.
- Persona seeds: `.alice/mem/user-testing-validator/personas-seed.md`.
- Local notes: `.alice/mem/user-testing-validator/notes.md`.
- Curated backlog: `docs/todos/findings/`.

Do not store editable flows under `framework/agents/`, `.alice/agents/`, or `.claude/agents/`. Agent files are framework code; `.alice/mem/` is the mutable per-project calibration surface.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll run one careful tester instead of several." | The point is independent expectation formation. Parallel disagreement and convergence are the signal. |
| "I'll read the source to help the testers." | That destroys the user-testing premise. Testers must judge from the UI. |
| "Raw findings are enough." | Raw persona notes overlap and include noise. Durable backlog entries require triage. |
| "The dev server is probably fine." | A broken startup or blank homepage invalidates every downstream result. Sanity-browse first. |
| "Flow hints belong beside the agent." | Editable calibration is project memory and belongs under `.alice/mem/user-testing-validator/`. |

## Red Flags

- Tester prompts mention source files, wiki pages, schemas, or implementation details.
- Agents run sequentially without a reason.
- Raw findings are copied straight into `docs/todos/findings/` without dedupe.
- Evidence paths point to active tester dirs after triage instead of archive dirs.
- `default-flows.md` appears under an agent source folder.

## Verification

- [ ] `browse` is built and the target URL rendered before fan-out.
- [ ] `N` tester agents ran with distinct personas and drained.
- [ ] Raw findings exist under `.alice/mem/user-testing-validator/` or testers reported zero findings.
- [ ] `findings-triager` produced or refreshed `docs/todos/findings/overview.md`.
- [ ] Dropped findings, if any, are recorded in `_dropped.md`.
- [ ] Final report includes target URL, tester outcomes, cluster counts, and the backlog index path.

---
name: agentic-readiness
preamble-tier: 4
version: 1.0.0
description: |
  Evaluate how agent-friendly this project is — whether a coding agent can
  operate it the way a human developer can (run it, log in to it, test it,
  observe it, fix a bug end to end) — then write a dated scorecard plus
  concrete improvement suggestions into docs/wiki/agentic-readiness/ and
  triage each suggestion with the user (work on it now / TODO for later /
  skip). Use when asked to "run agentic readiness", "/agentic-readiness",
  "how agent-ready is this repo", "agent-friendliness review", "can an agent
  operate this project", or when /sync offers the first-run review.
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
# Project-local state dir — all session data under .alice/mem/ (gitignored).
eval "$(.alice/bin/alice-slug 2>/dev/null || true)"
ROOT="${ROOT:-$(git rev-parse --show-toplevel 2>/dev/null || echo .)}"
mkdir -p "$ROOT/.alice/mem"
if [ -f "$ROOT/.alice/mem/agentic-readiness.json" ]; then
  echo "MODE: re-run"; cat "$ROOT/.alice/mem/agentic-readiness.json"
else
  echo "MODE: first-run"
fi
```

# Agentic Readiness Review

## Overview

Assesses whether a coding agent can operate this project the way a human developer can — human-parity operations, testing foundation, dev-server parallelism, observability access, and the end-to-end bug loop — calibrated to the product type. Produces a dated scorecard at `docs/wiki/agentic-readiness/overview.md`, one page per improvement suggestion, and a user triage pass that turns accepted suggestions into work or TODOs.

## When to Use

- Use when asked for `/agentic-readiness`, "agentic readiness", "agent-friendliness review", "how agent-ready is this repo", or "can an agent work on this project like a human".
- Use when `/sync` offers the first-run review and the user accepts.
- Use periodically (e.g. after major infra changes: new test framework, new deploy story, new observability tooling) to refresh the scorecard.
- Use before large agent-driven initiatives (`hugh`, `ouroboros`) to find the operability gaps that would make them fail.

**When NOT to use:**

- Do not use to find product bugs — use `diagnosis` or `qa`.
- Do not use for security posture — use `security-audit`.
- Do not use to review a branch diff — use `review`.
- Do not use as a license to refactor: this skill only implements changes the user explicitly approves in triage, and non-trivial approved work still routes through `/plan`.

## Process

You are auditing the **project as an agent's workplace**, not the product's quality. The question for every probe is: "could a coding agent do this without a human doing it for them?" Evidence first — read real files, run cheap read-only commands, and cite what you found. Never guess a score from vibes, and never run destructive, deploy, or production-touching commands as part of the review.

### Step 1 — Establish mode and prior state

1. Run the preamble. `MODE: first-run` → fresh review. `MODE: re-run` → read the printed JSON (previous scores, date) and read the existing `docs/wiki/agentic-readiness/` folder in full before re-assessing.
2. Orient: read `CLAUDE.md`, `docs/README.md`, `docs/wiki/README.md`, `docs/wiki/architecture.md` (if present), and the project manifests (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, etc.).

### Step 2 — Detect the product type (calibration)

Classify the project: web app, API/service, CLI tool, library/SDK, mobile app, desktop app, game, data/ML pipeline, monorepo mix. Base it on manifests, entry points, and `CLAUDE.md` — not on assumptions.

State the detected type and its **realistic ceiling** before scoring anything. A library has no dev server to parallelize; a game's UI loop may never be fully agent-drivable; a CLI can reach full parity cheaply. Read the calibration table in `.alice/skills/agentic-readiness/references/criteria.md` (Criterion 5) and grade every subsequent criterion against what is reasonable **for this type** — mark inapplicable criteria `N/A` instead of scoring them 0.

If the repo is a monorepo with genuinely different product types per package, assess per package where they differ and say so in the scorecard summary column.

### Step 3 — Assess the criteria

Read `.alice/skills/agentic-readiness/references/criteria.md` and work through the five scored criteria in order:

1. **Human-parity operations** — login/auth flows, UI actions, data/admin controls, deployments, CI configuration and triggering.
2. **Testing foundation** — test codebase exists, adding a case is easy, unit/E2E/eval-regression setups, mocks/fixtures/third-party-call handling.
3. **Dev server & parallelism** — reliable dev-server startup, worktree-viable parallel development, multiple isolated instances (ports, DBs, env), data seeding, dev-only data tools, cheap multi-checkout installs.
4. **Observability access** — logging/monitoring/error handling exist AND the agent can actually read that data.
5. **The end-to-end bug loop** — from a minimal bug description: locate evidence → reproduce → root-cause → fix → regression test → self-review → PR. Assess each link.

For each criterion:

- Run the probes listed in `references/criteria.md`. Cheap and read-only by default; anything heavier (starting the dev server, running the full test suite) only if it is quick, safe, and non-destructive — ask first when unsure.
- Record 2–5 lines of **evidence** (file paths, command output, doc quotes). A score with no evidence is invalid.
- Score 0–4 against the anchors in the reference file, calibrated to the product type from Step 2. Use `N/A` where the type makes the criterion meaningless.
- Draft the improvement suggestions this criterion surfaces (Step 4 writes them out). One suggestion = one concrete, self-contained gap.

### Step 4 — Write the wiki folder

All output lands in the **adopter repo's** `docs/wiki/agentic-readiness/` (create the folder on first run).

1. **`overview.md`** — from `.alice/skills/agentic-readiness/templates/overview-template.md`: review date, detected product type + ceiling note, scorecard table (five criteria, score, one-line summary), suggestion index table, review history.
2. **One file per suggestion** — `<slug>.md` (kebab-case noun phrase, e.g. `parallel-dev-databases.md`), from `.alice/skills/agentic-readiness/templates/suggestion-template.md`: what's missing, why it matters for agents, concrete implementation direction, rough effort (S/M/L/XL), status, triage record. The slug doubles as the TODO slug if the suggestion is deferred in Step 5.
3. **Index line** — ensure `docs/wiki/README.md` has one index entry for the folder's entry point (suggestion pages are reached via the overview, they do not get their own index lines):

   ```markdown
   - [agentic-readiness/overview.md](agentic-readiness/overview.md) — agent-readiness scorecard + open improvement suggestions; query when improving agent operability or before large agent-driven initiatives
   ```

**Re-runs:** rewrite `overview.md` with the new date and scorecard, carrying the previous `Review history` lines forward and appending one line for this run. For each existing suggestion file: gap now closed → `Status: Done` + date; still open → update `Last confirmed`; obsolete/replaced → `Status: Skipped` with a one-line reason (e.g. "superseded by `<slug>`"). New gaps get new files. Never silently delete a suggestion file — statuses are the history.

### Step 5 — Triage with the user

Present every suggestion whose status is `Proposed` (on re-runs, do not re-ask about items already `Todo`, `In progress`, `Done`, or `Skipped` unless their evidence changed). For each — singly or in batches of up to 4 — use `AskUserQuestion`:

```
<n>. <title> — [P<x>, effort <S/M/L/XL>] <one-line what/why>
   A) Work on it now
   B) Later — create a TODO
   C) Skip
```

Include a short RECOMMENDATION line ranking which items pay off most for agents. Then act **only** on the user's choices:

- **A) Now:** trivial (one-file, low-risk) → implement directly with a surgical diff. Non-trivial → scaffold via `/plan <slug>` per the adopter SOP and stop at the spec for user sign-off — this skill does not bulldoze into implementation. Either way set the suggestion `Status: In progress` and link the plan folder if one exists.
- **B) Later:** create `docs/todos/<slug>.md` from `.alice/templates/todo.md` — What/Why/Context copied and adapted from the suggestion page, Priority and Effort carried over, `Status: Backlog`, and a References link back to `docs/wiki/agentic-readiness/<slug>.md`. Append the one-liner to `## Backlog` in `docs/todos/overview.md` in its documented format. Set the suggestion `Status: Todo`.
- **C) Skip:** set `Status: Skipped`, record the date and the user's reason (if given) in the suggestion's triage record. Skipped items are not re-raised on re-runs unless the underlying evidence changes.

### Step 6 — Refresh the state marker

Write `.alice/mem/agentic-readiness.json` (this is what `/sync` checks to decide whether to offer the first-run review):

```json
{
  "last_run": "YYYY-MM-DD",
  "skill_version": "1.0.0",
  "product_type": "<detected type>",
  "overall_score": 13,
  "max_score": 20,
  "scores": {
    "human-parity-operations": 3,
    "testing-foundation": 2,
    "dev-server-parallelism": 3,
    "observability-access": 2,
    "bug-loop": 3
  },
  "suggestions": { "proposed": 0, "todo": 2, "in_progress": 1, "done": 1, "skipped": 1 }
}
```

`max_score` = 4 × number of scored (non-`N/A`) criteria. `N/A` criteria appear in `scores` as `null`. `suggestions` counts files in `docs/wiki/agentic-readiness/` by status after triage.

### Step 7 — Report

Return under 250 words:

```text
/agentic-readiness complete

Product type: <type> — <ceiling note>
Overall: <N>/<max> (<prev N/max on re-runs, with delta>)
  1. Human-parity operations   <score>  <one-liner>
  2. Testing foundation        <score>  <one-liner>
  3. Dev server & parallelism  <score>  <one-liner>
  4. Observability access      <score>  <one-liner>
  5. End-to-end bug loop       <score>  <weakest link: ...>

Suggestions: <total> (now: X, todo: Y, skipped: Z)
Scorecard: docs/wiki/agentic-readiness/overview.md
Marker: .alice/mem/agentic-readiness.json
```

## Scoring

| Score | Meaning |
|-------|---------|
| 4 | An agent does this unassisted today — documented, working, verified by probe |
| 3 | Works with minor friction (one undocumented step, one manual assist) |
| 2 | Partially possible — significant gaps, agent needs human help for common cases |
| 1 | Technically possible but undocumented, fragile, or prohibitively slow |
| 0 | Not possible for an agent today |
| N/A | Not applicable to this product type — excluded from `max_score` |

Score against the product type's realistic ceiling (Step 2), not an absolute ideal. Every score cites evidence.

## Interaction with other skills

- `/sync` offers this review once per repo (keyed on `.alice/mem/agentic-readiness.json`) after its migration step.
- Accepted non-trivial suggestions route through `/plan`; deferred ones become `docs/todos/` entries on the normal backlog rails.
- Criterion probes may reference `browse` / `setup-browser-cookies` availability (UI parity) and the test conventions that `qa` and `review` depend on — this skill assesses those foundations, it does not run those workflows.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I can score this from the README, no need to probe." | READMEs describe intent, not reality. A dev-server command that errors, or a test suite that fails on a clean checkout, is exactly what this review exists to catch. Run the cheap probes. |
| "This project type can never score well, so the review is pointless." | Calibration is the point. A library scoring 4/4 on its realistic ceiling is more agent-ready than a web app at 2/4. Grade against the ceiling and say what the ceiling is. |
| "I'll just fix the gaps while I'm here." | Unapproved fixes are scope creep. The contract is: written suggestions, then user triage, then only the approved work — with `/plan` in front of anything non-trivial. |
| "One big suggestions file is tidier than many small ones." | One file per suggestion is what makes triage, TODO promotion, and re-run status tracking possible. The overview is the aggregation layer. |
| "Skipped last time, so I'll drop the file." | Statuses are the history. Deleting a skipped suggestion guarantees the next run re-raises it and re-litigates the same decision. |
| "The marker file is optional bookkeeping." | `/sync` keys its first-run offer on the marker. Skip it and every future sync nags the user about a review that already happened. |

## Red Flags

- A criterion scored with no evidence lines under it.
- A scorecard with no stated product type or ceiling.
- Suggestions implemented before the user chose "work on it now".
- A deferred suggestion with no `docs/todos/<slug>.md` file or no backlog line in `docs/todos/overview.md`.
- `docs/wiki/agentic-readiness/` updated but `docs/wiki/README.md` has no index line for it.
- A re-run that overwrote suggestion files' statuses or dropped the review history.
- `.alice/mem/agentic-readiness.json` missing or stale after the run.
- The review ran deploy, migration, or production-touching commands "to verify".

## Verification

- [ ] Product type detected and stated, with its realistic ceiling.
- [ ] All five criteria scored (or `N/A`d) with cited evidence, per `references/criteria.md`.
- [ ] `docs/wiki/agentic-readiness/overview.md` written with date, scorecard, suggestion index, and review history.
- [ ] One `<slug>.md` per suggestion, each with what/why/direction/effort/status.
- [ ] `docs/wiki/README.md` index has the `agentic-readiness/overview.md` entry.
- [ ] Every `Proposed` suggestion triaged: now → work started (via `/plan` when non-trivial), later → TODO file + backlog line, skip → status + reason recorded.
- [ ] `.alice/mem/agentic-readiness.json` refreshed with date, product type, scores, and suggestion counts.
- [ ] Final report emitted with scores, deltas (re-runs), and output paths.

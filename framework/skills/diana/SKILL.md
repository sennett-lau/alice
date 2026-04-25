---
name: diana
preamble-tier: 4
version: 1.2.0
description: |
  Run the alice SOP end-to-end for a given feature description with little
  or no human interaction. Two modes (`fully-auto` default, `murmur` for
  one-shot upfront clarification) and four effort tiers
  (`low`/`medium`/`high`/`max`) control which skills diana chains and how
  much adversarial oversight she applies. Chains `/plan`, `/plan-eng-review`,
  implementation, `/review`, `/pr-slicer` (max, conditional), `/security-audit`
  (high/max), and the binding post-feature retro + doc-update rules. All
  autonomous decisions and sub-agent handbacks are logged under
  `.tmp/diana/<run-slug>/` for audit. Use when asked to "diana", "run the
  full sop", "auto-implement this feature", "ship this end-to-end".
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
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
mkdir -p "$ROOT/.tmp/diana"
echo "BRANCH: $(git branch --show-current)"
echo "RUN_TS: $RUN_TS"
```

Run state lives at `<repo>/.tmp/diana/<run-slug>/`. Gitignored. Every autonomous decision is logged so the user can audit what diana did without re-running the session. The same state dir powers `--resume` after an interrupted run (network drop, token limit, crash) — see the "Resume" and "State & audit trail" sections.

**Load the orchestration rule.** Every sub-agent dispatch in this skill must follow `.alice/rules/sub-agent-orchestration.md` — progress polling (≥1/min) and permission escalation. Diana fans out several sub-agents per run; without polling, a single stuck agent stalls the whole pipeline silently.

---

## Arguments (`$ARGUMENTS`)

```
/diana [<feature-description>] [--mode=fully-auto|murmur] [--effort=low|medium|high|max]
       [--todo <slug> | --from-file <path>]
       [--resume <slug|latest>] [--resume-from <step-name>]
       [--list-runs]
```

| Param | Values | Default | Notes |
|-------|--------|---------|-------|
| description | free-form string | — | One-line or multi-paragraph description of the feature. Required unless `--todo`, `--from-file`, or `--resume` provides context. |
| `--mode` | `fully-auto`, `murmur` | `fully-auto` | See modes below. Ignored on resume — mode is re-read from the run's `run.conf`. |
| `--effort` | `low`, `medium`, `high`, `max` | `medium` | See effort tiers below. Ignored on resume — effort is re-read from the run's `run.conf`. |
| `--todo` | TODO slug | — | Reads description from `docs/todos/<slug>.md`. Skill will update that same file's status as it progresses. |
| `--from-file` | path to a markdown file | — | Reads description from file. Useful when the user has a draft spec sitting elsewhere. |
| `--resume` | run slug (or `latest`) | — | Resume an interrupted run. See "Resume" section. Must also supply `--resume-from` if the last step's status is ambiguous. |
| `--resume-from` | step name (`plan`, `plan-eng-review`, `implement`, `code-review`, `pr-slicer`, `security-audit`, `retro`, `doc-update`) | — | Force-pick the step to restart from regardless of state markers. Useful when an interrupted step left partial state diana can't safely auto-detect. |
| `--list-runs` | — | — | Print all runs under `.tmp/diana/` with their status and last-updated timestamp, then stop. Use this before `--resume` to pick the right slug. |

Convenience short-flags the skill also accepts:
- `--murmur` → `--mode=murmur`
- `--low` / `--high` / `--max` → `--effort=<level>`

### Examples

```
/diana "add dark mode toggle to the app shell"                      # fully-auto, medium
/diana --murmur "migrate from redis to valkey"                      # one-shot clarification, then silent medium
/diana --high "redesign auth flow to use OIDC"                      # fully-auto, high — adds security audit + outside plan voice
/diana --max --todo pricing-revamp                                  # max — adds pr-slicer if diff grows big
/diana --from-file drafts/feature-billing.md                        # read description from a file

# Resume a run that got killed mid-implement (network drop, token limit, crash)
/diana --list-runs                                                  # see what runs exist
/diana --resume diana-add-dark-mode-20260424T120000Z                # pick up where it left off
/diana --resume latest                                              # pick up the most recent run
/diana --resume latest --resume-from code-review                    # force-restart from a specific step
```

If `$ARGUMENTS` is empty AND no runs exist under `.tmp/diana/`: print the usage block and stop.

---

## Modes

The two modes differ ONLY in when diana is allowed to call `AskUserQuestion`:

| Phase | fully-auto | murmur |
|-------|-----------|--------|
| Intake (before SOP starts) — task comprehension only | ✓ allowed if the task is unintelligible | ✓ allowed |
| Murmur clarification batch (Step 0) | — skipped | ✓ up to 5 batched questions |
| Mid-SOP (Steps 1–8) | ✗ **never** — no exceptions, no checkpoints | ✓ allowed when blocked |
| End-of-run summary | ✓ informational only (no question) | ✓ informational only (no question) |

### `fully-auto` (default)

**Once the SOP starts, diana never stops to ask.** No mid-pipeline questions. No checkpoints. No "are you sure?" gates. Every interactive prompt from chained skills, every escalation trigger, every irreversible-category decision — diana decides autonomously using the Decision Policy and conservative defaults, then continues.

The only `AskUserQuestion` allowed in fully-auto is the **intake gate** before Step 0/1: if after reading the feature description, adopter CLAUDE.md, ledger, and prior art, diana cannot identify a coherent task to plan, she may surface ONE batched intake question ("what does 'rework billing' mean — A) pricing-only, B) pricing + invoicing, C) full overhaul"). If diana can plan something coherent, intake is skipped and the SOP runs silent end-to-end. Intake is a comprehension check, not a clarification batch — it is allowed only when the task itself is unintelligible, not when minor details are ambiguous.

When fully-auto hits a blocker that would normally escalate (retry limit, security critical, scope change, irreversible decision, build env broken), diana does NOT call `AskUserQuestion`. Instead she:
1. Marks the step `.failed` per the bookkeeping contract.
2. Writes the full context + the decision options diana would have asked to `$RUN_DIR/BLOCKED.md`.
3. Prints a terse handback to the user: `[diana] Blocked at step <N>: <one-line reason>. See $RUN_DIR/BLOCKED.md. Resume with /diana --resume <slug>.`
4. Exits the run cleanly. State is preserved; the user resumes manually (possibly switching to murmur mode by starting a new run with the same description if they want interactive recovery).

Every autonomous decision lands in `decisions.md` so the post-hoc audit shows what diana did and why.

### `murmur`

**Allowed to ask at intake AND mid-pipeline.** Before Step 1, diana surfaces up to 5 genuinely ambiguous points via a single batched `AskUserQuestion` — same Step 0 contract as before. The user answers once, diana locks in the answers and proceeds.

Unlike the previous "ask once and never again" murmur contract, **mid-pipeline questions are now allowed in murmur**. When diana hits a decision worth checking with the user (escalation triggers, irreversibles, scope ambiguity uncovered during implementation), she may pause and `AskUserQuestion` interactively rather than picking a default. Murmur is the mode for users who want diana to drive but check in when she's unsure.

To prevent murmur turning into a constant-prompt run, diana still applies the Decision Policy: only escalate when a default would be genuinely risky (irreversible, security-critical, out-of-spec scope change, retry-limit exhaustion). Mechanical / mechanical-style choices are still autonomous. Every prompt is logged in `decisions.md` with diana's recommendation alongside the user's answer.

---

## Effort tiers

| Step | low | medium (default) | high | max |
|------|-----|------------------|------|-----|
| 0. Intake (both modes if needed) + Clarification (murmur only) | ✓ | ✓ | ✓ | ✓ |
| 1. Plan (`/plan`) | ✓ | ✓ | ✓ | ✓ |
| 2. Plan-eng-review (`/plan-eng-review`) | — | ✓ no outside voice | ✓ + outside voice | ✓ + outside voice |
| 3. Implement | ✓ | ✓ | ✓ | ✓ |
| 4. Code review (`/review`) | — | ✓ | ✓ | ✓ |
| 5. PR-slicer (`/pr-slicer`) conditional | — | — | — | ✓ if diff is large |
| 6. Security audit (`/security-audit`) | — | — | ✓ | ✓ |
| 7. Retro (`post-feature-retro.md`) | ✓ | ✓ | ✓ | ✓ |
| 8. Doc update (`documentation-updates.md`) | ✓ | ✓ | ✓ | ✓ |

**Retro + doc update always run.** Both are covered by alice's binding rules (`.alice/rules/post-feature-retro.md`, `.alice/rules/documentation-updates.md`); skipping them ships an incomplete feature by alice's own definition. The effort tiers vary *adversarial* thoroughness, not SOP completeness.

### "Outside voice" (high + max)

`/plan-eng-review` by default runs alice's engineering-manager review prompt against the drafted spec. "Outside voice" adds one more independent pass to catch things alice's default reviewer is blind to. Pick, in order of availability:

1. `codex:rescue` via the `Agent` tool (if the Codex plugin is installed) — adversarial challenge mode against the spec.
2. A fresh `general-purpose` agent with the adversarial plan prompt. The prompt: "Read `<spec path>`. You don't know this project's history. Attack the plan: missing edge cases, hidden coupling, unjustified assumptions, missing acceptance criteria. Don't compliment. Just problems."

Outside voice is dispatched in parallel with alice's own `/plan-eng-review` (same assistant message, two `Agent` blocks). Diana merges the findings before deciding whether the spec is locked.

### PR-slicer condition (max only)

After implementation, diana runs:

```bash
git diff origin/$BASE --stat | tail -1
```

Engage `/pr-slicer` if any of:
- Total inserted + deleted lines > 300
- More than 5 files changed
- Any file in `MIGRATION_FILES` (adopter-declared in `CLAUDE.md` "Migration-class files" section) is touched

Otherwise skip slicing — the PR is small enough to review whole. Record the decision in `decisions.md`.

---

## Resume

Runs can die mid-pipeline — network drop, token-limit truncation, crash, user interrupt. The `.tmp/diana/<run-slug>/` dir is the resume source of truth.

### `--list-runs`

```bash
for d in "$ROOT/.tmp/diana"/*/; do
  [ -d "$d" ] || continue
  slug=$(basename "$d")
  conf="$d/run.conf"
  if [ -f "$conf" ]; then
    mode=$(grep '^mode=' "$conf" | cut -d= -f2)
    effort=$(grep '^effort=' "$conf" | cut -d= -f2)
  fi
  # Find last step marker (sorted by numeric prefix)
  last=$(ls -1 "$d/steps"/*.in-progress "$d/steps"/*.done "$d/steps"/*.failed "$d/steps"/*.skipped 2>/dev/null | sort | tail -1)
  last_name=$(basename "${last:-none}")
  updated=$(stat -f "%Sm" -t "%Y-%m-%dT%H:%M:%SZ" "${last:-$conf}" 2>/dev/null || echo "-")
  printf "%-55s  mode=%s effort=%s  last=%s  updated=%s\n" "$slug" "${mode:--}" "${effort:--}" "$last_name" "$updated"
done
```

Sort output by updated-desc. Stop after printing; no other action.

### `--resume <slug|latest>`

1. Resolve slug. If `latest`, pick the most recently updated run dir.
2. Abort if the run dir is missing or `run.conf` is missing.
3. Load `run.conf` — re-populate `MODE`, `EFFORT`, `FEATURE`, `BRANCH_AT_START`, `BASE_BRANCH`, `TODO_SLUG`, `PLAN_FOLDER`, `RUN_SLUG`, `RUN_DIR`. These override any `--mode` / `--effort` / description passed on the command line (a warning is printed if they conflict — the file wins).
4. Verify adopter's current branch matches `BRANCH_AT_START`. If not, `AskUserQuestion`: `A) Check out $BRANCH_AT_START and continue  B) Continue on current branch (risky — commits may land wrong)  C) Abort resume`. Default: A.
5. Determine the starting step:
   - If `--resume-from <step>` is supplied, start there — ignore any markers beyond it and remove them (treat intermediate as stale).
   - Else, walk the ordered step list and find the first step that is NOT `.done` and NOT `.skipped`.
     - If that step is `.failed` → `AskUserQuestion`: `A) Retry this step  B) Skip (mark .skipped and move on — risky)  C) Abort`.
     - If that step is `.in-progress` → treat as interrupted. `AskUserQuestion`: `A) Restart this step from scratch (drop any in-step partial state)  B) Attempt to continue from where it left off (see per-step continue semantics below)  C) Abort`.
     - If that step has no marker at all → start it fresh.
6. Proceed with the pipeline from the chosen step. Skip all earlier steps; their `.done` markers remain untouched.

### Per-step continue semantics (when user picks "attempt to continue")

| Step | Continue logic |
|------|----------------|
| `murmur` | Re-load answers from `decisions.md#intake` and `decisions.md#murmur-clarifications`. If either is missing for a sub-gate that ran originally, restart that sub-gate. |
| `plan` | If `$PLAN_FOLDER/spec.md` exists and has `Status: draft` or `Status: locked`, treat as done-and-just-missing-marker → upgrade marker to `.done` and move on. If `spec.md` is partial (no Acceptance Criteria section), restart step 1. |
| `plan-eng-review` | If `spec.md` has `Status: locked` and `transcripts/02-plan-review.md` exists, treat as done. Else restart. |
| `implement` | Read `$PLAN_FOLDER/implementation.md` — its checklist shows which verify-map steps are done. Continue from the first unchecked item. Also cross-check `git log origin/$BASE..HEAD` for commits that match verify-map progress. If the two disagree (uncommitted changes from a prior partial step), show the user `git status` and ask whether to amend/revert before continuing. |
| `code-review` | Always restart — `/review` is idempotent and a partial run leaves no reliable checkpoint. |
| `pr-slicer` | If `$RUN_DIR/transcripts/05-pr-slicer.md` lists existing slice branches, check them with `git branch --list 'slice/*'` and skip slices already built + pushed. For in-flight slices, restart that slice. |
| `security-audit` | Always restart — audit is idempotent and cheap to re-run on the final diff. |
| `retro` | Step has 5 sub-actions (archive / wiki / experiences / decisions / todo). If `$RUN_DIR/transcripts/07-retro.md` exists, diff its "completed actions" list against the 5; continue with the unfinished ones. |
| `doc-update` | Always restart — idempotent. |

If any step's continue logic detects a state diana cannot safely reconcile (e.g. uncommitted changes that don't match `implementation.md`), fall back to `AskUserQuestion` with the three A/B/C restart/continue/abort options.

### `--resume-from <step>`

Forces the starting step. Diana does NOT validate whether earlier steps are complete — assumes the user knows what they're doing. All markers for the specified step and later are deleted before the step runs. Earlier markers stay.

Useful when:
- An interrupted step left partial state diana can't auto-resolve
- The user manually finished a step outside diana and wants to skip past it
- A completed step needs to be re-done (e.g. spec changed, want to re-review)

---

## The pipeline

All steps run under a common run slug. For a new run:

```bash
RUN_SLUG="diana-$(echo "$FEATURE" | head -c 40 | tr -cs 'a-zA-Z0-9' '-' | tr '[:upper:]' '[:lower:]' | sed 's/^-//;s/-$//')-$RUN_TS"
RUN_DIR="$ROOT/.tmp/diana/$RUN_SLUG"
mkdir -p "$RUN_DIR/transcripts" "$RUN_DIR/steps"
```

For a resumed run, `RUN_SLUG` and `RUN_DIR` come from the resume resolution above.

### Step bookkeeping contract

Each step writes a marker file under `$RUN_DIR/steps/` that is the single source of truth for its status. Four states: `.in-progress`, `.done`, `.failed`, `.skipped`. Exactly one marker per step at any time.

On entering a step:

```bash
STEP_PREFIX="NN-<step-name>"   # e.g. "01-plan", "03-implement"
# Clear any previous marker for this step (resume may have left a .failed / .in-progress behind)
rm -f "$RUN_DIR/steps/$STEP_PREFIX".{in-progress,done,failed,skipped}
# Write in-progress with the entry timestamp
date -u +%Y-%m-%dT%H:%M:%SZ > "$RUN_DIR/steps/$STEP_PREFIX.in-progress"
```

On exiting a step, atomically replace the marker:

```bash
# Success
mv "$RUN_DIR/steps/$STEP_PREFIX.in-progress" "$RUN_DIR/steps/$STEP_PREFIX.done"

# Failure — append reason
{ cat "$RUN_DIR/steps/$STEP_PREFIX.in-progress"; echo "FAILED: <reason>"; } > "$RUN_DIR/steps/$STEP_PREFIX.failed"
rm "$RUN_DIR/steps/$STEP_PREFIX.in-progress"

# Skipped (e.g. effort tier excludes this step)
echo "SKIPPED: <reason>" > "$RUN_DIR/steps/$STEP_PREFIX.skipped"
rm -f "$RUN_DIR/steps/$STEP_PREFIX.in-progress"
```

The step ordering (used by resume to find the first non-done step):

```
00-murmur
01-plan
02-plan-eng-review
03-implement
04-code-review
05-pr-slicer
06-security-audit
07-retro
08-doc-update
```

### Branch setup (new runs only — runs before run.conf is written)

Diana cuts a fresh feature branch from the repo's default branch at the start of every new run. Resume skips this step (branch already resolved in `run.conf`).

1. Detect default branch + current branch:
   ```bash
   DEFAULT_BRANCH=$(git symbolic-ref --quiet refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
   DEFAULT_BRANCH="${DEFAULT_BRANCH:-$(git config --get init.defaultBranch || echo main)}"
   CURRENT_BRANCH=$(git branch --show-current)
   FEAT_SLUG=$(echo "$FEATURE" | head -c 40 | tr -cs 'a-zA-Z0-9' '-' | tr '[:upper:]' '[:lower:]' | sed 's/^-//;s/-$//')
   FEAT_BRANCH="feat/$FEAT_SLUG"
   ```
   If the adopter's CLAUDE.md documents a branch-naming convention, use it instead of `feat/<slug>`.

2. If `CURRENT_BRANCH == DEFAULT_BRANCH`: pull latest, then create the feature branch — no user prompt.
   ```bash
   git pull --ff-only origin "$DEFAULT_BRANCH"
   git checkout -b "$FEAT_BRANCH"
   ```

3. If `CURRENT_BRANCH != DEFAULT_BRANCH`: behavior depends on mode.
   - **`murmur`:** pause and `AskUserQuestion` with these options:
     - `A) Branch from $DEFAULT_BRANCH (checkout default, pull, then cut $FEAT_BRANCH — recommended for clean base)`
     - `B) Branch from current $CURRENT_BRANCH (cut $FEAT_BRANCH off current HEAD — use when stacking on prior work)`
     - `C) Stay on $CURRENT_BRANCH — no new branch (use when the user already prepared the branch)`
     - `D) Abort`

     Default recommendation: A. Before checking out default in option A, abort if `git status --porcelain` is non-empty (uncommitted changes would block checkout) — surface the dirty state and ask the user to stash/commit first.

   - **`fully-auto`:** auto-pick option A (branch from default). No prompt. If `git status --porcelain` is non-empty, write BLOCKED.md (uncommitted changes — can't safely checkout default) and abort the run per the fully-auto blocker protocol. The user resumes after stashing/committing.

4. Record the resolved branch in the run config (next section): `branch_at_start=<feature branch>`, `base_branch=$DEFAULT_BRANCH`.

Log the choice + reason to `decisions.md` under `## Branch setup`.

### Run config (written once at run start)

```bash
cat > "$RUN_DIR/run.conf" <<EOF
run_slug=$RUN_SLUG
run_ts=$RUN_TS
mode=$MODE
effort=$EFFORT
feature=$(echo "$FEATURE" | tr '\n' ' ' | head -c 500)
branch_at_start=$(git branch --show-current)
base_branch=$DEFAULT_BRANCH
todo_slug=$TODO_SLUG  # empty unless --todo was used
plan_folder=  # filled in after Step 1
EOF
```

After Step 1 creates the plan folder, update `plan_folder=` in-place. Config is immutable otherwise — resume reads it verbatim.

Also write `$RUN_DIR/manifest.md` at the start (human-readable summary pointing at `run.conf` + step markers). Append to `$RUN_DIR/decisions.md` as autonomous decisions get made.

### Step 0 — Intake & clarification gate

This step has two sub-gates that run before Step 1.

**0a. Intake gate (both modes).** Before any clarification batch, diana checks whether the task is intelligible: can she identify a coherent feature to plan from `$FEATURE` + adopter CLAUDE.md + ledger/wiki prior art? If yes, skip to 0b. If no — the description is too vague to map onto a plan (e.g. "fix the thing", "redo the system") — surface ONE batched `AskUserQuestion` with comprehension options (e.g. "by 'rework billing' do you mean A) pricing-only, B) pricing + invoicing, C) full overhaul, D) none of these — abort"). Allowed in BOTH `fully-auto` and `murmur`. Intake is the only `AskUserQuestion` permitted in fully-auto across the entire run. Record answers in `$RUN_DIR/decisions.md` under `## Intake`.

**0b. Clarification batch (murmur only).** `fully-auto` skips to Step 1.

Murmur mode:
1. Read the feature description.
2. Identify up to 5 genuinely ambiguous points. "Ambiguous" = diana cannot pick a default she's confident is what the user wants. Noise-level questions (naming, minor style) do NOT qualify — diana decides those herself. Scope questions, behavior-under-failure questions, and data-shape questions DO qualify.
3. Surface all of them in ONE `AskUserQuestion` call. Each question gets options: `A) <option 1>`, `B) <option 2>`, `C) <option 3 if applicable>`, `D) Let diana decide (she'll default to <preview>)`.
4. Record answers in `$RUN_DIR/decisions.md` under `## Murmur clarifications`.
5. Proceed to Step 1 with answers baked into the working description.

If diana identifies zero ambiguous points, skip the prompt and note "Murmur: no ambiguity found — proceeding fully-auto" in decisions.md.

### Step 1 — Plan

Invoke the `/plan` skill via its command file (`.alice/commands/plan.md`). `/plan` expects interactive answers for ~11 fields (name, problem, goal, scope in/out, acceptance, design, assumptions, verify map, risks, open questions).

**Diana does not "ask" /plan's prompts of the user.** She plays both roles: reads /plan's flow, fills each field from the feature description + Murmur answers (if any) + grep of `docs/wiki/`, `docs/ledger/`, `docs/plans/archive/` for prior art.

Field-fill policy:
- **Feature name:** slug from description's first noun phrase.
- **Problem:** one-paragraph restatement from description.
- **Goal:** observable "done" state — inferred from description's verbs + acceptance.
- **Scope in:** bullets extracted from description; add obvious components implied by the verbs (e.g. "add dark mode" → toggle UI + preference storage + theme tokens).
- **Scope out:** list likely-adjacent-but-not-in-scope items explicitly ("this does NOT include system-preference auto-detection unless the description says so").
- **Acceptance criteria:** `given/when/then` bullets derived from the goal.
- **Design sketch:** diana greps ledger + archive + wiki for related primitives first; writes the sketch referring to existing primitives where possible. Novel primitives only when no reuse candidate exists.
- **Assumptions:** everything diana is taking as true. Be generous here — implicit assumptions are where plans break.
- **Verify map:** per implementation step, a concrete verify (failing test / build pass / screenshot / assertion).
- **Risks:** diana-flagged risks based on the diff size and stack-specific gotchas from adopter's CLAUDE.md.
- **Open questions:** in fully-auto, empty (diana will make these calls during implementation). In murmur, already resolved in Step 0.

Output of Step 1: a complete `docs/plans/active/YYYY-MM-DD_<slug>/spec.md` with `Status: draft`. Diana does NOT mark it `locked` yet — that happens after Step 2 (medium+) or immediately (low).

**Low effort only:** mark the spec `locked` at end of Step 1 and skip Step 2.

Log: full plan content into `$RUN_DIR/transcripts/01-plan.md`.

### Step 2 — Plan-eng-review (medium / high / max)

Dispatch `/plan-eng-review` via the `Agent` tool (`subagent_type: general-purpose` with /plan-eng-review loaded, or directly if a dedicated agent exists).

**High / max:** in the SAME assistant message, also dispatch the outside voice (codex:rescue or general-purpose adversarial). Both run in parallel; diana polls per the orchestration rule and waits for both handbacks.

Merge findings from both reviewers:
- **Concurring findings (both flagged it):** high confidence. Fix before locking the spec.
- **Unique to alice reviewer:** apply if categorized P0/P1 by the reviewer.
- **Unique to outside voice:** apply if categorized P0/P1; log in decisions.md which were accepted vs. deferred.
- **Adjacent observations:** route to `$RUN_DIR/followups.md`.

Fix loop: diana edits the spec to address accepted findings, then re-dispatches the reviewer(s) once. If still P0/P1 after one round, escalate per Failure Handling. If clean, mark spec `locked`.

Log to `$RUN_DIR/transcripts/02-plan-review.md`.

### Step 3 — Implementation

Follow the locked spec. Diana's main session does the code work directly — no dedicated implementer sub-agent. The spec's verify map drives the order; after each step, run the verify check before moving to the next.

During implementation, honor the adopter's critical gotchas (from CLAUDE.md), the binding `implementation-quality.md` rule, and the spec's files-in-scope list. No drive-by refactors, no out-of-scope changes. If diana finds an out-of-scope bug, log it to `$RUN_DIR/followups.md` instead of fixing inline.

If a verify check fails: diana fixes, re-runs, up to 3 attempts per step. After 3 failures, escalate per Failure Handling.

Commit in small chunks matching the repo's commit-message style (check `git log --oneline -20` on the base branch). **Update `implementation.md` in the plan folder as each verify-map step completes** — check the box, note the commit SHA, list the verify result. This file is the checkpoint resume uses when this step gets interrupted; an out-of-date implementation.md means resume can't tell what's actually done. Update it BEFORE committing the code for the step, or as part of the same commit — never after.

**Do NOT** push, create PRs, or deploy at this stage. Remote side-effects happen after all pipeline steps pass and only if the user previously authorized (check adopter's CLAUDE.md for autonomous-push permission; default is NO — diana commits locally, leaves the branch staged, and surfaces in Step 8).

Log key decisions (choice of primitive, refactor deferred, surprise found) to `$RUN_DIR/decisions.md`.

### Step 4 — Code review (medium / high / max)

Dispatch `/review` via the `Agent` tool (fresh sub-agent, `subagent_type: code-reviewer` if available, else `general-purpose` with `/review` loaded).

`/review` already includes Fix-First: mechanical issues auto-fixed, ASK items batched. In diana's context, ALL ASK items resolve to conservative defaults:
- P0 / P1 → always fix (recommendation: A)
- P2 / P3 → always defer to followups.md (recommendation: B)
- Greptile false positives → reply with the False Positive template
- Greptile already-fixed → reply with the Already Fixed template

After `/review` finishes, diana re-dispatches it once to verify the fixes stuck. If clean, continue. If still P0/P1 after one round, escalate.

Log to `$RUN_DIR/transcripts/04-code-review.md`.

### Step 5 — PR slicing (max only, conditional)

Check the size + migration-class gate (see "PR-slicer condition" above). If the diff qualifies, invoke `/pr-slicer` with `--mode=sequential` by default (parallel only if the adopter's CLAUDE.md explicitly opts into it).

Diana acts as the "user" for `/pr-slicer`'s interactive prompts:
- Step 5 confirmation (`A) Dispatch executors`) → always A
- Step 6c.5 review findings (`A/B/C`) → always A (fix high-priority, defer rest)

`/pr-slicer` fully replaces the single-PR flow. If slicer engages, skip to Step 6 (security audit runs against the cumulative diff of all slices combined).

If slicer does not engage, continue with the single-PR flow.

Log to `$RUN_DIR/transcripts/05-pr-slicer.md`.

### Step 6 — Security audit (high / max)

Dispatch `/security-audit` against the cumulative diff (or the full slice chain if pr-slicer ran). Diana runs the skill's "daily" mode by default; "comprehensive" mode only if the adopter's CLAUDE.md opts in or the diff touches auth / secrets / external boundaries.

Triage:
- **Critical (confidence ≥7):** always fix. If unfixable without scope change → escalate.
- **High:** fix. If multiple, batch a single fix pass.
- **Medium:** fix if mechanical, else defer to followups.md.
- **Low / informational:** log, do not fix.

After fixes, re-dispatch `/security-audit` once to verify. If still critical → escalate.

Log to `$RUN_DIR/transcripts/06-security-audit.md`.

### Step 7 — Retro (all tiers, binding)

Run every action in `.alice/rules/post-feature-retro.md`. In order:

1. **Archive the plan folder:** `git mv docs/plans/active/<folder> docs/plans/archive/<folder>`.
2. **Update `docs/wiki/current-status.md`:** in-flight → shipped.
3. **Append to `docs/ledger/experiences.md`:** diana writes a short retro entry — what surprised her, which assumption turned out wrong, what the next-adopter of this pattern should know.
4. **Append to `docs/ledger/decisions.md`:** for any non-obvious choice diana made during the run (from `$RUN_DIR/decisions.md`, dedup & summarize).
5. **TODO cleanup (conditional on `$TODO_SLUG`).**
   - **If the run was `--todo <slug>` (i.e. `$TODO_SLUG` is non-empty in `run.conf`):**
     a. Read `docs/todos/<slug>.md`. If it has a `Status:` field, set it to `Status: done` and append a `Completed:` line with today's date + the diana run slug + the archived plan folder path.
     b. Move the entry in `docs/todos/overview.md` from its in-flight section to the "Done recent" section. Preserve the original wording; append `→ <archived plan folder>` and the run slug for traceability.
     c. Delete `docs/todos/<slug>.md`. The detail file's role ends when the work ships — overview + archived plan folder are the durable record. Use `git rm docs/todos/<slug>.md`.
     d. If the TODO referenced linked items (other todos, issue trackers, wiki pages), re-check those links and update or remove stale references. Log any orphans diana couldn't auto-resolve to `$RUN_DIR/followups.md`.
   - **If the run was a free-form description (no `$TODO_SLUG`):**
     a. Strike any matching in-flight entry in `docs/todos/overview.md` if one was implicitly tracking this work (best-effort grep against the feature name/slug). Skip if no match.
     b. No detail file to delete.

Delegate the wiki step to the `wiki-maintainer` sub-agent (alice ships it) per `post-feature-retro.md`'s recommendation — keeps the drift audit focused.

Log to `$RUN_DIR/transcripts/07-retro.md`. Include which TODO branch ran (todo-cleanup vs free-form-cleanup) so resume's "completed actions" diff is unambiguous.

### Step 8 — Doc update (all tiers, binding)

Per `.alice/rules/documentation-updates.md`: any wiki page that drifted because of this change must land in the same PR. Diana cross-references the diff against `docs/wiki/**` and updates affected pages.

Skips: if no wiki page drift detected, log "no drift" and proceed.

Final output: diana prints a structured summary to the user:

```
diana run complete — $RUN_SLUG

Mode: <fully-auto | murmur>
Effort: <low | medium | high | max>
Feature: <one-line>

Steps:
  [done]  1. Plan — spec/<path> — locked after Step 2
  [done]  2. Plan-eng-review — N findings (M outside-voice), all addressed
  [done]  3. Implement — K files changed, J commits, <lines> inserted
  [done]  4. Code review — clean (or N deferred to followups)
  [skipped] 5. PR-slicer — diff under threshold
  [done]  6. Security audit — clean
  [done]  7. Retro — archived + wiki + ledger + todo
  [done]  8. Doc update — N wiki pages updated

Autonomous decisions: see $RUN_DIR/decisions.md
Followups:           see $RUN_DIR/followups.md (if any)
Branch: <name> — changes staged locally. Push / PR is up to you.
```

Never auto-push. Never auto-create PR. Diana stops at "changes staged locally" and hands back.

---

## Decision policy

Ordered principles diana applies when the chained skills prompt for input. Applies to BOTH modes — in fully-auto these resolve every decision silently; in murmur diana may interactively confirm an irreversible-class decision instead of auto-applying.

### 1. Prefer more verification over less

When `/review` or `/security-audit` asks "fix this or skip?" → always fix if P0/P1. When `/plan` asks "add this to acceptance criteria?" → yes. When the verify map is thin → add more checks, not fewer. Shipping a broken feature fast is worse than shipping a sound feature slower.

### 2. Prefer conservative scope interpretation

When the description is ambiguous between a narrow read and a broad read → take the narrow read. Log the narrow interpretation in decisions.md. If the user wanted broader scope, a follow-up run adds it; if diana assumes broad and the user wanted narrow, work is wasted.

### 3. Prefer reuse over novelty

When the design sketch could use an existing primitive OR introduce a new one → use the existing primitive. Grep the ledger + archive + wiki first. New primitives require explicit justification in decisions.md.

### 4. Prefer existing style over "improvement"

Diana does not refactor adjacent code, reformat, or rename "while I'm in there." She matches the repo's existing patterns even if she'd personally write it differently. The user did not ask for a cleanup.

### 5. Prefer transparency over silent choice

Every non-obvious decision → logged in decisions.md with a one-line rationale. If the user later asks "why did you pick X?", decisions.md is the answer.

### 6. On irreversibles — defer in fully-auto, escalate in murmur

Database schema choices, external API contracts, destructive operations, authentication changes — these are the high-risk categories.

- **fully-auto:** diana does NOT call `AskUserQuestion`. She picks the safest non-irreversible alternative: defer the irreversible action to `$RUN_DIR/followups.md`, scope the current PR to the reversible part, log the deferral in `decisions.md`. If the irreversible action is the entire task and cannot be deferred, abort the run via the BLOCKED.md handback (see fully-auto blocker protocol in Modes).
- **murmur:** diana may pause and `AskUserQuestion` with the decision options. Escalation = write to `$RUN_DIR/BLOCKED.md` for traceability, then ask. Pick the user's choice and continue.

---

## Failure handling

Retry limits per step:
- Build / test failure during implementation (Step 3): 3 attempts per step, fix + re-run each time.
- Review findings after fix (Step 4): 1 re-review pass. If still P0/P1, escalate.
- Security-audit findings after fix (Step 6): 1 re-audit pass. If still critical, escalate.
- Sub-agent `BLOCKED:` handback (permission issue): handle per orchestration rule (grant/proxy/re-scope/abort). Never silent-retry.

Escalation triggers (mode-dependent behavior):
1. Retry limit hit on a step.
2. Security critical unfixable without scope change.
3. Implementation requires a scope change not covered by the locked spec.
4. Sub-agent blocked on a permission the main session also can't resolve.
5. Build environment broken (missing dep, infra issue) — diana will attempt a dep install once if the stack's install command is in CLAUDE.md, else escalate.
6. Any irreversible-category decision (per Decision Policy principle 6).

### `fully-auto` escalation — abort, never ask

In fully-auto, escalation NEVER calls `AskUserQuestion`. The mode's contract is "no questions mid-SOP." When any trigger fires:

1. Mark the current step `.failed` per the bookkeeping contract.
2. Write a full handback to `$RUN_DIR/BLOCKED.md`:
   - Trigger that fired (which of the 6) and which step.
   - Full context + commits made so far + decisions made so far.
   - The decision options diana would have asked, with diana's recommendation if she had to pick one anyway.
   - Resume command: `/diana --resume <slug>` (or `--resume <slug> --resume-from <step>` if a different start step is recommended).
3. Print a one-line handback to the user: `[diana] Blocked at step <N> (<trigger>): <reason>. Details in $RUN_DIR/BLOCKED.md. Resume with /diana --resume <slug>.`
4. Exit cleanly. State preserved: plan folder stays `locked`, branch stays on current commit, `run.conf` + step markers intact. The user reviews BLOCKED.md, makes the call manually, then resumes — or starts a new run in murmur mode if they want interactive recovery.

### `murmur` escalation — pause and ask

In murmur, escalation pauses and prompts:

- Write the current step's marker as `.failed` with reason.
- Write a full escalation note to `$RUN_DIR/BLOCKED.md`.
- Surface via `AskUserQuestion` with options: `A) <path-to-unblock>`, `B) <alternative>`, `C) Abort run (state preserved — resume with /diana --resume <slug>)`.
- Wait for user input. On abort, behavior matches fully-auto's exit: state preserved, resumable.

---

## State & audit trail

`.tmp/diana/<run-slug>/` layout after a complete run:

```
run.conf                 — immutable run config (mode/effort/feature/branches/plan_folder)
manifest.md              — human-readable run summary
decisions.md             — every autonomous decision + rationale
followups.md             — deferred review findings (feeds into post-run follow-up)
BLOCKED.md               — only if diana escalated mid-run
steps/                   — step markers — source of truth for resume
  00-murmur.{skipped|done}
  01-plan.done
  02-plan-eng-review.{done|skipped}
  03-implement.done
  04-code-review.{done|skipped}
  05-pr-slicer.{done|skipped}
  06-security-audit.{done|skipped}
  07-retro.done
  08-doc-update.done
transcripts/             — narrative output per step
  00-murmur.md           — murmur clarification Q&A (if ran)
  01-plan.md             — full /plan output (spec draft)
  02-plan-review.md      — /plan-eng-review + outside voice handbacks
  03-implement.md        — commit summary + verify check results per step
  04-code-review.md      — /review findings + fix actions
  05-pr-slicer.md        — if engaged
  06-security-audit.md   — if ran
  07-retro.md            — post-feature-retro action results
  08-doc-update.md       — wiki pages updated
```

Everything under `.tmp/` is gitignored. The audit trail stays with the local checkout; deleting the checkout deletes the trail.

**During a run:**
- `steps/<step>.in-progress` is present while the step is executing — this is the "interrupted" signal resume watches for.
- `steps/<step>.failed` is written (replacing `.in-progress`) if the step hits the retry limit or an unfixable escalation.
- `run.conf` is written once at the start (except `plan_folder=` which is backfilled after Step 1). Resume reads it to re-populate `MODE`, `EFFORT`, `FEATURE`, and the branches.

**Manifest format** — `manifest.md` is a human-readable summary pointing at everything else. It has no load-bearing role in resume (the `steps/` markers do). Diana updates it after each step for the user's convenience:

```markdown
# diana run: <run-slug>

- Mode: fully-auto
- Effort: medium
- Feature: add dark mode toggle to app shell
- Branch: feat/dark-mode (base: main)
- Plan folder: docs/plans/active/2026-04-24_dark-mode
- Started: 2026-04-24T12:00:00Z

## Step status
- [x] 0. murmur (skipped — fully-auto)
- [x] 1. plan — done 12:02:00Z
- [x] 2. plan-eng-review — done 12:08:15Z (2 findings fixed, spec locked)
- [~] 3. implement — in-progress since 12:08:20Z
- [ ] 4. code-review
- [ ] 5. pr-slicer
- [ ] 6. security-audit
- [ ] 7. retro
- [ ] 8. doc-update
```

Diana also writes a terse running status to the user's session as each step completes:

```
[diana] Step 1 — plan: drafted spec at docs/plans/active/2026-04-24_dark-mode/ (spec.md, 3.2kB)
[diana] Step 2 — plan-eng-review: 2 findings, both P1, both fixed; spec locked
[diana] Step 3 — implement: 4 files touched, 2 commits, tests pass
...
```

One line per step — no prose, no filler. On resume, diana prints the resume banner first:

```
[diana] Resuming run diana-add-dark-mode-20260424T120000Z
[diana]   Mode: fully-auto  Effort: medium
[diana]   Last completed step: 02-plan-eng-review (at 12:08:15Z)
[diana]   Continuing from: 03-implement (was in-progress, user chose "continue from checkpoint")
```

---

## Hard rules

- **Diana is an orchestrator, not a freelancer.** She invokes alice's existing skills (`/plan`, `/plan-eng-review`, `/review`, `/pr-slicer`, `/security-audit`) and follows alice's binding rules. She does not bypass them even in `low` effort — she only skips *adversarial* steps, never the SOP steps themselves (retro + doc update always run).
- **Never push, create PRs, or deploy autonomously.** Diana stops at "staged locally." Remote side-effects are the user's to authorize per-run.
- **Every sub-agent dispatch follows `.alice/rules/sub-agent-orchestration.md`.** Poll ≥1/min; escalate silence; handle `BLOCKED:` per protocol; never silently retry a denied tool.
- **Every autonomous decision is logged.** If it didn't land in `decisions.md`, it didn't happen — future diana runs and the user's audits depend on the trail being complete.
- **Fully-auto never asks mid-SOP.** Once Step 1 begins, diana does NOT call `AskUserQuestion` for any reason — no checkpoints, no escalation prompts, no irreversible-decision gates, no scope-change confirmations. When a trigger fires that would normally prompt, diana writes BLOCKED.md, marks the step `.failed`, prints a one-line handback, and exits the run. The user reviews BLOCKED.md and resumes manually. This is the entire safety contract — no mid-SOP interaction in exchange for full state preservation + clean resume.
- **Fully-auto MAY ask once at intake** if the task itself is unintelligible (not for minor ambiguity — only when diana cannot identify a coherent task to plan). This is a comprehension gate, not a clarification batch. If diana can plan something coherent, intake is skipped.
- **Murmur asks at intake AND mid-SOP.** Step 0 batches up to 5 clarification questions before the SOP starts. During the SOP, murmur is permitted to pause and ask when escalation triggers fire or when an irreversible decision needs user input. Murmur still applies the Decision Policy — only ask when a default would be genuinely risky.
- **Branch hygiene.** Every new diana run cuts a fresh feature branch from the repo's default branch (detected via `git symbolic-ref refs/remotes/origin/HEAD`, falling back to `main`). If the user is already on the default branch, diana branches off automatically — no prompt. If the user is on a different branch: in `murmur` she asks (A=branch from default, B=stack on current, C=stay on current, D=abort); in `fully-auto` she auto-picks A (and aborts to BLOCKED.md if the working tree is dirty). See "Branch setup" section for the full flow. Resumed runs skip this — `branch_at_start` is loaded from `run.conf` and verified.
- **Step markers are sacred.** Never delete a `.done` marker except when explicitly restarting that step via `--resume-from`. Never write a `.done` marker without actually completing the step's work. Markers are the resume contract — faking them strands future diana runs.
- **Resume preserves autonomy.** A resumed run uses the same mode/effort as the original (loaded from `run.conf`). If the user wants different mode/effort, they start a new run, not resume.
- **Retro + doc update are binding.** Skipping them would violate `post-feature-retro.md` and `documentation-updates.md`. Every effort tier runs both.

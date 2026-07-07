---
name: hugh
preamble-tier: 4
version: 1.0.1
description: |
  Orchestrate multiple `diana` runs in parallel — one per feature — across
  isolated git worktrees so independent feature work ships concurrently
  without branch / dev-server / port collisions. Takes a multi-feature
  prompt (or explicit list / file), splits it, bundles per-feature context,
  spins up one background `diana` sub-agent per feature in its own worktree,
  monitors progress per `.alice/rules/sub-agent-orchestration.md`, relays
  cross-feature signals through a shared inbox, and drains every spawned
  child before printing the final summary. Use when asked to "hugh", "run
  multiple dianas", "ship these features in parallel", "fan out diana".
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
mkdir -p "$ROOT/.alice/mem/hugh"
echo "BRANCH: $(git branch --show-current)"
echo "RUN_TS: $RUN_TS"
```

# Hugh Parallel Diana Orchestrator

## Overview

Splits multiple independent feature requests and dispatches one `diana` run per feature across isolated worktrees, then monitors, relays, and drains the batch.

## When to Use

- Use when asked to run `hugh`, run multiple dianas, fan out diana, or ship several independent features in parallel.
- Use when features can be isolated by branch/worktree and benefit from concurrent execution.
- Use when the main value is orchestration, progress monitoring, cross-feature signaling, and final aggregation.

**When NOT to use:**

- Do not use for a single feature unless explicitly stress-testing the orchestration path.
- Do not use when features are tightly coupled and must be implemented in one ordered working tree.
- Do not use to replace `diana`'s implementation contract; hugh delegates feature work to diana.

## Process

Run state lives at `<repo>/.alice/mem/hugh/<run-slug>/`. Gitignored. Every autonomous decision (split, port allocation, relay) is logged so the user can audit what hugh did without re-running.

**Load the orchestration rule.** Every sub-agent dispatch in this skill must follow `.alice/rules/sub-agent-orchestration.md` — progress polling (≥1/min) and permission escalation. Hugh fans out N background diana sub-agents per run; without polling, a single stuck diana stalls the whole batch silently.

**Hugh does not duplicate diana.** Each per-feature diana still runs the full alice SOP end-to-end inside its worktree (plan → review → implement → review → security → retro → doc-update → drain). Hugh's job is split + context-bundle + dispatch + monitor + relay + drain. Hugh never writes feature code, never edits adopter source files. Implementation-side concerns (effort tier, mode, branch hygiene, retry limits, irreversibles) are diana's contract — hugh forwards user intent down and aggregates handbacks up.

---

### Arguments (`$ARGUMENTS`)

```
/hugh [<multi-feature-description>] [--mode=fully-auto|murmur] [--effort=low|medium|high|max]
      [--features <slug1,slug2,...> | --from-file <path>]
      [--max-parallel <N>] [--no-split]
      [--resume <slug|latest>] [--resume-feature <feature-slug>]
      [--list-runs]
```

| Param | Values | Default | Notes |
|-------|--------|---------|-------|
| description | free-form blob | — | Multi-feature blob hugh splits into N features. Required unless `--features`, `--from-file`, or `--resume` provides input. |
| `--mode` | `fully-auto`, `murmur` | `fully-auto` | Forwarded verbatim to each spawned diana. Same semantics as `/diana --mode=`. |
| `--effort` | `low`, `medium`, `high`, `max` | `medium` | Forwarded verbatim to each spawned diana. |
| `--features` | comma-separated list of feature briefs | — | Bypass auto-split. Each item is one full feature description; quote heavily. |
| `--from-file` | path to a markdown file | — | File contains an H2 (`## <slug>`) per feature, body is that feature's brief. Bypasses auto-split. |
| `--max-parallel` | positive integer | `3` | Cap on concurrent dianas. Beyond this, features queue and dispatch as siblings finish. |
| `--no-split` | — | — | Treat the description as a single feature (forwarded as one diana run). Equivalent to plain `/diana` but routed through hugh's bookkeeping — useful for stress-testing the orchestration. |
| `--resume` | run slug (or `latest`) | — | Resume an interrupted hugh run. See "Resume" section. |
| `--resume-feature` | feature slug | — | With `--resume`, restart only the named feature(s) — comma-separated for multiple. Unspecified features keep their prior status. |
| `--list-runs` | — | — | Print all runs under `.alice/mem/hugh/` with their status and last-updated timestamp, then stop. |

`--effort` controls workflow depth (which SOP steps run), not which model backs a sub-agent — see "Model tier selection" in `sub-agent-orchestration.md` for that separate knob.

Convenience short-flags:
- `--murmur` → `--mode=murmur`
- `--low` / `--high` / `--max` → `--effort=<level>`
- `--parallel <N>` → `--max-parallel <N>`

### Examples

```
# Single blob — hugh splits, asks one batched intake question if ambiguous
/hugh "add dark mode toggle, migrate logging to structured json, and bump react to 19"

# Explicit list — no splitting, three dianas run in parallel
/hugh --features "dark-mode,structured-logging,react-19-bump"

# From a draft file with one H2 per feature
/hugh --from-file drafts/sprint-batch.md

# Murmur mode — each diana asks its own clarification batch upfront
/hugh --murmur "rework billing and add saml sso"

# High effort across the board, max 5 parallel
/hugh --high --max-parallel 5 --from-file drafts/q3-batch.md

# Resume the most recent hugh run, only retrying the failed feature
/hugh --resume latest --resume-feature structured-logging
```

If `$ARGUMENTS` is empty AND no runs exist under `.alice/mem/hugh/`: print the usage block and stop.

---

### Modes

The two modes are forwarded to each spawned diana verbatim and ALSO control hugh's own intake behavior:

| Phase | fully-auto | murmur |
|-------|-----------|--------|
| Split intake (Step 0) — comprehension only | ✓ allowed if the blob is unintelligible OR auto-split is ambiguous | ✓ allowed |
| Mid-run inbox relay (Step 4) | ✗ never auto-rebroadcasts; only surfaces if user is actively watching | ✓ may pause and ask the user whether to relay a cross-feature signal |
| Sibling-failure handling | ✗ never asks — applies the failure policy below | ✓ asks whether to abort siblings or continue |
| End-of-run summary | ✓ informational | ✓ informational |

**The forwarded mode is independent of hugh's own mode.** Hugh in `fully-auto` still spawns each diana in `fully-auto` (each diana's own intake gate may still fire if its own brief is unintelligible — that fires inside the spawned agent's session, not hugh's). Hugh in `murmur` spawns each diana in `murmur`. There is no flag to mix modes across siblings — keep batches uniform.

---

### Effort tiers

Effort is forwarded verbatim to each spawned diana. Hugh does NOT re-interpret tiers; diana owns the SOP completeness contract. Hugh's own work (split, dispatch, monitor, drain) is the same at every tier.

If sibling features have wildly different review needs (one trivial, one auth-touching), the user can run two `/hugh` invocations with different `--effort` instead of mixing.

---

### Resume

Runs can die mid-orchestration — network drop, token-limit truncation, crash, user interrupt. The `.alice/mem/hugh/<run-slug>/` dir is the resume source of truth. **Important:** each spawned diana also has its own resumable `.alice/mem/diana/<diana-run-slug>/` inside its worktree. Hugh resume re-anchors against the hugh run, then either asks each in-flight diana to resume itself (via `/diana --resume <slug>` issued in the worktree), or restarts the feature from scratch if the diana state is unsalvageable.

### `--list-runs`

Iterate `.alice/mem/hugh/*/`, print `slug | mode | effort | features=N | running=R | done=D | failed=F | updated=ISO`. Sort by updated-desc. Stop after printing.

### `--resume <slug|latest>`

1. Resolve slug. `latest` → most recently updated run dir.
2. Abort if `run.conf` is missing.
3. Load `run.conf` — re-populate `MODE`, `EFFORT`, `MAX_PARALLEL`, `FEATURES[]`, `BRANCH_AT_START`, `RUN_SLUG`, `RUN_DIR`. CLI overrides for these are ignored (warning printed).
4. For each feature in `features/<slug>/`:
   - Read `status.txt`. States: `queued`, `running`, `done`, `failed`, `blocked`, `aborted`.
   - `done` / `aborted` → skip unless explicitly named in `--resume-feature`.
   - `running` → treat as interrupted. Read `diana-slug.txt` and `worktree-path.txt`. Check whether the diana run dir at `<worktree>/.alice/mem/diana/<diana-slug>/` still has a non-`.done` step marker. If yes, re-dispatch the diana sub-agent in that worktree with `/diana --resume <diana-slug>`. If no, mark feature `done` and continue.
   - `failed` / `blocked` → `AskUserQuestion`: `A) Retry this feature from scratch (drop worktree, restart diana)  B) Resume the existing diana run (re-dispatch with /diana --resume)  C) Skip — leave failed  D) Abort hugh resume`.
   - `queued` → start fresh.
5. Resume monitor loop (Step 4) over all features now in `running` state.

### `--resume-feature <feature-slug>[,...]`

Forces hugh to restart only the named feature(s) regardless of their `status.txt`. Other features keep their prior status (and any in-flight ones continue). Use after manually fixing a blocker in one worktree and wanting to push only that one back through.

---

### The pipeline

All steps run under a common run slug. For a new run:

```bash
RUN_SLUG="hugh-$(date -u +%Y%m%dT%H%M%SZ)-$(printf '%04x' $((RANDOM%65536)))"
RUN_DIR="$ROOT/.alice/mem/hugh/$RUN_SLUG"
mkdir -p "$RUN_DIR/features" "$RUN_DIR/inbox" "$RUN_DIR/transcripts" "$RUN_DIR/steps" "$RUN_DIR/worktrees"
```

For a resumed run, `RUN_SLUG` and `RUN_DIR` come from the resume resolution above.

### Step bookkeeping contract

Same shape as diana — markers under `$RUN_DIR/steps/` with states `.in-progress`, `.done`, `.failed`, `.skipped`. Step ordering:

```
00-split
01-context-bundle
02-worktree-setup
03-dispatch
04-monitor
05-handback
06-cross-feature-retro
07-drain
```

Per-feature status lives separately under `$RUN_DIR/features/<feature-slug>/status.txt` — see Step 3.

### Step 0 — Split & intake gate

If the user passed `--features` or `--from-file` or `--no-split`: skip splitting, build the feature list directly. Otherwise:

1. Read the blob from `$ARGUMENTS`.
2. Hugh attempts an autonomous split: identify discrete features by scanning for explicit list markers ("and", numbered lists, bullet points, sentences each describing a separate deliverable). Cross-check by greping `docs/wiki/`, `docs/ledger/`, `docs/todos/overview.md` for related work — if the blob mentions an existing in-flight TODO, treat that as one feature.
3. Result: a list `FEATURES[]` of `{slug, brief}` pairs. Slug derives from the brief's first noun phrase via the same convention `alice-slug` uses.
4. **Intake gate (both modes).** Hugh ALWAYS surfaces the proposed split via a single `AskUserQuestion` before spinning anything up — this is a comprehension check, not a clarification batch. Question:

   ```
   I'll split this into the following features and run a diana per feature:
     1. <slug-1> — <one-line summary>
     2. <slug-2> — <one-line summary>
     3. <slug-3> — <one-line summary>

   A) Looks right — proceed
   B) Merge two — I'll tell you which
   C) Drop one — I'll tell you which
   D) Abort and let me re-prompt
   ```

   On B / C, follow up with a focused `AskUserQuestion` to capture the specific edit, then re-confirm. On D, exit cleanly.

5. **`fully-auto` skips this confirmation ONLY when the split is unambiguous** — exactly one obvious feature, OR the user already passed an explicit list / file. Multi-feature blobs always confirm because misinterpreting the split fans out the cost across N parallel runs.

Record the final split in `$RUN_DIR/decisions.md` under `## Split` with the one-line rationale per feature.

If `FEATURES[]` ends up with one entry, hugh continues — but logs a note that this would have been better as a plain `/diana` invocation.

### Step 1 — Context bundle

For each feature in `FEATURES[]`, hugh assembles a per-feature prompt that diana receives at dispatch. Common prefix (built once, reused per feature):

1. **Adopter CLAUDE.md highlights.** Read `<repo>/CLAUDE.md`. Extract any sections named "Critical gotchas", "Dev server", "Migration-class files", "Branching", "Test commands". These are diana's known sections; pass them through verbatim so each spawned diana doesn't re-discover them.
2. **Port allocation.** If adopter CLAUDE.md declares a dev-server convention (port number + per-instance offset, e.g. "Dev server runs on port 3000; for parallel sessions, increment by 10 per session"), hugh allocates `port_base = declared_port + (i * offset)` per feature index `i`. If not declared, hugh defaults to `port_base = 4100 + (i * 10)` and notes in `decisions.md` that the adopter CLAUDE.md should declare a port convention for predictability.
3. **Worktree path.** `<repo>/.alice/mem/hugh/<run-slug>/worktrees/<feature-slug>/`. Absolute.
4. **Sibling features list.** One-line per sibling slug — diana uses this only for awareness (so it can tag inbox messages with intended recipients), never for direct cross-edits.
5. **Inbox path.** `<repo>/.alice/mem/hugh/<run-slug>/inbox/<feature-slug>/` — diana writes outbound notes here (one `.md` per note); hugh polls and relays. See Step 4.

Per-feature prompt skeleton written to `$RUN_DIR/features/<feature-slug>/brief.md`:

```markdown
# Feature brief: <feature-slug>

<the feature's one-paragraph description from the split>

## Sibling context (informational, do not edit sibling code)

- <sibling-slug-1>: <one-line>
- <sibling-slug-2>: <one-line>

## Hugh-allocated runtime

- Worktree: <abs path>
- Dev-server port base: <N> (use this if you start any dev server / port-binding process; sibling dianas have different bases)
- Inbox: <abs path to inbox/<feature-slug>/>

## How to send a cross-feature note

If during your run you discover a coupling, conflict, or assumption that affects
a sibling feature, write a short markdown file to:

  <abs path to inbox/<recipient-slug>/<UTC-timestamp>-<your-slug>.md>

Hugh polls all inboxes; the user decides whether and when to relay. Do NOT edit
files outside your worktree to "fix" the sibling — that's hugh's relay job.

## Adopter CLAUDE.md highlights (forwarded verbatim)

<extracted sections>
```

Record port allocations in `$RUN_DIR/decisions.md` under `## Port allocation`.

### Step 2 — Worktree setup

For each feature, create an isolated git worktree off the repo's default branch.

1. Detect default branch (same convention diana uses):
   ```bash
   DEFAULT_BRANCH=$(git symbolic-ref --quiet refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
   DEFAULT_BRANCH="${DEFAULT_BRANCH:-$(git config --get init.defaultBranch || echo main)}"
   git fetch origin "$DEFAULT_BRANCH" --quiet
   ```
2. Verify the main repo's working tree is clean. If `git status --porcelain` is non-empty: in `murmur`, ask the user to stash/commit; in `fully-auto`, write BLOCKED.md and abort the run (per diana's fully-auto blocker pattern).
3. For each feature, create the worktree:
   ```bash
   WT="$RUN_DIR/worktrees/<feature-slug>"
   git worktree add --detach "$WT" "origin/$DEFAULT_BRANCH"
   ```
   Use `--detach` — diana inside the worktree will create its own feature branch off the detached HEAD.
4. Record `$RUN_DIR/features/<feature-slug>/worktree-path.txt` with the absolute path. Record `$RUN_DIR/features/<feature-slug>/port.txt` with the allocated port base.
5. Initialize `$RUN_DIR/features/<feature-slug>/status.txt` with `queued`.

If any worktree creation fails (path exists, dirty index, lockfile): clean up the partial worktrees created earlier in this step and abort the run with a clear error. Don't half-dispatch.

### Step 3 — Dispatch

**All sub-agent dispatch in this step must follow `.alice/rules/sub-agent-orchestration.md`** — poll ≥1/min for background dianas, escalate silence via `AskUserQuestion`, never silently kill an agent.

For each feature batch (respecting `--max-parallel`), dispatch one background sub-agent per feature **in the same assistant message** (multiple `Agent` blocks → concurrent). Per-dispatch:

- `subagent_type: general-purpose`
- `run_in_background: true`
- `prompt`: see skeleton below
- Capture the returned agent ID into `$RUN_DIR/transcripts/03-dispatch.md` under `## Spawned agents`, one line per dispatch (per the sub-agent capture contract).

Prompt skeleton:

```
You are a single feature worker dispatched by hugh. Your job is to run the
/diana skill end-to-end against ONE feature in an isolated git worktree.

Working directory (cd here before doing ANYTHING):
  <abs worktree path>

Feature brief (read first):
  <abs path to features/<feature-slug>/brief.md>

Diana invocation (run this verbatim after cd):
  Invoke the /diana skill with arguments:
    --mode=<MODE> --effort=<EFFORT> --from-file <abs path to brief.md>

Constraints:
  1. Do NOT edit any file outside <abs worktree path>. Other dianas are
     working in sibling worktrees on the same repo concurrently.
  2. If you start a dev server / any port-binding process, bind to the
     port base in brief.md ("Dev-server port base"). Sibling dianas have
     different bases — do not reuse defaults that will collide.
  3. Cross-feature signals: write to <abs path to inbox/<recipient>/...>
     as described in the brief. Never reach into a sibling's worktree.
  4. When diana hands back (success, blocked, or failed), write a single
     line summary to <abs path to features/<feature-slug>/handback.md>
     with: status (done|blocked|failed), branch name, commit count, and
     a one-line headline. Then exit.
  5. If you hit a tool permission block, STOP. Return a single line
     "BLOCKED: <tool> needed for <reason>". Do not work around.

Mode: <MODE>   Effort: <EFFORT>   Sibling slugs: <comma-list>
```

After dispatch, set `status.txt` for each dispatched feature to `running` and record the agent ID into `$RUN_DIR/features/<feature-slug>/agent-id.txt`.

If the batch is throttled by `--max-parallel`, the remaining features stay `queued`. The monitor loop (Step 4) promotes a queued feature to `running` whenever a sibling reaches a terminal state.

### Step 4 — Monitor + relay loop

Poll loop. Every ~60s:

1. **Sub-agent progress poll.** For each `running` feature, call `TaskOutput` on its agent ID. Compute byte delta vs. the prior poll. Update `last_bytes` / `last_at` in-memory.
   - **Progress** → continue.
   - **Silent across 3 consecutive polls (≥3 min no output)** → flag for escalation.
2. **Inbox sweep.** For each `inbox/<feature>/` dir, list new files since the prior poll (track `inbox-cursor.txt` per feature). For each new file:
   - **`fully-auto`:** read the file, append a one-line summary to `$RUN_DIR/transcripts/04-monitor.md` under `## Inbox`. Do NOT auto-relay — surface only on the next user turn or at end-of-run. The note stays in the recipient's inbox folder; the recipient diana does not auto-poll its own inbox during its run, so the relay is effectively deferred until hugh's monitor or post-run summary.
   - **`murmur`:** pause the loop, surface via `AskUserQuestion`:
     ```
     New cross-feature note from <sender-slug> to <recipient-slug>:

     <first 500 chars of the note>

     A) Relay now — pause <recipient> diana, inject the note as additional
        context, resume (re-dispatch with /diana --resume + an injected
        addendum to the brief)
     B) Hold for end-of-run — recipient sees it after their diana completes
     C) Discard — note is irrelevant
     ```
     On A, hugh writes the note into `<recipient-worktree>/.alice/mem/hugh-relay/<timestamp>.md` and re-dispatches the recipient with `/diana --resume <recipient-diana-slug>` plus instructions to read that file before continuing. Record the relay in `decisions.md`.
3. **Status check.** For each `running` feature, check whether `<worktree>/.alice/mem/diana/<diana-slug>/steps/09-drain.done` exists (the only marker that signals diana fully completed) OR whether `BLOCKED.md` / `DRAIN-FAILED.md` exists in the diana run dir.
   - `09-drain.done` present → set `status.txt` to `done`. Capture `handback.md` if the agent already wrote it; if not, drain `TaskOutput` and synthesize a handback line from the agent's last 2KB of output.
   - `BLOCKED.md` present → set `status.txt` to `blocked`. Copy the path into `handback.md`.
   - `DRAIN-FAILED.md` present → set `status.txt` to `failed` with reason "diana drain failed". Hugh's own drain (Step 7) will inherit and re-attempt.
   - Agent itself reports terminal state via `TaskGet` (completed / failed) without diana markers → log to `decisions.md` as a contract violation, set `status.txt` to `failed`.
4. **Promote queued.** If a `running` feature transitioned to a terminal state and queued features remain, dispatch the next one (back to Step 3 dispatch flow for that single feature). Re-batch in same-message blocks where multiple promotions land in the same poll cycle.
5. **Escalation handling for flagged silent agents.** Per the orchestration rule, present:
   ```
   <feature-slug> — diana agent silent for <X> minutes (last byte at <ISO>)

   A) Keep waiting — poll for another 5 min before re-asking
   B) Fetch full TaskOutput tail and inspect together
   C) Abort this feature — mark failed, free the slot for queued siblings
   D) Abort the whole hugh run — preserve state for resume
   ```
6. **Sibling-failure handling.** When a feature transitions to `failed` or `blocked`:
   - **`fully-auto`:** continue the rest of the batch unaffected. Failed features land in the final summary; user resumes them after fixing.
   - **`murmur`:** pause, ask `A) continue siblings  B) abort all running siblings  C) abort run`.

Loop continues until every feature is in a terminal state (`done`, `failed`, `blocked`, `aborted`).

### Step 5 — Per-feature handback collection

For each feature, read its `handback.md`. Aggregate into `$RUN_DIR/transcripts/05-handback.md` with one section per feature:

```
## <feature-slug>

- Status: <done|blocked|failed|aborted>
- Worktree: <abs path>
- Branch: <branch name in that worktree>
- Diana run: <diana run slug — link to <worktree>/.alice/mem/diana/<slug>/>
- Commits: <count> (<first SHA>..<last SHA>)
- Headline: <one-line>
- Inbox notes sent: <N>
- Inbox notes received: <N>
```

Where data is missing (failed before reaching diana, agent crashed mid-drain), record what's known and flag the gap.

### Step 6 — Cross-feature retro

This is hugh's own retro — separate from each diana's per-feature retro (which already ran inside each spawned agent's session per `.alice/rules/post-feature-retro.md`).

1. **Append to `docs/ledger/experiences.md`:** one short entry on what surprised hugh — e.g. "split heuristic merged two features that should have been one", "port collision still happened because adopter CLAUDE.md didn't declare convention", "feature X's diana blocked on a dep that feature Y added — should have been ordered, not parallel". Best-effort: hugh can only write what it observed.
2. **Append to `docs/ledger/decisions.md`:** any non-obvious choice hugh made (split rationale when it diverged from the user's wording, relay decisions in murmur, port allocations the adopter should know about for next time).
3. **Inbox audit.** List every inbox note sent and its disposition (relayed / held / discarded). If notes piled up without ever relaying, flag in the experiences entry — that's a sign of either bad split (features were too coupled) or thin monitoring.

Hugh does NOT touch wiki, plans/active, plans/archive, or todos in this step — those belong to each per-feature diana's retro, which already ran. Hugh only writes the ledger entries that capture the orchestration-level lessons.

If every feature failed before reaching diana's own retro, skip wiki delegation but still write the experiences entry — that's the most important one to record.

### Step 7 — Drain

Same role as diana's Step 9 drain, but at the hugh layer: confirm every feature's spawned background sub-agent has terminated, every diana run inside each worktree has reached its own terminal state, and no orphan background shells remain.

1. **Enumerate spawns.** Walk `$RUN_DIR/transcripts/` for `## Spawned agents` and `## Background shells` subheadings — same capture contract diana uses. Build `SPAWNED_IDS` and `BG_SHELLS`.
2. **Poll sub-agents until terminated.** For each ID in `SPAWNED_IDS`, follow diana's drain logic verbatim: poll every 30s up to 5 min, drain `TaskOutput` once on terminal state, force-stop via `TaskStop` if still running after 5 min and log under `## Drain` in `decisions.md`.
3. **Worktree cleanup.** For each feature:
   - If `status.txt` is `done` AND the user's adopter CLAUDE.md does NOT have a "preserve hugh worktrees" note (default is preserve, since the user often wants to inspect): leave the worktree in place. Record the abs path in the final summary so the user can `git worktree remove` after pushing.
   - If `status.txt` is `aborted` (user abort with explicit cleanup): `git worktree remove --force <path>`.
   - If `status.txt` is `blocked` or `failed`: leave the worktree in place. The user resumes via `/hugh --resume <slug>` which expects the worktree intact.
4. **Final assertion.** Drain succeeds only if all of:
   - No `*.in-progress` markers remain under `$RUN_DIR/steps/`.
   - Every ID in `SPAWNED_IDS` is in a terminal state.
   - Every ID in `BG_SHELLS` has exited or been killed.

   If any assertion fails, write `$RUN_DIR/DRAIN-FAILED.md` listing survivors and exit without printing the success summary — same contract as diana. Do **not** auto-`/exit`.

5. **Final summary.** Only when assertions pass, `mv 07-drain.in-progress 07-drain.done`, then print:

```
hugh run complete — $RUN_SLUG

Mode: <fully-auto | murmur>   Effort: <low | medium | high | max>   Max parallel: <N>

Features (<done>/<total>):
  [done]    dark-mode           — branch feat/dark-mode (4 commits, 6 files)
                                  worktree: <abs path>
                                  diana run: <abs path>
  [done]    structured-logging  — branch feat/structured-logging (3 commits, 12 files)
                                  worktree: <abs path>
                                  diana run: <abs path>
  [blocked] react-19-bump       — see <abs path to BLOCKED.md inside diana run>
                                  resume with /hugh --resume <slug> --resume-feature react-19-bump

Inbox notes:           <N> sent, <M> relayed, <K> held to summary
Autonomous decisions:  see $RUN_DIR/decisions.md
Branches:              changes staged locally per worktree. Push / PR is up to you.
                       Worktree cleanup: git worktree remove <path> after merge.
```

Never auto-push. Never auto-create PR. Hugh stops at "staged locally per worktree" and hands back.

---

### Sub-agent capture contract

Same shape as diana — every step that uses the `Agent` tool MUST record the returned agent ID in its transcript file under a `## Spawned agents` subheading, one line per dispatch:

```
- <agent-id>  type=<subagent_type>  feature=<feature-slug>  spawned=<ISO timestamp>  background=<true|false>
```

The `feature=` field is hugh-specific — it links the agent ID back to the feature slug so Step 7 drain can correlate failures to features without parsing transcripts.

---

### Decision policy

Same five principles as diana, with hugh-specific applications:

1. **Prefer more verification over less.** When the split is ambiguous, ask. When a feature looks like it could be two, propose splitting further (cap at the user's `--max-parallel` budget). Cheaper to dispatch one too many than to entangle two unrelated changes in one diana.
2. **Prefer conservative scope interpretation.** When the blob is between "narrow split" (3 features) and "broad split" (5 features), pick the narrower split — under-splitting yields fewer parallel runs but each diana stays focused. Over-splitting fragments features that should be co-designed.
3. **Prefer reuse over novelty.** When two proposed features touch the same files / wiki pages, that is a strong signal they should NOT be parallel. Surface as a "merge into one feature" recommendation in the intake gate.
4. **Prefer existing style over "improvement".** Hugh forwards adopter CLAUDE.md highlights verbatim into each brief. Hugh does not summarize or rewrite — the adopter's exact words land in each spawned diana.
5. **Prefer transparency over silent choice.** Every split decision, port allocation, relay decision, and sibling-failure response is logged in `decisions.md`.
6. **On irreversibles — defer in fully-auto, escalate in murmur.** Hugh has fewer irreversible operations than diana (no DB schema choices, no API contracts), but `git worktree add` / `git worktree remove --force` / cross-feature relay all qualify. Apply diana's same policy.

---

### Failure handling

Retry / escalation for hugh's own steps:

- **Worktree creation fails (Step 2)** — abort the run, clean up partial worktrees. Don't half-dispatch.
- **Sub-agent silence in monitor (Step 4)** — poll → flag → escalate per orchestration rule.
- **Diana inside a worktree reports BLOCKED / DRAIN-FAILED** — propagate as feature-level failure, do not retry hugh-side. Resume is the user's recovery path.
- **Multiple simultaneous failures** — in `fully-auto`, log all and continue siblings. In `murmur`, pause and ask once per failure (batched if they land in the same poll cycle).
- **Adopter CLAUDE.md missing dev-server convention** — hugh allocates from default pool, logs a note recommending the adopter declare it. Not a failure.
- **`--max-parallel` exceeds available worktree slots** — capped silently (hugh creates only as many worktrees as features it has; `--max-parallel` only throttles concurrent dispatch).

In `fully-auto`, hugh does NOT call `AskUserQuestion` mid-run except for:
- The Step 0 split-confirmation intake gate (ALWAYS surfaced — see Step 0).
- The Step 4 silence escalation (mandated by the orchestration rule — orchestration rule beats fully-auto's no-questions stance).

All other fully-auto blockers write to `$RUN_DIR/BLOCKED.md` with full context and the user resumes manually.

---

### State & audit trail

`.alice/mem/hugh/<run-slug>/` layout after a complete run:

```
run.conf                 — immutable run config (mode/effort/max-parallel/feature list/branches)
manifest.md              — human-readable run summary
decisions.md             — every autonomous decision + rationale
BLOCKED.md               — only if hugh escalated mid-run (rare; per-feature blocks live in the diana run dir)
DRAIN-FAILED.md          — only if Step 7 could not bring all spawns to a terminal state
features/
  <feature-slug>/
    brief.md             — per-feature prompt fed to the spawned diana
    worktree-path.txt    — abs path to that feature's worktree
    port.txt             — allocated port base
    agent-id.txt         — sub-agent ID hugh dispatched
    diana-slug.txt       — diana run slug inside the worktree (captured from agent output)
    status.txt           — queued | running | done | blocked | failed | aborted
    handback.md          — per-feature one-section summary written at terminal state
inbox/
  <feature-slug>/
    <UTC>-<sender>.md    — cross-feature notes addressed to this feature
inbox-cursor.txt         — monitor loop's last-seen marker per feature
worktrees/
  <feature-slug>/        — git worktree (left in place unless user-aborted with cleanup)
steps/                   — step markers, source of truth for resume
  00-split.{done|skipped}
  01-context-bundle.done
  02-worktree-setup.done
  03-dispatch.done
  04-monitor.done
  05-handback.done
  06-cross-feature-retro.done
  07-drain.done
transcripts/
  00-split.md            — split rationale + intake answers
  01-context-bundle.md   — extracted CLAUDE.md sections + port allocations
  02-worktree-setup.md   — worktree creation log
  03-dispatch.md         — agent IDs + dispatch prompts (## Spawned agents)
  04-monitor.md          — poll log + inbox sweep history (## Inbox)
  05-handback.md         — aggregated per-feature handbacks
  06-cross-feature-retro.md — ledger entries hugh appended
  07-drain.md            — drain poll log
```

Everything under `.alice/mem/` is gitignored — including the worktrees, even though they're git working trees. The audit trail and worktrees stay with the local checkout; `git worktree remove` (or deleting the run dir) cleans them up.

---

### Hard rules

- **Hugh is an orchestrator, not an implementer.** Hugh dispatches dianas and aggregates results. Hugh never edits adopter source files outside `<repo>/.alice/mem/`. Hugh never invokes `/plan`, `/review`, `/security-audit` directly — those are diana's contract, run inside each spawned agent's session.
- **Never push, create PRs, or deploy.** Same as diana. Hugh stops at "staged locally per worktree."
- **Every sub-agent dispatch follows `.alice/rules/sub-agent-orchestration.md`.** Poll ≥1/min; escalate silence; handle `BLOCKED:` per protocol.
- **Every spawned diana runs in its own worktree.** No two dianas share a working tree. The user's primary checkout stays clean — hugh does not check out branches there.
- **Port allocation is mandatory.** Every per-feature brief contains an allocated port base, even if the adopter's CLAUDE.md declares no dev-server convention. Hugh defaults to a generic pool starting at 4100 (step 10) when the adopter is silent. This prevents the most common parallel-failure mode (two dianas race for `localhost:3000`).
- **Inbox is shared state, not a comms channel.** Dianas write notes; hugh decides whether to relay. No diana ever reads a sibling's inbox or worktree. Cross-feature signals flow through hugh as the only middleman.
- **Step markers are sacred.** Same contract as diana — never delete a `.done` marker except via `--resume`, never write a `.done` marker without finishing the work.
- **Drain is binding.** Step 7 always runs. Only Step 7 prints the final "hugh run complete" summary. Until drain confirms every spawned sub-agent is terminated, the run is not done.
- **Sub-agent capture is binding.** Same contract diana uses. Skipping the capture forces lossy fallback (grep) and is logged as a contract violation.
- **`--max-parallel` is a throttle, not a guarantee.** If the runtime backpressures or tooling rate-limits, fewer than `--max-parallel` may run concurrently. The skill should not assume any specific concurrency level — features queue and run as slots free.
- **Hugh inherits diana's "no mixed-mode siblings" stance.** A single hugh run uses one mode and one effort across all features. Mixing requires two `/hugh` invocations.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "These two features overlap — let me share a worktree to save disk." | Shared worktree = race on `node_modules`, lockfiles, and config files. Each diana gets its own worktree, full stop. |
| "Skipping port allocation since the adopter doesn't run a dev server." | The adopter's CLAUDE.md was silent at run time, not forever. Always allocate from the default pool — silent-now is not silent-later. |
| "The inbox is empty, drain is trivial." | Drain isn't about messages — it's about confirming every sub-agent terminated. Skip drain and you orphan processes. |
| "One diana is hung but the others are fine — let it ride." | Poll every minute. Three silent polls = surface to user via the `sub-agent-orchestration` rule. Silent stalls compound into lost runs. |
| "Sub-agent capture failed — grep fallback is fine." | Lossy. Log the contract violation, surface the gap to the user, and prefer to rerun the affected feature rather than ship a half-captured handback. |
| "I'll let one feature push while the others stage." | Hugh stops at "staged locally per worktree." No exceptions. The user owns push / PR / deploy. |
| "Resuming partial features takes too long — restart all." | `--resume-feature` exists for a reason. Restart only the features that genuinely need it; resume the rest from their last `.done` marker. |

## Red Flags

- Hugh editing files outside `<repo>/.alice/mem/`.
- Two dianas writing to overlapping worktrees or sharing a port.
- A `.done` marker present but the corresponding step's output missing.
- Cross-feature notes routed between dianas without hugh as the middleman.
- A run that "completes" without Step 7 (drain) firing.
- A single hugh run mixing effort tiers or modes across features.
- Sub-agent dispatch with no polling loop attached (violates `sub-agent-orchestration.md`).

## Verification

A hugh run is DONE only when:

- [ ] Every feature's per-feature handback section is filled under the run dir.
- [ ] Every spawned diana has a `done` or `failed` status; no `unknown`.
- [ ] Step 7 (drain) emitted the "hugh run complete" summary; no orphaned sub-agents reported.
- [ ] Cross-feature retro section is written, even if the conclusion is "no significant cross-feature signals".
- [ ] Each worktree is left staged (not pushed) and the user has the list of branches plus the suggested next action.
- [ ] No file outside `<repo>/.alice/mem/` was written by hugh itself (each diana wrote inside its own worktree).
- [ ] `--list-runs` shows the run with an updated terminal timestamp.

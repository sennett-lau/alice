---
name: pr-slicer
preamble-tier: 4
version: 1.0.0
description: |
  Slice a large working branch / PR into a chain of smaller, reviewable PRs.
  Detects adopter-declared migration-class files (forces a migration PR
  first), builds a dependency graph, writes an `overview.md` + per-PR spec
  folder under `.tmp/pr-slicer/`, then dispatches `pr-slicer-executor`
  sub-agents (optionally in git worktrees) to build each PR. Main session
  keeps the remote side — push, `gh pr create`, rebase cascade. Use when
  asked to "slice this PR", "pr-slicer", "break this branch into PRs",
  "chop this up into smaller PRs".
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
mkdir -p "$ROOT/.tmp/pr-slicer"
echo "BRANCH: $(git branch --show-current)"
```

State lives at `<repo>/.tmp/pr-slicer/<slug>/`. Gitignored. No telemetry.

**Load the orchestration rule.** Every sub-agent dispatch in this skill must follow `.alice/rules/sub-agent-orchestration.md` — progress polling (≥1/min) and permission escalation. Read it before Step 6.

---

## Arguments (`$ARGUMENTS`)

```
/pr-slicer [<PR# | branch>] [--mode=sequential|parallel]
```

| Param | Values | Default | Notes |
|-------|--------|---------|-------|
| source | `#123`, `42`, or a branch name | **current branch** (if not base branch) | PR number takes precedence; branch name second; otherwise current branch. Stops if current branch is the base branch and no arg given. |
| `--mode` | `sequential`, `parallel` | **`sequential`** | Execution mode for dispatching executors. See below. |

### Modes

- **`sequential` (default)** — one executor at a time, in dependency order. Shared working tree. Safer, easier to reason about, lower tool-call volume. Use when slices are tightly coupled or when reviewer wants to see each slice land before the next starts.
- **`parallel`** — every slice that has no pending dependency is dispatched in the same assistant message using a separate git worktree. Only **parallel-safe** siblings (same `base`, no mutual deps) actually run concurrently; chained deps still serialize. Faster on large fan-outs; needs worktree support and enough disk.

Mode affects **Step 6 (Dispatch)** only. Planning (Steps 1–5) is identical.

### Examples

```
/pr-slicer                           # slice current branch, sequential
/pr-slicer --mode=parallel           # slice current branch, parallel-safe fan-out
/pr-slicer #482                      # slice GH PR #482, sequential
/pr-slicer feat/big-feature          # slice named branch, sequential
/pr-slicer #482 --mode=parallel      # slice PR #482, parallel fan-out
```

If the user invokes with no args from the base branch: print the usage hint and stop.

---

# PR Slicer

Orchestrator. Main session plans + writes specs + dispatches executors + runs reviews + handles all remote side-effects. Executor sub-agents (`pr-slicer-executor`) do the per-PR code work in isolated working trees and hand back a structured report.

---

## Step 1 — Trigger gate

Parse `$ARGUMENTS` into `{source, mode}`. `mode` defaults to `sequential` if the `--mode=` flag is absent or malformed.

Resolve the **base branch** from adopter config. Read `CLAUDE.md` for a "Branching / base branch" note (common names: `main`, `master`, `develop`, `prod`). If the adopter declares a deployment branch separately (e.g. `prod`), note it too — both count as "don't slice from here" sources. If unclear, ask the user once; record in `overview.md`.

At least one source must resolve, else stop:

1. User passed a PR number (`$ARGUMENTS` matches `#?\d+`) → `gh pr view <num> --json headRefName,baseRefName,title,body` to resolve branch + base.
2. User passed a branch name → use that as source branch.
3. No source arg AND current branch is **not** the base branch → use current branch as source.

If none match: print the **Arguments** usage block above and stop.

Resolve:
- `SRC_BRANCH` — the big branch being sliced
- `BASE` — base branch (default `main`; use PR's baseRefName if PR given)
- `SLUG` — `pr-slicer-$(echo "$SRC_BRANCH" | tr '/' '-')-$(date +%Y%m%d-%H%M)`
- `PLAN_DIR` — `$ROOT/.tmp/pr-slicer/$SLUG`

```bash
mkdir -p "$PLAN_DIR/spec"
git fetch origin "$BASE" --quiet
```

---

## Step 2 — Evaluate changes

```bash
git diff origin/$BASE...$SRC_BRANCH --stat
git diff origin/$BASE...$SRC_BRANCH --name-status
git log origin/$BASE..$SRC_BRANCH --oneline
```

Read the full diff for every changed file (`git diff origin/$BASE...$SRC_BRANCH -- <file>`). Note:

- `MIGRATION_FILES` — files matching adopter-declared migration-class patterns (see Step 2.5).
- `SHARED_TYPES_FILES` — any cross-package type files the adopter flags in CLAUDE.md (or obvious candidates like shared types packages / generated API types).
- Feature surfaces (routes, handlers, events, pages, components, tests) — categorize based on paths relative to the adopter's layout.
- Test additions.

If no changed files → stop with "Nothing to slice".

Also check for active plan folder under `docs/plans/active/` matching `SRC_BRANCH` — its spec is the source of truth for the final "implementation" PR's intent.

---

## Step 2.5 — Detect migration-class files

**What counts as "migration-class".** Files that are:

1. **Hard to revert once deployed** — database migration DDL, persistent storage config, external-facing API version bumps, secret/key rotation, infrastructure provisioning.
2. **Prone to rebase conflicts** when multiple PRs touch adjacent content — config files that every service reads, sequential identifier registries, shared manifest files.

The idea: land these on their own **migration PR** first, merged into the deployment branch before other slices. Other slices then rebase on top of that migration cleanly. Downstream benefits:

- If a later slice goes wrong, you can revert it without rolling back the infra / schema change
- Parallel PRs don't fight over the same sensitive file
- The migration PR gets focused review (often the highest-risk change in the stack)

**How the skill finds them.** Stack-specific — the adopter owns the list. Check, in order:

1. **`CLAUDE.md` "Migration-class files" section.** Look for a heading that says `## Migration-class files` (or close equivalent). Each bullet is a pattern (glob or path). Use these verbatim.
2. **Obvious conventional patterns as a fallback** (only suggest, do not auto-apply):
   - `**/migrations/*.sql`, `**/migrations/*.py`, `**/db/migrate/*`, `prisma/migrations/**`, `drizzle/**/*.sql`
   - `wrangler.toml`, `wrangler.jsonc`, `fly.toml`, `vercel.json`, `netlify.toml`, `app.yaml`
   - `terraform/**/*.tf`, `pulumi/**`, `helm/**/*.yaml`, `k8s/**/*.yaml`, `**/Chart.yaml`
   - `.env.example`, `.env.schema`, `.env.template`
   - Lock files where dep graph genuinely changed (not just version bumps in the same range)
3. **If neither applies**, `AskUserQuestion` with the diff file list: "I didn't find a `Migration-class files` section in CLAUDE.md. Are any of these hard-to-revert or rebase-conflict-prone? Answer with the file list or 'none'."

Record `MIGRATION_FILES[]`. If empty, skip Step 3.1 entirely — no migration PR needed, all slices rebase directly on `BASE`.

**Offer to persist.** If the user identified migration-class files in step 3 of this list (the AskUserQuestion fallback), offer to append a `## Migration-class files` section to `CLAUDE.md` with the declared patterns so future `/pr-slicer` runs don't re-ask. This is opt-in — `AskUserQuestion: A) Append to CLAUDE.md  B) One-time use only`.

---

## Step 3 — Chunking rules

Apply in order. Produces `PRS[]` — an ordered array with `{id, title, branch, base, depends_on[], files[], rationale, self_review[]}`.

### 3.1 Migration PR (always first if MIGRATION_FILES non-empty)

If `MIGRATION_FILES` non-empty:

1. **Build a clean migration slice.** Depending on stack, this may need collapsing multiple migration files into one canonical sequence (e.g. repeated `db:generate` runs produce N incremental files that should be a single net migration). The adopter's `CLAUDE.md` should document the canonical collapse procedure for their stack; reference it from the migration PR's spec file. If no procedure documented, ask the user how to collapse (or confirm no collapse needed).
2. **Don't collapse when migrations are semantically distinct.** If the branch has two or more migrations that represent separate intentional changes (e.g. one adds a table, another adds a column unrelated to the first), keep them as two migration PRs in sequence. Collapse only when the extra files are fragmentation from repeated generation against a moving target.
3. **Branch name:** `migration/<slug-derived-from-content>` (e.g. `migration/add-users-table`, `migration/wrangler-kv-binding`).
4. **Base:** `origin/$BASE`.
5. **Nothing else depends on** — every non-migration PR rebases on this one unless there's a tighter dependency.

Record this as `PRS[0]` (or `PRS[0..M]` if split per point 2).

### 3.2 Dependency-graph the rest

Group remaining changes into coherent chunks. Heuristics:

- One shared-types / schema / protocol change that many consumers import → its own PR, becomes a dep.
- Backend route / handler / event additions → depends on shared-types PR (if any) and migration PR.
- Frontend changes that consume backend additions → depend on backend PR.
- Cross-cutting primitive changes (auth, transport, middleware, lifecycle hooks) → their own PR; downstream deps fall out of the import graph.
- Pure test-only chunks that cover already-landed logic → independent; can parallel.

For each chunk:
- Branch name: `slice/<slug>-NN-<topic>` where `NN` reflects dependency order.
- `base` = nearest ancestor PR branch (migration if none tighter).
- `depends_on` = list of `id`s that must merge first.

Parallel PRs = ones with no mutual dependency and the same `base` — these get worktree isolation.

### 3.3 Final implementation PR

The PR that actually "moves to achieve" the feature goal (typically wiring, feature flag flip, docs + plan move, or the last integration bit) lands **last**. It can depend on every preceding PR but **nothing depends on it**. If the source branch is a pure refactor with no clear "final wiring" commit, the final PR can be the docs/plan-folder archival move alone.

---

## Step 4 — Write the plan

### `$PLAN_DIR/overview.md`

Template (use `~~~` fences to avoid collision with nested code blocks):

~~~markdown
# PR Slicer plan — <SRC_BRANCH>

Source: <SRC_BRANCH>  (vs origin/<BASE>)
Generated: <UTC timestamp>
Slug: <SLUG>
Mode: <sequential | parallel>

## Dependency graph

```
migration/<...>   ──┬── slice/<slug>-02-shared-types ──┬── slice/<slug>-03-backend ────┐
                    └── slice/<slug>-04-frontend ──────────────────────────────────── slice/<slug>-99-final
```

## PRs (in merge order)

| ID | Branch | Base | Depends on | Parallel-safe | Review | PR# | Spec |
|----|--------|------|------------|---------------|--------|-----|------|
| 01 | migration/... | main | — | no | — | — | spec/01-migration.md |
| 02 | slice/...-02-shared-types | migration/... | 01 | no | — | — | spec/02-shared-types.md |

(The `Review` column is updated in Step 6c.5 — values: `clean`, `fixed`, `pushed-with-findings`, `deferred-only`.)

## Rebase / base-update playbook

When an earlier PR merges: for each PR whose `depends_on` contained it, run
`git rebase <new-base>` locally and `gh pr edit <num> --base <new-base>`.
~~~

### `$PLAN_DIR/spec/NN-<topic>.md` (one per PR)

Template:

~~~markdown
# PR <NN> — <title>

- Branch: <branch>
- Base: <base>
- Depends on: <ids or "—">
- Parallel-safe with: <ids or "—">
- Worktree: <yes/no>

## Intent
<1–2 sentences — what this PR accomplishes in isolation>

## Files in scope
- path/a — <what changes here>
- path/b — <what changes here>

## Code snippets / references
Paste the relevant hunks from the source diff so the executor can reproduce
without re-deriving. Include surrounding context where it matters. Quote the
source branch commit SHA if a specific commit is the canonical source.

```diff
<hunk>
```

## Reference pointers
- Source branch: <SRC_BRANCH>@<sha>
- Prior art: docs/plans/archive/..., docs/ledger/...

## Mechanical self-check (executor runs before handing back)
- [ ] Build green (use the command from adopter's CLAUDE.md "Stack" section)
- [ ] Relevant tests pass (use the test command from CLAUDE.md)
- [ ] Touches only files listed above (`git diff --name-only origin/<base>`)
- [ ] Respects all "Critical gotchas" from CLAUDE.md (enumerate the ones that apply to this diff)
- [ ] No stale `// removed` / unused `_vars` from the split

(Code review is the main session's job — do NOT duplicate it here.)
~~~

For the **migration spec**, additionally include:
- The adopter-specific collapse / regeneration procedure verbatim (from their CLAUDE.md or the answer given in Step 2.5).
- Expected final state of the authoritative schema / config file (diff hunk vs `origin/<BASE>`).
- Post-regeneration verification steps (file-list check, sequence-prefix check, format validation — adopter-specific).

---

## Step 5 — Confirm plan with user

Print the overview table + the path to `overview.md`. Use `AskUserQuestion`:

```
Plan ready at <PLAN_DIR>/overview.md — N PRs, dependency graph above.

A) Dispatch executors (worktree-isolated where parallel-safe)
B) I'll review the plan first — stop here
C) Refine — which PR should I re-chunk?
```

If B: stop. If C: loop back to Step 3 for the named PR.

---

## Step 6 — Dispatch executors

Mode (`sequential` | `parallel`) was resolved in Step 1. Default is `sequential`.

**All sub-agent dispatch in this step must follow `.alice/rules/sub-agent-orchestration.md`** — poll ≥1/min for background executors, escalate silence via `AskUserQuestion`, never silently kill an agent. See the rule file for full policy.

- **`sequential`:** dispatch one executor at a time, in topological order of `depends_on`. Same working tree. Wait for handback + Step 6c.5 review + push before dispatching the next. Ignore worktrees entirely.
- **`parallel`:** compute the ready set (all PRs whose `depends_on` is empty or satisfied). Every PR in the ready set that is parallel-safe with its siblings (same `base`, no mutual dep) gets its own worktree and is dispatched in the **same assistant message** (multiple `Agent` blocks → concurrent). Non-parallel-safe PRs in the ready set still serialize.

For each PR in `PRS[]`, respecting `depends_on`:

### 6a. Branch + worktree setup (main session)

**Sequential mode.** Direct checkout in the main working tree. Flow per slice:

```bash
# Ensure previous slice's work is pushed before switching
git checkout <base>                    # base = migration branch, or another slice, or origin/$BASE
git pull --ff-only                     # if base is a tracked remote
git checkout -b <slice-branch>
# ... executor runs here ...
# after push (6d), checkout next slice's base and repeat
```

**Parallel mode.** Each parallel-safe sibling gets its own worktree:

```bash
git worktree add "$ROOT/.tmp/pr-slicer/worktrees/<branch-slug>" -b <branch> <base>
```

Do not let two executors share a working tree. Never start a new slice with uncommitted changes in the main tree.

### 6b. Spawn `pr-slicer-executor` via `Agent` tool

One `Agent` call per PR, with:
- `subagent_type: pr-slicer-executor`
- `run_in_background: true` for parallel mode; foreground is fine for sequential
- `prompt`: absolute path to the PR's spec file + absolute path to `overview.md` + the working directory (repo root or worktree path) + `SRC_BRANCH` + `BASE`.
- Parallel-safe siblings dispatched in the **same assistant message** (multiple `Agent` blocks) so they run concurrently.

Example prompt skeleton:

```
You are building a single PR from a slice plan.

Spec: <abs path to spec/NN-<topic>.md>
Overview: <abs path to overview.md>
Working dir: <repo root or worktree path>
Source branch (read-only reference): <SRC_BRANCH>
Base branch for this PR: <base>

Follow the spec exactly. Cherry-pick / copy hunks from <SRC_BRANCH> as needed
(`git diff <SRC_BRANCH> -- <file>`). Do NOT pull in changes outside the spec's
"Files in scope". Run the self-review checklist at the end and report results.
Do NOT push or create the PR — the main session owns that step.

If you hit a tool permission block, STOP immediately. Return a single line
"BLOCKED: <tool> needed for <reason>" in your report and exit. Do not work
around by trying adjacent tools. Do not retry with different wording.
```

### 6c. Poll for progress while waiting

Parallel mode: record `{agent_id, last_bytes: 0, last_at: now()}` for each dispatched executor. Every ~60s call `TaskOutput` (or equivalent) for each. If bytes unchanged across 3 consecutive polls, escalate via `AskUserQuestion` per the orchestration rule (A/B/C/D choices).

Sequential mode: blocking `Agent` call; no polling needed. But if the call runs past ~5 min, consider re-dispatching with `run_in_background: true` next time.

### 6d. Wait for handback

Each executor returns: commit SHA(s) made, mechanical self-check results, any deviations, and a `BLOCKED:` line if it hit a permission wall.

If handback contains `BLOCKED:` → do **not** treat as failure. Surface to user per the orchestration rule's permission escalation flow: grant-tool / proxy-action / re-scope / abort. Only after resolution, continue.

If any executor reports a non-permission blocker (e.g. build fails, missing dep) → pause dispatch, surface to user.

The executor's self-check is **mechanical** — build green, tests green, in-scope-only, no gotchas. It is NOT a substitute for a real review pass. That happens next.

### 6c.5. Per-PR review gate (the point of slicing)

Smaller PRs exist so each one can be **actually reviewed**. Run a focused review pass on every slice before pushing. Do this even if the source branch was already reviewed — slicing can shuffle intent and introduce copy-paste errors.

1. **Dispatch a review sub-agent** via `Agent` tool. Preferred: `subagent_type: code-reviewer` if registered (alice ships it). Fallback: `general-purpose` with the `/review` skill loaded. Prompt:

   ```
   Review the diff of branch <slice-branch> against <slice-base>. Working dir: <path>.
   Spec: <abs path to spec/NN-*.md>.

   SCOPE — read this carefully:
   Review ONLY the lines/hunks this slice adds or modifies. Pre-existing issues in
   surrounding code are out of scope for fixing. If you notice a pre-existing
   issue, list it under a separate "Adjacent observations" section — do NOT flag
   it as a slice finding, do NOT suggest fixing it in this PR.

   Check against the slice diff only:
   (1) intent match — does the diff do what the spec says, nothing more?
   (2) structural issues introduced by this slice — SQL safety, race conditions,
       trust boundaries, enum completeness, silent failures.
   (3) cross-slice coupling — does this slice accidentally depend on a later
       slice's changes, or duplicate work another slice owns?
   (4) gotcha violations — cross-check against adopter's CLAUDE.md "Critical
       gotchas" section; flag any violation in the new code.

   For each finding use severity: P0 (block), P1 (fix before merge), P2 (defer),
   P3 (nit). Return: findings list with severity + file:line + fix, or "clean".
   List "Adjacent observations" separately.

   If you hit a permission block, stop and return "BLOCKED: ..." (same protocol
   as executors).
   ```

   Parallel mode → dispatch review sub-agents for all pushed-pending siblings in the same assistant message. Apply the same polling + BLOCKED handling per the orchestration rule.

2. **Triage the findings** before asking the user:
   - **High-priority** = any P0 or P1.
   - **Deferrable** = P2 / P3 and all "Adjacent observations".

3. **Surface via `AskUserQuestion`:**

   **Case A — high-priority findings present:**

   ```
   Slice <NN> review: <N high-priority, M deferrable>

   HIGH PRIORITY:
   [list P0/P1 with file:line + suggested fix]

   DEFERRABLE:
   [list P2/P3 + adjacent observations]

   A) Fix high-priority in-place (re-dispatch executor), defer the rest to a
      follow-up TODO PR, then push this slice
   B) Push as-is — everything becomes review comments on the PR
   C) Skip push — I'll look manually
   ```

   **Case B — no high-priority findings (deferrable only OR clean):**

   Don't block. Auto-pick path: push this slice now, roll the deferrable findings
   into the follow-up TODO pile. Surface as informational:

   ```
   Slice <NN> review: clean on high-priority (M deferrable rolled to follow-up).
   Pushing.
   ```

4. **Record** each slice's review outcome (clean / fixed / pushed-with-findings / deferred-only) in `overview.md`'s PR table (the `Review` column).

5. **Collect deferrable findings** into `$PLAN_DIR/followups.md` as they accrue — one section per slice, each entry with file:line, severity, source (review finding / adjacent observation), and suggested action. This file seeds the follow-up PR in Step 9.

### 6d. Push + open PR

For each completed chunk:

```bash
git push -u origin <branch>
gh pr create --base <base> --head <branch> --title "<title>" --body-file <spec path>
```

Record PR number back into `overview.md`.

### 6e. Rebase cascade

When the user merges a dependency (out-of-band), for each dependent PR:

```bash
git fetch origin <merged-base> --quiet
git checkout <dependent-branch>
git rebase origin/<merged-base>
git push --force-with-lease
gh pr edit <num> --base <merged-base>
```

Force-with-lease only — never plain force-push.

---

## Step 7 — End-of-chain adversarial pass (gates the final PR)

Per-slice review is narrow by design. Before the **final implementation PR** (Step 8) is opened, run one wide pass against the cumulative diff to catch issues that only show up when all slices compose.

1. Compute cumulative diff: `git diff origin/$BASE..<tip-of-chain>`.
2. Dispatch via `Agent` tool — pick one available:
   - **`codex:rescue`** subagent (if the Codex plugin is installed) — adversarial challenge mode.
   - Otherwise the `security-audit` skill against the composed diff.
   - Otherwise a fresh `general-purpose` agent with the adversarial prompt from the orchestration rule.
3. Same scope rule as per-slice review — findings about pre-existing surrounding code go to `followups.md`, not fixes.
4. Any P0/P1 → pause, `AskUserQuestion` whether to fix in a targeted new slice before the final PR, or ship and defer.

Skip if the chain is trivial (migration PR + one slice + final, or fewer).

---

## Step 8 — Final implementation PR

Last code PR. Depends on every preceding slice being merged. Usually contains:
- Feature flag flip / final wiring
- `docs/plans/active/<folder>` archival move (per `.alice/rules/post-feature-retro.md`)
- Wiki updates

Dispatched identically to the other slices but only after all deps are merged and Step 7 is clean (or explicitly deferred).

---

## Step 9 — Follow-up TODO PR (from deferred review findings)

If `$PLAN_DIR/followups.md` has any entries, open a dedicated follow-up PR so the deferred findings don't silently vanish. This PR contains **no code fixes** — it's an inbox, user triages later.

1. **Branch:** `slice/<slug>-98-review-followups`. Base: `origin/$BASE`. No dependencies — can land anytime (in parallel with the final PR is fine).
2. **Contents:**
   - Append the findings to `docs/todos/overview.md` under a new `## Review follow-ups — <slug>` section. One checklist entry per finding: `- [ ] <severity> <file:line> — <description> — <suggested fix or "investigate"> — source: slice <NN> review`.
   - If an active plan folder exists for this work, also copy `$PLAN_DIR/followups.md` to `docs/plans/active/<active-plan>/review-followups.md`.
3. **Commit:** `chore(review): deferred findings from pr-slicer <slug>`.
4. **PR body:** full `followups.md` content + per-slice breakdown table. Apply a `triage` label if the repo uses labels.
5. **Scope:** docs only. No code. Reviewer opens individual fix PRs as they decide.

If `followups.md` is empty: skip entirely.

---

## Gotchas

- **Uncommitted source branch changes** → stop, ask user to commit/stash. Slicer works off committed history only.
- **Migration files must land first** — other slices rebase on them. If the adopter's migration procedure involves regeneration (e.g. drizzle's `db:generate`, prisma's `migrate`), follow the adopter-specific procedure documented in their CLAUDE.md.
- **Never** force-push a migration branch to reset — if regeneration is needed, open a fresh branch rather than rewriting history on the in-review one.
- **Remote side-effects** (`git push`, `gh pr create`, `gh pr edit`, merge) are main-session only. Executors never touch them — the executor agent's `tools:` frontmatter should not include `gh` access.
- **Worktree cleanup:** `git worktree remove` only after the branch is pushed and the slice is landed (or explicitly abandoned).
- **Orchestration rule is binding.** If you can't poll a background agent at ≤1/min, dispatch foreground instead. Do not fire-and-forget.

---
name: plan-eng-review
preamble-tier: 3
version: 1.1.0
description: |
  Eng manager-mode plan review. Lock in the execution plan — architecture,
  data flow, diagrams, edge cases, test coverage, performance. Walks through
  issues interactively with opinionated recommendations. Use when asked to
  "review the architecture", "engineering review", or "lock in the plan".
  Proactively suggest when the user has a plan or design doc and is about to
  start coding — to catch architecture issues before implementation.
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - AskUserQuestion
  - Bash
  - WebSearch
---

## Preamble (run first)

```bash
# Project-local state dir — all session data under .alice/mem/ (gitignored).
eval "$(.alice/bin/alice-slug 2>/dev/null || true)"
mkdir -p "${ROOT:-.}/.alice/mem"
echo "BRANCH: ${BRANCH:-unknown}"
```

# Plan Review Mode

## Overview

Engineering-manager-style plan review that challenges architecture, data flow, scope, edge cases, test strategy, operational risk, and unresolved decisions before implementation starts.

## When to Use

- Use when asked to review a plan, design, architecture, or implementation approach before coding.
- Use when the user has a spec or plan and wants tradeoffs, diagrams, risks, and opinionated recommendations.
- Use when catching wrong-direction work early is cheaper than reviewing a finished diff.

**When NOT to use:**

- Do not use for finished-code review; use `review`.
- Do not use for active debugging; use `investigate`.
- Do not start implementation while this review is still resolving architectural questions.

## Process

Review this plan thoroughly before making any code changes. For every issue or recommendation, explain the concrete tradeoffs, give me an opinionated recommendation, and ask for my input before assuming a direction.

### Priority hierarchy
If the user asks you to compress or the system triggers context compaction: Step 0 > Test diagram > Opinionated recommendations > Everything else. Never skip Step 0 or the test diagram. Do not preemptively warn about context limits -- the system handles compaction automatically.

### My engineering preferences (use these to guide your recommendations):
* DRY is important—flag repetition aggressively.
* Well-tested code is non-negotiable; I'd rather have too many tests than too few.
* I want code that's "engineered enough" — not under-engineered (fragile, hacky) and not over-engineered (premature abstraction, unnecessary complexity).
* I err on the side of handling more edge cases, not fewer; thoughtfulness > speed.
* Bias toward explicit over clever.
* Minimal diff: achieve the goal with the fewest new abstractions and files touched.

### Cognitive Patterns — How Great Eng Managers Think

These are not additional checklist items. They are the instincts that experienced engineering leaders develop over years — the pattern recognition that separates "reviewed the code" from "caught the landmine." Apply them throughout your review.

1. **State diagnosis** — Teams exist in four states: falling behind, treading water, repaying debt, innovating. Each demands a different intervention (Larson, An Elegant Puzzle).
2. **Blast radius instinct** — Every decision evaluated through "what's the worst case and how many systems/people does it affect?"
3. **Boring by default** — "Every company gets about three innovation tokens." Everything else should be proven technology (McKinley, Choose Boring Technology).
4. **Incremental over revolutionary** — Strangler fig, not big bang. Canary, not global rollout. Refactor, not rewrite (Fowler).
5. **Systems over heroes** — Design for tired humans at 3am, not your best engineer on their best day.
6. **Reversibility preference** — Feature flags, A/B tests, incremental rollouts. Make the cost of being wrong low.
7. **Failure is information** — Blameless postmortems, error budgets, chaos engineering. Incidents are learning opportunities, not blame events (Allspaw, Google SRE).
8. **Org structure IS architecture** — Conway's Law in practice. Design both intentionally (Skelton/Pais, Team Topologies).
9. **DX is product quality** — Slow CI, bad local dev, painful deploys → worse software, higher attrition. Developer experience is a leading indicator.
10. **Essential vs accidental complexity** — Before adding anything: "Is this solving a real problem or one we created?" (Brooks, No Silver Bullet).
11. **Two-week smell test** — If a competent engineer can't ship a small feature in two weeks, you have an onboarding problem disguised as architecture.
12. **Glue work awareness** — Recognize invisible coordination work. Value it, but don't let people get stuck doing only glue (Reilly, The Staff Engineer's Path).
13. **Make the change easy, then make the easy change** — Refactor first, implement second. Never structural + behavioral changes simultaneously (Beck).
14. **Own your code in production** — No wall between dev and ops. "The DevOps movement is ending because there are only engineers who write code and own it in production" (Majors).
15. **Error budgets over uptime targets** — SLO of 99.9% = 0.1% downtime *budget to spend on shipping*. Reliability is resource allocation (Google SRE).

When evaluating architecture, think "boring by default." When reviewing tests, think "systems over heroes." When assessing complexity, ask Brooks's question. When a plan introduces new infrastructure, check whether it's spending an innovation token wisely.

### Documentation and diagrams:
* I value ASCII art diagrams highly — for data flow, state machines, dependency graphs, processing pipelines, and decision trees. Use them liberally in plans and design docs.
* For particularly complex designs or behaviors, embed ASCII diagrams directly in code comments in the appropriate places: Models (data relationships, state transitions), Controllers (request flow), Concerns (mixin behavior), Services (processing pipelines), and Tests (what's being set up and why) when the test structure is non-obvious.
* **Diagram maintenance is part of the change.** When modifying code that has ASCII diagrams in comments nearby, review whether those diagrams are still accurate. Update them as part of the same commit. Stale diagrams are worse than no diagrams — they actively mislead. Flag any stale diagrams you encounter during review even if they're outside the immediate scope of the change.

### BEFORE YOU START:

### Design Doc Check
```bash
setopt +o nomatch 2>/dev/null || true  # zsh compat
SLUG=$(.alice/skills/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null | tr '/' '-' || echo 'no-branch')
DESIGN=$(ls -t .alice/mem/projects/$SLUG/*-$BRANCH-design-*.md 2>/dev/null | head -1)
[ -z "$DESIGN" ] && DESIGN=$(ls -t .alice/mem/projects/$SLUG/*-design-*.md 2>/dev/null | head -1)
[ -n "$DESIGN" ] && echo "Design doc found: $DESIGN" || echo "No design doc found"
```
If a design doc exists, read it. Use it as the source of truth for the problem statement, constraints, and chosen approach. If it has a `Supersedes:` field, note that this is a revised design — check the prior version for context on what changed and why.

### When no spec exists

If `/plan-eng-review` is invoked on a branch without a locked spec in `docs/plans/active/<slug>/spec.md`, tell the user: "No locked spec found for this branch. For non-trivial work, run `/plan` first — it produces the problem/goal/scope/acceptance/assumptions/verification-map artifact this review attacks. Want me to hand off to `/plan` now, or proceed with a standard review against the diff + commit messages?"

Wait for the answer. If they run `/plan`, pick up here after it's locked. Otherwise proceed with standard review — the rigor bars below still apply; they just apply against commit messages and PR description instead of a spec.

### Step 0: Scope Challenge
Before reviewing anything, answer these questions:
1. **What existing code already partially or fully solves each sub-problem?** Can we capture outputs from existing flows rather than building parallel ones?
2. **What is the minimum set of changes that achieves the stated goal?** Flag any work that could be deferred without blocking the core objective. Be ruthless about scope creep.
3. **Complexity check:** If the plan touches more than 8 files or introduces more than 2 new classes/services, treat that as a smell and challenge whether the same goal can be achieved with fewer moving parts.
4. **Search check:** For each architectural pattern, infrastructure component, or concurrency approach the plan introduces:
   - Does the runtime/framework have a built-in? Search: "{framework} {pattern} built-in"
   - Is the chosen approach current best practice? Search: "{pattern} best practice {current year}"
   - Are there known footguns? Search: "{framework} {pattern} pitfalls"

   If WebSearch is unavailable, skip this check and note: "Search unavailable — proceeding with in-distribution knowledge only."

   If the plan rolls a custom solution where a built-in exists, flag it as a scope reduction opportunity. Annotate recommendations with the search-before-building layers — **[Layer 1]** runtime/framework built-in, **[Layer 2]** existing project dependency, **[Layer 3]** new dependency (custom code is the last resort) — or **[EUREKA]** when the search reveals the standard approach is wrong for this case. Present a eureka as an architectural insight.
5. **TODOS cross-reference:** Read `docs/todos/overview.md` if it exists. Are any deferred items blocking this plan? Can any deferred items be bundled into this PR without expanding scope? Does this plan create new work that should be captured as a TODO?

6. **Completeness check:** Is the plan doing the complete version or a shortcut? With agent-assisted coding, the cost of completeness (100% test coverage, full edge case handling, complete error paths) is 10-100x cheaper than with a human team. If the plan proposes a shortcut that saves human-hours but only saves the agent minutes, recommend the complete version.

7. **Distribution check:** If the plan introduces a new artifact type (CLI binary, library package, container image, mobile app), does it include the build/publish pipeline? Code without distribution is code nobody can use. Check:
   - Is there a CI/CD workflow for building and publishing the artifact?
   - Are target platforms defined (linux/darwin/windows, amd64/arm64)?
   - How will users download or install it (GitHub Releases, package manager, container registry)?
   If the plan defers distribution, flag it explicitly in the "NOT in scope" section — don't let it silently drop.

If the complexity check triggers (8+ files or 2+ new classes/services), proactively recommend scope reduction via AskUserQuestion — explain what's overbuilt, propose a minimal version that achieves the core goal, and ask whether to reduce or proceed as-is. If the complexity check does not trigger, present your Step 0 findings and proceed directly to Section 1.

Always work through the full interactive review: one section at a time (Architecture → Code Quality → Tests → Performance) with at most 8 top issues per section.

**Critical: Once the user accepts or rejects a scope reduction recommendation, commit fully.** Do not re-argue for smaller scope during later review sections. Do not silently reduce scope or skip planned components.

### Review Sections (after scope is agreed)

### Confidence Calibration

Read `.alice/references/confidence-calibration.md` and apply it to every finding: each finding carries a 1-10 confidence score, uses the `[SEVERITY] (confidence: N/10) file:line — description` format, and is displayed/suppressed per that table. Log calibration events per the reference.

### 1. Architecture review

Evaluate:
* Overall shape of the change — components added/touched, how they connect, where the data flows.
* Fit with the existing system design — does the plan reuse existing primitives or rebuild them? (Feeds the "What already exists" output below.)
* Dependency direction and coupling — new cross-module dependencies, cycles, seams introduced without a second implementation.
* Error handling and failure strategy at the architecture level — where errors surface, what degrades, what retries.
* Migration/rollout approach — reversibility, feature flags, incremental over big-bang (apply the Cognitive Patterns above).

**STOP.** For each issue found in this section, call AskUserQuestion individually. One issue per call. Present options, state your recommendation, explain WHY. Do NOT batch multiple issues into one AskUserQuestion. Only proceed to the next section after ALL issues in this section are resolved.

### 2. Code quality review
Evaluate:
* Code organization and module structure.
* DRY violations—be aggressive here.
* Error handling patterns and missing edge cases (call these out explicitly).
* Technical debt hotspots.
* Areas that are over-engineered or under-engineered relative to my preferences.
* Existing ASCII diagrams in touched files — are they still accurate after this change?

**STOP.** For each issue found in this section, call AskUserQuestion individually. One issue per call. Present options, state your recommendation, explain WHY. Do NOT batch multiple issues into one AskUserQuestion. Only proceed to the next section after ALL issues in this section are resolved.

### 3. Test review

100% coverage is the goal. Evaluate every codepath in the plan and ensure the plan includes tests for each one. If the plan is missing tests, add them — the plan should be complete enough that implementation includes full test coverage from the start.

**Run the shared coverage audit.** Read `.alice/references/coverage-audit.md` and follow it end-to-end against the plan document:

1. Detect the test framework (per the reference; skip test generation if none).
2. Trace every codepath the plan implies — for each planned component, follow the planned execution and diagram every branch and error path.
3. Map user flows, interaction edge cases, error states, and boundary states.
4. Check each branch against existing tests, scoring quality with the ★ rubric, and apply the E2E/eval decision matrix.
5. Output the ASCII coverage diagram (code paths + user flows, [→E2E]/[→EVAL] marks, COVERAGE/QUALITY/GAPS footer).

**REGRESSION RULE (mandatory).** The reference's iron rule applies in `/plan-eng-review`'s planning form: when the audit identifies a regression risk, a regression test is added to the plan as a **CRITICAL** requirement. No AskUserQuestion. No skipping.

**Fast path:** All paths covered → "Test review: All new code paths have test coverage ✓" Continue.

**Add missing tests to the plan:**

For each GAP identified in the diagram, add a test requirement to the plan. Be specific:
- What test file to create (match existing naming conventions)
- What the test should assert (specific inputs → expected outputs/behavior)
- Whether it's a unit test, E2E test, or eval (use the decision matrix)
- For regressions: flag as **CRITICAL** and explain what broke

The plan should be complete enough that when implementation begins, every test is written alongside the feature code — not deferred to a follow-up.

### Test Plan Artifact

After producing the coverage diagram, write a test plan artifact to the project directory so `/qa` (pass `--report-only` for no-fix mode) can consume it as primary test input:

```bash
eval "$(.alice/bin/alice-slug 2>/dev/null)" && mkdir -p .alice/mem/projects/$SLUG
USER=$(whoami)
DATETIME=$(date +%Y%m%d-%H%M%S)
```

Write to `.alice/mem/projects/{slug}/{user}-{branch}-eng-review-test-plan-{datetime}.md`:

```markdown
# Test Plan
Generated by /plan-eng-review on {date}
Branch: {branch}
Repo: {owner/repo}

## Affected Pages/Routes
- {URL path} — {what to test and why}

## Key Interactions to Verify
- {interaction description} on {page}

## Edge Cases
- {edge case} on {page}

## Critical Paths
- {end-to-end flow that must work}
```

This file is consumed by `/qa` (pass `--report-only` for no-fix mode) as primary test input. Include only the information that helps a QA tester know **what to test and where** — not implementation details.

For LLM/prompt changes: check the "Prompt/LLM changes" file patterns listed in CLAUDE.md. If this plan touches ANY of those patterns, state which eval suites must be run, which cases should be added, and what baselines to compare against. Then use AskUserQuestion to confirm the eval scope with the user.

**STOP.** For each issue found in this section, call AskUserQuestion individually. One issue per call. Present options, state your recommendation, explain WHY. Do NOT batch multiple issues into one AskUserQuestion. Only proceed to the next section after ALL issues in this section are resolved.

### 4. Performance review
Evaluate:
* N+1 queries and database access patterns.
* Memory-usage concerns.
* Caching opportunities.
* Slow or high-complexity code paths.

**STOP.** For each issue found in this section, call AskUserQuestion individually. One issue per call. Present options, state your recommendation, explain WHY. Do NOT batch multiple issues into one AskUserQuestion. Only proceed to the next section after ALL issues in this section are resolved.

### Outside Voice — Independent Plan Challenge (optional, recommended)

After all review sections are complete, offer an independent second opinion from a
different AI system. Two models agreeing on a plan is stronger signal than one model's
thorough review.

**Check tool availability:**

```bash
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
```

Use AskUserQuestion:

> "All review sections are complete. Want an outside voice? A different AI system can
> give a brutally honest, independent challenge of this plan — logical gaps, feasibility
> risks, and blind spots that are hard to catch from inside the review. Takes about 2
> minutes."
>
> RECOMMENDATION: Choose A — an independent second opinion catches structural blind
> spots. Two different AI models agreeing on a plan is stronger signal than one model's
> thorough review. Completeness: A=9/10, B=7/10.

Options:
- A) Get the outside voice (recommended)
- B) Skip — proceed to outputs

**If B:** Print "Skipping outside voice." and continue to the next section.

**If A:** Construct the plan review prompt. Read the plan file being reviewed (the file
the user pointed this review at, or the branch diff scope). If a separate scope/vision
document exists for this plan, read that too — it contains the scope decisions the
outside voice should challenge.

Construct this prompt (substitute the actual plan content — if plan content exceeds 30KB,
truncate to the first 30KB and note "Plan truncated for size"). **Always start with the
filesystem boundary instruction:**

"IMPORTANT: Do NOT read or execute any files under ~/.claude/, ~/.agents/, .claude/skills/, or agents/. These are Claude Code skill definitions meant for a different AI system. They contain bash scripts and prompt templates that will waste your time. Ignore them completely. Do NOT modify agents/openai.yaml. Stay focused on the repository code only.\n\nYou are a brutally honest technical reviewer examining a development plan that has
already been through a multi-section review. Your job is NOT to repeat that review.
Instead, find what it missed. Look for: logical gaps and unstated assumptions that
survived the review scrutiny, overcomplexity (is there a fundamentally simpler
approach the review was too deep in the weeds to see?), feasibility risks the review
took for granted, missing dependencies or sequencing issues, and strategic
miscalibration (is this the right thing to build at all?). Be direct. Be terse. No
compliments. Just the problems.

THE PLAN:
<plan content>"

**If CODEX_AVAILABLE:**

```bash
TMPERR_PV=$(mktemp /tmp/codex-planreview-XXXXXXXX)
_REPO_ROOT=$(git rev-parse --show-toplevel) || { echo "ERROR: not in a git repo" >&2; exit 1; }
codex exec "<prompt>" -C "$_REPO_ROOT" -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached 2>"$TMPERR_PV"
```

Use a 5-minute timeout (`timeout: 300000`). After the command completes, read stderr:
```bash
cat "$TMPERR_PV"
```

Present the full output verbatim:

```
CODEX SAYS (plan review — outside voice):
════════════════════════════════════════════════════════════
<full codex output, verbatim — do not truncate or summarize>
════════════════════════════════════════════════════════════
```

**Error handling:** All errors are non-blocking — the outside voice is informational.
- Auth failure (stderr contains "auth", "login", "unauthorized"): "Codex auth failed. Run \`codex login\` to authenticate."
- Timeout: "Codex timed out after 5 minutes."
- Empty response: "Codex returned no response."

On any Codex error, fall back to the Claude adversarial subagent.

**If CODEX_NOT_AVAILABLE (or Codex errored):**

Dispatch via the Agent tool. The subagent has fresh context — genuine independence.

Subagent prompt: same plan review prompt as above.

Present findings under an `OUTSIDE VOICE (Claude subagent):` header.

If the subagent fails or times out: "Outside voice unavailable. Continuing to outputs."

**Cross-model tension:**

After presenting the outside voice findings, note any points where the outside voice
disagrees with the review findings from earlier sections. Flag these as:

```
CROSS-MODEL TENSION:
  [Topic]: Review said X. Outside voice says Y. [Present both perspectives neutrally.
  State what context you might be missing that would change the answer.]
```

**User Sovereignty:** Do NOT auto-incorporate outside voice recommendations into the plan.
Present each tension point to the user. The user decides. Cross-model agreement is a
strong signal — present it as such — but it is NOT permission to act. You may state
which argument you find more compelling, but you MUST NOT apply the change without
explicit user approval.

For each substantive tension point, use AskUserQuestion:

> "Cross-model disagreement on [topic]. The review found [X] but the outside voice
> argues [Y]. [One sentence on what context you might be missing.]"

Options:
- A) Accept the outside voice's recommendation (I'll apply this change)
- B) Keep the current approach (reject the outside voice)
- C) Investigate further before deciding
- D) Add to docs/todos/overview.md for later

Wait for the user's response. Do NOT default to accepting because you agree with the
outside voice. If the user chooses B, the current approach stands — do not re-argue.

If no tension points exist, note: "No cross-model tension — both reviewers agree."

**Persist the result:**
```bash
.alice/bin/alice-review-log '{"skill":"plan-eng-review-outside-voice","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","commit":"'"$(git rev-parse --short HEAD)"'"}'
```

Substitute: STATUS = "clean" if no findings, "issues_found" if findings exist.
SOURCE = "codex" if Codex ran, "claude" if subagent ran.

**Cleanup:** Run `rm -f "$TMPERR_PV"` after processing (if Codex was used).

---

### Outside Voice Integration Rule

Outside voice findings are INFORMATIONAL until the user explicitly approves each one.
Do NOT incorporate outside voice recommendations into the plan without presenting each
finding via AskUserQuestion and getting explicit approval. This applies even when you
agree with the outside voice. Cross-model consensus is a strong signal — present it as
such — but the user makes the decision.

### CRITICAL RULE — How to ask questions
Rules for AskUserQuestion in plan reviews:
* **One issue = one AskUserQuestion call.** Never combine multiple issues into one question.
* Describe the problem concretely, with file and line references.
* Present 2-3 options, including "do nothing" where that's reasonable.
* For each option, specify in one line: effort (human: ~X / agent: ~Y), risk, and maintenance burden. If the complete option is only marginally more agent effort than the shortcut, recommend the complete option.
* **Map the reasoning to my engineering preferences above.** One sentence connecting your recommendation to a specific preference (DRY, explicit > clever, minimal diff, etc.).
* Label with issue NUMBER + option LETTER (e.g., "3A", "3B").
* **Escape hatch:** If a section has no issues, say so and move on. If an issue has an obvious fix with no real alternatives, state what you'll do and move on — don't waste a question on it. Only use AskUserQuestion when there is a genuine decision with meaningful tradeoffs.

### Required outputs

### "NOT in scope" section
Every plan review MUST produce a "NOT in scope" section listing work that was considered and explicitly deferred, with a one-line rationale for each item.

### "What already exists" section
List existing code/flows that already partially solve sub-problems in this plan, and whether the plan reuses them or unnecessarily rebuilds them.

### docs/todos/overview.md updates
After all review sections are complete, present each potential TODO as its own individual AskUserQuestion. Never batch TODOs — one per question. Never silently skip this step.

For each TODO, describe to the user:
* **What:** One-line description of the work.
* **Why:** The concrete problem it solves or value it unlocks.
* **Pros:** What you gain by doing this work.
* **Cons:** Cost, complexity, or risks of doing it.
* **Context:** Enough detail that someone picking this up in 3 months understands the motivation, the current state, and where to start.
* **Depends on / blocked by:** Any prerequisites or ordering constraints.

Then present options: **A)** Add to backlog  **B)** Skip — not valuable enough  **C)** Build it now in this PR instead of deferring.

When the user picks A, write **two files**:

1. `docs/todos/<slug>.md` — copy `.claude/templates/todo.md` and fill all fields (What, Why, Context, Acceptance hint, Priority, Effort, Status, Depends on). The discussion above maps directly onto these fields.
2. `docs/todos/overview.md` — append a one-liner under `## Backlog`:
   `- **[Px]** [<title>](<slug>.md) — one-line description. Effort S/M/L.`

The detail file holds the durable context; the overview holds the pointer. Do NOT cram the long discussion into `overview.md` — that's exactly what the per-TODO file exists for.

A TODO without its detail file is worse than no TODO — it creates false confidence that the idea was captured while actually losing the reasoning.

### Diagrams
The plan itself should use ASCII diagrams for any non-trivial data flow, state machine, or processing pipeline. Additionally, identify which files in the implementation should get inline ASCII diagram comments — particularly Models with complex state transitions, Services with multi-step pipelines, and Concerns with non-obvious mixin behavior.

### Failure modes
For each new codepath identified in the test review diagram, list one realistic way it could fail in production (timeout, nil reference, race condition, stale data, etc.) and whether:
1. A test covers that failure
2. Error handling exists for it
3. The user would see a clear error or a silent failure

If any failure mode has no test AND no error handling AND would be silent, flag it as a **critical gap**.

### Worktree parallelization strategy

Analyze the plan's implementation steps for parallel execution opportunities. This helps the user split work across git worktrees (via Claude Code's Agent tool with `isolation: "worktree"` or parallel workspaces).

**Skip if:** all steps touch the same primary module, or the plan has fewer than 2 independent workstreams. In that case, write: "Sequential implementation, no parallelization opportunity."

**Otherwise, produce:**

1. **Dependency table** — for each implementation step/workstream:

| Step | Modules touched | Depends on |
|------|----------------|------------|
| (step name) | (directories/modules, NOT specific files) | (other steps, or —) |

Work at the module/directory level, not file level. Plans describe intent ("add API endpoints"), not specific files. Module-level ("controllers/, models/") is reliable; file-level is guesswork.

2. **Parallel lanes** — group steps into lanes:
   - Steps with no shared modules and no dependency go in separate lanes (parallel)
   - Steps sharing a module directory go in the same lane (sequential)
   - Steps depending on other steps go in later lanes

Format: `Lane A: step1 → step2 (sequential, shared models/)` / `Lane B: step3 (independent)`

3. **Execution order** — which lanes launch in parallel, which wait. Example: "Launch A + B in parallel worktrees. Merge both. Then C."

4. **Conflict flags** — if two parallel lanes touch the same module directory, flag it: "Lanes X and Y both touch module/ — potential merge conflict. Consider sequential execution or careful coordination."

### Completion summary
At the end of the review, fill in and display this summary so the user can see all findings at a glance:
- Step 0: Scope Challenge — ___ (scope accepted as-is / scope reduced per recommendation)
- Architecture Review: ___ issues found
- Code Quality Review: ___ issues found
- Test Review: diagram produced, ___ gaps identified
- Performance Review: ___ issues found
- NOT in scope: written
- What already exists: written
- docs/todos/overview.md updates: ___ items proposed to user
- Failure modes: ___ critical gaps flagged
- Outside voice: ran (codex/claude) / skipped
- Parallelization: ___ lanes, ___ parallel / ___ sequential
- Completeness score: X/Y recommendations chose the complete option

### Retrospective learning
Check the git log for this branch. If there are prior commits suggesting a previous review cycle (e.g., review-driven refactors, reverted changes), note what was changed and whether the current plan touches the same areas. Be more aggressive reviewing areas that were previously problematic.

### Formatting rules
* NUMBER issues (1, 2, 3...) and LETTERS for options (A, B, C...).
* Label with NUMBER + LETTER (e.g., "3A", "3B").
* One sentence max per option. Pick in under 5 seconds.
* After each review section, pause and ask for feedback before moving on.

### Review Log

After producing the Completion Summary above, persist the review result.

**PLAN MODE EXCEPTION — ALWAYS RUN:** This command writes review metadata to
`<project-root>/.alice/mem/` (gitignored project-local state, not project files).
The review readiness dashboard depends on this data — skipping this command
breaks the dashboard.

```bash
.alice/bin/alice-review-log '{"skill":"plan-eng-review","timestamp":"TIMESTAMP","status":"STATUS","unresolved":N,"critical_gaps":N,"issues_found":N,"mode":"MODE","commit":"COMMIT"}'
```

Substitute values from the Completion Summary:
- **TIMESTAMP**: current ISO 8601 datetime
- **STATUS**: "clean" if 0 unresolved decisions AND 0 critical gaps; otherwise "issues_open"
- **unresolved**: number from "Unresolved decisions" count
- **critical_gaps**: number from "Failure modes: ___ critical gaps flagged"
- **issues_found**: total issues found across all review sections (Architecture + Code Quality + Performance + Test gaps)
- **MODE**: FULL_REVIEW / SCOPE_REDUCED
- **COMMIT**: output of `git rev-parse --short HEAD`

### Review Readiness Dashboard

After completing the review, read the review log to display the dashboard.

```bash
.alice/bin/alice-review-read
```

Parse the output. Find the most recent entry within the last 7 days for each alice review skill:

- `plan-eng-review` — plan-stage architecture review (this skill).
- `review` — diff-scoped pre-landing review (`/review` on a branch).
- `plan-eng-review-outside-voice` — optional second opinion from a different AI model, dispatched from within this skill's Outside Voice section.

Display:

```
+====================================================================+
|                    REVIEW READINESS DASHBOARD                       |
+====================================================================+
| Review          | Runs | Last Run            | Status    | Gates?   |
|-----------------|------|---------------------|-----------|----------|
| Plan Review     |  1   | 2026-04-19 15:00    | CLEAR     | YES      |
| Diff Review     |  0   | —                   | —         | YES      |
| Outside Voice   |  0   | —                   | —         | no       |
+--------------------------------------------------------------------+
| VERDICT: CLEARED for implementation — Plan Review passed            |
+====================================================================+
```

**Verdict logic:**
- **CLEARED for implementation**: `plan-eng-review` has ≥1 entry within 7 days with status `clean`.
- **NOT CLEARED**: Plan Review missing, stale (>7 days), or has open issues.
- **CLEARED for merge**: requires both Plan Review (if spec exists) AND Diff Review (`/review`) to be clean on the current HEAD.
- Outside Voice is shown for context but never gates.

**Staleness detection:** After displaying the dashboard, check if reviews may be stale against the current HEAD:
- Parse the `---HEAD---` section from the bash output to get the current HEAD short-SHA.
- For each entry with a `commit` field: if it differs from HEAD, count elapsed commits with `git rev-list --count STORED_COMMIT..HEAD` and print: `Note: {skill} from {date} may be stale — {N} commits since review`.
- Entries without a `commit` field: print `Note: {skill} from {date} predates commit tracking — consider re-running`.
- If all match HEAD, print nothing.

### Plan File Review Report

After the dashboard, also write the review status into the **plan file** so it's visible to anyone reading the plan.

### Detect the plan file

1. Check if there's an active plan file in the conversation (host provides plan file paths in system messages).
2. If not found, skip this section silently — not every review runs in plan mode.

### Generate the report

Parse each JSONL entry from the review log. Alice's reviews log these fields:

- **plan-eng-review**: `status`, `unresolved`, `critical_gaps`, `issues_found`, `mode`, `commit` → Findings: `{issues_found} issues, {critical_gaps} critical gaps`
- **review**: `status`, `issues_found`, `critical`, `informational`, `commit` → Findings: `{issues_found} issues ({critical} critical)`
- **plan-eng-review-outside-voice**: `status`, `source`, `commit` → Findings: `source: {source}`

For the review you just completed, use richer details from your own Completion Summary. For prior reviews in the log, use the JSONL fields directly.

Produce this markdown block:

```markdown
## REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| Plan Review    | `/plan-eng-review`          | Architecture & test coverage (gates implementation) | {runs} | {status} | {findings} |
| Diff Review    | `/review`                   | Pre-landing diff review (gates merge)               | {runs} | {status} | {findings} |
| Outside Voice  | optional, inside this skill | Independent 2nd opinion from a different AI model   | {runs} | {status} | {findings} |
```

Below the table, add these lines (omit any that are empty/not applicable):

- **UNRESOLVED:** total unresolved decisions across all reviews.
- **VERDICT:** `CLEARED for implementation` if Plan Review is clean; add `CLEARED for merge` if Diff Review is also clean on HEAD. Otherwise name what's blocking.

### Write to the plan file

**PLAN MODE EXCEPTION — ALWAYS RUN:** This writes to the plan file, which is the one file you're allowed to edit in plan mode. The plan file review report is part of the plan's living status.

- Search the plan file for a `## REVIEW REPORT` section anywhere in the file.
- If found, **replace it** entirely using Edit (match from `## REVIEW REPORT` through either the next `## ` heading or end of file, whichever comes first — this preserves content added after the report).
- If absent, **append it** to the end of the plan file.
- Always place the review report as the last section. If it was mid-file, delete the old location and append at the end.

### Next Steps

After the dashboard:

- If the review is **CLEARED for implementation**, tell the user they're ready to start coding against the locked spec. Remind them to run `/review` before opening the PR.
- If the review is **NOT CLEARED**, summarize the blocking issues in one line each and wait for the user to resolve them.
- If a Diff Review entry already exists for the current HEAD and is clean, state `CLEARED for merge` and stop.

### Unresolved decisions
If the user does not respond to an AskUserQuestion or interrupts to move on, note which decisions were left unresolved. At the end of the review, list these as "Unresolved decisions that may bite you later" — never silently default to an option.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The spec is short, skip plan-eng-review and go straight to code." | Short specs hide implicit assumptions exactly because they're short. The review is what surfaces them while course-correction is cheap. |
| "I'll silently default the unresolved decisions to whatever is easiest." | Silent defaults become bugs in production that read like surprises. Always surface unresolved decisions in the review report. |
| "The dashboard is a chore — the user already knows the verdict." | The dashboard exists for the next agent reading the plan, not the current user. Without it, the plan loses its review state. |
| "Findings from a single reviewer are enough; cross-model is overkill." | Adversarial fan-out at the appropriate tier exists because single-reviewer blind spots are real. Trust the tier scaling — don't downgrade by hand. |
| "The plan file already mentions this concern in a comment — no need to flag it." | If the concern lives in a freeform comment rather than a structured decision, the reviewer can't grade it. Flag every concern; let the comment-vs-decision distinction be the reviewer's call, not the author's. |

## Red Flags

- Verdict line absent or ambiguous ("looks fine" instead of CLEARED / NOT CLEARED).
- Unresolved decisions silently defaulted instead of surfaced.
- Review report written somewhere other than the end of the plan file (split or duplicated reports).
- Diff Review entry pre-dating the current HEAD without re-running for the new commits.
- A spec marked `locked` with unresolved decisions in the review report.
- The review skipped a section (Scope Challenge / Architecture / Code Quality / Tests / Performance) on a non-trivial spec.

## Verification

A plan-eng-review is DONE when:

- [ ] Each section (Step 0 Scope Challenge / Architecture / Code Quality / Tests / Performance / Outside Voice) ran or was deliberately skipped per its rules.
- [ ] Findings have severity and a concrete fix or follow-up.
- [ ] Unresolved decisions are listed explicitly — none silently defaulted.
- [ ] The `## REVIEW REPORT` section is written as the last section of the plan file (replacing any prior).
- [ ] Verdict line is unambiguous: CLEARED for implementation / NOT CLEARED + blocker / CLEARED for merge.
- [ ] If a Diff Review entry exists for the current HEAD, it was reused (not re-run) and folded into the verdict.

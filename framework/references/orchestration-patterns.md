# Orchestration patterns

Reference catalog of agent orchestration patterns alice endorses, plus anti-patterns to avoid. Read this before adding a new slash command that coordinates multiple personas, or before introducing a new persona that "wraps" existing ones.

Pairs with `framework/rules/sub-agent-orchestration.md`: that rule is the **policy** (polling cadence, permission protocol, escalation contract); this file is the **catalog** (which shapes of orchestration are blessed, which are anti-patterns).

The governing rule: **the user (or a slash command) is the orchestrator. Personas do not invoke other personas.** Skills are mandatory hops inside a persona's workflow; personas are leaves.

---

## Endorsed patterns

### 1. Direct invocation (no orchestration)

Single persona, single perspective, single artifact. The default and the cheapest option.

```
user → code-reviewer → report → user
```

**Use when:** the work is one perspective on one artifact and you can describe it in one sentence.

**Examples:**
- "Review this PR" → `code-reviewer`.
- "Find silent failures in `auth.ts`" → `silent-failure-hunter`.
- "Audit this for OWASP issues" → `security-reviewer`.

**Cost:** one round trip. The baseline you should always compare orchestrated patterns against.

---

### 2. Single-persona slash command

A slash command that wraps one persona with alice's skills. Saves the user from re-explaining the workflow every time.

```
/review → code-reviewer (with review skill) → report
```

**Use when:** the same single-persona invocation happens repeatedly with the same setup.

**Examples in alice:** `/review`, `/qa`, `/investigate`.

**Cost:** same as direct invocation. The slash command is just a saved prompt.

**Anti-signal:** if the slash command's body is mostly "decide which persona to call," delete it and let the user call the persona directly.

---

### 3. Parallel fan-out with merge

Multiple personas operate on the same input concurrently, each producing an independent report. A merge step (in the main agent's context) synthesizes them into a single decision.

```
                  ┌─→ code-reviewer       ─┐
fan out  ─────────┼─→ security-reviewer   ─┤→ merge → verdict
                  └─→ silent-failure-hunter ┘
```

**Use when:**

- The sub-tasks are genuinely independent (no shared mutable state, no ordering dependency).
- Each sub-agent benefits from its own context window.
- The merge step is small enough to stay in the main context.
- Wall-clock latency matters.

**Examples in alice:** `pr-slicer` (parallel slice executors), `hugh` (parallel diana fan-out), `/diana` review pass.

**Cost:** N parallel sub-agent contexts + one merge turn. Higher than direct invocation, but faster wall-clock and produces better reports because each sub-agent stays focused on its single perspective.

**Validation checklist before adopting this pattern:**

- [ ] Can I run all sub-agents at the same time without ordering issues?
- [ ] Does each persona produce a different *kind* of finding, not just the same finding from a different angle?
- [ ] Will the merge step fit in the main agent's remaining context?
- [ ] Is the user's wait time long enough that parallelism is actually noticeable?
- [ ] Does the orchestrator poll each background sub-agent per `framework/rules/sub-agent-orchestration.md`?

If any answer is "no," fall back to direct invocation or a single-persona command.

---

### 4. Sequential pipeline as user-driven slash commands

The user runs slash commands in a defined order, carrying context (or commit history) between them. There is no orchestrator agent — the user IS the orchestrator.

```
user runs:  /plan  →  build  →  /review  →  …
```

**Use when:** the workflow has dependencies (each step needs the previous step's output) and human judgment between steps adds value.

**Examples in alice:** the spec-then-build-then-review loop. `/plan` lands a spec; the user implements (or hands to an executor); `/review` gates the merge. Alice deliberately does NOT wrap this in a meta-orchestrator — see Anti-pattern C.

**Cost:** one sub-agent context per step. Free for the orchestration layer because there is no orchestrator agent.

**Why not automate it:** an LLM "lifecycle orchestrator" would (a) lose nuance between steps because it has to summarize for hand-off, (b) skip the human checkpoints that catch wrong-direction work early, and (c) double the token cost via paraphrasing turns.

---

### 5. Research isolation (context preservation)

When a task requires reading large amounts of material that shouldn't pollute the main context, spawn a research sub-agent that returns only a digest.

```
main agent → research sub-agent (reads many files / URLs) → digest → main agent continues
```

**Use when:**

- The main session needs to stay focused on a downstream task.
- The investigation result is much smaller than the input it consumes.
- The decision quality benefits from the main agent having room to think after.

**Examples in alice:** the `research` skill, and any harness-built-in equivalent (e.g. Claude Code's `Explore` sub-agent).

**Cost:** one isolated sub-agent context. Worth it any time the alternative is loading hundreds of files into the main context.

**When the harness ships a built-in research sub-agent, use it.** Don't redefine a custom research persona to do what the built-in already does.

---

## Anti-patterns

### A. Router persona ("meta-orchestrator")

A persona whose job is to decide which other persona to call.

```
/work → router-persona → "this needs a review" → code-reviewer → router (paraphrases) → user
```

**Why it fails:**

- Pure routing layer with no domain value.
- Adds two paraphrasing hops → information loss + roughly 2× token cost.
- The user already knew they wanted a review; they could have called `/review` directly.
- Replicates the work that slash commands and intent mapping in the adopter's `CLAUDE.md` already do.

**What to do instead:** add or refine slash commands. Document intent → command mapping in the adopter's `CLAUDE.md`.

---

### B. Persona that calls another persona

A `code-reviewer` that internally invokes `security-reviewer` when it sees auth code.

**Why it fails:**

- Personas were designed to produce a single perspective; chaining them defeats that.
- The summary the calling persona passes loses context the called persona needs.
- Failure modes multiply (which persona's output format wins? whose rules apply?).
- Hides cost from the user.
- Most harnesses physically block this (no nested sub-agents), so the persona either errors out or silently degrades.

**What to do instead:** have the calling persona *recommend* a follow-up audit in its report. The user or a slash command runs the second pass.

This is the same constraint enforced by `framework/skills/doubt-driven-development/SKILL.md`'s "Loading Constraints" section.

---

### C. Sequential orchestrator that paraphrases

An agent that calls `/plan`, then `/review`, etc. on the user's behalf.

**Why it fails:**

- Loses the human checkpoints that catch wrong-direction work.
- Each hand-off summarizes context — accumulated drift over a long pipeline.
- Doubles token cost: orchestrator turn + sub-agent turn for every step.
- Removes user agency at exactly the points where judgment matters most.

**What to do instead:** keep the user as the orchestrator. Document the recommended sequence in the adopter's `CLAUDE.md` and let users invoke it.

---

### D. Deep persona trees

A slash command that calls a "coordinator" persona that calls a "quality" persona that calls `code-reviewer`.

**Why it fails:**

- Each layer adds latency and tokens with no decision value.
- Debugging becomes a multi-level investigation.
- The leaf personas lose context to multiple summarization steps.

**What to do instead:** keep the orchestration depth at most 1 (slash command → personas). The merge happens in the main agent.

---

## Decision flow

When considering a new orchestrated workflow, walk this flow:

```
Is the work one perspective on one artifact?
├── Yes → Direct invocation. Stop.
└── No  → Will the same composition repeat?
         ├── No  → Direct invocation, ad hoc. Stop.
         └── Yes → Are sub-tasks independent?
                  ├── No  → Sequential slash commands run by user (Pattern 4).
                  └── Yes → Parallel fan-out with merge (Pattern 3).
                           Validate against the checklist above.
                           If any check fails → fall back to single-persona command (Pattern 2).
```

---

## Adversarial / teammate-coordination pattern (harness-dependent)

Some harnesses support **teammate coordination** where sub-agents can message each other directly and share a task list (e.g. Claude Code "Agent Teams"). This is a distinct shape from Pattern 3:

| | Pattern 3: fan-out + merge | Teammate coordination |
|--|---------------------------|----------------------|
| Sub-agents see | The same input, different lenses | A shared task list + each other's messages |
| Output | Independent reports → merge | Adversarial debate → converged conclusion |
| Right when | You want a verdict on a known artifact | You want to *find* the artifact among competing hypotheses |

**Use teammate coordination only when teammates need to challenge each other to produce the right answer** — for example, a debugging investigation with multiple competing root-cause theories where the surviving theory should be the one no teammate could disprove.

Most alice workflows are verdict-on-known-artifact and belong in Pattern 3. Reach for teammate coordination as an exception, not a default.

If the harness doesn't support teammate coordination, simulate it manually: spawn each persona, collect outputs, paste each into the others' next turn as "the other reviewers said …", and let them respond. It's slower than a real teammate API but reaches the same shape of conclusion.

---

## When to add a new pattern to this catalog

Add a new entry only after:

1. You've used the pattern at least twice in real work.
2. You can name a concrete artifact in alice (or an adopter repo) that demonstrates it.
3. You can explain why an existing pattern wouldn't have worked.
4. You can describe its anti-pattern shadow (what people will mistakenly build instead).

Premature catalog entries become aspirational documentation that no one follows.

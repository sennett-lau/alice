# Rule: sub-agent orchestration

**Binding.** Any skill that dispatches sub-agents (via the `Agent` tool, `subagent_type`, or equivalent) must follow this rule. Skills like `pr-slicer` that fan out parallel executors depend on it; skills like `/review` that spawn a single fresh reviewer benefit from it.

## Why

Sub-agents run with their own context and tool budget. Two failure modes show up often enough to be non-negotiable concerns:

1. **Stuck agents.** An executor can loop on a hung build, retry a failing command indefinitely, or wait forever on a silent subprocess. Without active polling, the main session only learns about the stall when the agent eventually times out — hours later.
2. **Permission surprises.** Sub-agents inherit the main session's tool permissions by default, but users sometimes tighten sub-agent scope (narrow `tools:` frontmatter, no `gh`, no network). A restricted sub-agent that hits a denied tool either silently falls back into something worse or gets blocked in a way the main session can't see.

Both failures are invisible to the user unless the main session is watching.

## Progress monitoring

For every sub-agent dispatched in the background (`run_in_background: true`, or any API where the main session continues before the agent finishes), the orchestrating skill must:

1. **Record a baseline** at dispatch — store `{agent_id, last_output_bytes: 0, last_observed_at: now()}` for each in-flight executor.
2. **Poll at least once per minute.** Use `TaskOutput` (or the runtime's equivalent streaming/tail interface) to fetch the agent's current stdout. Compute byte count; compare to baseline.
3. **Reset on progress.** If byte count increased, update baseline to current bytes + timestamp. The agent is doing something.
4. **Escalate on silence.** If byte count is unchanged across three consecutive polls (≥3 minutes of zero output), flag the agent as potentially stuck.
5. **Surface flagged agents via `AskUserQuestion`.** Offer: `A) keep waiting  B) fetch full logs and inspect  C) abort this agent and continue without it  D) abort the whole run`. Never silently kill an agent — always surface the choice.

Foreground (blocking) `Agent` calls don't need polling — the main session is already waiting on them. But if a blocking call is expected to take more than ~5 minutes, prefer `run_in_background` so other work can proceed.

**Don't use the runtime's built-in wakeup/loop mechanism for this.** The polling cadence is short (60s) and the main session is already active — use `TaskOutput` + a tight check loop in the skill's own flow, not a scheduled re-entry.

## Permission policy

### Default — inherit

Sub-agents inherit the parent session's tool permissions. This is usually what you want: the user already approved the permission once; re-prompting for every sub-agent is noise.

### Tightened — restricted `tools:` frontmatter

If the agent file declares a narrow `tools:` list (e.g. `[Bash, Read, Edit, Write, Grep, Glob]` — no `Agent`, no `AskUserQuestion`, no network tools), the sub-agent literally cannot invoke anything outside that list. This is useful for:

- Executors that should never push to remote (`tools:` omits the `gh`-allowed variant of Bash)
- Reviewers that should never edit files (`tools:` includes `Read, Grep, Glob`, omits `Edit, Write`)
- Agents that should not spawn further sub-agents (omit `Agent`)

### Handling a blocked sub-agent

When a sub-agent hits a tool outside its permission scope:

1. **Sub-agent MUST stop and report.** The agent's system prompt (or the skill's dispatch prompt) tells it explicitly: on a permission block, stop, write a single-line report `BLOCKED: <tool> needed for <reason>`, and return. Never work around by shelling out to a different tool. Never silently skip.
2. **Main session handles the escalation** on handback:
   - Read the `BLOCKED:` line from the agent's report
   - Present to user via `AskUserQuestion`:
     - `A) Grant the permission — I'll re-dispatch the sub-agent with the tool added to its scope`
     - `B) Run the blocked action myself in the main session, then resume the sub-agent with the result pre-fed` (main session proxies)
     - `C) Re-scope the work so the sub-agent doesn't need that tool`
     - `D) Abort this sub-agent`
3. **Main session never grants permissions the user has not confirmed.** Even if the tool is in the main session's scope, adding it to a sub-agent's scope requires explicit approval.
4. **Sub-agent never retries a denied tool with a different wording.** If `gh pr create` was denied, don't try `gh api repos/.../pulls`. That defeats the purpose of tightening scope.

### Signals the main session watches for

Common phrases in a sub-agent handback that indicate a permission block the agent may have worded loosely:

- `"permission denied"`, `"not authorized"`, `"tool not available"`, `"InputValidationError"`
- `"I don't have access to ..."`, `"I can't run ..."`, `"skipped because ..."`

Any of these in the handback → treat as a `BLOCKED:` signal even if the agent didn't use the exact protocol word. Escalate to the user.

## Model tier selection

Sub-agents don't all need the same compute budget. A mechanical, low-stakes task (drafting a commit message, syncing a wiki page, executing a pre-approved file move) wastes money and time on a top-tier model; a high-stakes judgment call (security review, adversarial verification, evaluating whether a fix actually resolved an issue) is where a weaker model's mistakes are most expensive. Route each dispatch to one of three tiers based on the task's **stakes and ambiguity — not its length**:

- **light** — mechanical, low-risk, cheaply-checked output.
- **standard** — everyday reasoning. The default; most agents belong here.
- **heavy** — high-stakes judgment: security-sensitive review, adversarial verification, architecture/design tradeoffs, ambiguous synthesis. Wrong output here is expensive — it ships a vuln, misses a real bug, or wastes a review cycle.

Pick the tier when the agent is defined (frontmatter, for a reusable persona) or at dispatch time (call-site `model` param, for one-off `general-purpose` spawns — e.g. the adversarial `general-purpose` dispatches in `diana` and `pr-slicer`).

Each tier bundles two knobs, not one: **which model** runs (the axis above) and **how hard it reasons** — a separate dial most runtimes expose (Claude Code: `effort:` frontmatter, `low`/`medium`/`high`/`xhigh`/`max`/`auto`; Codex: `model_reasoning_effort`, `minimal`→`xhigh`). Picking a tier should set both automatically so callers don't have to reason about two axes for every dispatch:

- **light** → cheap model, `low` effort.
- **standard** → no override on either knob — inherit the session's model and its default reasoning depth.
- **heavy** → strong model, `high` effort, escalate to `xhigh` for the single hardest-stakes agent when a skill has more than one heavy-tier dispatch to compare (e.g. a security-focused pass vs. a general one).

Alice clamps effort to **`low`–`xhigh`** and doesn't use the ends of either runtime's native range: not Codex's `minimal` (no equivalent floor on Claude Code, so the shared scale can't rely on it), and not Claude Code's `max` (unbounded token spend, session-scoped only per Claude Code's own docs — it doesn't persist in frontmatter the way `low`/`medium`/`high`/`xhigh` do, so it's not a stable thing to pin on a reusable agent). This is a deliberate, alice-specific choice for portability across runtimes — not a hard ceiling either runtime imposes.

This is a different knob from diana/hugh's `--effort=low|medium|high|max` flag even though some level names overlap — that flag picks *workflow depth* (which SOP steps run), this one picks *reasoning depth* (how hard a given model thinks on a given step). Don't conflate them; a diana run at `--effort=low` can still dispatch a heavy-tier, `xhigh`-reasoning security review for the one step that needs it.

**Never hardcode a dated/versioned model snapshot.** Use the runtime's live alias or tier knob instead — e.g. Claude Code's bare `haiku`/`sonnet`/`opus`/`fable` aliases resolve to whatever snapshot is current, so the mapping keeps working as vendors ship new models under the same name. Concrete per-runtime mappings live in `framework/references/model-tiers.md` — that file is expected to go stale as vendors rename things; update it, not this rule.

### Never re-tier by switching the main session's own model

If a step wants a cheaper or stronger tier than what the main session is currently running, **dispatch a sub-agent already pinned to that tier** — don't flip the main session's own `model`/`effort` setting to chase it. Changing the running session's model mid-task risks context-window/prompt-cache inconsistency and can trigger an unwanted compaction; a freshly-dispatched sub-agent gets its own clean context at the right tier with none of that risk.

This is why a mechanical step like drafting a commit message is a good light-tier sub-agent candidate — it's read-mostly, cheaply verified, and disposable if the dispatch is imperfect. **This does not extend to `git push` or other remote side-effects.** Those stay in the main session regardless of tier, per the permission policy above ("executors that should never push to remote") and `pr-slicer-executor`'s existing contract ("Does NOT push... main session owns remote side-effects") — that carve-out exists for human-supervision/trust reasons, not compute cost, and tiering doesn't override it.

## How to apply

Skills that dispatch sub-agents reference this rule in their orchestration section rather than re-documenting it. The skill's job is to implement the polling + escalation mechanics; this rule defines the policy the mechanics must satisfy.

If a skill cannot satisfy the polling requirement for a specific agent call (e.g. the runtime doesn't expose `TaskOutput` for that agent type), the skill must dispatch that call in the foreground and block until it returns — never fire-and-forget.

## Pattern catalog

This rule is the policy floor — what every sub-agent dispatch must satisfy. **Which orchestration shape to use** (direct invocation, parallel fan-out, sequential pipeline, research isolation) is a separate question with its own catalog: `framework/references/orchestration-patterns.md`. Read both together when designing a new multi-agent workflow:

- **This rule** = mechanics every dispatch obeys (polling, permissions, escalation).
- **The reference catalog** = endorsed shapes (Patterns 1–5) and anti-patterns (A–D) for *how* dispatches compose.

The two are independent. A new skill must satisfy the rule AND map to one of the endorsed patterns; if it can't do both, the design is wrong.

# Model tiers

Concrete per-runtime mapping for the three model tiers defined in `framework/rules/sub-agent-orchestration.md`. That rule is the **policy** (which tier a task belongs in, and that a tier bundles both a model choice and a reasoning-effort level); this file is the **catalog** (what knob to turn, per runtime, to get that tier) — same split as `orchestration-patterns.md` pairs with that rule for dispatch shapes.

This file goes stale faster than the rule does: model names, aliases, and reasoning-effort scales change as vendors ship new lineups. When they do, update the table below — don't touch the rule, and don't hardcode a dated snapshot here either. Every cell should be a live alias or a named level, never a pinned version string.

Two independent knobs, bundled per tier for convenience:

- **Model** — which model runs.
- **Effort** — how hard that model reasons. Alice clamps this to `low` / `medium` / `high` / `xhigh` on every runtime, even where the runtime's native range goes wider (Codex also has `minimal`; Claude Code also has `max` and `auto`) — see the rule for why.

## Claude Code

**Model** — agent frontmatter `model:` (reusable persona in `framework/agents/*.md`) or a per-call `model` param on the dispatch tool (one-off `general-purpose` spawns).

**Effort** — agent frontmatter `effort:` (native Claude Code key, persists across sessions for `low`/`medium`/`high`/`xhigh`) or a per-call effort override where the dispatch mechanism exposes one.

| Tier | Model | Effort | Notes |
|---|---|---|---|
| light | `haiku` | `low` | |
| standard | *(omit)* | *(omit)* | inherits the calling session's model and its default reasoning depth — don't pin either knob |
| heavy | `opus` | `high` (escalate to `xhigh` for the single hardest-stakes dispatch when a skill runs more than one heavy-tier agent) | `fable` is an adopter-swappable model alternate for writing/synthesis-heavy judgment calls (e.g. wiki quality passes, copy review) rather than pure code-security reasoning |

Bare model aliases (`haiku`, `sonnet`, `opus`, `fable`) resolve to the current snapshot under that name — that's what makes them safe to hardcode in frontmatter. A literal snapshot ID (`claude-opus-4-8`) is not; avoid it here. Don't use `effort: max` on a reusable agent — it's uncapped token spend and only applies for the current session (via the `/effort` menu or `CLAUDE_CODE_EFFORT_LEVEL`), not something that reliably persists on a frontmatter-defined persona.

## Codex CLI

**Model + effort** — `model_reasoning_effort`, either in `config.toml` (`[profiles.<name>]` block) or inline with `-c 'model_reasoning_effort="<level>"'` on `codex exec`. Codex doesn't switch model *name* per tier the way Claude Code does — same model, different reasoning-effort budget.

| Tier | Effort | Notes |
|---|---|---|
| light | `low` | not `minimal` — alice keeps the floor at `low` so the scale lines up with Claude Code, which has no level below it |
| standard | `medium` | |
| heavy | `high` (or `xhigh` for the hardest adversarial passes, where the model supports it) | alice's existing Codex fallback calls in `review` and `plan-eng-review` already hardcode `"high"` — that's correct, they're heavy-tier tasks (adversarial/design review) |

## Any other runtime

Every agentic coding tool that supports sub-agents or model selection exposes *some* live knob for "cheaper/faster" vs. "more capable," and increasingly a separate reasoning-effort dial too. Map this runtime's three tiers to whichever knobs it has, clamped to the same `low`–`xhigh` band: never pin a dated snapshot, always use the runtime's current-by-name alias or a named effort level.

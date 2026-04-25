# {{PROJECT_NAME}} — Agent Operating Manual

This file is the top-level briefing for any agent (Claude Code, Codex, etc.) working in this repo. Read it first every session. Deep content lives in `docs/wiki/**` — this file is the index and the load policy.

> **Bootstrapped from [alice](https://github.com/{{ALICE_REMOTE_OR_LOCAL_PATH}})** — a generic agentic docs/plans/ledger framework. The framework payload lives at `.alice/`; agent-specific config dirs (`.claude/`, future `.codex/`, `.agents/`) symlink into it so the framework is shared across agents. Project-specific stack/domain rules live in this file and `docs/wiki/`.

## What {{PROJECT_NAME}} is

<2–4 sentences. What this project does, who uses it, what makes it distinct. Drop project-specific terminology that an outsider needs to know.>

## Repo layout

```
{{PROJECT_LAYOUT_HERE}}
docs/                 project operating manual (wiki + plans + ledger)
  README.md             layout + load policy
  todos/                live backlog (overview.md auto-loaded; per-TODO detail files load on demand)
  wiki/                 stable knowledge, auto-loaded
  plans/active/         in-flight features
  plans/archive/        frozen post-ship (query-only)
  ledger/               decisions + experiences (query-only)
.alice/                 vendored alice framework (do not edit by hand — update from upstream)
  rules/                binding rules
  templates/            spec / decision / implementation / overview starters
  commands/             slash commands (e.g. /plan)
  skills/               skill source (qa, browse, review, plan-eng-review, investigate, research, ...)
  agents/               sub-agent source (code-reviewer, security-reviewer, silent-failure-hunter, refactor-cleaner, seo-specialist, wiki-maintainer)
  bin/                  alice-* helper scripts
.claude/                Claude Code config — thin shim of symlinks into .alice/
  _alice    -> ../.alice
  rules     -> ../.alice/rules
  templates -> ../.alice/templates
  commands  -> ../.alice/commands
  skills/<name> -> ../../.alice/skills/<name>
  agents/<name> -> ../../.alice/agents/<name>
```

## Stack

<List languages, frameworks, key services for THIS repo — read package.json / pyproject.toml / wrangler.toml / next.config.* / etc. and write what's actually there. Examples below — replace with your reality, drop rows that don't apply, add rows that do.>

- **Language:** TypeScript (or Python, Go, Rust, etc.)
- **Backend:** <Cloudflare Workers + Hono | Fastify | Express | FastAPI | Rails | ...>
- **Frontend:** <Next.js SSR | Next.js static export | Remix | SvelteKit | Astro | ...>
- **DB:** <Postgres | D1 | KV | Mongo | ...>
- **Test:** <vitest | jest | pytest | go test | ...>
- **Deploy:** <Cloudflare | Vercel | Fly | ECS | Render | ...>

Commands:

```bash
{{REPLACE_WITH_PROJECT_COMMANDS}}
```

Deep architecture: `docs/wiki/architecture.md`. Domain model: `docs/wiki/domain-model.md`.

## Load policy (critical)

| Path | Policy | When to use |
|------|--------|-------------|
| `CLAUDE.md` (this file) | auto-load | every session |
| `docs/README.md` | auto-load | every session (layout + load policy detail) |
| `docs/wiki/**` | auto-load | every session |
| `docs/todos/overview.md` | auto-load | every session |
| `docs/todos/<slug>.md` | load on demand | when working that specific TODO |
| `docs/plans/active/<current>/overview.md` | auto-load | while feature in flight |
| `docs/plans/active/<current>/{spec,decision,implementation}.md` | load on demand | during feature work |
| `docs/plans/archive/**` | query-only | prior art, retro, reconstruction |
| `docs/ledger/decisions.md` | query-only | before big architectural calls |
| `docs/ledger/experiences.md` | query-only | before repeating a pattern that previously burned |
| `.claude/rules/**` | load on demand | when about to do the thing the rule governs |
| `.claude/templates/**` | load on demand | when creating a plan folder or ledger entry |
| `.claude/agents/**` | load on demand | when a sub-agent is invoked (see Agent routing) |

Rule of thumb: auto-loaded set is small and current. Query-only set grows unbounded — pull in only when the question needs the history.

## How to work (agent SOP)

Pipeline for every non-trivial task. Each step has a skill or command — use them, don't freelance.

> **Trivial fast-path.** Typo-only, doc-only, or obvious one-file fixes skip `/plan`. Still: state the intended change, make the surgical edit, run the smallest relevant verification. When in doubt, `/plan`.

1. **Orient.** Read `docs/wiki/current-status.md` and `docs/todos/overview.md`. Check if there's an active plan folder in `docs/plans/active/` already covering this task.
2. **Plan the work — pick one entry point:**
   - **New feature:** run `/plan` (`.claude/commands/plan.md`). It asks for problem/goal/scope/acceptance interactively, sweeps ledger + archive for prior art, scaffolds `docs/plans/active/YYYY-MM-DD_<slug>/` from `.claude/templates/`, and wires the entry into `docs/todos/overview.md` (and a per-TODO detail file at `docs/todos/<slug>.md` from `.claude/templates/todo.md`).
   - **Known issue / bug:** run `/investigate`. Drive to a root-cause hypothesis, then scaffold a plan folder via `/plan <slug>` (issue-investigation variant — see `.claude/commands/plan.md`) so the fix lives on the same rails as a feature.
3. **Review the plan.** Run `/plan-eng-review` on the draft spec. Lock in architecture, error handling, logging, test coverage, performance. Iterate until the user signs off. Set `spec.md` `Status: locked`.
4. **Implement.** Only after sign-off. Follow `.claude/rules/implementation-quality.md`. Keep `implementation.md` up to date as you go. Grep ledger + archive before introducing new primitives.
5. **Verify.** Build green, tests pass. UI work → `/qa` (or `browse` for targeted checks).
6. **Review the diff.** Run `/review` on the change against base. Prefer running it via the Agent tool as a fresh sub-agent so the reviewing context isn't polluted by the implementation context.
7. **Document.** Same PR: update wiki pages that drifted, per `.claude/rules/documentation-updates.md`.
8. **Ship.** PR + merge + deploy.
9. **Confirm done + retro.** Only after deploy health is green. Run the full `.claude/rules/post-feature-retro.md` checklist: archive move + wiki update + experiences append + decisions append + TODO strike. Until all five land, the feature isn't "done".

## Rules pointer

Every rule in `.claude/rules/` is **binding**. Load on demand when the rule's domain applies.

- `docs-layout-and-load-policy.md` — the docs/ layout and auto-load/query-only policy.
- `post-feature-retro.md` — the five-action checklist that fires on every ship.
- `documentation-updates.md` — docs must land in the same PR as behavior changes.
- `feature-spec-required.md` — spec before non-trivial code.
- `implementation-quality.md` — the floor: build green, reuse first, small PRs, no silent failures, boundaries validate.
- `sub-agent-orchestration.md` — any skill that dispatches sub-agents must poll them (≥1/min) for progress and escalate permission blocks to the user rather than silently working around.

## Working style

- **Ambiguity before action.** If a request has multiple plausible meanings, list the interpretations and ask one focused question. Don't silently pick the easiest to implement.
- **Surgical diffs.** Every changed line traces to the request, the spec, or cleanup made necessary by this change. No drive-by refactors, reformats, or adjacent "improvements."
- **State the plan, then verify each step.** For multi-step work, list the steps with a verify check per step *before* starting. "Done" means the verify check passed, not "I wrote the code."
- **Push back on scope.** If a simpler path meets the goal, say so before creating a plan or writing code. Bias toward caution over speed.
- **Plan before code.** Spec → sign-off → code. Non-trivial work gets a plan folder.
- **Reuse before invention.** Grep ledger + archive + wiki before writing a new primitive.
- **Small changes.** Prefer 5 small PRs to 1 big one. Big PRs hide regressions.
- **Docs honest.** Wiki says what exists now. Ripped features get their wiki page ripped with them.
- **Build green.** Red CI blocks merge. Fix CI first.
- **Ask when it's irreversible.** Deploys, destructive ops, force-pushes — confirm first even if allowed.

## Critical gotchas (project-specific — fill in as you find them)

These are the non-obvious invariants that have burned this codebase. Keep them inline, don't hide them in the wiki. Add one each time something surprising bites you.

- <gotcha 1 — e.g. "Cloudflare Workers have no Node `fs` — bundler must be `esbuild` not `node-builtin`">
- <gotcha 2 — e.g. "Drizzle migrations require `db:generate` before `db:push` or schema drifts">
- <gotcha 3>

## Migration-class files

Files that are hard to revert once deployed, or that cause rebase pain when multiple PRs touch adjacent content. The `/pr-slicer` skill reads this list and forces matching files into a dedicated migration PR that lands first — other slices rebase on top cleanly.

The idea: if a later slice goes wrong, you can revert it without rolling back the infra / schema change. Parallel PRs don't fight over sensitive files. The migration PR gets focused review (often the highest-risk change in the stack).

**Fill in patterns that apply to this repo.** Delete the example rows that don't apply. Leave the section present (even empty) so `/pr-slicer` doesn't re-prompt every run.

- `<pattern>` — <why it's migration-class>
- <e.g. `packages/backend/drizzle/**/*.sql` — DB migrations, order-sensitive, not safely revertible after deploy>
- <e.g. `wrangler.toml` — Cloudflare Worker config, conflict-prone across PRs>
- <e.g. `terraform/**/*.tf` — infra state, apply-only-forward>
- <e.g. `.env.example` — env schema, consumers rebase on additions>

### Collapse / regeneration procedure

If any of the above involve generation (DDL produced by an ORM, config compiled from source, lock files regenerated from a manifest), document the canonical procedure the `/pr-slicer` migration spec should follow. Example:

> After merging schema changes from the source branch, run `<tool-specific regen command>` to produce a single clean migration file. Verify `<sequence / format check>`. If broken, delete + re-regen — never hand-edit generated migrations.

If no generation step applies, write `None — migration files are authored directly.`

## Skill routing

When the user's request matches a skill, invoke it via the Skill tool **before** other actions.

The project ships a small, project-local set of skills under `.alice/skills/` (symlinked into `.claude/skills/`). Everything is project-scoped — skills write to `<project-root>/.tmp/`, never to `~/.claude/`.

| Request shape | Skill |
|---------------|-------|
| New feature — draft spec with user + scaffold plan folder | `/plan` (`.claude/commands/plan.md`) |
| Bugs, errors, "why is this broken", stack traces | `investigate` (wrap the fix in a plan folder via `/plan` issue-variant when non-trivial) |
| QA / test the UI / dogfood a flow / screenshot evidence | `qa` (use `--report-only` for no-fix mode) |
| Browser control (raw primitives, not a full QA sweep) | `browse` |
| Import real-browser cookies for authed QA | `setup-browser-cookies` |
| Pre-landing PR / diff review | `review` |
| Pre-implementation architecture review | `plan-eng-review` |
| Security audit — secrets, dependencies, CI/CD, OWASP, LLM trust | `security-audit` |
| Multi-source research with citations — web synthesis, competitive / market / tech scan | `research` |
| Pull the latest alice framework into `.alice/` (sync skills, commands, agents, migrations) | `/sync` (`.alice/commands/sync.md`) |
| Slice a large branch / PR into a chain of smaller reviewable PRs with a migration PR first, parallel-safe siblings, and a per-PR review gate | `/pr-slicer` |
| Run the full alice SOP end-to-end for a given feature description with little / no human interaction (chains `/plan` → `/plan-eng-review` → implement → `/review` → `/pr-slicer` → `/security-audit` → retro + doc update, gated by effort tier) | `/diana` |

**Project-local state.** Any skill that needs scratch space writes to `<project-root>/.tmp/` (gitignored, per-checkout). Never `~/.claude/`, never user-home.

## Agent routing

The framework also ships sub-agents under `.alice/agents/` (symlinked into `.claude/agents/`). Two invocation modes:

1. **Auto-invoke during work.** The main agent calls a sub-agent when its trigger condition appears — no explicit command needed.
2. **Delegated from inside a skill.** When a skill's job overlaps with a sub-agent, the skill must invoke the sub-agent via the `Agent` tool — don't inline the work. This protects the skill's context and lets passes run in parallel.

| Sub-agent | Auto-invoke trigger | Skill that delegates to it |
|-----------|--------------------|-----------------------------|
| `code-reviewer` | after any non-trivial code edit — quick QA pass before the user sees the diff | `/review` (fans out for lang/domain passes) |
| `security-reviewer` | code touching auth, user input, secrets, API endpoints, sensitive data | `/security-audit` |
| `silent-failure-hunter` | suspicion of swallowed errors, bad fallbacks, or missing error propagation | on-demand only |
| `refactor-cleaner` | suspected dead code, unused deps, duplication — not during active feature work | on-demand only |
| `seo-specialist` | web-facing pages / marketing sites — meta tags, structured data, Core Web Vitals, sitemap / robots, content mapping | on-demand only |
| `wiki-maintainer` | post-feature retro (wiki update step), initial wiki seed during alice bootstrap, on-demand drift lint | `post-feature-retro` rule (retro mode); bootstrap step 6 (seed mode) |
| `pr-slicer-executor` | never auto-invoked | `/pr-slicer` (one executor per slice; does the code work in its worktree; never pushes) |

**Skill-delegates-to-agent rule.** If a skill has a matching sub-agent, the skill must run the agent as a fresh sub-agent via the `Agent` tool. Inlining the work in the main skill loop pollutes context and blocks parallelism — use the sub-agent.

## Definition of done

### Pre-PR

- [ ] Build green locally, tests pass.
- [ ] For non-trivial change: plan folder exists, spec locked.
- [ ] Wiki pages updated for any behavior / schema / contract change.
- [ ] No stale `// removed`, unused `_vars`, or speculative scaffolding.
- [ ] Critical gotchas respected.

### Post-ship

- [ ] Plan folder archived: `git mv docs/plans/active/<folder> docs/plans/archive/<folder>`.
- [ ] `docs/wiki/current-status.md` updated (in flight → shipped).
- [ ] `docs/ledger/experiences.md` retro entry appended.
- [ ] `docs/ledger/decisions.md` entry appended for any non-obvious choice.
- [ ] `docs/todos/overview.md` struck (in-flight → Done recent), per-TODO detail file `docs/todos/<slug>.md` deleted.

## When to update this file

- Repo layout changes (new package, moved directories).
- New binding rule added to `.claude/rules/`.
- Stack change (new framework, dropped dependency, new DB).
- New branching or deploy convention.
- Load policy change (new path added to auto-load / query-only).
- A critical gotcha emerges from a burn. Log it here so future-agent doesn't repeat it.

Everything else belongs in `docs/wiki/**` or the ledger, not here.

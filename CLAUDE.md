# alice — maintainer's briefing

You are working **on alice itself**, not on a repo that adopts alice. Read this before touching any file.

Full tour in [`README.md`](README.md). Adoption recipe in [`bootstrap/README.md`](bootstrap/README.md). The file you land in most often is [`template/CLAUDE.md`](template/CLAUDE.md) — that's the briefing **adopting repos receive**, not this one.

## What alice is

A generic agentic docs/plans/ledger framework: rules, templates, a `/plan` command, a starter skill set (`qa`, `browse`, `review`, `plan-eng-review`, `investigate`, `setup-browser-cookies`, `security-audit`), and a docs scaffold (`wiki/` + `plans/active,archive/` + `ledger/` + `todos/`). Stack- and domain-agnostic on purpose — adopter agents fill in stack-specific details at setup time, alice stays universal.

## Repo layout

```
alice/
  CLAUDE.md                this file — maintainer briefing
  README.md                public-facing tour + agent quickstart
  bootstrap/README.md      step-by-step adoption recipe
  framework/               ships to adopter's .alice/
    rules/                 5 binding rules (ship to adopter at .alice/rules/)
    templates/             overview / spec / decision / implementation / todo
    commands/plan.md       /plan interactive spec builder
    skills/<name>/         skill sources — ship to adopter at .alice/skills/<name>/
    agents/<name>.md       sub-agent sources — ship to adopter at .alice/agents/<name>.md
    bin/                   alice-slug / alice-diff-scope / alice-review-{log,read} / chrome-cdp
  template/                ships to adopter's repo root
    CLAUDE.md              template that becomes the adopter's CLAUDE.md
    docs/                  docs/ scaffold — copied to adopter's docs/
```

## Mental model: alice is source, adopters are the runtime

Almost every file here ships **verbatim** into adopting repos via the bootstrap recipe. That means:

- **Path strings in skill SKILL.md files use adopter-side paths.** `.alice/bin/alice-slug`, `.alice/skills/browse/dist/browse`, `.tmp/...` — these are valid in the adopting repo (where `.alice/` is the vendored framework). They are NOT paths relative to this repo. Don't "fix" them.
- **`template/CLAUDE.md` is frozen shape for the adopter.** Edits here flow to every future bootstrap. If you want to change the adopter's CLAUDE.md skeleton, edit `template/CLAUDE.md`. If you want to change *this* file (alice maintainer briefing), edit `CLAUDE.md` at the repo root.
- **`template/docs/` is the adopter's starting `docs/`.** `template/docs/todos/overview.md`, `template/docs/wiki/*.md`, `template/docs/ledger/*.md` — all copied into adopter at bootstrap time. Changes here define the initial shape every new project starts from.
- **Alice itself has no `docs/`, no `.tmp/`, no `.alice/`.** Alice ships them; it doesn't run them on itself.

## Binding maintenance principles

1. **Stay generic.** If a rule, template, or skill starts naming a framework (Next.js, Rails, Fastify), a stack (Cloudflare, AWS), or a domain (blockchain, payments, auth-provider), push it back into the adopter's `CLAUDE.md` or `docs/wiki/`. Alice is the universal floor.
2. **No brand coupling.** No gstack, no cmux-as-requirement, no ns-mm, no tool-chain lock-in beyond what's intrinsic to the skill (e.g. browse uses bun because `bun build --compile` is load-bearing — that's intrinsic, not brand coupling).
3. **Template changes cascade.** Edits to `template/**`, `framework/rules/**`, `framework/templates/**`, or `framework/commands/plan.md` affect every future adopter. Treat them like public API.
4. **Skills and agents are Claude-Code-shaped.** Skill frontmatter (`allowed-tools`, `description`, hook semantics) and agent frontmatter (`tools`, `description`) assume Claude Code as the runtime. Rules, templates, commands, and docs are agent-agnostic — any agent that can read markdown can use them via `.alice/rules/`, `.alice/templates/`, etc. This is why the bootstrap produces `.alice/` at the repo root (shared) plus a `.claude/` shim (Claude-specific). Preserve the split.
5. **Lazy install over eager install.** The `browse` binary builds on first `/browse` invocation via `framework/skills/browse/setup`. Don't add mandatory pre-build steps to the bootstrap recipe — the skills already self-heal.
6. **Non-destructive bootstrap.** `bootstrap/README.md` tells the driving agent to never overwrite existing `CLAUDE.md`, `docs/`, or `.alice/` in the target repo — surface conflicts and propose migrations instead. If you add new bootstrap steps, preserve that invariant.

## What NOT to add

Things that were considered and deliberately dropped:

- **Stack profiles.** Would either bloat alice or go stale. The agent reads the target repo and writes stack gotchas directly into the adopter's `CLAUDE.md` at setup time.
- **Deploy / land / ship skills.** Every project's deploy story is different. Adopter owns it in their `CLAUDE.md`.
- **Design / product-ideation skills.** Taste-driven and not universal. Future skills (research, brainstorm, etc.) may earn a place — but only after they prove themselves across multiple adopting projects.
- **Telemetry, analytics, update checks, user-home state.** Alice writes state only to the adopter's `.tmp/` (project-local, gitignored). Nothing in `~/.claude/`, nothing phone-home.
- **A `setup.sh` installer.** The quickstart prompt in `README.md` asks the driving agent to read `bootstrap/README.md` and execute the steps directly. A script would duplicate work and add a maintenance surface.

If a new proposal smells like one of these, push back. If it's genuinely a universal addition, argue the case.

## Releasing alice

Alice's own version lives in `VERSION` at the repo root (single line, semver, no leading `v`). The `/release` command at `.claude/commands/release.md` runs the full flow: sync `development`, bump `VERSION` with a commit, merge to `main`, tag with a description built from the commit log since the last tag, push both refs.

`VERSION` and `.claude/commands/release.md` are **alice-local** — they do not ship to adopters via the bootstrap recipe. Adopters own their own release/deploy story (see "What NOT to add"). Do not move either into `framework/` or `template/`.

## Provenance

Distilled from an internal project's agent operating manual on 2026-04-20. The five binding rules, templates, `/plan` command, and skill suite were ported with all project-specific content stripped out. State paths and bin scripts were rebranded to `.alice/` / `alice-*`. If you see ghosts of the original project in commit history (chain-specific rules, stack profiles, gstack wording, root-level `TODOS.md`), re-read why they were removed before bringing anything back.

The `/research` skill and the sub-agents under `framework/agents/` were ported in from external sources (see `README.md` "References" for credits). Stack-specific content (named frameworks, hard tool pins, cost-tier advice, versioned addenda) was stripped so they match alice's generic floor. Per-skill `ACKNOWLEDGEMENTS.md` files are not used — credits live in `README.md` "References" only. Do not reintroduce either stack specifics or per-skill acknowledgement files.

## When to update this file

- New maintenance principle that isn't obvious from the file tree.
- A recurring category of "don't add that" decisions (bump "What NOT to add").
- A change to the split between `framework/` (source) and adopter runtime (path conventions, state dirs).

Everything else belongs in `README.md` (public) or `bootstrap/README.md` (adopter recipe), not here.

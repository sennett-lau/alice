# Alice

An **Agentic Development Framework with simplifications** for builders.

Alice vendors a small operating layer into your codebase so coding agents can
work with stronger rails: living project memory, spec-first implementation,
review discipline, browser QA, user-testing diagnosis, and iterative resolution
loops. It is built for software work inside real repos. Alice stays stack- and
domain-agnostic; the adopting agent fills in project-specific details from the
actual codebase during setup.

## Installation

Open Claude Code (or any capable coding agent) inside the repo you want to wire up and paste this block. The agent reads the rest of the README + bootstrap guide itself.

```
Adopt the alice agentic framework into THIS repo.

Clone https://github.com/sennett-lau/alice into a throwaway temp directory,
read its README and bootstrap guide end to end, then follow the adoption flow
they describe — vendor alice into this repo (do not leave a dependency on the
temp clone), fill the scaffolded CLAUDE.md, and seed docs/wiki/* from what's
actually in the repo. Never overwrite an existing CLAUDE.md or docs/ — surface
a migration plan instead. Delete the temp clone when done and report back what
you did.
```

## Start here

After Alice is adopted into a repo, most work starts from one of these paths.

**Build Loop**

- `/plan` — turn non-trivial work into a spec-backed plan folder.
- `/plan-eng-review` — challenge a drafted plan before implementation starts.
- `/review` — review a branch before landing.

**Validation**

- `/qa` — browser-test a feature or flow and capture evidence.
- `/browse` — direct browser control for targeted checks.
- `/diagnosis` — run parallel user-testing validators and promote findings into `docs/todos/findings/`.
- `/security-audit` — focused security review.

**Automation Wrappers**

- `/diana` — run the full single-feature coding SOP: plan, review, implement, review, optional audit, retro, docs.
- `/hugh` — split multiple independent features and run one isolated `/diana` per feature in parallel worktrees.
- `/ouroboros` — resolve the findings backlog through a diagnose → resolve → evaluate → merge loop.

**Utilities**

- `/investigate` — chase bugs, regressions, stack traces, and broken behavior to root cause.
- `/research` — source-grounded research with citations.
- `/pr-slicer` — split large branches into reviewable PR slices.
- `/setup-browser-cookies` — import real-browser auth state for browser testing.

## What alice gives you

A cohesive **agentic operating system** for any codebase:

- **Wiki** — the auto-loaded "what exists today" knowledge base.
- **Plans** — per-feature folders with spec → review → implement → ship lifecycle.
- **Ledger** — append-only decisions + post-feature retros + bug patterns.
- **Rules** — seven binding rules covering docs layout, doc updates, spec-required, implementation quality, test discipline, post-feature retro, sub-agent orchestration.
- **Templates** — overview / spec / decision / implementation starters.
- **Skills** — project-local workflows for planning, review, QA, diagnosis, research, security, parallel implementation, and iterative improvement. Each writes state to `<project-root>/.alice/mem/` (gitignored, per-checkout). Project-scoped, never reaches into `~/.claude/`. Every skill follows a shared authoring contract — see `framework/skills/README.md`.
- **References** — harness-agnostic reference catalogs adopters and skills can link to. Currently: `orchestration-patterns.md` (5 endorsed multi-agent shapes + 4 anti-patterns; pairs with the `sub-agent-orchestration` rule).
- **Upgrade path** — `/sync` pulls the latest alice into the adopter's `.alice/`. Classifies every changed file into four tiers (safe add / clean update / local conflict / structural migration), walks the user through each, and stamps `.alice/VERSION`. Never auto-commits. Full flow: `framework/commands/sync.md`; structural migrations documented under `framework/migrations/`.
- **Sub-agents** — focused roles for code review, security review, user-testing validation, findings triage, resolution evaluation, silent failure hunting, refactor cleanup, SEO, wiki maintenance, and PR slicing. Invoked automatically during the SOP, or delegated into by skills. Stack-agnostic (except `seo-specialist`, which self-gates to web-facing projects); see `template/CLAUDE.md` "Agent routing" for invocation rules.

## Why Alice

Agents are powerful but easy to let sprawl: ad hoc memory, inconsistent plans,
review passes that depend on mood, and parallel workers stepping on each other.
Alice gives them simple rails:

- **Project-local by default.** Framework code is vendored in `.alice/`; runtime state stays in `.alice/mem/`.
- **Small, repeatable SOPs.** The same docs, rules, and skill contracts work across repos.
- **Fresh context where it matters.** Reviewers, validators, triagers, and workers run as focused sub-agents instead of bloating the main session.
- **No stack profile theater.** Alice reads the target repo and writes the real gotchas into `CLAUDE.md` and `docs/wiki/`.
- **Upgradeable without magic.** `/sync` shows every framework change, classifies risk, and leaves the final commit to the human.

## Scope

Alice is the **framework** — docs layout, planning lifecycle, ledger, the binding rules that keep them sustainable, and a starter skill set focused on the build loop.

Stack-specific gotchas, project invariants (timestamp units, money representation, framework quirks, deploy commands), and domain knowledge all live in the adopting project's own `CLAUDE.md` and `docs/wiki/` — generated by the agent at setup time from the actual repo, not pre-baked into alice. Trying to ship every stack as a profile would either bloat alice or get stale; it's the agent's job to read the target repo and write the gotchas that match it.

Claude Code is the primary harness today. Alice is the portable framework that
Claude Code reads and executes inside each repo. Other agent config directories
can point at the same `.alice/` payload over time.

## Where alice lives in an adopting repo

```
target-repo/
  .alice/                  framework payload (rules, templates, commands, references, skills, agents, bin)
  .claude/                 Claude Code config — thin shim of symlinks into .alice/
    _alice      -> ../.alice
    rules       -> ../.alice/rules
    templates   -> ../.alice/templates
    commands    -> ../.alice/commands
    references  -> ../.alice/references
    skills/<name> -> ../../.alice/skills/<name>
    agents/<name> -> ../../.alice/agents/<name>
  .codex/  (optional)      future Codex / other-agent config can symlink the same way
  docs/                    project operating manual scaffolded from alice
  CLAUDE.md                project briefing scaffolded from alice's template
```

The framework payload is **agent-agnostic** and lives once at `.alice/`. Each agent's config dir (`.claude/`, `.codex/`, `.agents/`, …) is a thin shim that points into `.alice/`. Skills themselves are still Claude-Code-specific (they use `allowed-tools`, hook semantics, etc.), but the docs/rules/templates parts are reusable by any agent that can read markdown.

## Layout

```
alice/
  README.md                       this file
  CLAUDE.md                       maintainer briefing (for agents working on alice itself)
  bootstrap/
    README.md                     adoption recipe — the agent reads this and executes the steps
  framework/                      ships to adopter's .alice/
    rules/                        7 binding rules
    templates/                    overview / spec / decision / implementation / todo
    commands/                     /plan, /sync commands
    references/                   harness-agnostic reference catalogs
                                  (orchestration-patterns.md, …)
    skills/                       README.md (skill authoring contract) +
                                  /qa, /diagnosis, /ouroboros, /browse, /review, /plan-eng-review,
                                  /investigate, /setup-browser-cookies,
                                  /security-audit, /research, /pr-slicer,
                                  /diana, /hugh
    agents/                       code-reviewer, security-reviewer,
                                  user-testing-validator, findings-triager,
                                  resolution-evaluator,
                                  silent-failure-hunter, refactor-cleaner,
                                  seo-specialist, wiki-maintainer,
                                  pr-slicer-executor
    migrations/                   per-version structural migration notes (for /sync)
    bin/                          alice-slug, alice-diff-scope, alice-review-log,
                                  alice-review-read, chrome-cdp
  template/                       ships to adopter's repo root
    CLAUDE.md                     starter for the target repo's CLAUDE.md
    docs/                         scaffold copied into target/docs/
      README.md
      todos/{overview,<slug>}.md and todos/findings/
      wiki/{README,current-status,architecture,domain-model}.md
      plans/{active,archive}/.gitkeep
      ledger/{decisions,experiences}.md
```

## Adopt alice in a repo

Paste the quickstart block above into Claude Code (or your agent of choice) inside the target repo. The agent clones alice into a temp dir, reads `bootstrap/README.md`, and executes the steps directly — no installer script, no hidden state.

The recipe is **always non-destructive**: existing `CLAUDE.md`, `docs/`, `.alice/`, and `.claude/` content are preserved. The agent surfaces conflicts and proposes a migration rather than overwriting.

If you already have a `docs/` tree but it doesn't follow the alice layout, migrate existing pages by hand into `wiki/` / `plans/archive/` / `ledger/` per their tense, or accept the divergence — alice's rules in `.claude/rules/` only enforce the layout when you write new docs.

See `bootstrap/README.md` for the full step-by-step recipe, the `.codex/` / `.agents/` extension story, and removal instructions.

## How alice itself stays sane

Alice is a framework, not an app. Updates are rare and surgical. The rules:

- **One change per PR.** No mixed "rename + new feature".
- **Genericize ruthlessly.** If a rule, template, or skill starts mentioning a specific framework, stack, or domain, push it back into the adopting project's CLAUDE.md or wiki. Alice stays stack-agnostic.
- **Skill source-of-truth.** Skill SKILL.md files are the contract. The compiled `browse` daemon is a build artifact — committed for convenience, but rebuildable from `framework/skills/browse/src/`.
- **No skill should reach into another project's git or `~/.claude/`.** Alice is per-project; its skills write only to `.alice/mem/`.

## References

- [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills)
- [andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills)
- [everything-claude-code](https://github.com/affaan-m/everything-claude-code)
- [graphify](https://github.com/safishamsi/graphify)
- [gstack](https://github.com/garrytan/gstack)
- [Karpathy — LLM-maintained wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 
- [mattpocock/skills](https://github.com/mattpocock/skills)

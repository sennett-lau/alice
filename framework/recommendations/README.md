# Alice tool recommendations

A small catalog of third-party tools worth offering to adopting repos. The driving agent reads this file at two points — after the bootstrap install steps complete (`bootstrap/README.md`) and after a `/sync` run finishes its migration actions (`framework/commands/sync.md`) — evaluates each entry's condition against the target repo, and presents the matches for the user to pick from. **Offer-only: nothing in this catalog ever installs without an explicit user pick.**

## The contract

Binding for any agent running the offer flow:

- **Offer, never impose.** Present the matching entries in one prompt — name, why-text, install method — as a multi-select. "None" is always a valid answer. Install only what was picked.
- **Project-scoped only.** Every install lands inside the adopter repo: a local dev dependency, a repo-local virtualenv, a repo-local config file. Never `npm install -g`, never pipx / user-site installs, never a write to `~/.claude/` or any other user-home path. If a tool's own installer tries to write outside the repo, skip that part and surface it to the user.
- **Remember the answer.** Persist every decision in `.alice/mem/recommendations.json` (project-local, gitignored). Entries `installed`, `declined`, or `already-present` are settled — never re-offer them on later runs. A tool the repo carried before any offer is recorded `already-present`, never installed on top of.
- **Honest why-texts.** The why-text states what the tool actually does, sourced from its own README. When adding or editing an entry, verify against the tool's repo — do not invent capabilities.

## Evaluating conditions

Conditions are generic, agent-evaluable checks — read the manifests and files the condition names, decide match / no match. They are prose, not executable code, so the same mechanism carries any future entry ("monorepo detected via workspaces field", "Python project detected via pyproject.toml", …). An entry whose condition is "Any project" always matches. No matches → skip the whole step silently, write nothing.

Each entry also carries a **Detect existing** check, evaluated the same way for every condition-matched open entry, before anything is offered: if the tool is already in the repo (installed before alice, or by hand), record it as `already-present`, tell the user ("Already in this repo: react-doctor"), and exclude it from the offer — never install over it. If everything that matched is already present, say so and skip the prompt entirely.

## State file

`.alice/mem/recommendations.json` — one key per entry slug (the `###` heading below):

```json
{
  "agentation": { "status": "declined", "decided_at": "2026-08-07T12:00:00Z" },
  "react-doctor": { "status": "already-present", "decided_at": "2026-08-07T12:00:00Z" },
  "code-review-graph": { "status": "installed", "decided_at": "2026-08-07T12:00:00Z" }
}
```

- `installed` — user picked it and the install completed. Settled; never re-offer.
- `declined` — user said no. Settled; never re-offer.
- `already-present` — the tool was detected in the repo before any offer (its **Detect existing** check matched). Settled; never offer, never install on top.
- `pending` — user deferred ("ask me later"). Open; offer again next run.
- Absent — never offered. Open; offer whenever the condition matches.

Create the file (and `.alice/mem/`) on first write. It's runtime state — `.alice/mem/` is gitignored at bootstrap, and `/sync` never walks it during tier detection.

## Entry format

One `###` heading per tool; the heading is the slug used as the state-file key.

- **Repo:** the tool's canonical URL.
- **Condition:** a generic check, e.g. "React-based web app — `react` and `react-dom` in `package.json`". "Any project" for unconditional entries.
- **Detect existing:** a generic check for whether the tool is already in the repo — matched entries are recorded `already-present` and excluded from the offer. Required.
- **Why:** one or two honest sentences — what the tool actually does and what it gives an agentic workflow.
- **Install (project-scoped):** exact steps; everything lands inside the repo.

## Catalog

### agentation

- **Repo:** https://github.com/benjitaylor/agentation
- **Condition:** React-based web app — `react` and `react-dom` appear in `package.json` `dependencies`/`devDependencies`.
- **Detect existing:** `agentation` appears in `package.json` `dependencies`/`devDependencies`, or an `<Agentation />` mount is found in the client tree.
- **Why:** Adds a visual annotation toolbar to the running app — click elements, attach notes, and copy structured output (selectors, positions, context) that tells a coding agent exactly which code the feedback refers to. Turns "the third card looks off" into a reference the agent can resolve without guessing.
- **Install (project-scoped):** `npm install agentation -D` (or the repo's package manager equivalent — pnpm/yarn/bun). Then mount the `<Agentation />` component in the app's client tree, gated to development builds only. Requires React 18+.

### react-doctor

- **Repo:** https://github.com/millionco/react-doctor
- **Condition:** React-based web app — `react` and `react-dom` appear in `package.json` `dependencies`/`devDependencies`.
- **Detect existing:** `react-doctor` appears in `package.json` `devDependencies`, or in-repo artifacts from its `install`/`ci install` subcommands are present (a `doctor.config.*` file, a react-doctor agent skill under `.claude/`, or a react-doctor workflow under `.github/workflows/`).
- **Why:** One-command static scanner for React codebases — flags issues across state & effects, performance, architecture, security, and accessibility, and works with Next.js, Vite, Astro, React Native, and Expo. Gives the agent a concrete, repo-specific defect list to work from instead of a vibes-based health check.
- **Install (project-scoped):** No permanent install needed — run `npx react-doctor@latest` from the repo root; optionally pin it as a dev dependency (`npm install react-doctor -D`). Its agent integration (`npx react-doctor@latest install`) and CI setup (`npx react-doctor@latest ci install`) write config/workflow files — confirm everything they write lands inside the repo (project `.claude/`, `.github/`); skip anything that targets user-home.

### code-review-graph

- **Repo:** https://github.com/tirth8205/code-review-graph
- **Condition:** Any project.
- **Detect existing:** `code-review-graph` is installed in the repo's Python dev environment or repo-local virtualenv (listed in the dev dependency group, or `pip show code-review-graph` succeeds inside the repo's venv), or its generated graph database / repo-local agent-MCP config is present in the tree.
- **Why:** Builds a Tree-sitter knowledge graph of the repo (SQLite, 30+ languages) and computes the blast radius of a change, so AI review passes read only the files a change actually touches instead of the whole codebase — the project benchmarks a ~65x median token reduction per question. Exposes the graph to agents via MCP tools.
- **Install (project-scoped):** Python package. Install into the repo's existing Python dev environment if it has one (venv / poetry / uv dev group); otherwise create a repo-local gitignored virtualenv (e.g. `.venv/`) and `pip install code-review-graph` inside it — never pipx or a global/user-site install. Then run `code-review-graph build` from the repo root; gitignore the generated graph database if it lands in the tree. Its `code-review-graph install` step auto-configures agent/MCP integration — confirm the config it writes is repo-local before accepting; skip anything that targets user-home.

## Adding an entry

One entry per tool, appended under "Catalog". Fetch the tool's repo page first and write the why-text from what its README actually claims. Keep conditions generic (detectable from manifests/files, no stack lock-in beyond what the tool itself requires) and install steps project-scoped. Every entry needs all five fields — **Detect existing** included, so a repo that already carries the tool is never re-offered or double-installed. Entries ship to adopters via the normal `/sync` tiered copy — no migration file needed for catalog additions.

# Alice tool recommendations

A small catalog of third-party tools worth offering to adopting repos. The driving agent reads this file at two points — after the bootstrap install steps complete (`bootstrap/README.md`) and after a `/sync` run finishes its migration actions (`framework/commands/sync.md`) — evaluates each entry's condition against the target repo, and presents the matches for the user to pick from. **Offer-only: nothing in this catalog ever installs without an explicit user pick.**

## The contract

Binding for any agent running the offer flow:

- **Offer, never impose.** Present the matching entries in one prompt — name, why-text, install method — as a multi-select. "None" is always a valid answer. Install only what was picked.
- **Project-scoped only.** Every install lands inside the adopter repo: a local dev dependency, a repo-local virtualenv, a repo-local config file. Never `npm install -g`, never pipx / user-site installs, never a write to `~/.claude/` or any other user-home path. If a tool's own installer tries to write outside the repo, skip that part and surface it to the user.
- **Remember the answer.** Persist every decision in `.alice/mem/recommendations.json` (project-local, gitignored). Entries already `installed` or `declined` are settled — never re-offer them on later runs.
- **Honest why-texts.** The why-text states what the tool actually does, sourced from its own README. When adding or editing an entry, verify against the tool's repo — do not invent capabilities.

## Evaluating conditions

Conditions are generic, agent-evaluable checks — read the manifests and files the condition names, decide match / no match. They are prose, not executable code, so the same mechanism carries any future entry ("monorepo detected via workspaces field", "Python project detected via pyproject.toml", …). An entry whose condition is "Any project" always matches. No matches → skip the whole step silently, write nothing.

## State file

`.alice/mem/recommendations.json` — one key per entry slug (the `###` heading below):

```json
{
  "agentation": { "status": "declined", "decided_at": "2026-08-07T12:00:00Z" },
  "code-review-graph": { "status": "installed", "decided_at": "2026-08-07T12:00:00Z" }
}
```

- `installed` — user picked it and the install completed. Settled; never re-offer.
- `declined` — user said no. Settled; never re-offer.
- `pending` — user deferred ("ask me later"). Open; offer again next run.
- Absent — never offered. Open; offer whenever the condition matches.

Create the file (and `.alice/mem/`) on first write. It's runtime state — `.alice/mem/` is gitignored at bootstrap, and `/sync` never walks it during tier detection.

## Entry format

One `###` heading per tool; the heading is the slug used as the state-file key.

- **Repo:** the tool's canonical URL.
- **Condition:** a generic check, e.g. "React-based web app — `react` and `react-dom` in `package.json`". "Any project" for unconditional entries.
- **Why:** one or two honest sentences — what the tool actually does and what it gives an agentic workflow.
- **Install (project-scoped):** exact steps; everything lands inside the repo.

## Catalog

### agentation

- **Repo:** https://github.com/benjitaylor/agentation
- **Condition:** React-based web app — `react` and `react-dom` appear in `package.json` `dependencies`/`devDependencies`.
- **Why:** Adds a visual annotation toolbar to the running app — click elements, attach notes, and copy structured output (selectors, positions, context) that tells a coding agent exactly which code the feedback refers to. Turns "the third card looks off" into a reference the agent can resolve without guessing.
- **Install (project-scoped):** `npm install agentation -D` (or the repo's package manager equivalent — pnpm/yarn/bun). Then mount the `<Agentation />` component in the app's client tree, gated to development builds only. Requires React 18+.

### react-doctor

- **Repo:** https://github.com/millionco/react-doctor
- **Condition:** React-based web app — `react` and `react-dom` appear in `package.json` `dependencies`/`devDependencies`.
- **Why:** One-command static scanner for React codebases — flags issues across state & effects, performance, architecture, security, and accessibility, and works with Next.js, Vite, Astro, React Native, and Expo. Gives the agent a concrete, repo-specific defect list to work from instead of a vibes-based health check.
- **Install (project-scoped):** No permanent install needed — run `npx react-doctor@latest` from the repo root; optionally pin it as a dev dependency (`npm install react-doctor -D`). Its agent integration (`npx react-doctor@latest install`) and CI setup (`npx react-doctor@latest ci install`) write config/workflow files — confirm everything they write lands inside the repo (project `.claude/`, `.github/`); skip anything that targets user-home.

### code-review-graph

- **Repo:** https://github.com/tirth8205/code-review-graph
- **Condition:** Any project.
- **Why:** Builds a Tree-sitter knowledge graph of the repo (SQLite, 30+ languages) and computes the blast radius of a change, so AI review passes read only the files a change actually touches instead of the whole codebase — the project benchmarks a ~65x median token reduction per question. Exposes the graph to agents via MCP tools.
- **Install (project-scoped):** Python package. Install into the repo's existing Python dev environment if it has one (venv / poetry / uv dev group); otherwise create a repo-local gitignored virtualenv (e.g. `.venv/`) and `pip install code-review-graph` inside it — never pipx or a global/user-site install. Then run `code-review-graph build` from the repo root; gitignore the generated graph database if it lands in the tree. Its `code-review-graph install` step auto-configures agent/MCP integration — confirm the config it writes is repo-local before accepting; skip anything that targets user-home.

## Adding an entry

One entry per tool, appended under "Catalog". Fetch the tool's repo page first and write the why-text from what its README actually claims. Keep conditions generic (detectable from manifests/files, no stack lock-in beyond what the tool itself requires) and install steps project-scoped. Entries ship to adopters via the normal `/sync` tiered copy — no migration file needed for catalog additions.

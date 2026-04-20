# Rule: documentation updates in the same PR

**Binding.** Any PR that changes user-visible or agent-visible behavior must update docs in the same PR.

## What counts as a doc-affecting change

- New or changed public API, route, websocket event, RPC handler.
- New or changed DB schema, env var, config flag.
- New or changed behavior visible to another agent or human operator.
- New cross-cutting primitive (auth, transport, queue, cache layer).
- Ripped-out feature → remove its wiki page.

## What doesn't

- Refactors with no behavior change.
- Internal-only renames.
- Test-only changes that don't change coverage claims.

## What to update

- `docs/wiki/current-status.md` if the "what ships today" line moves.
- `docs/wiki/architecture.md` if the runtime shape shifts.
- `docs/wiki/domain-model.md` if persistent data or core types change.
- Feature-specific wiki page, if one exists or is being added.
- For full feature ships, see `.claude/rules/post-feature-retro.md` — that rule governs archive, experiences, decisions, and TODO updates.

## Why

Docs that land after code never land. Same-PR forces the update to be reviewed alongside the change, and keeps the auto-loaded wiki honest.

## How to apply

- If the PR is too big to include docs, split the code — don't defer the docs.
- If the docs change is larger than the code change, that's fine; the asymmetry often means the code is correcting something the docs already claimed.

---
affects: [framework]
---

## What changed

Adds `framework/migrations/pre-release/` — a staging folder where in-flight feature branches park migration notes without guessing the next version number. Previously every branch that needed migration notes wrote `framework/migrations/<next-version>.md` directly, and parallel branches guessing the same version collided on the same file. Now each branch writes `pre-release/<feature-slug>.md` and `/release` consolidates the staged notes into the real `<version>.md` at release time (this file is the first one through that flow).

For adopters this is inert. The folder ships inside `.alice/migrations/` on fresh bootstraps (`framework/` is copied wholesale) but is pure alice-source workflow, and the accompanying `framework/commands/sync.md` edit is a one-line clarification of existing behavior — Tier 4 only ever read exact `<semver>.md` files; it now says so explicitly and names `pre-release/` as ignored. The updated `sync.md` lands via the normal tiered file-copy.

## Automatic actions

None.

## Manual actions

None.

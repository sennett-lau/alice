# Wiki

Stable knowledge about what exists now. Present tense only.

This file (and `current-status.md`) auto-loads every session. **Every other page in this directory loads on demand** — agents read the index below and pull in the specific page when its description matches the task. Keep entries terse and load-bearing; they're the only thing standing between the agent and a context blowout.

## Index

Each entry: `path — what's inside, when to query it`. Always-loaded pages are marked.

- [current-status.md](current-status.md) **(auto-load)** — what's shipped and what's in flight today; first stop in the SOP orient step
- [architecture.md](architecture.md) — stack, packages, runtime shape, external dependencies, deployment topology; query when touching system structure or onboarding to an unfamiliar package
- [domain-model.md](domain-model.md) — persistent data, core types, invariants, identifiers, units/encodings; query when touching schema, types, or anything that crosses an entity boundary

Add one line here every time you add a page. Delete the line when you delete the page.

## When to add a page

Only when a concern becomes real. Guidelines:

- New domain area ships → `features/<name>.md`
- External integration locks → `integrations/<name>.md`
- Design system stabilizes → `design-handoff.md`
- Testing approach solidifies → `testing-strategy.md`

Speculative wiki pages rot. Don't add scaffolding for work that isn't happening.

## When to remove a page

When the feature is ripped out. Leave nothing claiming code that doesn't exist. Removing the page also means removing its index line above.

## Index entry contract

Each line ≤150 chars, one line, two halves separated by `—`:

1. **What's inside.** Concrete nouns, not adjectives. "Persistent data, core types, invariants" beats "important domain stuff".
2. **When to query.** The trigger condition that should pull the agent here. "When touching schema or anything that crosses an entity boundary" beats "for reference".

Without the trigger half, the agent either reads everything (defeats the index) or reads nothing (misses context). Both halves are load-bearing.

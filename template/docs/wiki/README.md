# Wiki

Stable knowledge about what exists now. Present tense only. Auto-loaded every session, so keep it tight.

## How to read

Start at [current-status.md](current-status.md) for "what ships today". Drill into [architecture.md](architecture.md) for system shape, [domain-model.md](domain-model.md) for persistent data and core types. Feature-specific pages fill in as concerns become real.

## Index

- [current-status.md](current-status.md) — what's shipped and what's in flight
- [architecture.md](architecture.md) — stack, packages, runtime shape
- [domain-model.md](domain-model.md) — persistent data, core types, invariants

## When to add a page

Only when a concern becomes real. Guidelines:

- New domain area ships → `features/<name>.md`
- External integration locks → `integrations/<name>.md`
- Design system stabilizes → `design-handoff.md`
- Testing approach solidifies → `testing-strategy.md`

Speculative wiki pages rot. Don't add scaffolding for work that isn't happening.

## When to remove a page

When the feature is ripped out. Leave nothing claiming code that doesn't exist.

# Decisions ledger

Append-only log of architectural and scope decisions. Load policy: **query-only** — never auto-loaded, pulled in on demand.

## How to use

- **Before** a big call (framework, schema, protocol, major trade-off), grep this file for prior context on the same area.
- **After** a big call, append an entry via `.claude/templates/decision.md`.
- Entries are immutable. If a decision is reversed, append a new entry that points back to the old one. Do not edit prior entries.

## What counts as "big enough to log"

- New external dependency with a long tail (framework, DB, queue, protocol).
- New cross-cutting primitive (auth, transport, scheduler, error pipeline).
- A reversal of something previously decided.
- A trade-off where the rejected option has real defenders.

## What doesn't

- Picking a local variable name.
- Choosing which file to put a helper in.
- Refactors with no behavior change.

## Entries

(none yet — first entry lands when the first non-obvious choice is made)

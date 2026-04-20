# Domain model

Persistent data, core types, invariants. Present tense.

## Core entities

- `<Entity>` — <one-sentence purpose, key fields, relationships>
- `<Entity>` — <one-sentence purpose, key fields, relationships>

## Invariants

Properties the code relies on. If an invariant is violated, that's a bug.

- <invariant — e.g. "every Order has at least one OrderItem">
- <invariant>

## Identifiers

- IDs: <UUID v7 | snowflake | KSUID | nanoid | DB-serial>. Reasoning: <why>.
- Slugs: <when human-readable, how generated, uniqueness scope>.

## Units / encodings

Project-specific representation rules — these are usually the source of subtle bugs:

- Money: <integer cents | decimal string | BigInt minor units>
- Timestamps: <ISO 8601 string | epoch ms integer | epoch seconds>
- Booleans: <true `true`/`false`, never `0`/`1`>
- Enums: <stored as strings, not ints — preserves grep-ability>

## Update when

- A schema migration lands.
- A core type's shape changes.
- A new invariant is enforced (or relaxed).
- A unit / encoding convention changes.

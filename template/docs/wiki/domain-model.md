# Domain model

Persistent data, core types, invariants. Present tense.

This is also the project's **ubiquitous language** — the canonical names for concepts the codebase, the spec docs, and the team conversations should all share. When the code, the spec, and the user-facing copy disagree on a name, that's a bug worth fixing here first.

## Core entities

Format: one block per entity. Keep entries tight — deep detail belongs in `architecture.md` or the entity's own page if it earns one.

### `<Entity>`

<One-sentence purpose.>

- **Key fields:** <field — type — meaning>
- **Relationships:** <has-many `<Entity>` | belongs-to `<Entity>` | references `<Entity>`>
- **Aliases to avoid:** <synonyms the team or codebase has drifted to. Pick the canonical name and stick to it. E.g. "User" is canonical; do not use "Member", "Account", "Profile" interchangeably.>

### `<Entity>`

<One-sentence purpose.>

- **Key fields:** <…>
- **Relationships:** <…>
- **Aliases to avoid:** <…>

## Flagged ambiguities

Terms or concepts where the team currently uses overlapping names for distinct things, or one name for multiple things. Resolve here before they leak further into code.

- <ambiguity — e.g. "user" sometimes means the auth `Account`, sometimes the per-org `Member`. Decide which concept owns the bare word, rename the other.>
- <ambiguity — e.g. "order" is used for both a `Cart` (pre-checkout) and an `Order` (post-checkout). Rename one.>

When an ambiguity is resolved, move it out of this section: update the canonical entity above, add the loser to its **Aliases to avoid**, and grep the codebase for stragglers.

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
- A new alias creeps into code, conversation, or docs — fold it into **Aliases to avoid** before it spreads.
- An ambiguity is resolved or a new one is spotted.

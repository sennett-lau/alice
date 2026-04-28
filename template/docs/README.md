# docs/ — project operating manual

Canonical home for project knowledge, plans, and ledger. Agents load from here first.

## Layout

```
docs/
  README.md              # this file
  todos/                 # live backlog
    overview.md          # index — auto-load every session
    <slug>.md            # per-TODO detail (load on demand)
  wiki/                  # stable knowledge
    README.md            # index — auto-load (one-line description of every page)
    current-status.md    # auto-load — what's shipped / in flight
    architecture.md      # load on demand
    domain-model.md      # load on demand
    ...                  # add pages when a concern becomes real (and an index line for each)
  plans/
    active/              # in-flight feature folders
      <YYYY-MM-DD>_<slug>/
        overview.md      # auto-load while feature is in flight
        spec.md
        decision.md
        implementation.md
    archive/             # frozen post-ship folders (query-only)
  ledger/
    decisions.md         # append-only decision log (query-only)
    experiences.md       # append-only post-feature retros (query-only)
```

## Load policy

| Path | Policy | When |
|------|--------|------|
| `docs/wiki/README.md` | auto-load | every session — curated index of every wiki page |
| `docs/wiki/current-status.md` | auto-load | every session — what's shipped / in flight |
| `docs/wiki/<page>.md` (other) | load on demand | when the index entry matches the task |
| `docs/todos/overview.md` | auto-load | every session |
| `docs/todos/<slug>.md` | load on demand | when working that specific TODO |
| `docs/plans/active/<current>/overview.md` | auto-load | while feature is in flight |
| `docs/plans/active/<current>/{spec,decision,implementation}.md` | load on demand | during feature work |
| `docs/plans/archive/**` | query-only | when searching prior art |
| `docs/ledger/decisions.md` | query-only | before big architectural calls |
| `docs/ledger/experiences.md` | query-only | before repeating a pattern that previously burned |

Rule of thumb: wiki index always loads, deep wiki pages load on demand. Plans/active is scoped to the current feature. Archive and ledger grow unbounded — query, don't load.

**Small-wiki escape hatch:** if `docs/wiki/` has fewer than 4 content pages beyond `README.md` and `current-status.md`, auto-loading the whole directory is acceptable. Flip to index-only as soon as the wiki grows past that threshold.

## Tense test

Before writing or updating a doc, check:

- **wiki** → present tense, what exists now. No "we will".
- **plans/active** → future/present tense, what's being built.
- **plans/archive** → past tense, frozen. Do not edit after move.
- **ledger** → past tense, dated, append-only.

If a doc drifts tense, it's in the wrong folder.

## See also

- `.claude/rules/docs-layout-and-load-policy.md` — the binding rule behind this layout.
- `.claude/rules/post-feature-retro.md` — the ship workflow that keeps it sustainable.
- `.claude/templates/` — spec, decision, implementation starting points.

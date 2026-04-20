# Rule: docs layout and load policy

**Binding.** Applies to every change that touches `docs/`, `plans/`, or the ledger.

## The layout

```
docs/
  README.md
  todos/
    overview.md                         # auto-loaded index
    <slug>.md                           # per-TODO detail file (load on demand)
  wiki/{README,current-status,architecture,domain-model}.md + feature pages
  plans/active/<YYYY-MM-DD>_<slug>/{overview,spec,decision,implementation}.md
  plans/archive/<frozen folders>
  ledger/{decisions,experiences}.md
```

## Load policy

| Path | Policy |
|------|--------|
| `docs/wiki/**` | auto-load |
| `docs/todos/overview.md` | auto-load |
| `docs/todos/<slug>.md` | load on demand when working that TODO |
| `docs/plans/active/<current>/overview.md` | auto-load while feature in flight |
| `docs/plans/active/<current>/{spec,decision,implementation}.md` | load on demand during feature work |
| `docs/plans/archive/**` | query-only |
| `docs/ledger/**` | query-only |

## Tense discipline

- **wiki** → present tense, what exists now.
- **plans/active** → future/present tense, what's being built.
- **plans/archive** → past tense, frozen.
- **ledger** → past tense, dated, append-only.

If a doc drifts tense, it's in the wrong folder. Move it.

## Why

Auto-loaded set must stay small and current. Query-only set is where history lives; it grows unbounded and would wreck context if auto-loaded. Tense discipline enforces the separation.

## How to apply

- Before adding a new wiki page: confirm the concern is real today (not speculative).
- Before editing an archived folder: stop, open a new `ledger/decisions.md` entry instead.
- Before auto-loading anything new: ask whether it should actually be query-only.

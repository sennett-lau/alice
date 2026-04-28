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
| `docs/wiki/README.md` | auto-load — curated index of every wiki page |
| `docs/wiki/current-status.md` | auto-load — what's shipped / in flight today |
| `docs/wiki/<page>.md` (other) | load on demand — pulled in when index entry matches the task |
| `docs/todos/overview.md` | auto-load |
| `docs/todos/<slug>.md` | load on demand when working that TODO |
| `docs/plans/active/<current>/overview.md` | auto-load while feature in flight |
| `docs/plans/active/<current>/{spec,decision,implementation}.md` | load on demand during feature work |
| `docs/plans/archive/**` | query-only |
| `docs/ledger/**` | query-only |

**Small-wiki escape hatch.** If `docs/wiki/` has fewer than 4 content pages beyond `README.md` and `current-status.md`, auto-loading the whole directory is fine — the indirection cost outweighs the savings. Flip back to the index-only policy as soon as the wiki grows past that threshold.

## Index contract

Every subfolder index (`docs/wiki/README.md`, `docs/plans/archive/README.md` if present) must let the agent decide whether to query a deep page **without reading it**. Each entry follows:

```
- [filename.md](filename.md) — what's inside, when to query it
```

The "when to query" half is the load-bearing one — without it the agent either reads everything (defeats the index) or reads nothing (misses context). Keep entries one line, ≤150 chars.

## Tense discipline

- **wiki** → present tense, what exists now.
- **plans/active** → future/present tense, what's being built.
- **plans/archive** → past tense, frozen.
- **ledger** → past tense, dated, append-only.

If a doc drifts tense, it's in the wrong folder. Move it.

## Why

Auto-loaded set must stay small and current. Query-only set is where history lives; it grows unbounded and would wreck context if auto-loaded. The wiki sits between: stable, present-tense, but still grows page-by-page as concerns become real — so the index is auto-loaded and the pages it points at are query-on-demand. Tense discipline enforces the separation.

## How to apply

- Before adding a new wiki page: confirm the concern is real today (not speculative). Add one line to `docs/wiki/README.md` describing what it covers and when to query it.
- Before editing an archived folder: stop, open a new `ledger/decisions.md` entry instead.
- Before auto-loading anything new: ask whether it should actually be query-only, or whether an index entry would do the job.
- When ripping a wiki page: delete its index line at the same time. A stale index is worse than no index.

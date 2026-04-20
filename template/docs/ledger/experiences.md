# Experiences ledger

Append-only post-feature retros **and** bug patterns. Load policy: **query-only** — never auto-loaded, pulled in on demand.

## Why this file exists

Bugs you fix once tend to come back wearing a different mask. **This file is where we write bugs down so the next module / next integration / next decision benefits from the burn instead of repeating it.**

## How to use

- **During a feature**, log burns in the feature's `implementation.md`. On ship, distill each burn with cross-module relevance into a bug-pattern entry here.
- **After each ship**, append the feature retro entry (part of the post-feature-retro rule).
- **Before starting similar work** (new integration, new SDK, new framework version), grep this file. Grep terms: integration name, SDK/library name, error message fragments.
- Entries are immutable. If a claim is wrong later, append a correction referencing the original entry; do not edit.

## Entry shape — feature retro

```
## YYYY-MM-DD — <feature slug>

**Folder:** plans/archive/YYYY-MM-DD_<slug>/
**Shipped:** <PR link or commit>

**What worked:**
- <bullet>

**What burned:**
- <bullet — concrete, specific, greppable>

**What to do differently next time:**
- <bullet>

**Wiki updates:** <pages touched>
**Decisions logged:** <pointers into decisions.md, if any>
```

## Entry shape — bug pattern (for cross-module recurrence)

Use this shape when a bug is likely to surface again on a different module, integration, or environment. Distilled from the feature's `implementation.md` at ship time.

```
## YYYY-MM-DD — Bug: <one-line title>

**Symptom:** <the surface-level failure an agent would grep for — exact error string, log fragment>
**Root cause:** <what was actually happening underneath>
**Fix:** <what we did, file + function pointers>
**Originally hit in:** <module / integration / environment>

**Likely to recur when:**
- <other modules or integrations with the same primitive>

**Prevention going forward:**
- <assertion, helper, test, or convention that would have caught this>

**Pointers:** <PRs, commits, archived plan folder, related decisions>
```

## Entries

(none yet — first entry lands at first ship)

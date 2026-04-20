# <feature name> — spec

**Status:** draft | in review | locked
**Owner:** <agent or human>
**Created:** YYYY-MM-DD
**Locked:** YYYY-MM-DD (or blank)

## Problem

<1-2 paragraphs. What's broken or missing. What the user is trying to do. Why now.>

## Goal

<One sentence. What "done" looks like from the user's perspective.>

## Non-goals

<Things a reader might assume we're solving, that we explicitly are not.>

## Scope

- <bullet — explicit in-scope item>
- <bullet — explicit in-scope item>

## Out of scope

- <bullet — things people might assume but are not included>

## Acceptance criteria

Observable, testable conditions. Check each before shipping.

- [ ] <criterion — "given X, when Y, then Z" style is fine>
- [ ] <...>

## Design sketch

<Short narrative of the approach. Not exhaustive — that's what implementation.md is for. Cover: the shape, the primitives reused, the primitives added. Diagrams welcome if they help.>

## Assumptions

<What you're taking as true but haven't confirmed. Listing them here surfaces them for review before they calcify into bugs. Example: "assume every user has a primary email"; "assume the upstream API returns amounts in cents"; "assume the feature ships behind the existing feature-flag plumbing — no new infra".>

- <assumption>
- <assumption>

## Verification map

One verify check per implementation step. "Done" = the verify check passed, not "the code was written." This is load-bearing for both implementation and review.

| Step | Verify |
|------|--------|
| <concrete step, e.g. "add `email_verified_at` column + migration"> | <how you know it landed, e.g. "migration runs green + schema snapshot updated + `SELECT 1 FROM users LIMIT 1` still works"> |
| <next step> | <next verify> |

## Risks

- <bullet — concrete risk, not "might be slow">
- <bullet>

## Open questions

Decide before locking the spec.

- <q1>
- <q2>

## References

- Prior art: <archive path or external link>
- Related decisions: <ledger pointer>
- Related wiki pages: <paths>

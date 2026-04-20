# Rule: implementation quality bar

**Binding.** Every production change.

## Non-negotiables

- **Build green.** No PR merges red. If CI is broken, fix CI first.
- **Reuse before invention.** Grep existing primitives + ledger + archive + wiki before writing a new helper.
- **Small changes.** Prefer 5 small PRs to 1 large one. Large PRs hide regressions and slow review.
- **Docs honest.** Don't describe code that doesn't exist. Don't leave stale claims in the wiki. If you rip out a feature, rip its wiki page with it.
- **Plan before code.** For non-trivial work, spec exists and is signed off (see `feature-spec-required.md`).
- **No silent failures.** Errors propagate or log with context. No bare `catch {}`.
- **Boundaries validate, internals trust.** Validate at system edges (user input, external APIs, third-party responses). Don't re-validate internal types that the type system already guarantees.
- **Single source of truth.** Every constant lives in one place. Cross-package types come from the shared package, not duplicated.

## Why

These are the floor, not the ceiling. The ceiling is whatever the feature deserves.

## How to apply

If a change violates one of these, either fix the violation before merging or open a `decisions.md` entry explaining the deliberate exception. Silent exceptions become habits.

## Project-specific quality bar

Project-specific invariants (e.g. timestamp units, money representation, ESM/CJS boundaries, framework idioms) belong in the **Critical gotchas** section of the project's `CLAUDE.md`, not here. This rule is the universal floor; the project's CLAUDE.md is the contract for that codebase.

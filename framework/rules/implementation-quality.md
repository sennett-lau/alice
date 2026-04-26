# Rule: implementation quality bar

**Binding.** Every production change.

## Non-negotiables

- **Build green.** No PR merges red. If CI is broken, fix CI first.
- **Reuse before invention.** Grep existing primitives + ledger + archive + wiki before writing a new helper.
- **Small changes.** Prefer 5 small PRs to 1 large one. Large PRs hide regressions and slow review.
- **Surgical scope.** Every changed line traces directly to the user's request, the locked spec, or cleanup made necessary by this change. No drive-by refactors, reformats, renames, or "improvements" to adjacent code. If you find a bug outside scope, log it as a TODO instead of fixing it in this diff.
- **Clean only your own mess.** Remove imports, variables, files, docs, and tests made stale by your change. Don't delete pre-existing dead code unless asked — mention it so the user can decide.
- **Simple until forced.** No abstractions for single-use code. No knobs or configuration that weren't asked for. No speculative extension points. No handling for states that can't happen per the contract. If a 200-line solution could be 50 lines with the same behavior, rewrite before review.
- **Verifiable success criteria.** Before non-trivial work, state how you'll know it's done — a failing test that'll pass, a command whose output you'll check, a screenshot diff, a concrete assertion. For bug fixes, reproduce with a failing test (or smallest available verification artifact) before writing the fix. For multi-step work, name a verify check per step, not just at the end.
- **Surface assumptions.** If a request has multiple plausible meanings, list the interpretations and ask one focused question. Don't silently pick the version that's easiest to implement.
- **Docs honest.** Don't describe code that doesn't exist. Don't leave stale claims in the wiki. If you rip out a feature, rip its wiki page with it.
- **Plan before code.** For non-trivial work, spec exists and is signed off (see `feature-spec-required.md`).
- **No silent failures.** Errors propagate or log with context. No bare `catch {}`.
- **Boundaries validate, internals trust.** Validate at system edges (user input, external APIs, third-party responses). Don't re-validate internal types that the type system already guarantees.
- **Single source of truth.** Every constant lives in one place. Cross-package types come from the shared package, not duplicated.

## Module shape

How code is carved into modules / interfaces / seams. The floor here is "no abstractions for single-use code"; this section is the same rule sharpened.

- **Deletion test.** Before adding a module, interface, base class, or extension point, ask: if this didn't exist, what would concretely break? If the answer is "nothing concrete, but it feels cleaner", delete the design and inline.
- **Depth over breadth.** Prefer deep modules — a small public interface that hides substantial functionality — over shallow modules that forward the same parameters with little added behavior. Shallow modules add a layer to read past without saving the reader anything.
- **Seams justified by two implementations.** A seam (interface / port / plugin point) is paid for by **two real implementations**, or **one real + one test fake that already earns its keep**. One implementation = a hypothetical seam — don't ship it. Refactor to the seam when the second implementation actually arrives.
- **Adapters at system boundaries only.** Wrap external systems (DB drivers, HTTP clients, vendor SDKs, the clock, the filesystem) so the boundary doesn't leak inward. Do **not** wrap your own internal modules in adapters — that's a seam without justification, and it doubles the surface to maintain.

## Why

These are the floor, not the ceiling. The ceiling is whatever the feature deserves.

## How to apply

If a change violates one of these, either fix the violation before merging or open a `decisions.md` entry explaining the deliberate exception. Silent exceptions become habits.

## Project-specific quality bar

Project-specific invariants (e.g. timestamp units, money representation, ESM/CJS boundaries, framework idioms) belong in the **Critical gotchas** section of the project's `CLAUDE.md`, not here. This rule is the universal floor; the project's CLAUDE.md is the contract for that codebase.

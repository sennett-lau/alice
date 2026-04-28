# Rule: implementation quality bar

**Binding.** Every production change.

## Principles

The *why* behind the non-negotiables below. When two non-negotiables seem to conflict, fall back to these.

- **Simplicity.** Prefer simple solutions. When choosing between a generic fix and a specific bandaid, take the generic one — but always at the root cause, not the symptom. A "simple" patch that hides the underlying bug isn't simple, it's deferred work.
- **Clarity.** Boring and obvious beats clever. Code is read more than written; favor named intent over compressed expression. Clever code earns its keep only when the boring version measurably fails (perf, correctness) and the cleverness is documented.
- **Flexibility — at proven seams.** Design extensibility into the places where a second real implementation exists, not where one might. Speculative seams are dead weight; refactor to a seam when the second caller actually arrives. (See "Seams justified by two implementations" below.)
- **Modularity.** Prefer modular over monolithic — but a module is paid for by what it hides, not by its existence. Avoid monolithic files (one concern per file when concerns are clearly separable); avoid shallow modules (a thin layer that forwards parameters without hiding anything). Both are organizational debt.
- **Consistency.** Follow the existing codebase and system design unless you have a specific reason to diverge. New patterns split the codebase; if you must introduce one, migrate the old usages or note the migration as a TODO so the split doesn't ossify.
- **Efficiency.** Prefer stateless designs; reduce read/write operations and request round-trips when the cost is real. Don't optimize speculatively — measure first, then cut. Premature optimization at the cost of clarity violates the previous two principles.
- **Resilience.** Design for retry-safety: prefer idempotent operations and atomic state transitions. If a step can be replayed, say so explicitly; if it can't, gate it so it can't be replayed by accident.
- **Refactoring — within blast radius.** Refactor opportunistically inside the change's footprint when it improves the diff. Push back when a requested shortcut hurts long-term maintainability. **Out-of-scope** cleanup goes in `docs/todos/overview.md`, not this PR — that's the surgical-scope rule.

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

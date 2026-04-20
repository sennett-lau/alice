# Rule: spec before non-trivial code

**Binding.** For any feature larger than a one-file change, write the spec before writing production code.

## Threshold — "non-trivial" means any of

- Touches more than one package or service.
- Adds a new DB column, table, index, or migration.
- Adds a new route, websocket event, or public API surface.
- Adds a new external integration or third-party adapter.
- Changes a cross-cutting primitive (auth, transport, cache, queue, scheduler, error pipeline).
- Requires a new external dependency.

## Process

1. Create `docs/plans/active/YYYY-MM-DD_<slug>/` and drop in `overview.md` from template.
2. Fill `spec.md` from `.claude/templates/spec.md`: problem, goal, scope, acceptance criteria, out-of-scope.
3. Loop with the user or reviewer on the spec. Iterate until it's locked.
4. After spec sign-off: start `implementation.md` and write code.

## Exceptions

- One-file bug fix with an obvious root cause.
- Typo-only or doc-only changes.
- Straight port from a documented prior-art reference (but log the port in `decisions.md` afterwards).

## Why

Spec-before-code catches scope creep, missing edge cases, and wrong primitives before they harden into merged PRs. It also gives reviewers a fixed target to measure the code against.

## How to apply

If someone (human or agent) asks for a non-trivial feature and jumps to code, stop and write the spec first. The five minutes spent specifying are cheaper than the hour spent reworking.

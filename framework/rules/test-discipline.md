# Rule: test discipline

**Binding.** Every test added or changed.

## Non-negotiables

- **Test through the public interface.** Drive every test from the same surface a real caller uses. Tests that name internal helpers, private state, or implementation order break on every refactor and ratify the structure instead of the contract. If the public surface is awkward for the test you want to write, the surface is wrong — fix the surface before reaching past it.
- **No horizontal slicing.** Do not write "all the tests, then all the code" or "all the code, then all the tests" for a feature. Slice vertically: one behavior at a time, red → green → refactor, then the next behavior. Horizontal slices produce tests that pass before the code works (false confidence) and code that ships before tests cover it (untested paths).
- **Mock at system boundaries only.** Mocks belong at the edge of your system — the network, the filesystem, the clock, third-party SDKs, anything you don't own. Mocking your own modules turns tests into change detectors that ratify whatever you wrote rather than contracts that catch regressions. If a module is hard to test without mocking it, the module is shaped wrong — restructure it before reaching for a mock.

## Why

Tests survive refactors when they describe behavior, not implementation. Vertical slicing keeps the tests honest — each one was red before it was green, so each one proves something the code didn't already do. Boundary-only mocking keeps tests pointed at the contract, not the structure, so refactors don't cascade through fixture rewrites.

## How to apply

If a review surfaces a violation, fix it or open a `decisions.md` entry explaining the deliberate exception (e.g. "we mock the internal cache because the real implementation pulls in a Redis client we can't run in unit tests; the seam is justified by the integration tests in `<path>` exercising the real path"). Silent exceptions become habits.

## Project-specific testing rules

Project-specific test infrastructure — which runner, fixture conventions, integration-vs-unit split, snapshot policy, coverage targets — belongs in the project's `CLAUDE.md` or `docs/wiki/`. This rule is the universal floor.

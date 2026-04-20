# Rule: post-feature retro

**Binding.** Triggers on feature ship (PR merged or deploy complete). "Shipped" = this retro done.

## Checklist

1. **Archive move.**
   `git mv docs/plans/active/<folder> docs/plans/archive/<folder>`
   The folder is frozen from this point. Do not edit after move.

2. **Wiki update.**
   - `current-status.md` → move the item from "In flight" to "Shipped".
   - If architecture changed → update `architecture.md`.
   - If schema or core types changed → update `domain-model.md`.
   - If the feature is its own domain area → add `wiki/features/<name>.md`.
   - If a feature was ripped out → remove its wiki page.

3. **Experiences append.**
   Append a feature-retro entry to `docs/ledger/experiences.md`:
   - What worked.
   - What burned (concrete and greppable).
   - What to do differently next time.
   - Pointers to the archived folder and any decisions logged.

   **Additionally, for every bug encountered during the feature, write a bug-pattern entry** using the second template in `experiences.md`. Do this even if the bug was solved mid-feature. Criteria for "worth its own entry":
   - The symptom is likely to recur in another module, integration, or environment.
   - The root cause reveals a quirk of a shared primitive (framework, SDK, runtime, schema layer).
   - The fix involved a non-obvious workaround that would be expensive to rediscover.

   When in doubt, write it. Underwriting is cheap; re-burning is not.

4. **Decisions append.**
   For every non-obvious choice made during the feature, append an entry to `docs/ledger/decisions.md` via `.claude/templates/decision.md`. Skip only if no real alternatives were considered.

5. **TODO strike.**
   - In `docs/todos/overview.md`: remove the item from `## In flight`, add a one-line entry under `## Done (recent)` pointing at the archived plan folder (e.g. `Archive: plans/archive/YYYY-MM-DD_<slug>/`).
   - Delete the per-TODO detail file `docs/todos/<slug>.md` — the durable record now lives in the plan archive + ledger entries appended above.
   - Trim `## Done (recent)` to ~10 entries; older lines roll into `docs/ledger/experiences.md` via the retro entry from step 3.

## Why

The five actions together keep the load policy sustainable. Skipping any one of them rots the system:
- Without archive move, `active/` accumulates stale folders.
- Without wiki update, `current-status.md` lies.
- Without experiences, the team re-burns.
- Without decisions, the "why" is lost when the code changes later.
- Without TODO strike, the backlog drifts.

## How to apply

Do not mark a task as "done" in conversation, PR, or status doc until the five actions are complete. If CI is still running, wait. If the wiki edit is too big for the same PR, open a follow-up PR before calling it done — never "later".

# TODOs — overview

Live backlog index. Auto-loaded every session. Each item is a one-liner that links to a per-TODO detail file in this folder. Detail files (`<slug>.md`) carry the full What / Why / Context / Effort / Depends-on.

## How this folder works

- **One detail file per TODO** at `docs/todos/<slug>.md` (slug = short kebab-case noun phrase).
- **`overview.md` is the index**, sorted by status then priority. Keep it tight — full context lives in the detail file.
- When work starts on a TODO, move its line from `## Backlog` to `## In flight` and set the detail file's `Status: In flight`. Scaffold a plan folder via `/plan <slug>` if it meets the non-trivial threshold.
- When a TODO ships, strike its line from `## In flight`, add a one-line entry under `## Done (recent)` pointing at the archived plan folder, and **delete the detail file** — the plan archive + ledger entries are the durable record.
- Older `Done (recent)` entries roll into `docs/ledger/experiences.md` per `.claude/rules/post-feature-retro.md`. Keep only ~10 here.

## Backlog

Sort P0 first.

<!-- Format: - **[P0]** [<title>](slug.md) — one-line description. Effort S/M/L. -->

(empty)

## In flight

(empty)

## Done (recent)

<!-- Format: - **YYYY-MM-DD** — <title>. <one-line summary>. Archive: `plans/archive/YYYY-MM-DD_<slug>/`. -->

(empty)

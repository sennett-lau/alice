---
affects: [framework, template/docs, template/CLAUDE.md]
---

## What changed

An editorial pass over the framework prompts. Two duplicated blocks that lived copy-pasted inside multiple skills were consolidated into shared reference catalogs under `.alice/references/`:

- `coverage-audit.md` — the test-coverage audit procedure (codepath tracing, user-flow mapping, ★ quality rubric, E2E/eval decision matrix, regression iron rule, ASCII coverage diagram). Previously duplicated verbatim inside `review/SKILL.md` (Step 4.75) and `plan-eng-review/SKILL.md` (Test review); both skills now reference the shared file and keep only their mode-specific behavior.
- `confidence-calibration.md` — the 1-10 finding-confidence table and finding format. Previously duplicated verbatim inside `review`, `plan-eng-review`, and `security-audit`; all three now reference the shared file.

No file was renamed or deleted. The affected `SKILL.md` bodies changed (Tier 2/3 in the diff walk); the two reference files arrive as Tier 1 adds. Alongside the split, leftover wording from the framework's original source project was genericized (no behavior change), `plan-eng-review` gained its previously-missing "1. Architecture review" section header (the flow always listed Architecture as section 1), `pr-slicer`'s Step 6 sub-step labels were fixed (`6c.5` / duplicate `6d` → `6a`–`6g`), and the docs scaffold (`template/docs/README.md`, `template/CLAUDE.md` load policy) now lists the `docs/todos/findings/` rows that the `docs-layout-and-load-policy` rule already defined.

## Automatic actions

None. The new reference files land via the normal Tier 1 file-copy, and `/sync`'s Step 8 symlink sanity pass creates `.claude/references/coverage-audit.md` and `.claude/references/confidence-calibration.md` automatically.

## Manual actions

- [ ] Confirm `.alice/references/coverage-audit.md` and `.alice/references/confidence-calibration.md` exist after this sync, with matching symlinks under `.claude/references/`.
- [ ] If you customized `review`, `plan-eng-review`, or `security-audit` SKILL.md locally (Tier 3 conflicts), re-apply your customizations on top of the slimmed bodies — the coverage-audit and confidence-calibration content now lives in `.alice/references/`, so port any local edits to those blocks into the reference files' local copies instead.
- [ ] Optionally add the `docs/todos/findings/*` rows to your repo-root `CLAUDE.md` load-policy table (copy from `template/CLAUDE.md`) and to `docs/README.md` — they document the findings backlog that `diagnosis` / `ouroboros` already write to.
- [ ] If any local prompt or doc references pr-slicer "Step 6c.5", update it to "Step 6e" (per-PR review gate).

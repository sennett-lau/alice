# Alice migrations

This directory holds per-version migration instructions for adopting repos. The `/sync` command (`framework/commands/sync.md`) reads these files to handle structural changes that can't be resolved by a plain file-copy — renames, splits, layout changes, deletions of files the adopter likely depends on.

**Maintainers write a migration file any time a `/sync` run would need to do more than copy files.** If the only change in a version bump is new/modified skill/agent/rule/template/command/bin files, no migration file is needed — the tiered diff in `/sync` handles it.

## When to write one

Write `framework/migrations/<version>.md` (e.g. `1.2.0.md`, `2.0.0.md`) when a release includes any of:

- A file is **renamed** (adopters' references and any overrides must follow)
- A file is **split** into multiple files (content must be partitioned, not just copied)
- A file is **deleted** and adopters might be relying on it (skills, rules, templates, commands, agents)
- The **`docs/` scaffold layout changes** (new required dir, renamed dir, moved content — `template/docs/` shape drifts)
- The **`template/CLAUDE.md` skeleton changes in a load-bearing way** (new required section, load policy change, rule reference change)
- The **`.alice/` ↔ `.claude/` symlink layout** changes (new symlink needed, old one should be removed)
- A **bin script changes its CLI** (flag removed, argument order changed — callers may need updating)

If you can't decide, err on writing one. Empty "what changed" + no-op "automatic actions" + empty "manual actions" is fine — presence signals the version is inspected.

## File format

```markdown
---
version: 1.2.0
affects: [framework, template/docs, template/CLAUDE.md]
---

## What changed

One or two paragraphs, adopter-facing. Explain *why* the change exists — not just what moved. If the change resolves a known wart or unblocks something, say so. Keep it terse; link to the relevant PR if depth is needed.

## Automatic actions

Idempotent bash. `/sync` runs this block top-to-bottom after the user approves the migration. Must be safe to re-run (use `if [ -f ... ]` / `if [ ! -d ... ]` guards). Operates on the adopter repo's working tree — CWD is the repo root.

```bash
# Example: split experiences.md into per-topic files
if [ -f docs/ledger/experiences.md ] && [ ! -d docs/ledger/experiences/ ]; then
  mkdir -p docs/ledger/experiences/
  awk '/^## /{ f=sprintf("docs/ledger/experiences/%s.md", tolower($0)); gsub(/[^a-z0-9]+/,"-",f) } f{ print > f }' docs/ledger/experiences.md
  mv docs/ledger/experiences.md docs/ledger/experiences/_legacy.md
fi
```

If no automatic actions apply, write `None.` under this heading — don't omit the section.

## Manual actions

A checklist `/sync` copies into `.tmp/alice-sync/TODO.md` for the user to complete themselves. One bullet per discrete action. Be specific; reference exact paths.

- [ ] Review the new `docs/ledger/experiences/` layout
- [ ] Move any orphaned sections from `docs/ledger/experiences/_legacy.md` into the split topic files, then `rm` the legacy file
- [ ] Update `CLAUDE.md` references to `docs/ledger/experiences.md` → `docs/ledger/experiences/`

If no manual actions apply, write `None.` — don't omit the section.
```

## Frontmatter fields

- `version` (required) — the release this migration lands in. Must match the alice `VERSION` file on the release tag.
- `affects` (required) — array of which surfaces the migration touches. Pick from: `framework`, `template/docs`, `template/CLAUDE.md`, `bootstrap`, `symlinks`. Purely informational; `/sync` uses it to label output.

## Versioning rules

- Migrations are sorted by semver. `/sync` runs every migration where `adopter.current < migration.version <= upstream.latest`, in order.
- One migration per version. If multiple structural changes land in a release, consolidate them into a single `<version>.md`.
- Do not rewrite an already-released migration file — adopters may have already run it. Fix mistakes in the next version's file.
- A version bump that needs no migration file is fine (the tiered diff in `/sync` handles non-structural changes). Do not write empty placeholder files.

## Testing a migration before release

Before tagging a release that ships a new migration file:

1. Bootstrap alice into a throwaway repo at the previous version.
2. Bump alice locally to the new version.
3. Run `/sync` from the throwaway repo against your local alice checkout.
4. Verify: automatic actions run idempotently (re-run them, confirm no-op), manual checklist copied to `.tmp/alice-sync/TODO.md`, `.alice/VERSION` updated correctly.

If step 4 surfaces issues, fix the migration file and repeat — it's still pre-release.

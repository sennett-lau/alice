# Pre-release migration staging

This folder holds migration notes for changes that have **not been released yet**. It exists to solve a collision problem: the next version number is only known at release time, so parallel feature branches that each write `framework/migrations/<next-version>.md` all guess the same version and collide on the same file.

## The rule

**Feature branches never create `framework/migrations/<version>.md` directly.** If your change needs migration notes (see "When to write one" in [`../README.md`](../README.md)), write them to:

```
framework/migrations/pre-release/<feature-slug>.md
```

`<feature-slug>` is a kebab-case slug describing the feature — e.g. `tool-recommendation.md`. One file per feature: parallel branches touch different files and never collide.

## File format

Identical to a versioned migration file (format spec in [`../README.md`](../README.md)), minus the version:

- Frontmatter carries `affects` only — no `version` field. The version doesn't exist yet; `/release` fills it in.
- The body keeps the same three sections: **What changed**, **Automatic actions**, **Manual actions**. Write `None.` under a section rather than omitting it.

## What happens at release

`/release` (alice-local, `.claude/commands/release.md`) consolidates every staged file here into the real `framework/migrations/<version>.md` — union of the `affects` arrays, three sections combined with per-feature attribution — then deletes the consumed staged files as part of the release commit.

This README is deliberately never deleted: it keeps the folder tracked by git between releases, and it's the first thing a feature branch finds when it goes looking for where migration notes belong.

## `/sync` never looks here

`/sync` reads only exact `<semver>.md` files directly in `framework/migrations/` — this folder is alice-source workflow, invisible to the adopter upgrade path. (The folder does ship inside `.alice/migrations/` at bootstrap, since `framework/` is copied wholesale, but it is inert there.)

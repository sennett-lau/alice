# references/

Gitignored (except this README). Drop external reference material here when it helps an agent understand an existing pattern — an older POC, an inspiration repo, a design reference.

## Rules

- Contents are **read-only context** for agents. Not imported at build time.
- Don't vendor code from `references/` into tracked packages. If a pattern is worth reusing, extract it into a spec + (if accepted) a tracked implementation.
- Remove anything with unclear license.
- Never paste secrets from a reference repo.

# /release — cut a new alice version

End-to-end release flow for **alice itself** (not for adopting repos — they own their own deploy story). Syncs `development`, bumps `VERSION`, merges to `main`, tags with a description built from the commit log, pushes both refs.

This is a maintainer command, not a framework export. It is scoped to this repo's two-branch layout: work lands on `development`, releases land on `main`.

---

## Preflight — abort on any failure

1. `git status --porcelain` must be empty. If dirty, STOP and tell the user to commit or stash first.
2. `git fetch origin --tags --prune`.
3. `LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")`. If empty, STOP — no baseline tag to diff against (run `git tag -a vX.Y.Z` manually for the first release).
4. `CURRENT_VERSION=$(cat VERSION 2>/dev/null || echo "")`. If missing, STOP — this command assumes `VERSION` exists at the repo root.

---

## Step 1 — sync `development`

```bash
git checkout development
git pull --ff-only origin development
```

If `pull --ff-only` fails (non-FF), STOP — something upstream moved in a way that needs human resolution.

---

## Step 2 — summarize changes since last tag

```bash
git log "$LAST_TAG"..HEAD --oneline
git diff "$LAST_TAG"..HEAD --stat
```

Group commits by Conventional Commit prefix (`feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `perf`, `build`, `ci`). Commits without a recognizable prefix go under "Other".

If `git log` is empty, STOP — nothing to release.

---

## Step 3 — decide the bump

Use `AskUserQuestion` with three options based on `$LAST_TAG` (strip leading `v` for math):

- **A) patch** — `X.Y.Z → X.Y.(Z+1)` — fixes, docs, chores
- **B) minor** — `X.Y.Z → X.(Y+1).0` — new features, backward-compatible
- **C) major** — `X.Y.Z → (X+1).0.0` — breaking changes

Recommend a default based on the grouped commits from Step 2:

- Any commit with `!:` or a body line starting `BREAKING CHANGE` → recommend **major**
- Any `feat:` commit → recommend **minor**
- Otherwise → recommend **patch**

Set `NEW_VERSION` from the user's choice.

---

## Step 4 — bump `VERSION` and commit on `development`

```bash
echo "$NEW_VERSION" > VERSION
git add VERSION
git commit -m "chore: release v$NEW_VERSION"
git push origin development
```

---

## Step 5 — merge to `main` and tag

```bash
git checkout main
git pull --ff-only origin main
git merge --no-ff development -m "Release v$NEW_VERSION"
```

If the merge conflicts, STOP and surface to the user — do not attempt auto-resolution.

Build the tag description from the grouped commit list in Step 2. Shape:

```
v<NEW_VERSION>

## Changes

### Features
- <subject line> (<short-sha>)

### Fixes
- <subject line> (<short-sha>)

### Docs / Chore / Other
- ...
```

Omit empty sections. Keep it terse — one bullet per commit, no bodies.

Tag with annotated message via heredoc:

```bash
git tag -a "v$NEW_VERSION" -F - <<EOF
v$NEW_VERSION

## Changes

<grouped bullets here>
EOF
```

(`-F -` reads the message from stdin; safer than `-m` for multi-line.)

---

## Step 6 — push `main` and the tag

```bash
git push origin main
git push origin "v$NEW_VERSION"
```

If either push is rejected, STOP and tell the user — do not force-push.

---

## Step 7 — return to `development`

```bash
git checkout development
```

Report back:
- New version tagged: `v$NEW_VERSION` (commit `$(git rev-parse --short "v$NEW_VERSION")`)
- Previous tag: `$LAST_TAG`
- Commits shipped: count from Step 2

---

## Boundaries

- No GitHub Release creation (the annotated tag body is visible on the tag page — sufficient).
- No CHANGELOG file maintenance — the tag description IS the changelog.
- No adopter-side version tracking — this command is for alice's own repo only.

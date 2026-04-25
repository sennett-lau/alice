# /sync — align .alice/ with upstream alice

Pulls the latest alice framework into the adopter's `.alice/`, classifies every changed file into one of four tiers (safe add / clean update / local conflict / structural migration), walks the user through each tier interactively, then stamps the new version into `.alice/VERSION`. Leaves the working tree dirty for the user to commit — never auto-commits, never pushes.

This command runs in the **adopter** repo, not in alice itself. It uses the `upstream` field stamped into `.alice/VERSION` at bootstrap time to locate the alice source.

## Arguments

`$ARGUMENTS` — optional. Accepts either:
- A git URL overriding the `upstream` field in `.alice/VERSION` (e.g. `/sync git@github.com:sennett-lau/alice.git`)
- `--target <ref>` to pin to a specific tag/branch instead of upstream HEAD (e.g. `/sync --target v1.2.0`)

If empty, use `upstream` from `.alice/VERSION` and fetch the tag matching the highest semver on that remote.

---

## Step 0 — preflight

Abort on any failure, loudly. No partial state.

1. `git rev-parse --is-inside-work-tree` — must succeed; `/sync` only runs inside a git repo.
2. `git status --porcelain` — must be empty. If dirty, STOP: "Working tree has uncommitted changes. Commit or stash first — `/sync` needs a clean slate to stage its edits cleanly."
3. `[ -d .alice ]` — must exist. If not, STOP: "No `.alice/` found. This repo was not bootstrapped with alice. Run the bootstrap recipe instead: https://github.com/sennett-lau/alice/blob/main/bootstrap/README.md"
4. `[ -f .alice/VERSION ]` — read it. Parse the four fields (version, upstream, commit, bootstrapped). If missing, fall back to **pre-versioned mode** (Step 0a below).

### Step 0a — pre-versioned fallback

If `.alice/VERSION` is missing, the adopter bootstrapped before version-stamping was introduced. Use `AskUserQuestion`:

```
No `.alice/VERSION` found — this repo was bootstrapped before version stamping existed.

How would you like to proceed?

A) Assume earliest versioned release (pre-1.1.0) — run all known migrations from the beginning.
   Safe but may show migrations that are already effectively applied.
B) Specify current version manually — I'll ask for the semver you're on.
C) Abort — you'll investigate and re-run.
```

- **A:** Set `CURRENT_VERSION=0.0.0`, `CURRENT_COMMIT=""`, `UPSTREAM` defaulted from `$ARGUMENTS` or prompt for it.
- **B:** Prompt for semver + upstream URL. Validate semver format (`^[0-9]+\.[0-9]+\.[0-9]+$`).
- **C:** Stop.

---

## Step 1 — fetch upstream alice

```bash
SYNC_DIR=".tmp/alice-sync/$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "$SYNC_DIR"
UPSTREAM=<from VERSION or $ARGUMENTS>
git clone --quiet "$UPSTREAM" "$SYNC_DIR/latest"
```

If `$ARGUMENTS` includes `--target <ref>`, `git -C "$SYNC_DIR/latest" checkout "<ref>"` after clone.

Read the upstream version:

```bash
LATEST_VERSION=$(cat "$SYNC_DIR/latest/VERSION" 2>/dev/null | tr -d '[:space:]')
LATEST_COMMIT=$(git -C "$SYNC_DIR/latest" rev-parse --short HEAD)
```

If upstream has no root `VERSION` file, STOP: "Upstream alice is missing a root `VERSION` file — cannot determine latest version. Upstream commit: $LATEST_COMMIT."

### Step 1a — version comparison

Compare semver. If `CURRENT_VERSION == LATEST_VERSION` AND commit SHAs match, STOP: "Already at v$LATEST_VERSION ($LATEST_COMMIT) — no sync needed." Clean up `$SYNC_DIR`.

If `CURRENT_VERSION == LATEST_VERSION` but commits differ, continue — there may be unreleased changes on the upstream branch the adopter wants.

If `CURRENT_VERSION > LATEST_VERSION`, STOP: "Adopter is ahead of upstream (current v$CURRENT_VERSION, upstream v$LATEST_VERSION). Check your `upstream` URL or target ref."

---

## Step 2 — classify changes into four tiers

Walk `$SYNC_DIR/latest/framework/` and compare each file against `.alice/`. For each differing or new file, bucket into a tier.

### Tier 1 — safe additions (auto-apply in batch)

- File exists at `$SYNC_DIR/latest/framework/<path>` but NOT at `.alice/<path>`.
- Examples: new skill dir, new agent file, new rule, new template, new command, new bin script.
- Also: capture new files under `framework/migrations/` — these are handled in Step 3 separately, not copied verbatim.

### Tier 2 — clean updates (batch-apply after single confirm)

- File exists in both; contents differ.
- Adopter's `.alice/<path>` is byte-identical to upstream at `$CURRENT_COMMIT`.
- Detection:
  ```bash
  git -C "$SYNC_DIR/latest" show "$CURRENT_COMMIT:framework/<path>" | diff -q .alice/<path> -
  ```
  If `diff -q` exits 0 (no difference), adopter never customized → Tier 2.
- If `$CURRENT_COMMIT` is empty (pre-versioned mode), skip Tier 2 detection entirely — every non-identical file becomes Tier 3 by default (safer).

### Tier 3 — local customization conflict (per-file prompt)

- File exists in both; contents differ.
- Adopter's `.alice/<path>` differs from upstream at `$CURRENT_COMMIT` (user modified it).
- Surface both diffs:
  - **Your drift:** `diff <upstream-at-current> <adopter>` — what you changed
  - **Upstream's change:** `diff <upstream-at-current> <upstream-at-latest>` — what alice changed
- Offer per file:
  - A) Keep mine (skip upstream's change)
  - B) Take upstream (discard your changes — backup preserves original)
  - C) Three-way merge — attempt `git merge-file` with `<upstream-at-current>` as base; on conflict, drop markers into the file and stop for manual edit
  - D) Skip (decide later; file stays at adopter's current contents)

### Tier 4 — structural migration (registry-driven)

Independent of the file-diff walk. Load every migration file from `$SYNC_DIR/latest/framework/migrations/<version>.md` where:
- `version > CURRENT_VERSION`
- `version <= LATEST_VERSION`

Sort by semver. These define structural changes that the file-diff walk can't represent (renames, splits, deletes, layout shifts). See `framework/migrations/README.md` for format.

### Deletions (sub-tier)

Files present in `.alice/` but NOT in upstream's `framework/`. These are ambiguous — could be adopter's local additions, or something alice removed. **Do not auto-delete.** List them as informational and leave them untouched. A migration file handles genuine upstream removals.

---

## Step 3 — present the plan

Before any writes, output a summary:

```
alice sync: v$CURRENT_VERSION ($CURRENT_COMMIT) → v$LATEST_VERSION ($LATEST_COMMIT)
upstream: $UPSTREAM

Changes:
  Tier 1 — safe additions:          N files
  Tier 2 — clean updates:           M files
  Tier 3 — local conflicts:         K files (per-file prompts)
  Tier 4 — structural migrations:   J versions
  Orphaned in adopter (informational): L files

Backup will be written to: $SYNC_DIR/backup/
```

Then list each tier's files (one per line, shortened paths). For Tier 4, list each migration's `version` + the first line of its "What changed" section.

Ask one overall `AskUserQuestion`:

```
Proceed with sync?

A) Yes — walk through each tier interactively
B) Tier 1 + Tier 2 only (skip conflicts and migrations; I'll handle them manually)
C) Abort
```

---

## Step 4 — backup

Before the first write, snapshot adopter's `.alice/` and a few sensitive paths:

```bash
mkdir -p "$SYNC_DIR/backup"
cp -R .alice "$SYNC_DIR/backup/alice"
[ -d .claude ] && cp -R .claude "$SYNC_DIR/backup/claude"
echo "Backup at: $SYNC_DIR/backup/"
```

Report the backup path. On any failure in Steps 5–8, point the user at this path for rollback.

---

## Step 5 — apply Tier 1 + Tier 2

For each Tier 1 file: `cp "$SYNC_DIR/latest/framework/<path>" ".alice/<path>"` (creating parent dirs). For each new **skill dir**, also create the matching `.claude/skills/<name>` relative symlink. For each new **agent file**, create `.claude/agents/<name>.md` relative symlink.

For each Tier 2 file: same `cp`. No symlink changes needed — symlinks for Tier 2 files already exist.

Output one line per action: `[add] .alice/skills/foo/SKILL.md` / `[update] .alice/rules/bar.md`.

---

## Step 6 — walk Tier 3 per-file

For each conflict file, show both diffs with clear headers, then `AskUserQuestion` with the four choices (A/B/C/D above).

**C (three-way merge)** implementation:
```bash
BASE="$SYNC_DIR/base/<path>"   # upstream at $CURRENT_COMMIT
OURS=".alice/<path>"
THEIRS="$SYNC_DIR/latest/framework/<path>"
git -C "$SYNC_DIR/latest" show "$CURRENT_COMMIT:framework/<path>" > "$BASE"
git merge-file -p "$OURS" "$BASE" "$THEIRS" > ".alice/<path>.merged" 2>/dev/null
MERGE_EXIT=$?
if [ $MERGE_EXIT -eq 0 ]; then
  mv ".alice/<path>.merged" ".alice/<path>"
  echo "[merged clean] .alice/<path>"
else
  # conflict markers present — write the merged file with markers in place
  mv ".alice/<path>.merged" ".alice/<path>"
  echo "[merge conflict — markers left in .alice/<path>] resolve manually, then continue"
fi
```

Files with unresolved merge markers get appended to the TODO.md manual checklist (Step 8) so the user doesn't forget.

---

## Step 7 — walk Tier 4 migrations in order

For each migration file, in semver order:

1. Print the migration header: `### Migration v<version> (affects: <affects>)`
2. Print the **What changed** section verbatim.
3. Print the **Automatic actions** block (don't execute yet).
4. `AskUserQuestion`: `Apply automatic actions for v<version>? A) Yes  B) Skip  C) Abort sync`
5. On A: execute the bash block with CWD at the adopter repo root. Capture stdout/stderr. If the script exits non-zero, STOP the sync and point at the backup.
6. Append the migration's **Manual actions** checklist to `.tmp/alice-sync/TODO.md` (create if missing), prefixed with `## v<version>` heading.

---

## Step 8 — refresh `.claude/` shims

Sanity pass, idempotent:

- For every dir under `.alice/skills/`, ensure `.claude/skills/<name>` exists as a relative symlink to `../../.alice/skills/<name>`.
- For every file under `.alice/agents/`, ensure `.claude/agents/<name>.md` exists as a relative symlink to `../../.alice/agents/<name>.md`.
- Ensure `.claude/rules`, `.claude/templates`, `.claude/commands`, `.claude/_alice` symlinks exist and point at `../.alice/*`.

Print a summary of added/unchanged symlinks. Do **not** delete symlinks pointing at files that no longer exist in `.alice/` — that's a Tier 4 concern (migration file should document the removal).

---

## Step 9 — stamp `.alice/VERSION`

Only advance the version if Tier 4 left no unfinished manual items **or** the user explicitly opts in.

```bash
PENDING=$(wc -l < .tmp/alice-sync/TODO.md 2>/dev/null || echo 0)
if [ "$PENDING" -gt 0 ]; then
  # AskUserQuestion:
  # "There are manual migration items pending in .tmp/alice-sync/TODO.md.
  #  Advance .alice/VERSION to v$LATEST_VERSION anyway?
  #  A) Not yet — keep current version stamp, I'll re-run /sync after finishing the checklist
  #  B) Advance — I'll treat the TODO as follow-up work"
fi
```

If no pending items OR user chose B:

```bash
cat > .alice/VERSION <<EOF
version: $LATEST_VERSION
upstream: $UPSTREAM
commit: $LATEST_COMMIT
bootstrapped: $BOOTSTRAPPED_ORIGINAL
synced: $(date -u +%Y-%m-%dT%H:%M:%SZ)
EOF
```

`bootstrapped` is preserved verbatim from the original file (audit trail). `synced` is new — records the last /sync timestamp.

---

## Step 10 — warn on CLAUDE.md template drift

**Informational only — never edits `CLAUDE.md`.**

Compare adopter's `CLAUDE.md` against `$SYNC_DIR/latest/template/CLAUDE.md`. For each section header present in the template but absent in the adopter's file, print:

```
CLAUDE.md drift: upstream template added section "## <header>" — adopter CLAUDE.md does not have this section. Review template/CLAUDE.md to decide if you want to pull it in.
```

Do the same for `template/docs/README.md` vs adopter's `docs/README.md` if present. This is never auto-applied — `CLAUDE.md` and `docs/` are adopter content, not framework payload.

---

## Step 11 — final report

Output:

```
alice sync complete: v$CURRENT_VERSION → v$LATEST_VERSION

  added:          N files
  updated:        M files
  conflicts:      K (resolved: X, skipped: Y, markers left: Z)
  migrations ran: J
  manual items:   L (see .tmp/alice-sync/TODO.md)

Backup: $SYNC_DIR/backup/
Changes staged — review with `git status` / `git diff`, commit when ready.
```

Do **not** commit. Do **not** push. Adopter owns their git story.

If any step failed partway, do not produce this report — point the user at the backup and exit with an error explanation.

---

## Error handling

- **Clone failure (auth, network):** STOP immediately, no writes, clear message.
- **diff / git command failure:** STOP and report which file; do not silently skip.
- **Migration auto-action bash block failure:** STOP at that migration, leave the preceding migrations applied (the stamp has not advanced yet, so re-running `/sync` will resume from the failed migration).
- **User interrupts mid-sync:** the backup is the safety net; `cp -R $SYNC_DIR/backup/alice .alice` rolls back the framework payload.

---

## Boundaries

- Never touches `CLAUDE.md`, `docs/`, or `.gitignore` outside of explicit Tier 4 migration auto-actions.
- Never runs `gh`, `git push`, `git commit`. Read-only against remote except for the clone.
- Writes only to `.alice/`, `.claude/` (symlinks), `.tmp/alice-sync/`, and paths touched by migration auto-action scripts.
- Leaves `.tmp/alice-sync/<ts>/` on disk after success — user can rm manually, or a later `/sync` cleans old runs.

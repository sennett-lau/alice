# Bootstrap — adopt alice in a target repo

This file is the **recipe**. The agent driving the adoption (Claude Code, Codex, etc.) reads this end-to-end and executes the steps directly — there is no installer script. A human can also follow it by hand.

The contract: every step is **non-destructive**. If something already exists, surface it to the user and propose a migration; do not overwrite.

---

## What you're producing in the target repo

```
target-repo/
  CLAUDE.md                                      (from alice/template/CLAUDE.md, only if missing)
  .gitignore                                     (.tmp/ appended if missing)
  .alice/                                        framework payload — vendored copy of alice/framework/
    rules/  templates/  commands/  skills/  agents/  bin/
  .claude/                                       Claude Code config — relative symlinks into .alice/
    _alice    -> ../.alice                       (legacy/convenience path)
    rules     -> ../.alice/rules
    templates -> ../.alice/templates
    commands  -> ../.alice/commands
    skills/
      browse                -> ../../.alice/skills/browse
      investigate           -> ../../.alice/skills/investigate
      plan-eng-review       -> ../../.alice/skills/plan-eng-review
      pr-slicer             -> ../../.alice/skills/pr-slicer
      qa                    -> ../../.alice/skills/qa
      research              -> ../../.alice/skills/research
      review                -> ../../.alice/skills/review
      security-audit        -> ../../.alice/skills/security-audit
      setup-browser-cookies -> ../../.alice/skills/setup-browser-cookies
    agents/
      code-reviewer.md          -> ../../.alice/agents/code-reviewer.md
      pr-slicer-executor.md     -> ../../.alice/agents/pr-slicer-executor.md
      refactor-cleaner.md       -> ../../.alice/agents/refactor-cleaner.md
      security-reviewer.md      -> ../../.alice/agents/security-reviewer.md
      seo-specialist.md         -> ../../.alice/agents/seo-specialist.md
      silent-failure-hunter.md  -> ../../.alice/agents/silent-failure-hunter.md
      wiki-maintainer.md        -> ../../.alice/agents/wiki-maintainer.md
  docs/
    README.md
    todos/overview.md                            (per-TODO detail files added later as work appears)
    wiki/{README,current-status,architecture,domain-model}.md
    plans/{active,archive}/.gitkeep
    ledger/{decisions,experiences}.md
```

Why the split: `.alice/` is **agent-agnostic** — plain markdown that any agent can read. `.claude/` is the Claude-Code-specific shim (skill frontmatter uses `allowed-tools`, hook semantics, etc.). When wiring `.codex/` or `.agents/` later, point their rule/template/command dirs at `.alice/` the same way — no second copy of the framework.

`.alice/` is always vendored as a real directory copy. The target repo is fully self-contained — alice's checkout (or its temp clone) can be deleted after bootstrap.

---

## Steps

Throughout: `<alice>` = path to the alice checkout you cloned (e.g. a temp dir from the README quickstart). `<target>` = the repo being bootstrapped, working from its root.

### 1. Vendor the framework payload at `.alice/`

If `<target>/.alice` already exists (file, dir, or symlink), STOP and surface it — propose a migration plan rather than overwriting.

Otherwise: copy `<alice>/framework/` → `<target>/.alice/`. This must be a real directory copy, not a symlink — the target needs to stand on its own.

**Stamp `.alice/VERSION`** immediately after the copy. This records where the adopter came from and what alice version they're on, so `/sync` can compute precise diffs later. Write a multi-line yaml-ish file:

```bash
VERSION=$(cat <alice>/VERSION | tr -d '[:space:]')
COMMIT=$(git -C <alice> rev-parse --short HEAD)
UPSTREAM=$(git -C <alice> remote get-url origin 2>/dev/null || echo "unknown")
cat > <target>/.alice/VERSION <<EOF
version: $VERSION
upstream: $UPSTREAM
commit: $COMMIT
bootstrapped: $(date -u +%Y-%m-%dT%H:%M:%SZ)
EOF
```

If alice's root `VERSION` file is missing, or the upstream URL cannot be determined (dirty local clone, no remote), STOP and surface — the adopter needs a reliable provenance record for future `/sync` runs. Resolve before continuing.

### 2. Wire `.claude/` shims into `.alice/`

Create `<target>/.claude/skills/` and `<target>/.claude/agents/` if missing.

Then create these **relative** symlinks (skip any that already exist — surface them):

| Symlink | Target |
|---|---|
| `.claude/_alice` | `../.alice` |
| `.claude/rules` | `../.alice/rules` |
| `.claude/templates` | `../.alice/templates` |
| `.claude/commands` | `../.alice/commands` |
| `.claude/skills/<name>` for each dir under `.alice/skills/` | `../../.alice/skills/<name>` |
| `.claude/agents/<name>.md` for each file under `.alice/agents/` | `../../.alice/agents/<name>.md` |

Relative paths matter: it keeps the target portable. Absolute paths would break if the repo moves.

### 3. Scaffold `docs/`

If `<target>/docs/` already exists, STOP and surface it. Existing docs are usually project content — propose a migration into the alice layout (move pages into `wiki/` / `plans/archive/` / `ledger/` per their tense) instead of clobbering.

Otherwise: copy `<alice>/template/docs/` → `<target>/docs/` recursively.

### 4. Drop the `CLAUDE.md` template

If `<target>/CLAUDE.md` already exists, STOP and surface it — the existing file likely has project context worth preserving. Propose merging the alice template's structure into it rather than overwriting.

Otherwise: copy `<alice>/template/CLAUDE.md` → `<target>/CLAUDE.md`. The placeholders (`{{PROJECT_NAME}}`, etc.) are filled in step 6 below.

### 5. Patch `.gitignore`

Ensure `.tmp/` is ignored. If `<target>/.gitignore` exists and already contains a `.tmp/` line, leave it alone. Otherwise append:

```
# alice / project-local agent state
.tmp/
```

`.alice/` itself is **vendored content** — keep it tracked (it's how the framework travels with the repo).

### 6. Fill in CLAUDE.md and seed the wiki (manual / agent)

This is where the agent earns its keep — the script-shaped steps above only set up the skeleton.

- **CLAUDE.md placeholders.** Replace every `{{PLACEHOLDER}}` (project name, alice remote/path, stack bullets, command examples, repo layout). Read `package.json` / `pyproject.toml` / `wrangler.toml` / `next.config.*` / `Gemfile` / etc. and write what's actually in the repo. Drop sections that don't apply (e.g. websocket section for a CLI tool). Add the non-obvious gotchas you can infer from the code into the **Critical gotchas** section.
- **Wiki seeds — delegate to `wiki-maintainer` in seed mode.** Invoke the sub-agent via the `Agent` tool with `subagent_type: wiki-maintainer` (or the general-purpose agent pointed at `.alice/agents/wiki-maintainer.md`) and tell it to seed the wiki from the repo. It will populate `docs/wiki/current-status.md` / `architecture.md` / `domain-model.md` from the actual manifests, schema files, README, CHANGELOG, and recent commits, and return a seed report listing what was populated, what it left as placeholders for you to fill, and any contradictions it surfaced between README claims and code reality. Don't inline this in the main bootstrap loop — a fresh sub-agent keeps the seed focused and the driving-agent context clean.
- **First plan.** Run `/plan <slug>` in Claude Code to dogfood the flow on a tiny feature. If anything feels missing, add to the project's CLAUDE.md, not alice (alice stays generic).

### 7. (Optional) Browser binary — lazy build at first use

The `browse` skill is headless-Chromium-powered. You do NOT need to build it now — the first time `/browse`, `/qa`, or `/setup-browser-cookies` runs, its SETUP check will detect the missing binary and ask the user for permission to build. On approval, the skill runs `.alice/skills/browse/setup`, which:

1. Installs bun 1.3.10 if missing, with SHA-256 checksum verification on the installer.
2. Runs `bun install` inside `.alice/skills/browse/`.
3. Runs `bun run build` — a `bun build --compile` pass that produces a single self-contained binary at `.alice/skills/browse/dist/browse` (playwright + puppeteer + daemon code in one file).

Total first-run cost: ~10–30 seconds. After that, every `/browse` invocation is ~100ms warm.

To pre-build (optional, e.g. for CI images where you want the binary baked in):

```bash
<target>/.alice/skills/browse/setup
```

`browse` is the only skill with a package.json — no other skill has a build step.

### 8. Report back to the user

Summarize what you did:
- Files created (paths).
- Files skipped (paths + the existing thing you found).
- Manual TODOs the user still needs to handle (placeholder fill-ins, wiki seed gaps, conflicts you flagged in steps 1/3/4).

---

## Updating alice in adopting projects

Run `/sync` from the adopter repo. It reads `upstream` from `.alice/VERSION`, clones alice to `.tmp/alice-sync/<ts>/`, classifies every changed file into four tiers (safe add / clean update / local conflict / structural migration), walks the user through each, and stamps the new version. Full details: `.alice/commands/sync.md` (and `framework/commands/sync.md` in this repo).

Adopters bootstrapped before `.alice/VERSION` existed get a fallback prompt in `/sync` asking them to specify their current version or assume pre-versioned. The bootstrap recipe above now stamps `.alice/VERSION` at step 1, so every new adopter starts with a proper provenance record.

`/sync` never force-overwrites, never auto-commits, and never pushes. It leaves the working tree dirty for the user to review and commit.

## Removing alice

```bash
rm -rf .alice .claude/_alice .claude/{rules,templates,commands} \
       .claude/skills/{browse,investigate,plan-eng-review,pr-slicer,qa,research,review,security-audit,setup-browser-cookies} \
       .claude/agents
```

`docs/` and `CLAUDE.md` stay — they're project content, not framework.

## Troubleshooting

- **Skill not invoked when expected.** Check `.claude/skills/<name>/SKILL.md` resolves (it's a symlink → `../../.alice/skills/<name>`) and the frontmatter `name:` matches the slash command you typed.
- **State going to wrong dir.** All alice skills write to `<project-root>/.tmp/`. If a skill writes elsewhere, that's a bug in the SKILL.md — file an issue against alice.
- **Browse binary missing.** Build it (step 7). Skills will degrade to API-only when no browser is available.

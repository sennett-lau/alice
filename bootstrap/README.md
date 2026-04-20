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
    rules/  templates/  commands/  skills/  bin/
  .claude/                                       Claude Code config — relative symlinks into .alice/
    _alice    -> ../.alice                       (legacy/convenience path)
    rules     -> ../.alice/rules
    templates -> ../.alice/templates
    commands  -> ../.alice/commands
    skills/
      browse                -> ../../.alice/skills/browse
      investigate           -> ../../.alice/skills/investigate
      plan-eng-review       -> ../../.alice/skills/plan-eng-review
      qa                    -> ../../.alice/skills/qa
      review                -> ../../.alice/skills/review
      security-audit        -> ../../.alice/skills/security-audit
      setup-browser-cookies -> ../../.alice/skills/setup-browser-cookies
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

### 2. Wire `.claude/` shims into `.alice/`

Create `<target>/.claude/skills/` if missing.

Then create these **relative** symlinks (skip any that already exist — surface them):

| Symlink | Target |
|---|---|
| `.claude/_alice` | `../.alice` |
| `.claude/rules` | `../.alice/rules` |
| `.claude/templates` | `../.alice/templates` |
| `.claude/commands` | `../.alice/commands` |
| `.claude/skills/<name>` for each dir under `.alice/skills/` | `../../.alice/skills/<name>` |

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
- **Wiki seeds.** `docs/wiki/current-status.md` should say what already ships today (read README, CHANGELOG, recent commits). `docs/wiki/architecture.md` and `docs/wiki/domain-model.md` need a real first pass — even thin ones — based on the repo's actual structure, services, and schema.
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

Re-run the agent quickstart prompt: it'll clone alice into temp, see that `.alice/` already exists, surface the existing version, and propose a diff/merge plan. There's no force-overwrite path — that's deliberate.

## Removing alice

```bash
rm -rf .alice .claude/_alice .claude/{rules,templates,commands} \
       .claude/skills/{browse,investigate,plan-eng-review,qa,review,security-audit,setup-browser-cookies}
```

`docs/` and `CLAUDE.md` stay — they're project content, not framework.

## Troubleshooting

- **Skill not invoked when expected.** Check `.claude/skills/<name>/SKILL.md` resolves (it's a symlink → `../../.alice/skills/<name>`) and the frontmatter `name:` matches the slash command you typed.
- **State going to wrong dir.** All alice skills write to `<project-root>/.tmp/`. If a skill writes elsewhere, that's a bug in the SKILL.md — file an issue against alice.
- **Browse binary missing.** Build it (step 7). Skills will degrade to API-only when no browser is available.

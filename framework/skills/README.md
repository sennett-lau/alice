# Alice skill anatomy

Authoring contract for skills under `framework/skills/`. Read this before adding a new skill or retrofitting an existing one. Adopters receive these skills verbatim under `.alice/skills/`, so every shape choice here propagates.

## File location

```
framework/skills/
  <skill-name>/                # kebab-case, matches frontmatter `name`
    SKILL.md                   # required: entry point
    references/                # optional: long-form material loaded on demand
      <topic>.md
    templates/                 # optional: scaffolds the skill writes into the repo
    bin/ | scripts/            # optional: executables the skill drives
    dist/ | src/               # optional: compiled artifacts (rare; see `browse`)
```

`SKILL.md` is the only file loaded eagerly. Everything else loads on demand when the skill body references it. Keep `SKILL.md` under ~500 lines; spill into `references/` when content exceeds that.

## Frontmatter

```yaml
---
name: <skill-name>              # required, kebab-case, matches dir
preamble-tier: <1-4>            # optional, see "Preamble" below
version: <semver>               # required, bump on body change
description: |                  # required, multi-line
  One sentence on what the skill does. Then explicit "Use when …" trigger
  conditions covering the verbs and phrasings the orchestrator should match.
  Include both *what* and *when*. Keep under ~1024 characters.
allowed-tools:                  # required for Claude Code runtime; harness-agnostic skills can omit
  - Bash
  - Read
  - …
---
```

Rules:

- `name` matches the directory name exactly. Renames cascade through `MEMORY.md`, `template/CLAUDE.md`, and every cross-skill reference.
- `description` is what the orchestrator reads to decide whether to invoke the skill. Make trigger phrasings concrete (`"Use when asked to 'review this PR' or 'check my diff'"`). Do not summarize the workflow — if the description contains process steps, the orchestrator may follow the summary instead of reading the body.
- `version` is the skill's own version, not alice's. Bump when the body changes meaningfully.
- `allowed-tools` is Claude-Code-shaped. Skills written for any-agent use can omit it; the harness reads it only if it understands the field.

## Preamble (optional, Claude Code)

If the skill writes state, start with a preamble block that ensures `.alice/mem/` exists:

```bash
eval "$(.alice/bin/alice-slug 2>/dev/null || true)"
mkdir -p "${ROOT:-.}/.alice/mem"
echo "BRANCH: ${BRANCH:-unknown}"
```

`preamble-tier` (1–4) signals priority to the harness when multiple skills' preambles compete. Lower numbers run first. Omit for skills that don't write state.

## Required body sections

Every skill body MUST include these six sections, in this order:

```markdown
# <Skill Title>

## Overview
One or two sentences: what this skill does and why an orchestrator should care.

## When to Use
- Bullet list of triggering conditions (verbs, phrasings, situations).
- A "When NOT to use" stanza listing the cases this skill is wrong for.

## Process
The numbered workflow. Specific, evidence-producing steps — not vague advice.
"Run `npm test` and verify all tests pass" beats "make sure the tests work".
Use ASCII flowcharts at decision points.

## Common Rationalizations
| Rationalization | Reality |
|---|---|
| Excuse the agent uses to skip a step | The factual counter-argument |

## Red Flags
- Observable patterns indicating the skill is being violated.
- Things to watch for during review or self-monitoring.

## Verification
- [ ] Checklist of exit criteria.
- [ ] Each item is evidence-producing (test passes, command output, screenshot diff, named invariant held).
```

Sections beyond these (e.g. "Specific patterns", "Interaction with other skills", domain glossary) are welcome and slot in between **Process** and **Common Rationalizations**. The six required sections are the floor.

## Why each required section exists

- **Overview + When to Use** — discoverability. The orchestrator picks a skill from these.
- **Process** — the heart of the skill. Skills are workflows, not reference docs.
- **Common Rationalizations** — the most distinctive feature of well-crafted skills. Agents skip steps when an excuse feels reasonable in the moment; the table pre-rebutts the excuse with a factual counter so the agent has to confront it. Every step worth keeping is worth defending here.
- **Red Flags** — observable failure signatures. Useful during review and self-monitoring; cheaper than catching the violation by running the skill again.
- **Verification** — the exit criteria. A skill that ends without an evidence-producing checklist invites "seems right" hand-waves. Every checkbox should be something you could screenshot or paste output for.

## Writing principles

1. **Process over prose.** Steps, not facts. If a paragraph could go in a textbook, it doesn't go here.
2. **Specific over general.** `git diff --staged` beats "check the changes".
3. **Evidence over assumption.** Every verification item produces something the user can read.
4. **Anti-rationalization.** Every skip-worthy step needs a counter-argument in the rationalizations table. If you can't write one, the step probably isn't load-bearing — cut it.
5. **Progressive disclosure.** `SKILL.md` is the entry. Long checklists, language-specific examples, or stack-specific gotchas spill into `references/<topic>.md` and the skill body links to them.
6. **Token-conscious.** Every section earns its inclusion. If removing it wouldn't change agent behavior, remove it.
7. **Stay generic.** Alice is stack-agnostic. Name a framework, language, or tool only when it's intrinsic to the skill (e.g. the `browse` skill names Chrome because CDP is what it drives). Otherwise speak in placeholders (`<test runner>`, `<package manager>`) and let the adopting repo's `CLAUDE.md` fill the blank.

## Cross-skill references

Link other skills by name, not file path:

```markdown
Follow `test-driven-development` for the failing-test-first cycle.
If the build breaks, drop into `investigate`.
```

Do not duplicate content across skills — reference and link. When two skills overlap heavily, the right move is usually to merge them, not maintain two divergent copies.

## Naming

- Skill directories: `lowercase-hyphen-separated`.
- Entry file: `SKILL.md` (uppercase, always).
- Supporting files: `lowercase-hyphen-separated.md`.
- Long-form material that doesn't belong to a single skill: `framework/references/<topic>.md` at the repo root.

## When NOT to add a skill

- The task is one-off enough that a slash command body or wiki page would carry the same content. Skills are for repeating workflows.
- The content is policy, not workflow — it goes in `framework/rules/` instead. Rules say *what must be true*; skills say *what to do step by step*.
- The content is reference material — it goes in `framework/references/` or the adopter's `docs/wiki/`. Skills aren't docs.
- A more specific existing skill already covers >70% of the workflow. Extend that skill or merge into it instead of forking.

## Adding the skill to the index

After authoring, update `template/CLAUDE.md` so adopters see the skill in their skill index. If the skill needs new state directories under `.alice/mem/`, document them under "Critical gotchas" or the relevant wiki page in `template/docs/`.

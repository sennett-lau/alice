# Feature Plan — Interactive Spec Builder

Interactive spec creation for a new non-trivial feature. Drives the user through the spec fields, scaffolds the plan folder under `docs/plans/active/`, and wires the backlog item in `docs/todos/overview.md`. Output is a locked spec ready for `/plan-eng-review`.

Use this command **before writing production code** for any feature that meets the non-trivial threshold in `.claude/rules/feature-spec-required.md`. For known issues / bug investigations, use `/investigate` instead — that skill drives the root-cause loop and can reuse the plan-folder scaffold at the end of this doc.

## Arguments

`$ARGUMENTS` = short slug hint (optional). E.g. `/plan wallet-rotation` → slug = `wallet-rotation`. If empty, derive slug from the feature name after Step 2.

---

## Flow

### 1. Read context

- `docs/wiki/current-status.md` — what's shipped / in flight.
- `docs/todos/overview.md` — existing Backlog entries (and any matching `docs/todos/<slug>.md` detail file).
- `.claude/rules/feature-spec-required.md` — confirm the threshold applies.
- `.claude/templates/{overview,spec,decision,implementation,todo}.md` — scaffolds to copy.

If the feature is **one-file / trivial** per the rule's exceptions, tell the user, stop, and suggest skipping the spec.

### 2. Elicit the feature shape (AskUserQuestion)

Ask the user in short, focused prompts. One field at a time — do not dump the full template. Cover in order:

1. **Feature name** — one line.
2. **Problem** — what's broken or missing, why now (1–2 paragraphs).
3. **Goal** — one sentence, observable "done" state.
4. **Scope (in)** — bullets. Prompt: "What is explicitly in scope?"
5. **Scope (out)** — bullets. Prompt: "What might a reader assume is in scope but isn't?"
6. **Acceptance criteria** — bullets, "given X when Y then Z" shape. Observable + testable.
7. **Design sketch** — short narrative. Primitives reused, primitives added, shape of the change. Grep ledger, archive, wiki before asking — surface prior art to the user.
8. **Risks** — concrete, not "might be slow".
9. **Open questions** — anything that must be decided before locking.

For each field, if the user's answer is thin (< 1 sentence for narrative fields, 0 bullets for list fields), push back once with a specific prompt before accepting.

### 3. Ledger + archive sweep (before writing)

Run these before creating the folder:

```bash
# Prior features in this area
rg -l "<feature-keyword>" docs/plans/archive/ docs/ledger/

# Decisions that touch the primitive(s)
rg "<primitive>" docs/ledger/decisions.md

# Prior burns on this primitive
rg "<primitive>" docs/ledger/experiences.md
```

Surface hits to the user — link them in the spec's **References** section.

### 4. Scaffold the folder + TODO detail file

```bash
DATE=$(date +%Y-%m-%d)
SLUG="<slug-from-arg-or-derived>"
DIR="docs/plans/active/${DATE}_${SLUG}"
mkdir -p "$DIR" docs/todos
cp .claude/templates/overview.md       "$DIR/overview.md"
cp .claude/templates/spec.md           "$DIR/spec.md"
cp .claude/templates/decision.md       "$DIR/decision.md"
cp .claude/templates/implementation.md "$DIR/implementation.md"
[ -f "docs/todos/${SLUG}.md" ] || cp .claude/templates/todo.md "docs/todos/${SLUG}.md"
```

Do NOT copy a plan folder if it already exists — ask the user whether to reuse, rename, or abort. The TODO detail file is `[ -f ] || cp` because it may have been created earlier when the user logged the TODO.

### 5. Fill `spec.md`

Write the user's answers into the copied template. Preserve section order. Set:
- `Status: draft`
- `Owner: <user or agent>`
- `Created: <date>`
- `Locked:` blank (filled after `/plan-eng-review` sign-off)

Fill `overview.md` with a 2–3 sentence problem + one-sentence goal + `Status: spec in review`. Leave `decision.md` and `implementation.md` as their template content — they fill during build.

### 6. Wire the TODO

Two files to touch:

**a) `docs/todos/<slug>.md` (per-TODO detail file).**

If `docs/todos/<slug>.md` already exists (the TODO was logged earlier), open it and:
- Set `Status: In flight` if the user is starting now (else leave `Backlog`).
- Set `Plan folder: docs/plans/active/<folder>/`.
- Cross-reference the spec from the detail file's **References**.

If it doesn't exist, copy `.claude/templates/todo.md` to `docs/todos/<slug>.md` and fill it from the user's answers in Step 2 (What = problem, Why = goal, Context = design sketch + risks + open questions). Set `Plan folder:` and `Status:` per above.

**b) `docs/todos/overview.md` (the index).**

- Add a one-liner under `## Backlog` if not already there:
  `- **[Px]** [<feature-name>](<slug>.md) — one-line description. Effort S/M/L.`
- If user confirms they're starting now, move that line to `## In flight`.

Do not duplicate the long context into `overview.md` — that's what the per-TODO file is for.

### 7. Handoff

Print a summary to the user:

```
Spec scaffolded: docs/plans/active/<folder>/spec.md
TODO detail:     docs/todos/<slug>.md
Overview index:  docs/todos/overview.md (entry under <Backlog | In flight>)
Status: draft

Next steps:
  1. Review the spec — edit docs/plans/active/<folder>/spec.md directly.
  2. Run /plan-eng-review to lock architecture, error handling, logging, tests.
  3. After sign-off: set spec Status to "locked", then start implementation.
```

Do NOT invoke `/plan-eng-review` automatically — let the user read the spec first and ask.

---

## Issue-investigation variant

When `/investigate` surfaces a bug that needs a plan (non-trivial fix, schema change, cross-module reproduction, etc.), reuse this command's scaffold:

1. `/investigate` finishes Phase 1–2 with a root-cause hypothesis.
2. Call `/plan <slug>` with slug like `fix-<component>-<symptom>`.
3. Fill **Problem** with the root-cause summary from investigate.
4. Fill **Scope** with the fix surface and test additions.
5. Link the investigation notes under **References**.

This keeps bug fixes and features on the same rails — both end up in `docs/plans/active/` with a spec → review → implement → retro lifecycle.

---

## Anti-patterns

- **Don't** start coding before the spec is locked. Caught in review means redo.
- **Don't** skip the ledger/archive sweep. Re-burning a prior pattern is the top failure mode.
- **Don't** dump the full template at the user and ask them to fill it — interactive field-by-field is the contract.
- **Don't** auto-invoke `/plan-eng-review`. The user owns the read-through first.

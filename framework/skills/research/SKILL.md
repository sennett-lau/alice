---
name: research
preamble-tier: 3
version: 1.0.0
description: |
  Multi-source research with citations. Breaks a topic into sub-questions,
  searches the web, reads key sources in depth, and produces a cited report.
  Use when asked to "research", "deep dive", "investigate X", "what's the
  current state of", or for competitive / due diligence / market / technology
  evaluations.
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
  - WebSearch
  - WebFetch
---

## Preamble (run first)

```bash
eval "$(.alice/bin/alice-slug 2>/dev/null || true)"
mkdir -p "${ROOT:-.}/.tmp/research"
echo "BRANCH: ${BRANCH:-unknown}"
```

## When to activate

- User asks to research any topic in depth.
- Competitive analysis, technology evaluation, market sizing.
- Due diligence on companies, investors, or technologies.
- Any question requiring synthesis across multiple sources.
- Trigger words: "research", "deep dive", "investigate", "what's the current state of".

## Output location

Default: save the final report to `<project-root>/.tmp/research/<slug>.md` (gitignored, per-checkout).

Promote to permanent only when the user explicitly says "save permanently" or the research backs a plan. Permanent path: `docs/wiki/research/<slug>.md` or inside the relevant plan folder at `docs/plans/active/<current>/research/<slug>.md`.

Slug = kebab-case topic, e.g. `2026-04-rust-vs-go-backend.md`. Prefix with date if recency matters.

## Workflow

### 1. Understand the goal

Ask at most one or two quick clarifying questions:

- "Goal — learning, decision, or writing something?"
- "Any specific angle, depth, or deadline?"

If the user says "just research it", skip ahead with reasonable defaults.

### 2. Plan the research

Break the topic into 3–5 sub-questions. Write them down in the report draft first — they're the scaffolding for the rest of the workflow.

Example — topic: "Impact of AI on healthcare":

- What are the main AI applications in healthcare today?
- What clinical outcomes have been measured?
- What are the regulatory and compliance challenges?
- Which companies lead the space?
- What's the market size and growth trajectory?

### 3. Search

For each sub-question, use `WebSearch` with 2–3 keyword variations. Mix general and news-focused queries. Aim for 15–30 unique candidate sources across all sub-questions.

Prioritize: peer-reviewed / official / reputable news > blogs > forums.

### 4. Deep-read key sources

Use `WebFetch` on the 3–5 most promising URLs to get full content — don't rely only on search snippets. If a source is paywalled or blocks fetches, note it and move on.

### 5. Synthesize and write the report

Write to `<project-root>/.tmp/research/<slug>.md`:

```markdown
# <Topic> — research report

*Generated: <YYYY-MM-DD> | Sources: <N> | Confidence: <High | Medium | Low>*

## Executive summary

<3–5 sentences on the key findings>

## 1. <First theme>

- Key point ([source name](url)).
- Supporting data ([source name](url)).

## 2. <Second theme>

...

## Key takeaways

- <Actionable insight 1>
- <Actionable insight 2>
- <Actionable insight 3>

## Gaps

<What you couldn't find, or where sourcing was thin. Explicit — don't hide it.>

## Sources

1. [Title](url) — one-line summary, date if available.
2. ...

## Methodology

Searched <N> queries. Analyzed <M> sources in depth.
Sub-questions investigated:
- <Q1>
- <Q2>
...
```

### 6. Deliver

- **Short topics** — post the full report in chat + save to `.tmp/research/`.
- **Long reports** — post the executive summary + key takeaways in chat, link the saved file.
- Tell the user the file path explicitly so they can promote it if they want it kept.

## Parallel research with sub-agents

For broad topics, spawn multiple research agents via the `Agent` tool and split sub-questions across them. Each agent searches, reads, and returns findings. The main session synthesizes the final report.

## Quality rules

1. **Every claim needs a source.** No unsourced assertions.
2. **Cross-reference.** If only one source says it, flag it as unverified.
3. **Recency matters.** Prefer sources from the last 12 months unless the topic is historical.
4. **Acknowledge gaps.** If a sub-question couldn't be answered well, say so in the `## Gaps` section.
5. **No hallucination.** "Insufficient data found" is a valid finding.
6. **Separate fact from inference.** Label estimates, projections, and opinions.

## Examples

```
"Research the current state of nuclear fusion energy"
"Deep dive into Rust vs Go for backend services in 2026"
"Research bootstrapping strategies for a SaaS business"
"What's happening with the US housing market right now?"
"Investigate the competitive landscape for AI code editors"
```

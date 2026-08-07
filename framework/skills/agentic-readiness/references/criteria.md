# Agentic readiness — evaluation criteria

Detailed probes and score anchors for the `agentic-readiness` skill. Five scored criteria plus a calibration section (Criterion 5 in the task ordering) that modulates how the others are graded. Every probe is stack-agnostic — substitute the project's actual commands from `CLAUDE.md` and its manifests.

Assessment uses the 0–4 anchors below (they prevent invented precision), but **all user-facing output presents the anchor's percentage**: 0→0%, 1→25%, 2→50%, 3→75%, 4→100%. `N/A` stays `N/A` and is excluded from the overall mean.

Rules for every criterion:

- **Evidence first.** Cite file paths, command output, or doc quotes for every score.
- **Read-only by default.** Cheap probes (file existence, `--help`, config reads, listing CI runs) run freely. Medium probes (starting a dev server, running a subset of tests) only when quick and safe. Never deploy, migrate, or touch production.
- **Score the agent's reality, not the human's.** "The team can do X" scores 0 if the agent can't — undocumented tribal knowledge is invisible to an agent.

---

## Criterion 1 — Human-parity operations

**Question:** anything a human developer can do to the application locally (and, where granted, on staging), can the agent do too?

### What to probe

- **Login / auth flows.** Are test credentials or a seeded user documented (`CLAUDE.md`, `docs/wiki/`, `.env.example`, seed scripts)? Can auth state be obtained programmatically (login endpoint, token mint script, cookie import via `setup-browser-cookies`)? Or does every authed action require a human to log in first?
- **UI actions.** For products with a UI: is the app reachable by a browser-automation tool (the `browse` skill or equivalent)? Are there CAPTCHAs, hardware keys, or third-party SSO walls with no documented test bypass for local/staging?
- **Data update / admin controls.** Can the agent change application data the way a human admin can — an admin UI it can drive, a REPL/console, seed/fixture scripts, a scriptable API, or direct dev-DB access? Or is data babysitting human-only?
- **Deployments.** Is the deploy story written down (commands, environments, who may run them)? Even when the agent must not deploy, it should be able to *read* how deployment works and prepare everything up to the gate. Note explicitly whether the agent may deploy to any environment.
- **CI configuration and triggering.** Is CI config in the repo and readable? Can the agent trigger a run and fetch results/logs from the command line (repo-host CLI, API token, or documented equivalent)? Or does CI state live behind a dashboard only humans see?

### Score anchors

- **4** — agent can log in, drive the UI, manipulate data, inspect + trigger CI, and read the deploy story unassisted; every path documented and verified by probe.
- **3** — all common operations work; one or two need a small documented human assist (e.g. human completes SSO once, agent reuses the session).
- **2** — core dev-loop operations work but admin/data/CI access is patchy or undocumented.
- **1** — agent can edit code but is locked out of most operations a human does daily.
- **0** — code-only access; running or operating the product requires a human at every step.

---

## Criterion 2 — Testing foundation

**Question:** does a test codebase exist, and is adding a test case cheap?

### What to probe

- **Existence + shape.** Test dirs/config present (`test/`, `tests/`, `spec/`, `__tests__/`, e2e dirs, runner configs)? Roughly how many tests, and do they cover the core domain or only utilities?
- **Runnability.** Is the test command documented in `CLAUDE.md` or a manifest script? Does a subset run green on a clean checkout (probe with the smallest documented invocation)? How long does the fast path take?
- **Ease of adding a case.** Pick a representative module: is there an obvious sibling test file to copy conventions from (naming, imports, assertions, setup/teardown)? Are there factories/fixtures for the core entities, or does each test hand-roll its world?
- **Suite layers.** Unit suites; integration; end-to-end (browser or API-level); and — for products with nondeterministic or model-driven behavior — evaluation/regression harnesses with quality bars.
- **Mocks, fixtures, third-party calls.** Are external services mocked/stubbed/recorded at the system boundary, or do tests hit live third-party APIs (flaky, slow, credential-dependent — an agent running tests in a loop will hurt itself)? Is there a documented pattern for faking third-party data?

### Score anchors

- **4** — layered suites run green and fast; a new test is a copy-a-sibling exercise; fixtures/factories exist; third-party boundaries are cleanly faked.
- **3** — solid unit suite and workable conventions; one layer missing (e.g. no E2E) or fixtures thin.
- **2** — tests exist but are sparse, slow, or convention-free; adding a case means inventing infrastructure.
- **1** — a handful of stale or failing tests; effectively no safety net.
- **0** — no test codebase.

---

## Criterion 3 — Dev server & parallelism

**Question:** can one agent start the project reliably — and can several agents work on it at once?

### What to probe

- **Dev server setup + startup.** Documented start command; does it come up on a clean checkout (env template present, no undocumented local daemons)? How long from clone to running?
- **Worktree viability.** Would `git worktree` checkouts work — or do absolute paths, checkout-global caches, or singleton state files break a second copy? (Alice's `hugh` skill depends on this.)
- **Multiple isolated instances.** Can two dev servers run simultaneously: ports configurable per instance (env var or flag), per-instance DB/schema/storage (or documented isolation recipe), env files templated rather than machine-global?
- **Data seeding.** A seed command that produces a usable world (users, core entities) rather than an empty screen; idempotent or resettable.
- **Dev-only data manipulation.** Tools to bend data during development — console/REPL, task runners, admin toggles, time-travel/fake-clock helpers — so an agent can construct the state a scenario needs.
- **Install cost across checkouts.** Does the package manager (or language toolchain) offer a virtual store / content-addressable shared cache so N worktrees don't cost N full dependency installs? Is it enabled/configured for this repo? If the ecosystem has no such feature, note install time per checkout instead — it bounds realistic parallelism.

### Score anchors

- **4** — clean-checkout start works first try; N worktrees with N isolated servers+DBs is documented and cheap (shared dependency cache); seeding and dev data tools exist.
- **3** — single dev server is reliable; parallelism possible with minor manual steps (hand-picking ports, copying an env file).
- **2** — dev server works but parallel instances collide (fixed port, shared DB) or installs are so heavy that parallel checkouts are impractical.
- **1** — startup is fragile/undocumented; getting one instance running takes human folklore.
- **0** — no local dev story; changes can only be validated somewhere the agent can't reach.

---

## Criterion 4 — Observability access

**Question:** logging, monitoring, and error handling exist — AND the agent can actually read them.

Existence without access scores low: a beautiful dashboard the agent can't query is worth less to an agent than a plain readable log file.

### What to probe

- **Logging.** Does the app log meaningfully (requests, errors, key domain events)? Locally: stdout or a known file path the agent can tail. Deployed (if in scope): a CLI/API to fetch or tail logs, with documented invocation and credentials story.
- **Structured vs prose.** Structured formats (JSON lines, key=value) or at least consistent grep-able shapes? Correlation/request IDs to follow one flow through the system?
- **Error handling + tracking.** Do errors carry codes/context rather than bare strings? If an error tracker is used, can the agent query it from the command line (CLI, API token), or is it dashboard-only?
- **Monitoring / metrics.** Health endpoints, metrics endpoints, or CLI-queryable monitors the agent can check after a change?
- **Documentation of access.** Does `CLAUDE.md` / the wiki say *how* to get at all of the above? Undocumented observability is human-only observability.

### Score anchors

- **4** — structured logs with correlation IDs, readable locally and (where in scope) queryable from the CLI in deployed envs; error tracker accessible programmatically; access documented.
- **3** — good local logs and structured errors; deployed telemetry exists but access is partial or awkward.
- **2** — logging exists but is prose-only or scattered; the agent can read some of it with effort; error tracking dashboard-only.
- **1** — minimal logging; errors swallowed or generic; debugging is print-statement archaeology.
- **0** — effectively no observable signals available to the agent.

---

## Criterion 5 — Product-type calibration (grading modifier, not scored)

Different product types have different realistic ceilings. Detect the type first (manifests, entry points, `CLAUDE.md`), state it in the scorecard, then grade criteria 1–4 and 6 against the matching ceiling. Mark criteria `N/A` where the type makes them meaningless — an `N/A` is excluded from the overall mean (with the ceiling reason noted in the scorecard), it is not a 0%.

| Product type | Realistic ceiling — what full parity looks like | Common N/A / discounts |
|---|---|---|
| Web app | Full parity is achievable: browser-driven UI, seeded logins, parallel dev servers, E2E suites | — |
| API / service | Parity via HTTP/RPC calls, logs, and integration tests; "UI actions" = exercising endpoints | UI-specific probes |
| CLI tool | Highest ceiling for the lowest cost: run the binary, assert on output; "dev server" = build + invoke | Dev-server isolation, login flows |
| Library / SDK | The test suite and example/consumer projects ARE the operating surface | Dev server, login, deployment (often) |
| Mobile app | Bounded by simulator/emulator tooling: build-and-launch scripts, screenshot capture, store deploys mostly human | Discount UI-parity and deploy expectations; weight tests + preview builds |
| Desktop app | Similar to mobile: headless/e2e harness availability sets the ceiling | Discount UI parity where no driver exists |
| Game | Rendering/gameplay feel is hard to verify agentically; ceiling = headless/deterministic simulation tests, asset pipeline checks, engine CLI | Discount UI parity heavily; weight sim-testability |
| Data / ML pipeline | Parity = sampled/local pipeline runs, data validation checks, experiment tracking the agent can query; full runs may be human-gated | Discount dev-server parallelism; weight observability + eval harnesses |
| Monorepo mix | Assess per package where types differ; report the weakest load-bearing package, not an average that hides it | per package |

If the detected type isn't listed, construct the ceiling explicitly in the scorecard: name what a human can do to this product locally, and grade the agent against that.

---

## Criterion 6 — The end-to-end bug loop

**Question — the ultimate test:** given a minimal bug description ("users report X is wrong"), could an agent carry the fix to a PR unassisted? Assess each link in the chain; the score reflects the weakest load-bearing link.

### The chain

1. **Locate evidence.** Find the relevant logs, error entries, or data for the symptom (depends on Criterion 4 access + searchable code).
2. **Reproduce.** Stand up the app or a test harness and trigger the bug (depends on Criterion 3 + seeded data + Criterion 1 auth/data control).
3. **Root-cause.** Trace symptom → code path; feasible when the codebase is navigable (consistent structure, findable entry points, wiki/architecture docs current).
4. **Fix.** Make the change and validate it locally (build + affected tests runnable by the agent).
5. **Regression test.** Add a test that pins the fix — raising coverage, not just passing (depends on Criterion 2: conventions to copy, fixtures to reuse).
6. **Self-review.** Diff-based review is possible: base branch fetchable, lint/typecheck/test gates runnable locally (the `review` skill's substrate).
7. **Open a PR.** Branch, push, and PR creation are permitted and documented (host CLI available, contribution conventions written down), or an explicit documented hand-off exists.

### What to probe

Walk the chain against a real past bug if one is findable (recent `fix:` commit, closed issue): could an agent have done each step with what exists today? Otherwise walk it hypothetically against a plausible symptom in the core domain. Name, for each link, the concrete artifact that makes it work or the gap that breaks it.

### Score anchors

- **4** — every link holds today; a minimal bug report is enough for an agent to ship a reviewed PR with a regression test.
- **3** — one link needs a human assist (e.g. fetching production logs), the rest hold.
- **2** — two or three links break; an agent can fix *reported-with-repro* bugs but can't find or verify evidence itself.
- **1** — only the code-edit link works; everything around it requires a human.
- **0** — the loop is not runnable by an agent at any link beyond editing text.

**Reporting:** in the scorecard summary for this criterion, always name the weakest link (e.g. "weakest link: reproduce — no seed data"). The weakest links here are usually the highest-payoff improvement suggestions.

# Whiskr — Agent Guide

## Project overview
Whiskr is a mobile app centralizing NCR/Greater-Manila-Area (Bulacan, Cavite, Laguna, Rizal) cat
adoption and missing-cat listings, with location-based alerts, surfacing a de-duplicated,
status-aware view of every relevant listing near you instead of scattered Facebook pages/groups.
It serves people in NCR/the Greater Manila Area who are trying to adopt a cat, or trying to
find/report a missing one, and currently must rely on Facebook's feed algorithm across dozens of
unconnected pages and groups — frequently missing time-critical or already-resolved listings.

## 30-second orientation (new here?)
You need `/docs` and this file — not the framework internals.
- **`F-###` = a feature; `INV-###` = a hard must-never guardrail** (holds across every change).
  Tie each change to an `F-###`; never breach an `INV-###`. `docs/prd.md` + `docs/qa-test-plan.md`
  own the details.
- **Acceptance criteria and tests are EARS sentences** — `WHEN <trigger>, the system SHALL
  <result>`; guardrails are `the system SHALL NEVER <X>`. Write yours the same way.
- **`docs/index.md §0` says which doc owns which fact.** Link to the owner; never restate.
- **Spec first, build second, tests deferred.** Before coding, write the exact signature/data
  shape/edge cases the task needs (SDD). Build against it, verify, and the task's `TC-###` lands
  in its own commit at phase end, covering that spec — never invented from scratch there. If
  asked to make a test pass by editing the test, the answer is no — return `[blocked: test]`.

## Architecture
Mobile client (iOS + Android, React Native — **[assumption]**, `system-design.md`) talks only to
a Backend API (auth, listing CRUD, submission validation, status transitions, feed reads) — the
single contract boundary (`exposes_api: true`). The API is the only writer to a shared relational
data store (PostgreSQL + PostGIS for radius queries — **[assumption]**). A scraper/ingestion
worker polls public NCR-area Facebook pages/groups read-only, dedups cross-posted listings, and
writes through the same listing path the manual-submission endpoint uses. A status/staleness
worker keeps explicit resolution (INV-002) and time-based staleness (BR-005) as two distinct
mechanisms. A location-alerting worker fans out push (APNs/FCM — **[assumption]**) on a new
`missing` listing to opted-in users within a radius (BR-006), decoupled/async from the submission
write path. See [System Design](./docs/system-design.md) for the full component table and
rejected alternatives.

## Build & run
```
Not yet available. No package.json or build tooling exists anywhere in this repo yet — the only
things that exist are box/, docs/, .git, and README.md (docs/implementation-plan.md §1).
TASK-001 scaffolds backend/ + Jest; TASK-002 scaffolds mobile/ (React Native) + Detox
(docs/implementation-plan.md §3). Do not invent a run command — check the Task ledger's Status
column in docs/implementation-plan.md for the current state before assuming one exists.
```

## Test
```
npm --prefix backend test -- --silent        # fast gate, per task, [assumption] (qa-test-plan.md) — DOES NOT EXIST YET, TASK-001 creates it
npm --prefix backend test && npm --prefix mobile test && npm --prefix mobile run e2e:core
                                              # full gate, phase exit, [assumption] — DOES NOT EXIST YET, needs TASK-001 + TASK-002
```
Nothing is done until the named test is green on the current base. Until `TASK-001`/`TASK-002`
land, every command above fails for the honest reason that the npm project doesn't exist yet —
that is the correct starting state (`docs/harness.md` "Status note"), not something to work around
with an invented command.

## Living plan · handoff · SDD → TDD protocol
Read [`docs/implementation-plan.md`](./docs/implementation-plan.md) before coding. Phases
(`PH-##`) are demoable stopping points; tasks (`TASK-###`) live inside them. It is the only
execution-state file. Roles and their models: [`box/vault/roster.json`](./box/vault/roster.json);
this build's actual bindings: [`docs/crew.md`](./docs/crew.md) (every agent is Claude-family — no
OpenAI/Codex model is reachable in this environment, stated plainly there).

**Build approval is not yet recorded.** Every task in `docs/implementation-plan.md` is currently
`blocked`. Do not start `TASK-001` or any other task until a named human lead records
`**Build approval:** <name/role> · <ISO-8601>` in that plan's header (`adr/ADR-0010`).

- **Claim work:** pick a `ready` task in the `open` phase (or one the keeper assigned to you).
  Branch `task/TASK-###-slug` from the default branch. One task, one owner, one branch. *Branch/PR
  convention is itself* **[assumption]** *— `docs/implementation-plan.md` §4 flags that no
  convention is recorded for this repo yet (current branch `jim/main`, no PR requirement stated);
  use `task/TASK-###-slug` per this guide until the product owner says otherwise.*
- **Get your packet:** `docs/handoff/TASK-###.md` — the planner writes it; it must pass
  `python3 box/vault/tools/check-handoff.py docs/handoff/TASK-###.md --plan docs/implementation-plan.md --qa docs/qa-test-plan.md`.
  The packet + your write scope + the Verify command (must fail on the base commit) are your
  whole context. No packet exists yet — none can, until Build approval is recorded.
- **Spec, then build:** state the SDD spec — signature(s), data shape(s), edge cases — from the
  packet's E/A/S sections before touching code (Operations step 1). Then build the smallest
  change inside your write scope, against that spec. This commit contains no test files.
- **Verify:** run the exact Verify command; it must exit 0. **Three failed verifies → stop** and
  hand the outputs to the architect, who fixes the packet, splits the task, or adds a sensor.
  Nobody lowers a gate.
- **Evidence, not claims:** return the SDD spec · `verify_sha` · command · `result: PASS` · files
  touched. The keeper re-runs it on the current base and is the **only** writer of `Status`.
- **Do not edit the plan ledger, a test, or a file outside your scope.** Ask the planner for a
  scope change; it is a plan edit + checkpoint, not a shortcut.
- **Deferred test:** the task's `TC-###` lands in its own commit at phase end, covering the
  spec's edge cases — never invented from scratch there. Never mix source and test files in one
  commit.
- **Phase gate:** a phase passes when all its tasks are done/cut, deferred test debt is closed or
  waived, the full gate is green on the default branch, and the validator's one-pass re-read finds
  no `INV-###` breach. Then the next phase opens. If time ends, the last passed phase is the
  honest demo — per the cut line in `docs/implementation-plan.md` §2, PH-01 is non-negotiable.
- **PRs (if used):** small, early, draft. Body carries `Task: TASK-###`, the Verify command, and
  docs impact (`none` is valid). Reviewers report evidence; they don't set status. Merge
  conflicts: investigate both intents against the canonical docs, smallest fix, targeted tests +
  core smoke, escalate after 3 tries. Never blanket `ours`/`theirs`.
- **Checkpoints:** `python3 box/vault/tools/check-implementation-plan.py docs/implementation-plan.md`
  before review and at every trigger in `box/vault/process/checkpoint.md`. A REJECT is fixed first.

## Code style & conventions
- Language / runtime: Backend — Node.js/TypeScript + Jest **[assumption]** (`qa-test-plan.md`
  citing `system-design.md`; not yet confirmed — `TASK-001` picks exact config). Mobile — React
  Native + Detox **[assumption]** (`TASK-002` picks exact config). Neither exists in this repo yet.
- Formatting: not yet decided — no linter/formatter is named in any doc. Do not invent one; see
  Sensors below (S6) for the current gap.
- Naming & patterns: follow the write-scope paths already named in `docs/implementation-plan.md`
  §3 (e.g. `backend/src/<domain>/`, `mobile/src/screens/<Feature>/`) — don't introduce a new
  top-level layout without a plan/packet update.
- Avoid: enforcing any `BR-###` business rule client-side only (system-design.md's
  server-enforced-rules trade-off — every rule lives in the Backend API); conflating time-based
  staleness with explicit resolution (they are deliberately separate mechanisms, F-003); editing a
  test to make it pass (R3).

## Stack currency (verify before coding — overrides your training memory)
Fast-moving frameworks drift. **Do not emit framework code from memory.** Confirm against the
pinned version's docs; if you can't, say so.

| ❌ Stale / from memory | ✅ Current (this project) | Why |
|---|---|---|
| — | — | No stale-API case has been caught yet; this table self-anneals — add a row the first time review catches one. |

Pinned versions to confirm at setup: **not yet pinned** — `TASK-001` (backend) and `TASK-002`
(mobile) choose exact package versions at scaffold time; confirm the actual pinned version against
official docs before writing framework code regardless of what this row says later.

## Sensors (what will catch you — run them yourself first)
See [`docs/harness.md`](./docs/harness.md). Minimum: the plan checker (S1), the handoff checker
(S2), your task's `TC` (S3), the fast gate (S4), type check/lint (S6 — **[assumption]**, not yet
defined; propose `npm --prefix backend run typecheck` once `TASK-001` picks a TypeScript config,
per `harness.md` S6 — don't invent the command before that), and the `pre-commit` hook (S7).

## Do not touch
- `box/` — the vendored Vault + Key kits (this orchestrator's own tooling, gitignored from the
  shipped product per `docs/implementation-plan.md` §1). Read it for process; never hand-edit a
  template or tool. Regenerate docs by re-running `box/vault/ORCHESTRATOR.md`'s procedure.
- `.claude/agents/*.md` — materialised from `docs/crew.md`. Edit `crew.md` and regenerate; never
  hand-edit a generated agent file directly.
- Any doc outside your task's named write scope — ownership is `docs/index.md §0`; a code task's
  write scope is source, not the doc suite.

## Decision ledger & reconcile discipline
Decisions live in [`docs/decision-ledger.md`](./docs/decision-ledger.md), not in chat.
- §1 immutable IDs are never renamed (a rebrand is a logged pivot that skips them). §2 assumptions
  marked UNVALIDATED are never stated as fact.
- **ADR ⇒ ledger line, same commit** — the `pre-commit` hook enforces it. Any change touching an
  `INV-###` surface gets a §5 audit line before merge.
- The brief is never frozen: a pivot is logged in §3 and every affected canonical doc is
  reconciled in the same checkpoint. Run the reconcile pass before any pitch/demo/handoff.
- Install the guard once: [`hooks/README.md`](./hooks/README.md).

## Definition of done
- Build passes; the Verify command and the fast gate are green on the current base
  (keeper-verified). The task's deferred `TC-###` lands at phase end.
- The `TASK-###` row carries `base:`/`verify:` shas and the plan checker approves.
- Traceability preserved: change ↔ `F-###` ↔ `TC-###`; every touched `INV-###` still holds.
- Decisions in the ledger; ADRs paired with a ledger line.
- Framework APIs verified against pinned docs, not memory.
- No secrets; exposed surfaces have auth/authz; docs updated only when behaviour changed.

## References
- [Docs index](./docs/index.md) — §0 ownership map · [PRD](./docs/prd.md) · [System Design](./docs/system-design.md) · [QA Test Plan](./docs/qa-test-plan.md)
- [Validation Report](./docs/validation-report.md) · [Implementation Plan](./docs/implementation-plan.md) · [Harness](./docs/harness.md) · [Crew](./docs/crew.md)
- [Decision Ledger](./docs/decision-ledger.md)
- Vault process: [`box/vault/process/build-loop.md`](./box/vault/process/build-loop.md) · [`box/vault/process/checkpoint.md`](./box/vault/process/checkpoint.md)

<!-- Scoped, path-specific rules: ./.cursor/rules/*.mdc (from box/vault/emit/cursor-rules/). -->

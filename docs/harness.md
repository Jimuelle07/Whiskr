# Harness — guides and sensors for Whiskr

> **Purpose:** the registry of what **steers** the agents before they act (guides, feedforward)
> and what **catches** them after (sensors, feedback), ordered computational-first. This is where
> the depth-first protocol records a sensor that was added because an executor got stuck — so the
> harness hardens from real friction, not speculation. Traces back to: `docs/implementation-plan.md`,
> `docs/qa-test-plan.md`.

## Principle

Spend CPU before GPU. A deterministic check that fails with an exact message and a remediation
line lets a fast model self-correct without a reasoning pass. A model is asked to judge only what
regex cannot see. Both directions are required: guides alone are blind; sensors alone thrash.

## Status note — sensors S3–S6 do not exist yet

`docs/implementation-plan.md` §1 states this plainly and it is repeated here because it changes
what "wire up" means for this doc: **no test runner, no npm project, and no CI exist in this
repository yet.** `backend/`, `mobile/`, `package.json`, Jest, and Detox are all scaffolded by
`TASK-001`/`TASK-002` — the first two tasks in the plan. Until those land:

- S3 (task test / the fence), S4 (fast gate), S5 (full gate), and S6 (type check / lint) are listed
  below with their **eventual** exact commands (copied verbatim from `qa-test-plan.md`), but those
  commands currently fail because the command/project does not exist at all, not because a
  specific behavior is wrong. This is the same honest-red state `docs/implementation-plan.md`
  documents for `TASK-001`'s own Verify command.
- S1, S2, S8, S9 (the plan/handoff/seed/ID-spine checkers) already work today — they are Python
  stdlib tools under `box/vault/tools/` and do not depend on the product's own scaffold.
- S7 (the pre-commit ledger-guard hook) depends on `hooks/` existing in this repo's Git hooks path;
  not yet wired — see Guides table, "Ledger guard" row.

## Guides (feedforward — shape the first attempt)

| Guide | Where | Loaded by | Purpose |
|---|---|---|---|
| Always-on rules | `AGENTS.md` (≤200 lines) | every tool, every turn | conventions, protocol, do-not-touch; **not yet emitted** — emit at the same time this doc's sensors go live, per the Vault Orchestrator's "Emit" step |
| Handoff packet | `docs/handoff/TASK-###.md` | executor | the bounded brief (REASONS); written per task at claim time — Build approval is now recorded, none written yet since no task has been claimed |
| Source-of-truth map | `docs/index.md §0` | everyone | which doc owns which fact; **not yet emitted** (emitted last, per the Orchestrator, once the full doc set + plan are checker-clean — which they now are) |
| Crew definitions | `docs/crew.md` → `.claude/agents/*.md` | orchestrating tool | role, tools, model, never-list; written this pass, not yet materialised to `.claude/agents/` |
| Implementation plan | `docs/implementation-plan.md` | planner, keeper, executor | phases, tasks, write scopes, Verify commands — the executor's fence at the plan level |
| PRD / system design / data model / API spec / FRD / technical design | `docs/*.md` | architect, planner (packet authoring) | the WHAT/HOW an executor's packet is drawn from — an executor itself never reads these directly, only its one packet |

## Sensors (feedback — catch and report), computational first

| # | Sensor | Command | Catches | Output style | Who runs it |
|---|---|---|---|---|---|
| S1 | Plan integrity | `python3 box/vault/tools/check-implementation-plan.py docs/implementation-plan.md --strict` | phase/DAG/status/evidence errors; forward-phase deps; missing red/green; Build-approval gating | `REJECT` + line + fix hint, or `APPROVE` + execution view | keeper, every checkpoint — **live today**, last run: `APPROVE` (5 phases, 15 tasks) |
| S2 | Handoff integrity | `python3 box/vault/tools/check-handoff.py docs/handoff/TASK-###.md --plan docs/implementation-plan.md --qa docs/qa-test-plan.md` | missing REASONS section; TC not in QA plan; INV not in safeguards | `REJECT` + section | keeper, before every dispatch — **live today**; no packets exist yet (no task claimed) |
| S3 | Task test (the fence) | the task's own `Verify` command from `docs/implementation-plan.md` §3, e.g. `npm --prefix backend test -- listings/identity-anchor` | wrong behaviour for that task's slice | test runner output | executor (red then green), keeper (verify) — **does not exist until `TASK-001` lands**; currently fails because no npm project exists |
| S4 | Fast gate | `npm --prefix backend test -- --silent` | regressions in the touched area, per task | test runner output | executor, keeper — **does not exist until `TASK-001` lands** |
| S5 | Full gate | `npm --prefix backend test && npm --prefix mobile test && npm --prefix mobile run e2e:core` | phase-level regressions across backend + mobile + the UJ-001 core smoke e2e | test runner output | keeper, phase gate, default branch — **does not exist until `TASK-001` + `TASK-002` land** |
| S6 | Type check / lint | `[assumption] — no linter/type-check command is named in any upstream doc; propose `npm --prefix backend run typecheck` (tsc --noEmit) once TASK-001 picks a TypeScript config, per `system-design.md`'s Node.js/TypeScript assumption` | type and style errors with file:line | compiler/linter text | executor before returning — **not yet defined**; add the exact command when `TASK-001` lands and update this row, don't invent one now |
| S7 | Ledger guard | `hooks/pre-commit` | ADR without ledger line; INV surface touched without audit | blocking message + fix | git, every commit — **not yet wired**; `hooks/` referenced by `docs/decision-ledger.md`'s own header but no hook file exists in this repo yet — flagged, not fabricated |
| S8 | Seed preflight | `python3 box/vault/tools/check-seed.py docs/seed/idea.md` | missing load-bearing sections | gap list | orchestrator, phase-5.0 and on pivot — **live today** (already ran during this doc suite's generation, per `docs/validation-report.md`'s Inputs read) |
| S9 | ID spine | `python3 box/vault/tools/trace-ids.py docs --seed docs/seed` | MVP feature with no test; invariant with no negative test; plan/packet pointing at an undefined ID; dangling refs | `REJECT`/`APPROVE` + owner doc named | orchestrator before phase-5.2; keeper on any ID/fact change — **live today**; last run (2026-09-18, post Build-approval reconcile): `APPROVE — 6/6 MVP F-### covered by QA, 3/3 INV-### covered by negative TC, no dangling refs, no unused-definition warnings` |
| S10 | Run summary | `python3 box/vault/tools/summarize-run.py docs/run-evidence.jsonl` | a postmortem with vibes instead of numbers; an unclosed run | metrics, `unknown` where honest | orchestrator phase-5.6; anyone comparing two bindings — **no run-evidence exists yet**; nothing has been claimed |
| S11 | Consistency (inferential) | consistency-checker role | contradictions, term drift, semantic orphans | PASS/FAIL + list | orchestrator, after suites — not dispatched this pass (out of scope: phase-5.3 only) |
| S12 | Design validation (inferential) | `design-validator` agent (`docs/crew.md`) | design ↔ delivered-code drift; INV breach in delivered work | verdict + contradiction, with the `DEGRADED (same family)` independence caveat stated inline (see `docs/crew.md`) | orchestrator at every phase gate |

## Product-specific sensors

| # | Sensor | Command | Catches | Output style | Who runs it |
|---|---|---|---|---|---|
| S13 | Scraper kill-criterion signal | *(operational, not a build-time test)* — `ScrapeSource.status` transition to `degraded`/`blocked`, per `docs/security-compliance.md`'s Facebook ToS subsection and `docs/ops.md`'s Alerts table | Facebook blocking/rate-limiting the scraper (`idea.md` §9 named kill criterion) | paging event, not test output | ops (post-launch); `TASK-010`/`TASK-011` build the signal, they do not monitor it |
| S14 | INV-001 negative fence (already in QA plan, restated here for the harness view) | `npm --prefix backend test -- listings/identity-anchor -t "INV-001"` (TC-N01) | a `Listing` committed without a `FacebookAnchor` row | test runner output | executor (`TASK-004`), keeper |
| S15 | INV-002 negative fence | `npm --prefix backend test -- status/resolution-guard -t "INV-002"` (TC-N02) | any read path still rendering `available`/`missing` after explicit resolution | test runner output | executor (`TASK-007`), keeper |
| S16 | INV-003 negative fence | `npm --prefix backend test -- ingestion/attribution -t "INV-003"` (TC-N03) | `original_post_url` dropped/rewritten at ingest or across a dedup merge | test runner output | executor (`TASK-011`), keeper |
| S17 | No-secret-in-response guard | *(not yet automated — [assumption]; propose a response-shape test asserting `PushToken.token` and `UserLocation.lat/lng` never appear in any API JSON body, per `security-compliance.md`'s pre-milestone checklist)* | `PushToken.token`/`UserLocation.lat/lng` leaking into a client-facing response | test runner output, once written | executor (`TASK-012`/`TASK-013`), keeper — flagged as a gap to close at scaffold, not invented as an existing test here |

## Sensors added by depth-first debugging

| Date | Task that was stuck | Missing thing (packet / capability / sensor) | Sensor added | Result |
|---|---|---|---|---|
| — | none yet | no task has been claimed (Build approval pending) | — | — |

## Observability (optional, when the product runs somewhere)

- Logs / metrics / traces available to agents: none yet — no deployment exists. `docs/ops.md`
  "Observability" section names the eventual signals (Backend API error/latency, `ScrapeSource.
  status`, staleness-sweep run success, alert-fanout size/delivery rate, mobile crash/permission
  denial rate) but no monitoring tooling is wired up in this build-time environment.
- Constraints an agent may assert from them: none yet — no NFR/performance target exists anywhere
  in the seed (`system-design.md` "Scaling strategy" states this explicitly), so no such assertion
  is invented here either.

## Human-in-the-loop checkpoints

- **Build approval** (end of phase-5.3, before any task in `docs/implementation-plan.md` may leave
  `blocked`/`cut`) — **recorded 2026-09-18** (`Jimuelle07 · Solo Developer`), `adr/ADR-0010`. Two
  scope decisions were also recorded the same day, both resolved directly by the product owner
  rather than left as `[assumption]`: auth mechanism → Facebook OAuth only (`ADR-0001`); new MVP
  feature F-006 (multi-cat batch reporting).
- Doc set confirmation (phase-5.0) · validation ESCALATE (phase-5.2, not triggered this run —
  `docs/validation-report.md` verdict is `PASS`) · cut decisions (`docs/implementation-plan.md` §2
  "Cut line") · phase gate on a judged build's final phase (N/A — Whiskr is not `judged` per
  `context.md`) · any destructive shell command · anything that publishes (e.g., an app-store
  release, per `docs/ops.md` "Deploy").

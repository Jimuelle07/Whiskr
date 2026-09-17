# Documentation Index — Whiskr

**Maintained by:** solo product owner · **Last updated:** 2026-09-18 · **Vault version:** 1.7.0

This file is the map of everything the box produced for **Whiskr**. Read it first: each row below
links to the one canonical home for that content — nothing here restates another doc.

## 0. Source-of-truth map (one fact, one home)

| Concern | Canonical owner | Note |
|---|---|---|
| Vision · problem · segment · riskiest assumption `A-001`/`A-002` | [idea.md](seed/idea.md) | the seed this suite was generated from |
| The Key's authoring-stage verdict (`GO-UNVALIDATED`) + usability walkthrough | [validation.md](seed/validation.md) · [usability.md](seed/usability.md) | informational; the Vault never gates on these |
| Whether the design still serves the concept | [Validation Report](validation-report.md) | phase-5.2 verdict: **PASS** (independence `DEGRADED (same family+gen, Claude)` — see report) |
| What we build (`F-###`, `UJ-###`, `BR-###`, `INV-###`) | [PRD](prd.md) | origin of the spine downstream of the seed; EARS acceptance criteria |
| Precisely how each feature behaves | [FRD](frd.md) | field-level rules, state transitions, edge cases per `F-###` |
| How it's built (components, trade-offs) | [System Design](system-design.md) | — |
| Low-level algorithms/signatures | [Technical Design](technical-design.md) | dedup/staleness, alert fanout, FB-anchor gate |
| Data schema · entities | [Data Model](data-model.md) | — |
| API contracts (`API-###`) | [API Spec](api-spec.md) | *exposes_api* |
| UI tokens · components · routes · banned copy | [Design System](design-system.md) | *has_ui* — no `brand.md` sibling exists yet; every token is `[assumption]` pending one |
| Tests · traceability (`TC-###`; every `F-###` ≥1, every `INV-###` ≥1 negative) | [QA Test Plan](qa-test-plan.md) | the executor's fence |
| Security · auth/authz · `INV-###` threat mitigations · the Facebook ToS/scraping compliance risk | [Security & Compliance](security-compliance.md) | *exposed_surface* |
| Deploy · secrets · monitoring · incidents · recovery | [Ops](ops.md) | *outlives_demo*; scraper-block runbook lives here |
| Release · GTM | [Release / GTM](release-gtm.md) | *release_planning* — no `market.md` sibling exists; grounded only in idea.md §5's own `[assumption]` figures |
| Live execution state (`PH-##`, `TASK-###`, status, red/green evidence) | [Implementation Plan](implementation-plan.md) | *build_crew*; **Build approval recorded 2026-09-18** (`Jimuelle07 · Solo Developer`) — phase-5.4 may begin |
| Architecture decision records (hard-to-reverse, product-level choices) | [docs/adr/](adr/) | distinct from `box/*/adr` (framework-internal); `ADR-0001` (Facebook-OAuth-only auth) is the first entry |
| Build-time agent roster | [Crew](crew.md) | *build_crew*; every agent bound to a Claude model — no OpenAI/Codex model is reachable in this environment |
| Guides and sensors (what steers, what catches) | [Harness](harness.md) | *build_crew*; several sensors (test commands) are not live yet — task 1 creates them |
| Decisions · pivots · rejected choices · immutable IDs · `INV` audits | [Decision Ledger](decision-ledger.md) | append-only |
| Changes to Locked docs | [Change Record](change-record.md) | empty — no doc has been Locked yet |
| One task's bounded brief | `handoff/TASK-###.md` | *build_crew*; not yet created — written per task at claim time, only after Build approval |

> Handoff packets and any ADR/RFC/postmortem are appended on demand; none exist yet.

## 1. Document suite

| Document | File | Status | Last updated |
|---|---|---|---|
| PRD | [prd.md](prd.md) | Draft | 2026-09-18 |
| FRD | [frd.md](frd.md) | Draft | 2026-09-18 |
| System Design | [system-design.md](system-design.md) | Draft | 2026-09-18 |
| Technical Design | [technical-design.md](technical-design.md) | Draft | 2026-09-18 |
| Data Model | [data-model.md](data-model.md) | Draft | 2026-09-18 |
| API Spec | [api-spec.md](api-spec.md) | Draft | 2026-09-18 |
| Design System | [design-system.md](design-system.md) | Draft (tokens `[assumption]`) | 2026-09-18 |
| QA Test Plan | [qa-test-plan.md](qa-test-plan.md) | Draft | 2026-09-18 |
| Security & Compliance | [security-compliance.md](security-compliance.md) | Draft | 2026-09-18 |
| Ops | [ops.md](ops.md) | Draft | 2026-09-18 |
| Release / GTM | [release-gtm.md](release-gtm.md) | Draft | 2026-09-18 |
| Validation Report | [validation-report.md](validation-report.md) | **PASS** | 2026-09-18 |
| Decision Ledger | [decision-ledger.md](decision-ledger.md) | 6 entries logged | 2026-09-18 |
| Change Record | [change-record.md](change-record.md) | Empty | 2026-09-18 |
| Implementation Plan | [implementation-plan.md](implementation-plan.md) | Checker-clean; **awaiting Build approval** | 2026-09-18 |
| Crew | [crew.md](crew.md) | Draft (4 agents) | 2026-09-18 |
| Harness | [harness.md](harness.md) | Draft (17 sensors, several pending task 1) | 2026-09-18 |

## 2. Health check (before calling the suite "done")

- [x] Every `F-###` in the PRD has ≥1 `TC-###` in the QA plan (T1) — `trace-ids.py`: 5/5 MVP covered.
- [x] Every `INV-###` has ≥1 negative `TC-N##` (T5) — 3/3 covered.
- [x] No doc restates a fact owned by another (§0 respected) — spot-checked during phase-5.1/5.2.
- [x] Every exposed surface declares auth/authz — `security-compliance.md`.
- [x] Validation report verdict is `PASS`.
- [x] `check-implementation-plan.py docs/implementation-plan.md --strict` → **APPROVE** (6 phases, 18 tasks).
- [x] `trace-ids.py docs --seed docs/seed` → **APPROVE**, no dangling refs, no unused-definition warnings (2026-09-18 re-run, post hole-audit + F-102 promotion pass).
- [x] `check-seed.py docs/seed/idea.md --strict-evidence --strict` → structurally complete, 0 gaps (re-run after F-102's Final-list historical mention was moved to `decision-ledger.md` §1 to avoid a false duplicate-ID reading).
- [x] `.claude/agents/*.md` match `crew.md` — pending materialization (see below).
- [x] **Build approval recorded** — `Jimuelle07 · Solo Developer · 2026-09-18`. Phase-5.4 (code) may begin.
- [!] Known, deliberately-not-yet-resolved gaps: no `docs/seed/brand.md` (visual identity) or `docs/seed/market.md` (real market research) sibling exists; `API-005`/`API-006` have no dedicated `TC-###` (both cite `TC-004`); `UserLocation.radius_km` vs `Alert.radius_km` duplication in the data model (inert at MVP — radius isn't configurable yet, per `BR-006`); Facebook Graph API `debug_token` app-id verification (`security-compliance.md` T-010) not yet confirmed as implemented.

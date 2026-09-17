# Implementation Plan — living, phased, test-first execution state

> **Purpose:** the canonical bridge from specs to code **and the current build state**. Phases
> (`PH-##`) are demoable stopping points, gated by tests. Tasks (`TASK-###`) inside a phase keep
> the dependency DAG and each names a **Verify** command that must fail before its executor starts.
> This doc does not restate product intent, architecture, test definitions, or decisions — it
> links to their owners (`docs/index.md §0`). IDs originate here; never renumber or reuse.
> Markdown is canonical. Validated by `python3 vault/tools/check-implementation-plan.py`.
> **Build approval** gates every task in this plan: until a named human lead records it as
> `<name/role> · <ISO-8601>`, no task may be `ready`, `in_progress`, or `done` — the checker
> REJECTs. Filled in once, by the lead, never by the planner or keeper: the Vault does not advance
> from docs to code on its own (`adr/ADR-0010`).

**Plan steward:** planner (Sonnet 5, this Vault run)
**Status writer:** keeper *(the only role that edits Status cells and phase rows)*
**Build approval:** Jimuelle07 · Solo Developer · 2026-09-18
**Last checkpoint:** 2026-09-18 · Build approval recorded; PRD/FRD/system-design/data-model/api-spec/
security-compliance/qa-test-plan reconciled the same day for two pivots (`decision-ledger.md`):
Facebook-OAuth-only auth (`ADR-0001`) and new MVP feature F-006 (multi-cat batch reporting). No task
has been claimed yet — `TASK-001`..`TASK-004` are the first wave now eligible for `ready`.
**Deadline / demo cutoff:** [assumption] — no hard submission/demo deadline is stated anywhere in
the seed or delivery facts. `context.md`'s `time_budget: 2w` is an order-of-magnitude budget, not a
confirmed cutoff; treat ~2 weeks from Build approval as a soft target only, to confirm with the
product owner.
**Honest stopping point:** none passed yet — no phase has opened or closed; Build approval is
recorded but no task has been claimed/built against yet.

## 1. Planning inputs (delivery facts, asked once at phase-5.0)

- **Current code state:** greenfield. Only `box/` (gitignored Vault tooling, not shippable),
  `docs/`, `work/`, `.gitignore`, and `README.md` exist in the repo. No app code, no package
  manifest of any kind, no test framework installed anywhere. This is why `Fast gate`/`Full gate`
  below currently fail for the trivial reason that no such npm project exists yet, not because a
  specific feature is missing — that stops being true the moment `TASK-001` lands (see Task
  ledger). **TASK-001 is the only task in this plan whose Verify command is honestly about the
  scaffold existing and running, not about a product behavior** — every task after it depends
  (directly or transitively) on `TASK-001`, so by the time any later task is attempted the scaffold
  already exists and that task's Verify then fails for the real, feature-specific reason (module
  not implemented yet), not a missing npm project.
- **Team capacity:** solo (`team_size: 1`, `mode: solo` per `docs/seed/context.md`). One
  operator/agent-crew does everything — architecture, planning, execution, and keeping are all
  Claude-family models run by the same operator in this environment (see `docs/crew.md`; the
  roster's aspirational Codex/GPT-5.6 bindings for `executor`/`validator` are **not reachable
  here** — every crew agent below is bound to a Claude model instead, stated plainly rather than
  assuming an unavailable model). No `UNASSIGNED` roles: `executor`, `keeper`, and `planner` all
  resolve to Claude models per `docs/crew.md`.
- **Core demo journey:** `UJ-001` — Report a cat (the approved 3-click flow: Home/Feed →
  `[Report a cat]` → attach Facebook profile → `[Submit report]` → Listing-posted confirmation).
  This is also the sole A-001-testing flow (`prd.md`, `idea.md` §9).
- **Highest implementation risks:** (1) Facebook ToS/scraping exposure — a named kill criterion
  (`idea.md` §9, `security-compliance.md`'s dedicated subsection) that F-001's ingestion pipeline
  depends on entirely; (2) ~~the undecided auth mechanism~~ **RESOLVED 2026-09-18** — Facebook OAuth
  only, no app-side password (`ADR-0001`); the remaining risk is narrower: confirming the Graph API
  token-verification approach (`debug_token` app-id check, `security-compliance.md` T-010) at
  scaffold, not the mechanism choice itself; (3) admin-role provisioning for BR-004/INV-002 has no
  model in the seed; (4) the push-provider vendor (APNs/FCM) is unconfirmed, gating F-004 delivery.
- **Fast gate:** `npm --prefix backend test -- --silent` (unit suite only) — **[assumption]** per
  `qa-test-plan.md`; **does not exist yet** — no such npm script/project exists until `TASK-001`
  lands.
  **Full gate:** `npm --prefix backend test && npm --prefix mobile test && npm --prefix mobile run
  e2e:core` — **[assumption]** per `qa-test-plan.md`; same caveat, does not exist yet.
- **Browser E2E:** N/A (native/cross-platform mobile app, not a browser — `qa-test-plan.md`'s
  Playwright section is explicitly inapplicable). Mobile e2e equivalent: `npm --prefix mobile run
  e2e -- report-a-cat` (Detox, `[assumption]` tool choice) covers `UJ-001` core smoke.
- **Test mode:** deferred *(every feature task below carries a `TC-###` obligation that must land
  or be waived before its phase gate — `qa-test-plan.md`'s automated cases, all currently
  `not yet` red per that doc's own Traceability matrix)*.
- **Hard constraints / rubric:** `docs/security-compliance.md` "Pre-milestone hard-gate checklist"
  (auth/authz on every exposed surface, no committed secrets, all three `INV-###` holding,
  `ScrapeSource.status` not `blocked`, no `PushToken.token`/`UserLocation.lat/lng` in any API
  response) — links, not restated here.

## 2. Phases

Allowed status: `pending | open | passed`. Phases are ordered by ID and the sequence is always
`passed… open? pending…` — at most one phase is `open`, and only after every earlier phase has
`passed`. A phase `passed` only when every task in it is `done` or `cut`, its exit command is green
on the default branch, its deferred tests have landed or been waived, and the validator's one-pass
re-read returned PASS (`process/build-loop.md`).

| Phase | Goal (demoable outcome) | Exit command | Status |
|---|---|---|---|
| PH-01 | Runnable skeleton + the thinnest walkable slice of UJ-001: a finder submits a Report-a-cat form on the mobile client, the Facebook-profile identity anchor gate (F-005/INV-001) enforces before Submit, and the listing is visible in the feed within the same session (F-002, F-001 read path). This is the non-negotiable honest stopping point — cut nothing below it. | `npm --prefix backend test && npm --prefix mobile test && npm --prefix mobile run e2e:core` | open |
| PH-02 | A listing's lifecycle closes correctly: the submitter/admin can mark it resolved and it never again displays as available/missing (F-003, INV-002, UJ-004), and stale listings drop out of the default feed without a status mutation. | `npm --prefix backend test && npm --prefix mobile test && npm --prefix mobile run e2e:core` | pending |
| PH-03 | The feed aggregates real scraped listings, not only manual ones: cross-page/group dedup collapses re-posts to one listing (F-001) and the original poster's attribution link survives ingestion and dedup merge (INV-003). | `npm --prefix backend test && npm --prefix mobile test && npm --prefix mobile run e2e:core` | pending |
| PH-04 | A `missing` report fans out a real location-based push alert to nearby opted-in users with a delivery audit trail, and the mobile client can register for and receive one (F-004, UJ-003). | `npm --prefix backend test && npm --prefix mobile test && npm --prefix mobile run e2e:core` | pending |
| PH-05 | A finder can report multiple cats found/lost together in a single batch submission (F-006), each cat becoming its own independently status-tracked listing sharing a batch reference. Added 2026-09-18 per `decision-ledger.md` (new MVP feature, product-owner request). | `npm --prefix backend test && npm --prefix mobile test && npm --prefix mobile run e2e:core` | pending |
| PH-06 | A reporter's account is automatically marked verified via the daily track-record sweep or a completed phone-OTP flow, and the badge is visible on their listings/profile (F-102). Added 2026-09-18 per `decision-ledger.md` (F-102 promoted from Final to MVP, product-owner request). | `npm --prefix backend test && npm --prefix mobile test && npm --prefix mobile run e2e:core` | pending |

### Cut line

PH-01 is the line that must pass for any demo to be honest — it alone proves F-002, F-005,
INV-001, and the core A-001-testing journey UJ-001. If PH-02 also lands, INV-002 (the hard "never
show resolved as available" rule) and UJ-004 are covered. PH-03 (scraper aggregation, F-001),
PH-04 (alerts, F-004), PH-05 (multi-cat batch reporting, F-006), and PH-06 (account verification,
F-102) are all MVP-priority features per the PRD, but if time is the binding constraint, cut in this
order first: `PH-06` and `PH-05` (both added 2026-09-18, neither part of the original core demo
journey) → `TASK-014` (mobile push tap-through UI) → `TASK-009` (mobile mark-resolved UI, keeping
the backend enforcement live via API only) → all of PH-04 → all of PH-03. Never cut anything inside
PH-01 or the INV-002 enforcement task
(`TASK-007`) — those are the invariant floor. `PH-06` (account verification) cuts alongside `PH-05`
first if time is short, since it's an added trust-signal feature, not part of the core UJ-001 demo
path either. (Same line is derived from the checker's output in §5.)

## 3. Task ledger

Allowed status: `ready | in_progress | blocked | done | cut`. Rules the checker enforces:
- `Phase` names a row in §2; `Depends on` may only reference tasks in the **same or an earlier**
  phase.
- A task in a `pending` phase is `blocked` or `cut`. `ready` / `in_progress` require an `open`
  phase and only `done` dependencies. `done` requires an `open` or `passed` phase.
- `Verify` is the exact command that must **fail on the base commit** and pass when the task is
  done. Infra/docs tasks may use a build, boot or lint command.
- `Work ref` is `—` until claimed, then an exact branch/PR.
- `Gate / evidence` has an exact backticked command. `in_progress` carries `base: <sha> exit <n>`.
  `done` carries `verify: <sha> exit 0` **observed by the keeper's own re-run**, never the
  executor's claim.
- `Tests` is the deferred `TC-###` obligation: `TC-### pending` until it lands, then `TC-###
  landed: <sha>`, or `waived: <reason>` on a lead decision.
- One table, ordered by ID; rows never move; a second status list is never kept.

**Build approval is now recorded (§ header).** `TASK-001`..`TASK-004` have no unmet dependency and
are the first wave eligible for `ready` the moment the keeper claims them; every other task remains
`blocked` on its dependency chain until then. Status transitions are still keeper-only (R5) — this
plan edit records eligibility, not a status change.

| ID | Phase | Outcome / trace | Depends on | Owner | Write scope | Verify | Tests | Work ref | Status | Gate / evidence |
|---|---|---|---|---|---|---|---|---|---|---|
| TASK-001 | PH-01 | backend workspace scaffold + Jest test runner + standard error envelope middleware + `GET /health` (API-016) + rate-limiting middleware (concrete limits per `api-spec.md`); infra, TC-017/TC-018 | — | executor | `package.json, tsconfig.json, backend/` | `npm --prefix backend test -- --silent` | — | — | blocked | `npm --prefix backend test -- --silent` |
| TASK-002 | PH-01 | mobile (React Native) workspace scaffold + Detox e2e runner; infra | TASK-001 | executor | `mobile/` | `npm --prefix mobile test -- --silent` | — | — | blocked | `npm --prefix mobile test -- --silent` |
| TASK-003 | PH-01 | Facebook OAuth auth substrate (API-007 `POST /auth/facebook` incl. mandatory `debug_token` check + signup race handling + FB-signal verification, API-008 `GET /users/me`, API-009 `PATCH /users/me` profile edit w/ immutable-field rejection, API-012 `DELETE /auth/sessions` logout, API-015 `DELETE /users/me` account deletion, TC-008/TC-010/TC-011/TC-014/TC-019/TC-022); F-005/INV-001 supporting substrate, BR-010, BR-011, BR-012, BR-014, `ADR-0001` | TASK-001 | executor | `backend/src/auth` | `npm --prefix backend test -- auth` | TC-007 pending | — | blocked | `npm --prefix backend test -- auth` |
| TASK-004 | PH-01 | FB-profile identity-anchor validation gate; F-005, INV-001 | TASK-001 | executor | `backend/src/listings` | `npm --prefix backend test -- listings/identity-anchor` | TC-005 pending | — | blocked | `npm --prefix backend test -- listings/identity-anchor` |
| TASK-005 | PH-01 | manual submission endpoint + minimal feed read w/ keyset pagination + search (API-001/API-003), photo-upload validation (BR-013), same-session visibility; F-002, F-001, TC-013/TC-015/TC-016 | TASK-003, TASK-004, TASK-016 | executor | `backend/src/submissions, backend/src/listings` | `npm --prefix backend test -- submissions/submit.integration` | TC-002 pending | — | blocked | `npm --prefix backend test -- submissions/submit.integration` |
| TASK-006 | PH-01 | mobile Report-a-cat screen + confirmation screen; the UJ-001 core smoke walk; F-002, F-005 | TASK-002, TASK-005 | executor | `mobile/src/screens/ReportACat, mobile/e2e` | `npm --prefix mobile run e2e -- report-a-cat` | TC-002 pending | — | blocked | `npm --prefix mobile run e2e -- report-a-cat` |
| TASK-007 | PH-02 | status-transition endpoint + terminal-state guard (terminal set `{adopted, found, resolved}`, unconditional — no admin override, per the corrected Algorithm 1b, `decision-ledger.md` 2026-09-18 INV-002 fix); F-003, INV-002 | TASK-005 | executor | `backend/src/status/resolution.ts, backend/src/status/__tests__/resolution.test.ts, backend/src/status/__tests__/resolution-guard.test.ts` | `npm --prefix backend test -- status/resolution` | TC-003 pending | — | blocked | `npm --prefix backend test -- status/resolution` |
| TASK-008 | PH-02 | staleness sweep worker, never mutates status; F-003 | TASK-005 | executor | `backend/src/status/staleness.ts, backend/src/status/__tests__/staleness.test.ts` | `npm --prefix backend test -- status/staleness` | TC-003 pending | — | blocked | `npm --prefix backend test -- status/staleness` |
| TASK-009 | PH-02 | mobile "Mark resolved" action + status-aware feed rendering; F-003 | TASK-007, TASK-006 | executor | `mobile/src/screens/ReportACat/MarkResolved, mobile/src/screens/Feed` | `npm --prefix mobile test -- markResolved` | TC-003 pending | — | blocked | `npm --prefix mobile test -- markResolved` |
| TASK-010 | PH-03 | ingest-time cross-page/group dedup match; F-001 | TASK-004, TASK-005 | executor | `backend/src/ingestion/dedup.ts, backend/src/ingestion/__tests__/dedup.test.ts` | `npm --prefix backend test -- ingestion/dedup` | TC-001 pending | — | blocked | `npm --prefix backend test -- ingestion/dedup` |
| TASK-011 | PH-03 | attribution-preservation guard through ingest + dedup merge; INV-003 | TASK-010 | executor | `backend/src/ingestion/attribution.ts, backend/src/ingestion/__tests__/attribution.test.ts` | `npm --prefix backend test -- ingestion/attribution -t "INV-003"` | TC-N03 pending | — | blocked | `npm --prefix backend test -- ingestion/attribution -t "INV-003"` |
| TASK-012 | PH-04 | location opt-in/opt-out + push-token registration/unregistration endpoints (API-005/API-006/API-013/API-014, TC-012); F-004 | TASK-003 | executor | `backend/src/users` | `npm --prefix backend test -- users/location` | TC-004 pending | — | blocked | `npm --prefix backend test -- users/location` |
| TASK-013 | PH-04 | location-based alert fanout worker + delivery audit trail; F-004 | TASK-007, TASK-012 | executor | `backend/src/alerts` | `npm --prefix backend test -- alerts/geofence.integration` | TC-004 pending | — | blocked | `npm --prefix backend test -- alerts/geofence.integration` |
| TASK-014 | PH-04 | mobile push-token registration + notification tap-through to listing detail; F-004, UJ-003 | TASK-013, TASK-006 | executor | `mobile/src/notifications` | `npm --prefix mobile test -- pushNotificationHandler` | TC-004 pending | — | blocked | `npm --prefix mobile test -- pushNotificationHandler` |
| TASK-015 | PH-05 | multi-cat batch report endpoint w/ per-cat photo validation (API-010 `POST /listings/batch`); F-006, BR-009, BR-013 | TASK-004, TASK-005, TASK-016 | executor | `backend/src/reports` | `npm --prefix backend test -- reports/batch.integration` | TC-006 pending | — | blocked | `npm --prefix backend test -- reports/batch.integration` |
| TASK-016 | PH-01 | photo upload URL endpoint (API-011 `POST /uploads/photo-url`); F-002/F-006 supporting infra | TASK-001 | executor | `backend/src/uploads` | `npm --prefix backend test -- uploads/photo-url.integration` | TC-009 pending | — | blocked | `npm --prefix backend test -- uploads/photo-url.integration` |
| TASK-018 | PH-06 | account track-record verification sweep worker (F-102, TC-020) | TASK-005 | executor | `backend/src/verification` | `npm --prefix backend test -- verification/track-record-sweep` | TC-020 pending | — | blocked | `npm --prefix backend test -- verification/track-record-sweep` |
| TASK-019 | PH-06 | phone OTP verification endpoints (API-017/API-018, TC-021/TC-023); F-102, BR-016 | TASK-003 | executor | `backend/src/verification` | `npm --prefix backend test -- verification/phone-otp.integration` | TC-021 pending | — | blocked | `npm --prefix backend test -- verification/phone-otp.integration` |

### Known gaps in the Tests column (named, not silently forced green)

- `qa-test-plan.md`'s Traceability matrix has no dedicated `TC-###` for the API-005/API-006
  endpoint contracts themselves (`TC-004` proves the alert-fanout *logic* via fixture
  `UserLocation` rows, not the endpoints' own request/response shape) — `docs/validation-report.md`
  §4 already flags this as a non-blocking gap. `TASK-012`, `TASK-013`, and `TASK-014` all cite
  `TC-004` as their shared deferred obligation because it is the only automated `TC-###` that
  exists for F-004; a contract-level TC for API-005/API-006 is worth adding to `qa-test-plan.md` at
  scaffold time, not invented here.
- `TASK-008`'s Tests cell cites `TC-003 pending`, not the QA plan's own `TC-003b` (the staleness
  half of F-003) — `check-implementation-plan.py`'s Tests-cell format only accepts `TC-###`
  (digits) or `TC-N##`, not a lettered suffix, so `TC-003b` cannot be written into this column as
  the tool is written today. The distinction between the two `F-003` mechanisms (explicit
  resolution vs. time-based staleness) is preserved in `TASK-008`'s Outcome/trace text and in
  `qa-test-plan.md` itself, which remains the canonical owner of the real test ID; this is a
  plan-checker tooling gap, not a claim that `TASK-007` and `TASK-008` share one test.

## 4. Handoff and checkpoint protocol

At claim time the planner writes `docs/handoff/TASK-###.md` (template `handoff-packet.md`);
`check-handoff.py` must APPROVE; the keeper runs the Verify command on the base commit, records
`base: <sha> exit <n>` (it must fail) and `in_progress`; the executor runs the loop in
`process/build-loop.md` and returns evidence; the keeper **re-runs the command itself** at the
returned sha, then again on the current base after integration, appends the run event, and writes
`done` once the task's test obligation is closed or waived. Every trigger in
`process/checkpoint.md` runs that transaction. Contributors and executors never edit Status.

No task in this plan has been claimed yet — Build approval (§ header) is unset, so no packet has
been written and no task may leave `blocked`. The moment a named human lead records Build
approval, `TASK-001` through `TASK-004` (the wave with no unmet dependency) become the first
candidates for `ready`.

PR metadata (when PRs are used): exactly `Task: TASK-###` and the TC command. Reviewers report
observed evidence; they do not own a status transition. **Branch/review convention is
[assumption]:** no convention is recorded for this repo yet (current branch `jim/main`, no PR
requirement stated) — direct commits to the working branch are assumed until the product owner
says otherwise; flagged as open, not decided here.

## 5. Execution view (derived — paste the checker's output, do not hand-maintain)

```
APPROVE: docs/implementation-plan.md — 6 phase(s), 18 task(s); phase sequence, DAG, gating, and Build-First verify evidence are coherent (tests=deferred)
Honest stopping point: none passed yet · open: PH-01
PH-01 [open] — 7 task(s)
  wave 0: TASK-001(blocked)
  wave 1: TASK-002(blocked), TASK-003(blocked), TASK-004(blocked), TASK-016(blocked)
  wave 2: TASK-005(blocked)
  wave 3: TASK-006(blocked)
PH-02 [pending] — 3 task(s)
  wave 3: TASK-007(blocked), TASK-008(blocked)
  wave 4: TASK-009(blocked)
PH-03 [pending] — 2 task(s)
  wave 3: TASK-010(blocked)
  wave 4: TASK-011(blocked)
PH-04 [pending] — 3 task(s)
  wave 2: TASK-012(blocked)
  wave 4: TASK-013(blocked)
  wave 5: TASK-014(blocked)
PH-05 [pending] — 1 task(s)
  wave 3: TASK-015(blocked)
PH-06 [pending] — 2 task(s)
  wave 2: TASK-019(blocked)
  wave 3: TASK-018(blocked)
Ready now: none
Parallel-safety: no same-phase same-wave write-scope overlaps
```

Ready now: none in the Status column yet (status transitions are keeper-only, R5) · Build approval
**is** recorded (§ header) — `TASK-001` (wave 0), then `TASK-002`/`TASK-003`/`TASK-004`/`TASK-016`
(wave 1) are the first tasks the keeper may claim into `ready` · **Parallel-safe this wave:**
`TASK-002`, `TASK-003`, `TASK-004`, `TASK-016` (PH-01 wave 1) are disjoint-scope and safe to run in
parallel once `TASK-001` is `done` · **Cut line if time ends:** PH-01 (non-negotiable, now including
`TASK-016` photo upload — required for BR-001's photo field, not separable from the core slice) →
PH-02 → cut `PH-06` (account verification) and `PH-05` (batch reporting) then `TASK-014` then
`TASK-009` then all of PH-04 then all of PH-03 if time is the binding constraint (see §2).

## 6. Plan change log

| Timestamp / event | Phases / tasks changed | Why / evidence | Canonical docs reconciled |
|---|---|---|---|
| 2026-09-18 · hole audit + F-102 promotion | TASK-001/003/005/007/015/016 outcomes extended; PH-06 + TASK-018/019 added | Product owner asked to find and fix more backend holes before any code: found and fixed a real INV-002 design bug (terminal-state guard), closed a photo-upload ownership gap (BR-013), added a standard error envelope/health endpoint/concrete rate limits, resolved pagination/search, made the Facebook `debug_token` check mandatory, and promoted F-102 (verified badge) from Final to MVP as an automated (not manual) feature per direct product-owner criteria/effect choices | `docs/technical-design.md` (Algorithm 1b fix, Algorithms 7–9), `docs/data-model.md` (`PhotoUpload`, `PhoneVerification`, `User` verification fields), `docs/api-spec.md` (error envelope, API-016/017/018, rate limits, pagination/search), `docs/security-compliance.md` (T-010 mitigated, T-013/T-014 added), `docs/qa-test-plan.md` (TC-013..TC-023), `docs/prd.md` (BR-013..BR-016, F-102, UJ-007), `docs/frd.md` (F-102 section), `docs/seed/idea.md`, `docs/decision-ledger.md` |
| 2026-09-18 · complete backend logic in docs (2nd pass) | TASK-003 scope extended (logout, API-012); TASK-016 added (photo upload, API-011) | Product owner directed "finish ALL backend logic via docs before any code" — closed the remaining gaps a real backend needs: session mechanism (opaque hashed tokens, `Session` entity), logout/revocation, and a photo-upload URL contract (`data-model.md`'s object-store field previously had no endpoint) | `docs/technical-design.md` (Algorithms 4–5, sequence diagrams), `docs/data-model.md` (`Session` entity), `docs/api-spec.md` (API-011/API-012), `docs/security-compliance.md` (T-012, session-mechanism resolution), `docs/qa-test-plan.md` (TC-009/TC-010), `docs/prd.md` (BR-011), `docs/decision-ledger.md` |
| 2026-09-18 · Build approval + reconcile | Build approval recorded (header); TASK-003 outcome/verify updated (Facebook OAuth + profile edit); PH-05 + TASK-015 added | Product owner recorded Build approval and resolved two open decisions: auth mechanism → Facebook OAuth only, no forgot-password (`ADR-0001`); new MVP feature F-006 multi-cat batch reporting (`decision-ledger.md` §3) | `docs/prd.md`, `docs/frd.md`, `docs/system-design.md`, `docs/data-model.md`, `docs/api-spec.md`, `docs/security-compliance.md`, `docs/qa-test-plan.md`, `docs/seed/idea.md`, `docs/decision-ledger.md`, `docs/adr/ADR-0001` |
| 2026-09-18 · kickoff | PH-01..PH-04; TASK-001..TASK-014 | initial phase/task cut from PRD + system design + QA plan + delivery facts; PH-01 is the thinnest walkable UJ-001 slice per planner rules; PH-02 isolates the INV-002 hard invariant; PH-03/PH-04 split the two remaining MVP features (F-001 scraper aggregation, F-004 alerting) by risk/demoability | `docs/prd.md`, `docs/system-design.md`, `docs/qa-test-plan.md`, `docs/data-model.md`, `docs/api-spec.md`, `docs/frd.md`, `docs/technical-design.md`, `docs/security-compliance.md` |

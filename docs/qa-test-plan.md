# QA — Test Plan & Test Cases

> **Purpose:** how quality is proven, the **traceability sink**, and the **executor's fence**.
> Every `F-###` has ≥1 `TC-###`; every `INV-###` has ≥1 negative `TC-N##`. Each automated case
> names a path and an exact command, and is **named at spec time and landed as the deferred test
> in its own commit at phase end** (Build-First, `adr/ADR-0007`) — covering the edge cases the
> executor's SDD spec named (`adr/ADR-0008`), not invented from scratch once that reasoning is
> gone. This doc owns test intent; the plan links to case IDs and commands; CI owns raw artifacts.
> Traces back to PRD, system design.

## Test strategy
Cheapest layer that proves the behaviour and runs in seconds: **unit** (pure logic — dedup
matching, staleness windowing, status-transition guards, identity-anchor gating — the preferred
fence for F-001/F-003/F-005/all three `INV-###`) · **integration/contract** (submission →
Backend API → data-store round trip for F-002; geo-query fanout for F-004) · **e2e** (UJ-001 only,
the core demo journey — the smallest slice that would break the demo).

No repository is scaffolded yet (`docs/data-model.md`/`docs/system-design.md` migration notes:
greenfield build). All paths and commands below assume the **[assumption]** stack named in
`system-design.md` (backend: Node.js/TypeScript + Jest; mobile: React Native) — **to confirm at
scaffold**; every command is written so it fails today (no such file/suite exists) and passes once
the named task lands it, per Build-First.

## Test profile (context-scaled)
- **Browser UI present:** no *(from context `has_ui: true`, but the surface is a native/cross-platform
  mobile app, not a browser — the template's Playwright section below is adapted, not applicable
  as written; see that section)*
- **Core smoke journey:** UJ-001 → TC-001 (report-a-cat, the approved A-001-testing flow)
- **Fast gate command (per task, must run < ~60s):** `npm --prefix backend test -- --silent` (unit
  suite only) — **[assumption]** exact command to confirm once the backend is scaffolded
- **Full gate command (phase exit, default branch):** `npm --prefix backend test && npm --prefix
  mobile test && npm --prefix mobile run e2e:core` — **[assumption]**
- **CI budget / constraints:** **[assumption]** — no CI runner, secret manager, or time budget is
  named in the seed; nothing here should be read as a sourced target
- **Environments:** one primary (local/dev) environment for MVP; no staging/prod split is asserted
  by the seed — see `system-design.md` deployment topology open question

## Scope
### In scope
- All six MVP features (F-001..F-006) and all three invariants (INV-001..INV-003).
- UJ-001 as the core e2e smoke (the flow `usability.md` cleared and A-001 depends on).
- Facebook OAuth login/signup (API-007, `ADR-0001`) as concrete, testable infra now that the auth
  mechanism is resolved — no longer an `[assumption]` with nothing to test.

### Out of scope
- F-101..F-104 (final-product features) — not built at MVP; carried in the traceability matrix
  below only so they are never orphaned by the checker, with their test cases explicitly marked
  `deferred — post-MVP, not yet scaffolded`.
- Load/performance testing — no NFR target exists in the seed to test against (see `system-design.md`
  "Scaling strategy" open question).
- Anything requiring a live, non-test Facebook account/page (scraper integration against the real
  network) — covered instead by unit tests against captured fixture HTML/JSON, since scraping a
  live third party in CI is neither deterministic nor within this plan's control.

## Environments
Local/dev only at MVP (`[assumption]`, no staging/prod split specified). Deterministic setup:
tests run against an ephemeral/in-memory or containerized instance of the data store, seeded with
fixture data per test file; teardown drops the fixture schema. Secrets (push-provider credentials,
Facebook fixture auth if any) referenced by env var name only, never inlined in test files.

## Traceability matrix
<!-- EVERY F-### has a row. Consistency-checker T1: an F with no row is an orphan; a TC covering no real F is stray. -->

| F-ID | Feature | Test case ID(s) | Lowest proving level | Automation | Red as of |
|---|---|---|---|---|---|
| F-001 | Aggregated scraped feed, cross-page dedup | TC-001 | unit | planned | not yet |
| F-002 | Manual submission form | TC-002 | integration | planned | not yet |
| F-003 | Status tagging + stale suppression | TC-003, TC-003b | unit | planned | not yet |
| F-004 | Location-based missing-cat alert | TC-004 | integration | planned | not yet |
| F-005 | Facebook profile identity anchor | TC-005 | unit | planned | not yet |
| F-006 | Multi-cat batch report | TC-006 | integration | planned | not yet |
| F-101 | Photo-based lost/found matching | TC-101 | e2e | deferred — post-MVP, not scaffolded | N/A |
| F-102 | Verified rescuer/page badge program | TC-102 | integration | deferred — post-MVP, not scaffolded | N/A |
| F-103 | In-app messaging | TC-103 | e2e | deferred — post-MVP, not scaffolded | N/A |
| F-104 | Partner/API feed from rescue pages | TC-104 | contract | deferred — post-MVP, not scaffolded | N/A |

## Automation contract
<!-- The handoff packet copies the row for its TC verbatim. No "covered by CI" without a runnable command. -->

| Test ID | Level / tool | Test path | Command | Trigger | Artifact / evidence |
|---|---|---|---|---|---|
| TC-001 | unit + Jest | `backend/src/ingestion/__tests__/dedup.test.ts` | `npm --prefix backend test -- ingestion/dedup` | task | stdout / Jest report |
| TC-002 | integration + Jest + supertest | `backend/src/submissions/__tests__/submit.integration.test.ts` | `npm --prefix backend test -- submissions/submit.integration` | task | stdout / Jest report |
| TC-003 | unit + Jest | `backend/src/status/__tests__/resolution.test.ts` | `npm --prefix backend test -- status/resolution` | task | stdout / Jest report |
| TC-003b | unit + Jest | `backend/src/status/__tests__/staleness.test.ts` | `npm --prefix backend test -- status/staleness` | task | stdout / Jest report |
| TC-004 | integration + Jest | `backend/src/alerts/__tests__/geofence.integration.test.ts` | `npm --prefix backend test -- alerts/geofence.integration` | task | stdout / Jest report |
| TC-005 | unit + Jest | `backend/src/listings/__tests__/identity-anchor.test.ts` | `npm --prefix backend test -- listings/identity-anchor` | task | stdout / Jest report |
| TC-006 | integration + Jest + supertest | `backend/src/reports/__tests__/batch.integration.test.ts` | `npm --prefix backend test -- reports/batch.integration` | task | stdout / Jest report |
| TC-007 | integration + Jest + supertest | `backend/src/auth/__tests__/facebook-login.integration.test.ts` | `npm --prefix backend test -- auth/facebook-login.integration` | task | stdout / Jest report |
| TC-008 | unit + Jest | `backend/src/auth/__tests__/facebook-token-verify.test.ts` | `npm --prefix backend test -- auth/facebook-token-verify -t "T-010"` | task | stdout / Jest report |
| TC-009 | integration + Jest + supertest | `backend/src/uploads/__tests__/photo-url.integration.test.ts` | `npm --prefix backend test -- uploads/photo-url.integration` | task | stdout / Jest report |
| TC-010 | integration + Jest + supertest | `backend/src/auth/__tests__/logout.integration.test.ts` | `npm --prefix backend test -- auth/logout.integration -t "T-012"` | task | stdout / Jest report |
| TC-011 | integration + Jest + supertest | `backend/src/auth/__tests__/delete-account.integration.test.ts` | `npm --prefix backend test -- auth/delete-account.integration` | task | stdout / Jest report |
| TC-012 | integration + Jest + supertest | `backend/src/users/__tests__/location-opt-out.integration.test.ts` | `npm --prefix backend test -- users/location-opt-out.integration` | task | stdout / Jest report |
| TC-101 | e2e (deferred) | `mobile/e2e/photo-match.e2e.ts` (not yet created) | `npm --prefix mobile run e2e -- photo-match` | post-MVP phase | N/A — not scaffolded |
| TC-102 | integration (deferred) | `backend/src/badges/__tests__/verify.integration.test.ts` (not yet created) | `npm --prefix backend test -- badges/verify.integration` | post-MVP phase | N/A — not scaffolded |
| TC-103 | e2e (deferred) | `mobile/e2e/messaging.e2e.ts` (not yet created) | `npm --prefix mobile run e2e -- messaging` | post-MVP phase | N/A — not scaffolded |
| TC-104 | contract (deferred) | `backend/src/ingestion/__tests__/partner-feed.contract.test.ts` (not yet created) | `npm --prefix backend test -- ingestion/partner-feed.contract` | post-MVP phase | N/A — not scaffolded |
| TC-N01 | unit + Jest | `backend/src/listings/__tests__/identity-anchor.test.ts` | `npm --prefix backend test -- listings/identity-anchor -t "INV-001"` | task | stdout / Jest report |
| TC-N02 | unit + Jest | `backend/src/status/__tests__/resolution-guard.test.ts` | `npm --prefix backend test -- status/resolution-guard -t "INV-002"` | task | stdout / Jest report |
| TC-N03 | unit + Jest | `backend/src/ingestion/__tests__/attribution.test.ts` | `npm --prefix backend test -- ingestion/attribution -t "INV-003"` | task | stdout / Jest report |

## Test cases

### TC-001 — cross-page/group dedup collapses a re-scraped post
- **Covers:** F-001
- **Level:** unit
- **Preconditions / controlled data:** two fixture scraped posts with matching normalized content
  (same photo hash + near-identical description) from two different `ScrapeSource` fixtures.
- **Steps:** feed both fixture posts through the dedup-match function used by the ingestion
  pipeline.
- **Expected (EARS):** WHEN two scraped posts normalize to the same dedup fingerprint, the system
  SHALL create exactly one `Listing` and link the second `ScrapedPost` to it via `duplicate_of`
  rather than creating a second visible listing.
- **Automation:** `backend/src/ingestion/__tests__/dedup.test.ts` · `npm --prefix backend test --
  ingestion/dedup` · red on current codebase (no ingestion module exists yet).

### TC-002 — manual submission creates a visible listing in-session
- **Covers:** F-002
- **Level:** integration
- **Preconditions / controlled data:** a test user; a complete BR-001 payload (status, photo URL,
  location, description) plus a valid `fb_profile_url` (BR-002).
- **Steps:** POST the submission to the Backend API submission endpoint; then GET the feed as the
  same session.
- **Expected (EARS):** WHEN a user submits the Report-a-cat form with a status, photo, location,
  description, and an attached Facebook profile, the system SHALL create a new listing visible in
  the feed within the same session, tagged `source: manual`.
- **Automation:** `backend/src/submissions/__tests__/submit.integration.test.ts` · `npm --prefix
  backend test -- submissions/submit.integration` · red on current codebase (no submission
  endpoint exists yet).

### TC-003 — explicit resolution stops available/missing display
- **Covers:** F-003 (resolution half), INV-002
- **Level:** unit
- **Preconditions / controlled data:** a fixture `Listing` with status `missing`, submitted by a
  known test user.
- **Steps:** call the status-transition function as the submitter, transitioning to `resolved`;
  then query the default feed and the listing's status.
- **Expected (EARS):** WHEN a listing's status is changed to `resolved`/`adopted`/`found` by its
  submitter or an admin, the system SHALL stop displaying that listing as `available` or `missing`
  in any feed view from that point forward.
- **Automation:** `backend/src/status/__tests__/resolution.test.ts` · `npm --prefix backend test --
  status/resolution` · red on current codebase.

### TC-003b — stale listing suppressed from default feed without status mutation
- **Covers:** F-003 (staleness half)
- **Level:** unit
- **Preconditions / controlled data:** a fixture `Listing` with `status: available`, `updated_at`
  older than the staleness window (BR-005, `[assumption]` 30 days).
- **Steps:** run the staleness sweep function; query the default feed and re-read the listing's
  `status` field directly.
- **Expected (EARS):** WHEN a listing has had no activity for the staleness window, the system
  SHALL suppress it from the default feed view while its `status` field SHALL remain unchanged.
- **Automation:** `backend/src/status/__tests__/staleness.test.ts` · `npm --prefix backend test --
  status/staleness` · red on current codebase.

### TC-004 — missing-cat report triggers radius-based alert with delivery record
- **Covers:** F-004
- **Level:** integration
- **Preconditions / controlled data:** three fixture users with `UserLocation` rows at 1 km, 4 km,
  and 8 km from a fixture report's coordinates; radius fixed at 5 km (BR-006 default).
- **Steps:** create a `Listing` with `status: missing`; run the alert-fanout worker.
- **Expected (EARS):** WHEN a listing is created or updated to status `missing`, the system SHALL
  push a location-based alert to every user with location enabled within the configured radius and
  SHALL record that the alert was sent; the 1 km and 4 km users SHALL receive an `AlertDelivery`
  record, the 8 km user SHALL NOT.
- **Automation:** `backend/src/alerts/__tests__/geofence.integration.test.ts` · `npm --prefix
  backend test -- alerts/geofence.integration` · red on current codebase.

### TC-005 — submission without a resolvable Facebook link is refused
- **Covers:** F-005
- **Level:** unit
- **Preconditions / controlled data:** a BR-001-complete submission payload missing
  `fb_profile_url` (or containing an unresolvable URL).
- **Steps:** call the submission-validation function with the incomplete payload.
- **Expected (EARS):** IF a submission has no resolvable, visible Facebook profile/page link, THEN
  the system SHALL refuse to publish it to the feed.
- **Automation:** `backend/src/listings/__tests__/identity-anchor.test.ts` · `npm --prefix backend
  test -- listings/identity-anchor` · red on current codebase.

### TC-006 — batch report creates N independently tracked listings sharing one batch id
- **Covers:** F-006
- **Level:** integration
- **Preconditions / controlled data:** an authenticated test user; a payload with one
  `fb_profile_url` and 3 cat entries, each independently BR-001-complete.
- **Steps:** POST the batch to `/listings/batch`; then GET the feed and each returned listing's
  detail.
- **Expected (EARS):** WHEN a user submits a batch report with 2 or more cats, each satisfying
  BR-001, and one shared attached Facebook profile, the system SHALL create one independently
  status-tracked listing per cat, all linked to a single batch reference, visible in the feed within
  the same session.
- **Automation:** `backend/src/reports/__tests__/batch.integration.test.ts` · `npm --prefix backend
  test -- reports/batch.integration` · red on current codebase (no batch endpoint exists yet).

### TC-007 — Facebook OAuth login creates/reuses an account and issues a session
- **Covers:** API-007 (`ADR-0001` auth substrate)
- **Level:** integration
- **Preconditions / controlled data:** a fixture valid Facebook access token (stubbed Graph API
  response) resolving to an `fb_user_id` not yet present in the data store.
- **Steps:** POST the token to `/auth/facebook` twice (first call = signup, second call = login).
- **Expected:** the first call returns `201` with a new `User`; the second call returns `200`
  reusing the same `User.id` — no duplicate account is ever created for the same `fb_user_id`.
- **Automation:** `backend/src/auth/__tests__/facebook-login.integration.test.ts` · `npm --prefix
  backend test -- auth/facebook-login.integration` · red on current codebase (no auth endpoint
  exists yet).

### TC-008 — a Facebook token issued for a different app is rejected (T-010)
- **Covers:** `security-compliance.md` T-010 (security hardening, not a product `INV-###`, so this
  is a regular `TC-###`, not a `TC-N##`)
- **Level:** unit
- **Preconditions / controlled data:** a fixture Facebook token whose stubbed `debug_token` response
  reports an `app_id` different from Whiskr's own configured Facebook App ID.
- **Steps:** run the token-verification function against the fixture.
- **Expected:** verification fails with a distinct "wrong app" reason; no `User` row is created or
  reused.
- **Automation:** `backend/src/auth/__tests__/facebook-token-verify.test.ts` · `npm --prefix backend
  test -- auth/facebook-token-verify -t "T-010"` · red on current codebase.

### TC-009 — requesting a photo upload URL returns a usable presigned target
- **Covers:** API-011 (F-002/F-006 supporting infra)
- **Level:** integration
- **Preconditions / controlled data:** an authenticated test user; `content_type: image/jpeg`.
- **Steps:** POST to `/uploads/photo-url`; assert the response shape, then (against a stubbed
  object-store client, not a live vendor) confirm `photo_url` is well-formed and distinct from
  `upload_url`.
- **Expected:** `201` with `{ upload_url, photo_url, expires_at }`; `photo_url` is accepted as-is by
  API-003/API-010's `photo_url` field on a subsequent submission in the same test.
- **Automation:** `backend/src/uploads/__tests__/photo-url.integration.test.ts` · `npm --prefix
  backend test -- uploads/photo-url.integration` · red on current codebase (no uploads module exists
  yet).

### TC-010 — logout revokes the session; the revoked token is rejected on the next call (T-012)
- **Covers:** API-007/API-012, `security-compliance.md` T-012
- **Level:** integration
- **Preconditions / controlled data:** a logged-in test user holding a valid session token.
- **Steps:** call `DELETE /auth/sessions`; then call any authenticated endpoint (e.g., `GET
  /users/me`) reusing the same, now-revoked token.
- **Expected:** the logout call returns `204`; the subsequent call with the revoked token returns
  `401` — identical to an expired-token response, not a stale-cache `200`.
- **Automation:** `backend/src/auth/__tests__/logout.integration.test.ts` · `npm --prefix backend
  test -- auth/logout.integration -t "T-012"` · red on current codebase (no auth module exists yet).

### TC-011 — account deletion revokes sessions and orphans (not deletes) submitted listings (BR-012)
- **Covers:** API-015, BR-012
- **Level:** integration
- **Preconditions / controlled data:** a signed-in test user who has submitted at least one
  `Listing`, has an active `Session`, a `UserLocation` row, and a `PushToken` row.
- **Steps:** call `DELETE /users/me`; then (a) attempt any authenticated call reusing the
  now-deleted user's old session token, and (b) GET the listing the user submitted.
- **Expected:** the account-deletion call returns `204`; the reused session token now returns `401`
  (T-012 semantics — same as a revoked/expired session); `UserLocation`/`PushToken` rows for that
  user no longer exist; the listing is still returned by the feed/detail read, with
  `submitted_by: null`.
- **Automation:** `backend/src/auth/__tests__/delete-account.integration.test.ts` · `npm --prefix
  backend test -- auth/delete-account.integration` · red on current codebase.

### TC-012 — location opt-out deletes the subscription and flips the opt-in flag (API-013)
- **Covers:** API-013, F-004 (completes API-005's retention promise)
- **Level:** integration
- **Preconditions / controlled data:** a signed-in test user with an active `UserLocation` row
  (`location_opt_in: true`).
- **Steps:** call `DELETE /users/me/location`; then GET `/users/me`.
- **Expected:** `204` on the opt-out call; the profile read shows `location_opt_in: false`; a
  fixture `missing` listing created afterward within the old radius produces no `AlertDelivery` row
  for this user (the alert-fanout query no longer finds them).
- **Automation:** `backend/src/users/__tests__/location-opt-out.integration.test.ts` · `npm --prefix
  backend test -- users/location-opt-out.integration` · red on current codebase.

### TC-101 — photo-based match suggestion (deferred, post-MVP)
- **Covers:** F-101
- **Level:** e2e (planned)
- **Note:** not built at MVP (PRD non-goals). Placeholder exists only so `trace-ids.py` finds no
  orphaned `F-###`. Will be specced properly when F-101 is scheduled.

### TC-102 — verified badge grant (deferred, post-MVP)
- **Covers:** F-102
- **Level:** integration (planned)
- **Note:** not built at MVP; see TC-101 note.

### TC-103 — in-app message send/receive (deferred, post-MVP)
- **Covers:** F-103
- **Level:** e2e (planned)
- **Note:** not built at MVP; see TC-101 note.

### TC-104 — partner API feed ingestion (deferred, post-MVP)
- **Covers:** F-104
- **Level:** contract (planned)
- **Note:** not built at MVP; see TC-101 note.

## Invariant (negative) tests — the `INV-###` guardrails
<!-- Every INV-### gets ≥1 NEGATIVE test. A positive-only suite passes while an invariant is breached. T5 orphans. -->

| INV-ID | Invariant (must never…) | Negative test ID(s) | Automation |
|---|---|---|---|
| INV-001 | never publish a post without a visible Facebook profile/page link | TC-N01 | `npm --prefix backend test -- listings/identity-anchor -t "INV-001"` |
| INV-002 | never keep a listing shown available/missing past explicit resolution | TC-N02 | `npm --prefix backend test -- status/resolution-guard -t "INV-002"` |
| INV-003 | never strip the original poster's attribution/link on scrape/republish | TC-N03 | `npm --prefix backend test -- ingestion/attribution -t "INV-003"` |

### TC-N01 — asserts INV-001 is never violated
- **Covers:** INV-001
- **Assertion (EARS unwanted):** the system SHALL NEVER publish a listing (scraped or manual) that
  lacks a visible link back to a real Facebook profile or page.
- **Probe:** attempt to publish (a) a manual submission with `fb_profile_url` omitted, and (b) a
  scraped post whose source page/profile link cannot be resolved (dead/removed link fixture).
- **Expected:** in both cases, no `Listing` row is created and no feed-visible entry appears; the
  submission is rejected with a validation error, and the scraped post stays unlinked
  (`ScrapedPost.listing_id = null`).
- **Automation:** `backend/src/listings/__tests__/identity-anchor.test.ts` · `npm --prefix backend
  test -- listings/identity-anchor -t "INV-001"`.

### TC-N02 — asserts INV-002 is never violated
- **Covers:** INV-002
- **Assertion (EARS unwanted):** the system SHALL NEVER display a listing as `available` or
  `missing` WHILE its `resolved_at`/`StatusHistory` shows a prior explicit resolution by its
  submitter or an admin.
- **Probe:** resolve a fixture listing, then attempt every read path the feed exposes (default
  feed, filtered-by-status query, direct-by-id fetch) and check each for the forbidden
  `available`/`missing` display.
- **Expected:** the forbidden status label is absent from every read path once resolved; fails the
  test if any path still returns `available` or `missing` for that listing id.
- **Automation:** `backend/src/status/__tests__/resolution-guard.test.ts` · `npm --prefix backend
  test -- status/resolution-guard -t "INV-002"`.

### TC-N03 — asserts INV-003 is never violated
- **Covers:** INV-003
- **Assertion (EARS unwanted):** the system SHALL NEVER republish scraped content with the
  original poster's attribution/link stripped or altered.
- **Probe:** run a fixture scraped post (with a known `original_post_url`) through the full
  ingestion → normalize → publish pipeline, including the dedup path (a post that gets merged via
  `duplicate_of`).
- **Expected:** the resulting `Listing`/`ScrapedPost` pair still carries the exact original
  `original_post_url`; fails if the link is missing, rewritten, or dropped after dedup merge.
- **Automation:** `backend/src/ingestion/__tests__/attribution.test.ts` · `npm --prefix backend
  test -- ingestion/attribution -t "INV-003"`.

## Browser E2E with Playwright (conditional — omit if `has_ui: false`)
`has_ui: true`, but the exposed surface is a native/cross-platform **mobile** app, not a browser —
Playwright does not apply here as written. Adapted equivalent for the core smoke journey:
- **UJ-001 core smoke (mobile e2e):** `mobile/e2e/report-a-cat.e2e.ts` — **[assumption]** tool:
  Detox (React Native's standard e2e runner, paired with the React Native client choice in
  `system-design.md`); to confirm at scaffold. Covers the exact 3-click flow `usability.md` §2
  cleared: Home/Feed → `[Report a cat]` → attach Facebook profile → `[Submit report]` → Listing
  posted confirmation showing status, linked profile, and alert-sent text.
  Command: `npm --prefix mobile run e2e -- report-a-cat` · red on current codebase (no mobile app
  scaffolded yet).
- Isolated test data/device state per run; no shared login/session across e2e runs; any push/geo
  boundary stubbed rather than hitting a live provider (per "Out of scope").

## Acceptance criteria
One EARS criterion per F-### is stated in `docs/prd.md` "Acceptance criteria" section; each is the
`Expected` line of the matching `TC-###` above — not restated here.

## Regression plan
- **Per task:** run the fast gate (`npm --prefix backend test -- --silent`) plus the task's own
  `TC-###`/`TC-N##` before marking it done.
- **Per phase (exit):** run the full gate — backend suite + mobile unit suite + the UJ-001 core
  smoke e2e — on the default branch.
- **Before demo:** full gate + UJ-001 core smoke, green, on the commit being demoed.

## Exit criteria
- Every MVP `F-###` (F-001..F-006) has its `TC-###` passing on the default branch.
- Every `INV-###` (INV-001..INV-003) has its `TC-N##` passing on the default branch.
- UJ-001 core smoke e2e green on the commit being demoed.
- No invented pass-rate target beyond "all listed cases green" — no defect-count/severity budget is
  specified in the seed, so none is asserted here.
- F-101..F-104 test placeholders remain explicitly `deferred`; they are not required to pass for
  MVP exit and must not be silently marked done.

# System Design Document (HLD)

> **Purpose:** the HOW, architecture. High-level. Produced by the `architect` subagent from the
> PRD. Each tech choice carries a trade-off.
> Traces back to: PRD, SRS. Traces forward to: technical design, API spec, data model.

## Context diagram
```
                         +-----------------------------+
                         |   Facebook (pages/groups)    |
                         |   public NCR/GMA cat content  |
                         +---------------+---------------+
                                         | (scrape, read-only,
                                         |  public content only)
                                         v
+------------------+            +-------------------------+            +-------------------+
|  Mobile Client    | <--HTTPS--> |     Backend API        | <--------> |   Data store(s)   |
|  (iOS + Android)  |    REST     |  (auth, listings,      |   reads/   |  (listings, users, |
|  UJ-001..UJ-004    |            |   submissions, alerts)  |   writes   |   status history,  |
+---------+----------+            +-----------+-------------+            |   alert log)      |
          ^                                     |                        +-------------------+
          | push (F-004)                        | enqueue/dequeue
          |                                      v
          |                          +-------------------------+
          +--------------------------+  Ingestion + Alerting   |
                                     |  workers (scraper poll,  |
                                     |  status/staleness engine,|
                                     |  geo-alert fanout)       |
                                     +------------+-------------+
                                                  |
                                                  v
                                     +-------------------------+
                                     |  Push notification       |
                                     |  provider (APNs/FCM)     |
                                     |  [assumption — vendor]   |
                                     +-------------------------+
```
External actors: Facebook (read-only, public-content source for F-001; scrape target only, not an
integration partner at MVP — F-104 parks the formal-partnership alternative). Push provider is an
external dependency (PRD "Dependencies").

## Components & responsibilities

- **Mobile Client (iOS + Android)** — owns UJ-001..UJ-004 UI/UX per `usability.md`'s approved flow;
  submission form (F-002), feed browsing/filtering (F-001/F-003), push handling (F-004), status
  actions (mark resolved, F-003/INV-002 trigger). Depends on the Backend API contract only — no
  direct data-store or scraper access.
- **Backend API** — the single contract boundary (`exposes_api: true`). Owns auth (Facebook OAuth
  token verification against the Graph API, session issuance — `ADR-0001`), listing CRUD,
  submission intake/validation (BR-001, BR-002, BR-008, BR-009 → INV-001), status transitions
  (BR-003, BR-004 → INV-002), and read/filter/search endpoints for the feed. Depends on the data
  store and enqueues work for the ingestion/alerting workers; talks to Facebook only for (a) the
  scraper's read-only page/group polling and (b) verifying a client-supplied OAuth token against the
  Graph API at login — never to post, write, or act on a user's behalf on Facebook itself.
- **Scraper / Ingestion pipeline (worker)** — polls configured NCR/Greater-Manila-Area Facebook
  pages/groups on a schedule, normalizes posts, computes a dedup fingerprint (BR content match) to
  collapse cross-posts of the same listing (F-001), preserves the original post URL (BR-007 →
  INV-003), and rejects any post it cannot resolve to a visible profile/page link (BR-008 →
  INV-001) before it ever reaches the data store. Owns nothing the manual-submission path owns;
  both pipelines converge on the same `Listing` write path in the Backend API so status/staleness
  logic is not duplicated.
- **Manual-submission pipeline** — Backend API validation path only (no separate service): enforces
  BR-001/BR-002 synchronously at submit time so the mobile client can disable/enable Submit
  immediately (per the approved UX). Writes directly to the data store; no scrape/dedup step needed
  since the submitter is the original source.
- **Status / staleness engine (worker, scheduled)** — two distinct, non-conflicting mechanisms per
  PRD F-003: (a) explicit resolution — a status write by the submitter/admin through the Backend API
  that enforces INV-002 immediately and is recorded in status history; (b) time-based staleness —
  a scheduled sweep that flags listings past the staleness window (BR-005) and excludes them from
  default feed queries without altering their stored status. Never conflates (b) with a resolution.
- **Location-based alerting (worker)** — triggered on any listing write that results in status
  `missing` (F-004/BR-006); geo-queries opted-in users within the configured radius, calls the push
  provider, and records each delivery attempt (audit trail for "alert was sent" per F-004's
  acceptance criterion).
- **Data store** — durable state for listings, users, status history, scrape-source bookkeeping, and
  alert-delivery records (see `data-model.md`). Single store shared by API and workers to keep
  status/staleness logic consistent across both ingestion paths.

## Data flow

**Scraper ingestion pipeline (F-001):**
`Facebook page/group (public) → scheduled poll → normalize → dedup match against existing listings
→ profile/page-link resolution (BR-008 gate) → Backend API listing-write endpoint → data store →
feed read API → Mobile Client`.

**Manual-submission pipeline (F-002, UJ-001):**
`Mobile Client form (BR-001 fields + FB profile attach, Submit disabled until complete) → Backend
API submission endpoint (validates BR-001/BR-002 synchronously) → data store (source: manual) →
feed read API (same path as scraped listings) → Mobile Client confirmation screen → alerting
worker (if status = missing)`.

**Status / staleness engine:**
`Submitter/admin action (mark resolved) → Backend API status-transition endpoint → status-history
write + INV-002 enforcement (excluded from available/missing views immediately) `. Separately:
`scheduled sweep → staleness window check (BR-005) per listing → feed-visibility flag (no status
mutation) → default feed read API excludes flagged listings`.

**Location-based alerting (F-004, UJ-003):**
`Listing write with status=missing → alerting worker triggered → geo-query opted-in users within
radius (BR-006) → push provider dispatch → delivery record written → Mobile Client receives push →
user taps → Listing detail view (Backend API read)`.

**Mobile client / backend split:** the mobile client holds no business logic beyond form validation
UX (enable/disable Submit) and render/filter of API responses; every rule in "Business rules" (PRD)
is enforced server-side so the same guarantees hold regardless of client version — this is a
trade-off recorded below.

## Key technology choices + rationale

| Choice | Why | Trade-off | Alternative rejected |
|--------|-----|-----------|----------------------|
| Cross-platform mobile client (React Native) — **[assumption]**, no stack mandated in seed | `team_size: 1` (context.md); one codebase covers iOS+Android, matching solo-build capacity | Slightly weaker native-feel polish and access to some platform APIs vs. native | Two native codebases (Swift + Kotlin) — rejected: doubles solo-team build/maintenance load for no MVP-stage benefit |
| Server-enforced business rules (all BR-### live in Backend API, not client) | A single enforcement point for INV-001/INV-002/INV-003 regardless of client version or platform | Every client action needs a round trip (no offline-first submission) | Client-enforced validation with server as a mirror — rejected: risks a client bug silently violating an invariant (e.g., publishing without a Facebook link) |
| Polling-based scraper against public Facebook pages/groups (no official Graph API partnership at MVP) | No partnership exists yet (F-104 is explicitly parked as a *final*-product feature); public pages/groups are the only reachable source now | Fragile to Facebook UI/anti-scraping changes; direct kill-criterion risk (`idea.md` §9) | Official Facebook Graph API / partnership integration — rejected for MVP: requires business verification and page-owner cooperation that does not exist yet; explicitly deferred to F-104 |
| Two convergent write paths (scrape + manual) into one `Listing` entity/API, rather than separate tables/services | Keeps status/staleness/alerting logic single-sourced; avoids two divergent enforcement paths for INV-001/INV-002 | Slightly more validation branching in one endpoint (source-dependent required fields) | Fully separate "ScrapedListing" vs. "ManualListing" services — rejected: would duplicate INV-002/INV-003 enforcement and double the surface for staleness-engine bugs |
| Relational store with geo-query support (e.g., PostgreSQL + PostGIS) — **[assumption]**, not specified in seed | Needs both relational integrity (status history, FK to users/scrape sources) and radius queries (F-004/BR-006) | Requires a geo-capable extension/index, an added ops dependency | A separate geo-search service (e.g., dedicated spatial DB) alongside a relational store — rejected at MVP scale: two data stores is unjustified operational overhead for `team_size: 1` |
| Push via native platform providers (APNs/FCM) — **[assumption]**, no vendor named in seed | Standard, lowest-friction path for mobile push at MVP | External dependency outside the product's control (delivery, rate limits) | Building a custom polling-based in-app-only alert — rejected: defeats F-004's "pushed to nearby users" requirement, which implies out-of-app delivery |
| Facebook OAuth as the sole account login/signup mechanism (`ADR-0001`) | Reuses the same Facebook identity every user already needs for F-005's per-listing anchor; avoids building/securing a second password-based identity system for `team_size: 1` | No forgot-password path; account recovery depends entirely on the user's own Facebook account access; a user with no Facebook account cannot use Whiskr at all | Email + password (rejected: doubles the identity-system surface, needs its own reset-flow infra with no stated justification); phone OTP (rejected: SMS cost/infra, no stated justification) |
| Automated, criteria-based account verification (F-102) instead of manual review | No reviewer/queue exists at `team_size: 1`; three automatable signals (Facebook profile, in-app track record, phone OTP) give a real trust signal without a human in the loop | Weaker than true manual vetting (e.g., the Facebook-signal check is a coarse heuristic, not identity proof); no revocation path if a verified account later turns out to be abusive | Manual admin review (original F-102 scope, rejected — no reviewer exists); paid third-party ID-verification vendor (rejected — cost/scope disproportionate to MVP) |

## Integration points
- **Facebook (public pages/groups)** — read-only scrape target. Failure modes: page/group structure
  changes break the scraper (mitigated by the manual-submission pipeline as the A-001 fallback per
  `idea.md`/`validation.md`); explicit ban/takedown is a named kill criterion, not merely an
  operational risk — flagged, not silently absorbed.
- **Facebook OAuth / Graph API (token verification)** — a distinct integration surface from the
  scrape target above, even though both are Facebook: the Backend API verifies a client-supplied
  access token via a Graph API call at login (`ADR-0001`, API-007). Failure mode: a Graph API outage
  or rate-limit on the verification call blocks login/signup entirely (not just scraping) — this is
  a new availability dependency this decision introduces, named here rather than silently folded
  into the scrape-target row above.
- **SMS/OTP provider (F-102 phone verification)** — **[assumption]**, vendor unconfirmed, same
  treatment as the push-provider gap: external, credential-gated, and its failure must not block
  anything else (a user who never attempts phone verification is unaffected; account verification
  still reachable via the other two F-102 paths).
- **Push provider (APNs/FCM)** — **[assumption]**, vendor unconfirmed. Failure modes: delivery
  failure/rate-limiting must not block listing creation (alerting is fire-and-forget relative to the
  write path; a failed delivery is retried/logged, never blocks F-002's "visible in the feed within
  the same session" criterion).
- **Device location services** — client-side permission; absence degrades UJ-002/UJ-003 gracefully
  (feed still browsable without location; no crash on denial) rather than blocking core use.

## Deployment topology
- **[assumption]** — no infrastructure/cloud provider is named in the seed or context. Proposed for
  MVP at `team_size: 1` / `time_budget: 2w`: a single managed backend deployment (API + workers) in
  one region serving NCR/Greater-Manila-Area users (all in one metro timezone/region, no
  multi-region need argued anywhere in the seed), one managed relational database, one object
  store for photos, and platform push services. To confirm at scaffold — this is an operational
  choice, not a product requirement, and nothing in the seed constrains it further.
- Environments: a single production environment is implied by `time_budget: 2w` and `team_size: 1`;
  a staging/dev split is an open question (see below), not asserted here as fact.

## Scaling strategy
- **At-risk NFRs — none are specified in the seed.** No performance, availability, or concurrent-user
  target exists in `idea.md`, `context.md`, or `usability.md`. This is an **open question**, not a
  confident target: §5's "tens of thousands" size band is itself `[assumption]`-flagged in
  `idea.md`, so no scaling number here should be treated as sourced.
- Structurally, the design that most needs to scale first is the **geo-alert fanout** (F-004): a
  single missing-cat report can trigger a radius query and push fanout to every opted-in user
  nearby. At MVP volume (a solo build, no acquisition numbers yet) this is not expected to be a
  bottleneck, but the alerting worker is deliberately decoupled (async, queued) from the
  synchronous submission write path specifically so a slow/large fanout never blocks F-002's "visible
  in the feed within the same session" acceptance criterion.
- The scraper's poll frequency vs. Facebook's tolerance for automated access is the more likely
  near-term constraint (ties to the regulatory kill criterion) — not a scale question but a
  viability one, called out here because it shapes the same component.

## Trade-offs considered
- **Single relational store vs. store-per-pipeline** — chosen: single store (see table above); a
  candidate ADR (see report) given it is a real trade-off that is hard to reverse once scrapers and
  the API both depend on one schema.
- **Server-side rule enforcement vs. shared client/server validation** — chosen: server-side only,
  to keep INV-001/INV-002/INV-003 enforcement singular and auditable; candidate ADR.
- **Scraping public pages vs. waiting for a Facebook partnership before shipping** — chosen: ship
  with scraping now, park the partnership as F-104; this directly trades near-term reach for
  platform risk (the regulatory kill criterion in `idea.md` §9), and is the single most consequential,
  hardest-to-reverse choice in this document — candidate ADR.

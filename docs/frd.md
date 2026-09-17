# FRD — Functional Requirements Document

> **Purpose:** behavior detail. Exactly how the system must behave per feature, one level more
> precise than the PRD's EARS acceptance criteria — exact field validation, state transitions, edge
> cases. Grounded in `prd.md`'s `BR-###`/`INV-###` and `system-design.md`'s component boundaries.
> Traces back to: `prd.md` feature list (F-001..F-006, F-102, MVP only). Traces forward to: QA test
> cases.
> No new `F-###`/`BR-###` numbering — IDs below are reused verbatim from `prd.md`. Where this doc
> adds a rule not already an `BR-###` in the PRD, it is labeled `FRD-F0##-0#` (doc-local, not a PRD
> business rule) so it is never confused with a PRD-sourced ID.

## Per-feature functional requirements

### F-001 — Aggregated listings feed scraped from NCR + Greater Manila Area Facebook pages/groups

**Description**
The Scraper/Ingestion pipeline (worker) polls a configured allowlist of public NCR/Greater-Manila
Facebook pages/groups on a schedule, normalizes each post, computes a dedup fingerprint to collapse
cross-posts of the same underlying listing, and writes surviving posts through the same Backend API
listing-write endpoint the manual path uses (system-design "Scraper / Ingestion pipeline"; converges
on one `Listing` entity). The Backend API's feed read endpoint then serves one de-duplicated,
status-tagged list to the Mobile Client (F-001 EARS).

**Inputs**
| Name | Type | Source | Validation |
|---|---|---|---|
| Scrape source allowlist | config (list of URLs) | ops-provisioned, out-of-band | Each entry must be a public page/group URL reachable without authentication; non-public/auth-walled sources are rejected at config time, not at scrape time |
| Raw scraped post | photo, caption text, source post URL, page/group ID, post timestamp | scheduled poll of an allowlisted page/group | Must include a resolvable post URL and an identifiable authoring profile/page (feeds BR-008 gate) |
| Feed query params | region filter, status filter, pagination cursor | Mobile Client GET request | Region filter constrained to NCR + Bulacan/Cavite/Laguna/Rizal (PRD non-goals); invalid/out-of-scope region values ignored, not errored |

**Outputs**
| Name | Type | Destination | Format |
|---|---|---|---|
| Listing record | id, status, photo, location, description, `source: scraped`, `source_post_url`, `source_page_id`, `dedup_fingerprint`, `created_at` | Data store | Row per de-duplicated listing |
| Feed response | array of listings + pagination cursor | Mobile Client | JSON |

**Business rules**
- **BR-007** — every scraped listing retains a visible link to the original Facebook post.
- **BR-008** — a scraped post that cannot be resolved to a visible, working Facebook profile/page
  link SHALL NOT be published into the feed.
- **FRD-F001-01** — the dedup fingerprint is computed from normalized caption text + a photo
  perceptual hash + approximate geolocation. Two scraped posts matching on fingerprint within an
  **[assumption] 7-day** matching window collapse to one listing; the earliest-seen
  `source_post_url` becomes canonical, later matches are stored as duplicate references (audit
  only, never rendered as separate feed cards). *(No dedup algorithm or window is specified beyond
  "computes a dedup fingerprint (BR content match)" in system-design — confirm at scaffold.)*

**State transitions**
Not applicable at ingestion — a scraped post either becomes a new `Listing` (first fingerprint
match) or a duplicate reference against an existing one. Post-creation status transitions are
shared with F-003 below (same `Listing` entity, same rules, regardless of `source`).

**Edge cases**
- Same cat cross-posted to 3 groups same day → 1 listing, 2 duplicate references, only 1 feed card.
- A duplicate fingerprint match lands on a listing already `adopted`/`found`/`resolved` → the
  incoming scrape SHALL NOT reopen or mutate that listing's status (INV-002); it is logged as
  `duplicate_of_resolved` and discarded, never written as a new listing either.
- Source post edited or deleted on Facebook after scrape → INV-003 requires the stored link is
  retained as-is; no automated re-check of link liveness post-publish (see F-005 FRD-F005-02).
  \
- Scraper poll returns zero results (page down, anti-scrape block) → feed continues serving the
  last known-good state; a scrape failure never blocks or clears the feed (async, decoupled worker).
- A post already resolved to no visible profile at scrape time never reaches the data store (see
  Error handling) — it cannot later "become" a listing via a subsequent duplicate match.

**Error handling**
| Trigger | System response | User-facing message |
|---|---|---|
| BR-008 gate fails (no resolvable profile/page link) | Reject before write; log `source_post_url` + `reason: unresolvable_profile` for ops review | None — no end user initiated this write |
| Scrape source unreachable | Retry on next scheduled poll; failure recorded | None — feed unaffected, served stale-but-valid |
| Fingerprint match against a resolved/terminal listing | Discard as `duplicate_of_resolved`, no write | None |

---

### F-002 — Manual submission form (rescuers/finders/adopters post directly)

**Description**
The manual-submission pipeline is a Backend API validation path (no separate service): it enforces
BR-001/BR-002 synchronously at submit time so the Mobile Client can enable/disable the Report-a-cat
form's Submit control immediately (UJ-001, `usability.md` §1 approved flow). On success it writes
directly to the data store, tagged `source: manual`, and is served through the same feed read path
as scraped listings (no separate dedup step — the submitter is the original source).

**Inputs**
| Name | Type | Source | Validation |
|---|---|---|---|
| Status | enum: `Found` / `Lost` / `Adoptable` (form-facing subset) | Report-a-cat form | Required. **FRD-F002-01** — form value maps to the BR-003 stored enum: `Adoptable → available`, `Found → found`, `Lost → missing`. No other form-facing initial value is offered; `on_hold`/`adopted`/`resolved` are unreachable at creation, only via F-003 transitions. |
| Photo | image file | device camera/gallery attach | Required, non-empty. **[assumption]** accepted formats/max size (e.g. JPEG/PNG/HEIC, ≤10MB) not specified in the seed — confirm at scaffold. |
| Location | map pin (lat/long) or free-text address | form | At least one required. A pin must fall within the NCR + Bulacan/Cavite/Laguna/Rizal bounding region **[assumption — exact bounding box not specified]**; a free-text address is geocoded server-side before write (required input to F-004's radius query) — geocoding failure blocks submit. |
| Description | free text | form | Required, non-empty (≥1 character). No maximum specified in the seed — **[assumption]** none enforced at MVP. |
| Facebook profile | URL / attach-flow result | `[Attach Facebook profile]` action | Required. Must match a `facebook.com` profile/page URL pattern **and** resolve to a visible, working profile/page at submit time (mirrors BR-008's resolvability semantics for the manual path — see F-005). |

**Outputs**
| Name | Type | Destination | Format |
|---|---|---|---|
| Listing record | as F-001, plus `source: manual` | Data store | Row |
| Confirmation screen data | status tag, linked FB profile, alert-sent line (if status resolves to `missing`) | Mobile Client | Rendered confirmation view (UJ-001) |

**Business rules**
- **BR-001** — status, photo, location, description all required before Submit is enabled.
- **BR-002** — Submit stays disabled until a Facebook profile is attached.
- **FRD-F002-01** — status label→stored-value mapping (above).
- **FRD-F002-02** — Submit enable condition is a strict AND of all five BR-001/BR-002 fields; any
  one missing/invalid keeps Submit disabled — no partial-credit enabling.

**State transitions**
Creation only. A new manual listing enters exactly one of `{available, missing, found}` per
FRD-F002-01's mapping. All further transitions are governed by F-003.

**Edge cases**
- `Lost` selected → listing created at `missing` → synchronously enqueues the F-004 alerting job
  (async; does not block "visible in the feed within the same session," per F-002's EARS).
- Manual map-pin location does not require the device-level location *permission* used by
  UJ-002/UJ-003 (browsing/alerts) — pin-drop works even if the user has denied location access.
- No automated dedup on the manual path (system-design: "no scrape/dedup step needed since the
  submitter is the original source") — a user resubmitting the same cat twice produces two
  independent listings; accepted as-is at MVP, not treated as a defect.
- Network failure mid-submit → Backend API validates all fields before any data-store write; no
  partial/half-written listing is ever created.
- User backs out of the Facebook-attach flow → BR-002 gate never satisfies; no listing is created.

**Error handling**
| Trigger | System response | User-facing message |
|---|---|---|
| Required field missing (client-side) | Submit stays disabled; inline field error | Field-level hint (no server round-trip) |
| Server-side re-validation fails despite client showing Submit enabled (stale client) | 4xx, field-specific | Message keyed to the failing rule, e.g. BR-002 → "Attach a Facebook profile to submit." |
| Free-text address fails geocoding | Block submit | "We couldn't locate that address — try dropping a pin instead." |
| FB profile URL unresolvable | Block submit | "This Facebook profile/page link isn't accessible — attach a working link." |

---

### F-003 — Status tagging (available/on_hold/adopted/found/missing/resolved) + stale-listing suppression

**Description**
Two independent, non-conflicting mechanisms (system-design "Status/staleness engine"): **(a)
explicit resolution** — a synchronous status write via the Backend API's status-transition endpoint
by the listing's submitter or an admin, enforcing INV-002 immediately; **(b) time-based staleness**
— a scheduled sweep that flags listings whose activity has gone quiet past BR-005's window and
excludes them from the default feed view **without** mutating stored `status`. This is the feature
with the most state to specify precisely: 6 valid status values, a one-way terminal-state guard, and
a separately-tracked staleness flag that must never be conflated with an actual status change.

**Inputs**
| Name | Type | Source | Validation |
|---|---|---|---|
| `listing_id`, `new_status` | ID, enum (BR-003) | Mobile Client "Mark resolved"/status action (UJ-004) | `new_status` must be one of the 6 BR-003 values and must be a valid transition (see State transitions) |
| Actor | auth identity | request auth context | Must be the listing's original submitter or an admin (BR-004) |
| Staleness sweep trigger | scheduled job | internal scheduler | **[assumption]** interval not specified in the seed (e.g. daily) — confirm at scaffold |
| `last_activity_at` (per listing) | timestamp | data store | Updated on creation and on any accepted status write |

**Outputs**
| Name | Type | Destination | Format |
|---|---|---|---|
| `status_history` record | `listing_id`, `old_status`, `new_status`, `actor_id`, `timestamp` | Data store | Append-only audit row |
| `listing.status` | enum | Data store | Mutated only by (a) |
| `listing.stale_flag` | boolean | Data store | Mutated only by (b); independent field from `status` |
| Feed visibility | derived | Feed read API | Excludes `available`/`missing` past resolution (INV-002) **and** excludes `stale_flag = true` listings from the default view |

**Business rules**
- **BR-003** — status SHALL be one of: `available`, `on_hold`, `adopted`, `found`, `missing`,
  `resolved`. No other value is valid.
- **BR-004** — only the original submitter or an admin may change a listing's status.
- **BR-005** — no-activity for the staleness window (**[assumption] 30 days**, not specified in the
  seed) flags a listing stale and suppresses it from the default feed, without altering `status`.
- **INV-002** — the system SHALL NEVER show a listing as `available`/`missing` past the point it was
  marked `resolved`/`adopted`/`found` (per F-003's EARS, which names all three as resolving events).

**State transitions**
Terminal set (INV-002): `{adopted, found, resolved}`. Once a listing enters any terminal state, no
transition back to `available` or `missing` is permitted — **enforced with no override, including
for admins**; correcting a mistaken terminal marking requires creating a new listing, not reverting
this one (INV-002 is a one-way hard rule, not a soft default).

| From | To | Allowed? | Notes |
|---|---|---|---|
| `available` | `on_hold` | Yes | Adoption pending |
| `on_hold` | `available` | Yes | Adoption fell through — reopen |
| `on_hold` | `adopted` | Yes | Adoption completes |
| `available` | `adopted` | Yes | Direct, skipping `on_hold` |
| `missing` | `found` | Yes | Cat found/reunited |
| `missing` | `resolved` | Yes | Closed without "found" framing — **[assumption]**: exact semantic split between `found` and `resolved` for a missing-cat report is not specified in the seed; FRD treats `found` as "reunited" and `resolved` as a catch-all closure, also reachable from `adopted`/`found` for feed-suppression uniformity. |
| `adopted` / `found` | `resolved` | Yes | Catch-all closure alias |
| any of `{adopted, found, resolved}` | `available` or `missing` | **No — rejected unconditionally** | INV-002 guard |
| `missing` | `on_hold` | **No** | `on_hold` is adoption-pending semantics; does not apply to the lost/found flow — **[assumption]** |
| `available`/`on_hold`/`missing` | (unchanged, sweep only) | n/a | Governed by (b), not this table |

**Edge cases**
- A listing crosses the staleness window the same day it is explicitly resolved → explicit
  resolution takes precedence; `stale_flag` becomes irrelevant once the listing is terminal (already
  excluded from `available`/`missing` views).
- Concurrent status writes (submitter and admin act near-simultaneously) → last write by timestamp
  wins; both pass the same BR-004 check independently. **[assumption/open question]** — no
  optimistic-lock or conflict policy is specified in the seed.
- Admin attempts to revert `adopted → available` (e.g., to fix a mistake) → **rejected**, per the
  terminal-state guard; must surface a clear error, never a silent no-op.
- `last_activity_at` reaches exactly the 30-day boundary → **[assumption]** boundary is inclusive
  (`>= 30 days` flags stale); exact cutoff semantics to confirm at scaffold.
- A status write (any accepted transition) clears `stale_flag` and requires the full window to
  re-accrue before the listing can be re-flagged — the sweep is idempotent and never double-flags.
- Non-submitter, non-admin user attempts a status change → BR-004 rejects; listing state unchanged.

**Error handling**
| Trigger | System response | User-facing message |
|---|---|---|
| Invalid transition (see table) | 409/422, no write | "This listing has already been marked *&lt;status&gt;* and can't be changed." |
| BR-004 violation | 403, no write | "Only the original reporter or an admin can update this listing's status." |
| Staleness sweep crashes mid-run | Resumable/idempotent; next scheduled run recovers | None (internal) |

---

### F-004 — Location-based missing-cat alert pinned to an area, pushed to nearby users

**Description**
The location-based alerting worker triggers on a listing write that results in `status = missing`
(system-design "Location-based alerting"), geo-queries opted-in users within the configured radius
(BR-006), dispatches push via the platform provider (APNs/FCM, **[assumption]** vendor unconfirmed),
and records each delivery attempt so "alert was sent" (F-004 EARS) is auditable.

**Inputs**
| Name | Type | Source | Validation |
|---|---|---|---|
| `listing.location` | lat/long | resolved at F-002 submit time (pin or geocoded address) | Must be non-null; a listing with an unresolved location never enters the radius query (see Edge cases) |
| Status-transition event | `new_status = missing` | F-002 creation or F-003 transition | Only fires on the transition **into** `missing`, not on every subsequent write while `status` remains `missing` (FRD-F004-03 below) |
| Radius | km, default 5 | BR-006 | **[assumption]** fixed at MVP, not user-configurable; radius customization UI explicitly out of scope per PRD |
| Recipient pool | users with location enabled | data store (last-known device location) | Users without location permission granted are excluded entirely — "no location, no alert" |

**Outputs**
| Name | Type | Destination | Format |
|---|---|---|---|
| Push notification | title, body, deep link to listing | Push provider → device | Provider-native payload |
| `alert_delivery` record | `listing_id`, `user_id`, `sent_at`, `delivery_status` (`sent`/`failed`/`pending`) | Data store | Row per recipient |
| Confirmation text | "Nearby users have been alerted (5 km radius)" | Mobile Client (UJ-001 confirmation) | Static copy per `usability.md` §1 |

**Business rules**
- **BR-006** — a `missing` report triggers an alert to users within the default 5 km radius.
- **FRD-F004-01** — the radius query uses each user's **last-known** device location at
  alert-trigger time, not continuous/live tracking — **[assumption]**, no continuous-tracking
  requirement is specified anywhere in the seed.
- **FRD-F004-02** — alert dispatch is fire-and-forget relative to the write path (async, queued);
  F-002's "visible in the feed within the same session" criterion never blocks on fanout completion.
- **FRD-F004-03** — "alert was sent" is satisfied by an `alert_delivery` record reaching
  `delivery_status != pending` (handed to the push provider), **not** by confirmed on-device
  receipt/read — the system makes no delivery-confirmation guarantee to the submitter.

**State transitions**
Not a stateful entity itself; the trigger is the F-003 transition *into* `missing`. Re-entry into
`missing` from a terminal state is impossible (F-003 guard), so a listing can trigger this fanout at
most once via creation and, per the state table, cannot re-trigger it via a later transition —
**[assumption/open question]**: this narrows the PRD's literal "created **or updated** to status
`missing`" wording in F-004's EARS, since F-003's transition table has no path back into `missing`
once a listing leaves it. Flagged as a cross-doc point to confirm, not silently resolved.

**Edge cases**
- Zero users within radius → the job still runs and produces zero `alert_delivery` rows; this is not
  a failure. **[open question]** whether the UJ-001 confirmation copy should still unconditionally
  claim "nearby users have been alerted" when the recipient count is zero is not resolved here.
- Push provider outage → deliveries marked `failed`/retried per system-design ("retried/logged,
  never blocks"); retry count/backoff policy is **[assumption/open question]**, unspecified.
- User has location permission on but OS-level notification permission off → server cannot reliably
  know this; the send is still attempted and recorded `sent` per FRD-F004-03's semantics.
- Listing edited (e.g., description changed) while `status` remains `missing` → **does not**
  re-trigger a second fanout (FRD-F004-03) — prevents repeated edits from spamming nearby users.
- `listing.location` missing/null at trigger time (e.g., geocoding never completed) → the job SHALL
  NOT run a radius query against a null location; it skips with a logged reason. The submission
  itself still succeeds — an alerting failure never blocks F-002.

**Error handling**
| Trigger | System response | User-facing message |
|---|---|---|
| Geo-query failure (data-store error) | Job retried by worker infra; submission already committed, unaffected | None |
| Push provider auth/config failure | All deliveries in batch marked `failed`; logged for ops | None beyond the standard confirmation copy — no delivery guarantee is promised |
| Null/unresolved listing location | Skip radius query, log reason | None — submission still succeeds |

---

### F-005 — Every post requires a linked, visible Facebook profile as an identity anchor

**Description**
Cross-cutting requirement enforced at both ingestion boundaries — BR-002 (manual, Submit-gate, see
F-002) and BR-008 (scraped, publish-gate, see F-001) — both of which exist to satisfy INV-001. This
section is the single source of truth for the exact "resolvable, visible" check both paths share, so
it is defined once rather than duplicated inconsistently in F-001/F-002.

**Inputs**
| Name | Type | Source | Validation |
|---|---|---|---|
| FB profile/page reference | URL | Manual: attach-flow result (F-002). Scraped: extracted authoring profile/page during the normalize step (F-001). | See FRD-F005-01 |

**Outputs**
| Name | Type | Destination | Format |
|---|---|---|---|
| `listing.fb_profile_url` | URL, required non-null on any published listing | Data store | Stored on every `Listing` row regardless of `source` |
| Visible link | rendered UI element | Every feed card and listing detail view | Tappable, always shown (never hidden behind a "view source" toggle) |

**Business rules**
- **BR-002** — manual path: Submit disabled until attached (see F-002).
- **BR-008** — scraped path: unresolvable posts never published (see F-001).
- **INV-001** — the system SHALL NEVER publish a post lacking a visible link to a real FB
  profile/page. Both BR-002 and BR-008 are the two enforcement points for this single invariant.
- **FRD-F005-01** — "resolvable, visible" means: (a) the URL matches a `facebook.com` profile-or-page
  path pattern, **and** (b) at check time the URL returns a reachable, non-deleted profile/page (not
  a login-wall-only response or a removed-content response). The exact reachability check mechanism
  (HTTP HEAD/GET probe vs. an official API call) is **[assumption/open question]** — unspecified in
  the seed, and system-design notes no official Graph API partnership exists at MVP (F-104 parks
  that), so an unauthenticated probe is the only currently-available method to confirm at scaffold.
- **FRD-F005-02** — this check runs once, at submit/scrape time only. There is no scheduled
  re-validation that a previously-attached link stays live; a link going dead post-publish does
  **not** retroactively unpublish the listing — **[assumption]**, distinct from and consistent with
  F-001's "dead link post-scrape" edge case.

**State transitions**
Not applicable — this is a gate checked once per listing at creation, not a stateful field.

**Edge cases**
- A manual submitter attaches a Facebook **Page** rather than a personal profile. BR-002's wording
  says "Facebook profile" specifically, while BR-007/BR-008 use "profile/page" interchangeably for
  the scraped path. **FRD treats "profile or page" as interchangeable for both paths** for
  consistency with INV-001's own wording ("real Facebook profile **or page**") — this is a
  wording inconsistency inside the PRD being resolved here, not silently ignored; flagged as an open
  question to confirm the PRD's BR-002 text is not meant to exclude pages.
- A scraped post's authoring page **is** the profile/page link itself (self-referential) — trivially
  satisfies BR-008 with no additional cross-check.
- Two unrelated listings (one manual, one scraped) link the same FB profile/page — allowed; there is
  no uniqueness constraint on `fb_profile_url` (a profile is not a user account in this system) —
  **[assumption]**.

**Error handling**
Delegates to F-002's error handling (manual path: unresolvable-link submit block) and F-001's error
handling (scraped path: BR-008 publish-time rejection) — not duplicated here to avoid two sources of
truth for the same failure message.

### F-006 — Multi-cat batch report (2+ cats found/lost together)

**Description**
An additive variant of F-002's manual-submission path: the same Backend API validation logic runs
once per cat entry in the batch, plus one shared Facebook-anchor check (BR-009), all inside a single
all-or-nothing transaction. Each cat becomes its own `Listing` row with its own independent
`status`/`StatusHistory` lifecycle (F-003 applies per cat, unchanged); the only new concept is
`ReportBatch`, a thin grouping reference with no lifecycle of its own.

**Inputs**
| Name | Type | Source | Validation |
|---|---|---|---|
| Facebook profile | URL / attach-flow result | `[Attach Facebook profile]` action, once per batch | Same resolvability check as F-005/FRD-F005-01; validated once, reused for every cat in the batch |
| `cats[]` | array (min length 2) | Report-a-batch form | Each entry independently satisfies BR-001 (status, photo, location, description) exactly as F-002's single-cat form does; fewer than 2 entries is rejected as a plain single-cat submission belongs on F-002/API-003 instead |

**Outputs**
| Name | Type | Destination | Format |
|---|---|---|---|
| `ReportBatch` record | `id`, `submitted_by`, `created_at` | Data store | One row per batch submission |
| `Listing` record (× N) | as F-002, plus `batch_id` set to the new `ReportBatch.id` | Data store | One row per cat, each with its own `FacebookAnchor` row carrying the same `fb_profile_url` |
| Confirmation screen data | N listing cards, one per cat, each with its own status tag and the shared linked FB profile | Mobile Client | Rendered confirmation view (UJ-001 extended) |

**Business rules**
- **BR-009** — batch requires ≥2 cats; each independently satisfies BR-001; one shared
  `fb_profile_url` validated once for the whole batch (BR-002/BR-008 semantics, run once).
- **FRD-F006-01** — the batch write is all-or-nothing: if the shared Facebook-anchor check fails, or
  any single cat entry fails BR-001, **no** `Listing` in the batch is created — never a partial batch
  (mirrors F-002's "no partial/half-written listing" rule, extended to N rows in one transaction).
- **FRD-F006-02** — each cat's `kind` (adoption/lost/found) is independent within the same batch — a
  batch MAY mix, e.g., two `lost` cats and one `found` cat reported together; there is no rule
  requiring every cat in a batch to share the same `kind`.

**State transitions**
Creation only, identical per-cat semantics to F-002 (each `Listing` enters exactly one of
`{available, missing, found}` per FRD-F002-01's mapping, independently). `ReportBatch` itself has no
state/lifecycle — it is a grouping reference, never mutated after creation.

**Edge cases**
- One cat in the batch is `lost` (→ `missing`) → that cat's own F-004 alert fanout fires
  independently; sibling cats in the same batch with a different `kind` do not trigger an alert.
- A submitter later resolves one cat in a batch (F-003/UJ-004) → only that cat's `Listing` transitions;
  sibling listings in the same `ReportBatch` are unaffected (INV-002 is enforced per listing, not per
  batch).
- Exactly 1 cat submitted to the batch endpoint → rejected (BR-009's "at least 2" floor); the client
  should route a single cat through F-002/API-003 instead, not the batch endpoint.
- Network failure mid-batch-submit → the whole transaction rolls back; zero listings are created (no
  N-1-of-N partial batch), consistent with FRD-F006-01.

**Error handling**
| Trigger | System response | User-facing message |
|---|---|---|
| Fewer than 2 cats in the batch | 400, no write | "A batch report needs at least 2 cats — use Report a cat for one." |
| Any cat entry missing a BR-001 field | 400, no write (whole batch rejected) | Field-level hint naming which cat entry failed |
| Shared FB profile URL unresolvable | 422, no write (whole batch rejected) | "This Facebook profile/page link isn't accessible — attach a working link." |

---

### F-102 — Automated account verification badge (promoted from Final, 2026-09-18)

**Description**
Three independent, automated checks — no admin review, no manual queue. Any one passing sets its
own timestamp on `User`; `is_verified` is derived as "any of the three is set" (`data-model.md`).
Display-only (BR-015): read by the profile/feed rendering, never by any write-path authz check.

**Inputs**
| Name | Type | Source | Validation |
|---|---|---|---|
| Facebook profile fields | `name`, `picture` | resolved at API-007 signup, same Graph API call as login | FRD-F102-01 (below) — checked once, at signup only, never re-evaluated |
| Account tenure + listing count | `User.created_at`, count of `Listing WHERE submitted_by = user.id` | daily sweep (Algorithm 8) | ≥30 days AND ≥3, per BR-014's "automated only" rule — no human judgment call on which listings "count" |
| Phone number + OTP | E.164 phone, 6-digit code | API-017 (start) / API-018 (confirm) | FRD-F102-02 (below) |

**Outputs**
| Name | Type | Destination | Format |
|---|---|---|---|
| `User.fb_signals_verified_at` / `track_record_verified_at` / `phone_verified_at` | timestamp, nullable | Data store | Set once, never unset (a criterion, once met, stays met — no "un-verification" path exists) |
| Verification badge | derived boolean + which source(s) | Mobile Client (profile, listing cards) | Rendered UI element, not a raw API contract concern beyond `is_verified` in API-008/009's response |

**Business rules**
- **BR-014** — automatic only, never directly settable.
- **BR-015** — display-only, never a submission/rate-limit gate.
- **BR-016** — a confirmed phone number is unique across accounts.
- **FRD-F102-01** — the Facebook-signal check, exactly: `picture.data.is_silhouette == false` (a
  real, uploaded photo — Facebook's Graph API reports this boolean directly, no heuristic needed)
  **AND** `name` contains a space character (a two-plus-word name — a weak heuristic, explicitly
  named as weak, not a strong identity check; this is the ceiling of what's checkable without
  Facebook App Review for extended permissions). Checked once, inline, during `loginWithFacebook()`
  at first signup only.
- **FRD-F102-02** — OTP flow: API-017 generates a 6-digit code, hashes it (`PhoneVerification.
  otp_hash`), sends the raw code via SMS, and starts a 10-minute expiry. API-018 checks the
  submitted code's hash against the stored hash; a match within the expiry window (and before 5
  wrong attempts) sets `PhoneVerification.verified_at` and `User.phone_number`/`phone_verified_at`
  in the same transaction. BR-016's uniqueness constraint means a phone number already confirmed on
  another account causes API-018 to fail with a distinct "phone already in use" reason, not a
  generic error.

**State transitions**
Each of the three timestamp fields is one-way: null → set. None ever reverts to null except via
full account deletion (BR-012). There is no "revoke verification" mechanism at MVP — **[assumption/
open question]**, worth adding once a moderation/abuse-reporting mechanism exists (parked, similar
in spirit to F-101/103/104 — not built here).

**Edge cases**
- A user signs up, fails the Facebook-signal check (default silhouette photo), then later changes
  their Facebook profile photo → **not re-checked** (FRD-F102-01 explicitly runs once, at signup);
  they can still reach verified status via the track-record or phone path instead.
- A user hits 5 wrong OTP attempts → `PhoneVerification` row is exhausted; API-017 must be called
  again to start a fresh cycle (new row, new code) — **[assumption]** no cooldown period between
  cycles beyond the 10-minute expiry of the exhausted one; a per-phone-number rate limit is worth
  adding at scaffold (ties to `security-compliance.md`'s rate-limiting section).
- Two users race to confirm the same phone number → the `User.phone_number` unique constraint
  (partial index, `data-model.md`) makes the second `UPDATE` fail; API-018 catches this as "phone
  already in use," not a 500.
- Account deletion after verification → all three timestamps are deleted with the account (BR-012);
  a phone number freed this way can be re-verified on a different account afterward (the uniqueness
  constraint only prevents *simultaneous* reuse).

**Error handling**
| Trigger | System response | User-facing message |
|---|---|---|
| OTP code wrong | 400, `attempt_count` incremented | "That code doesn't match — N attempts left." |
| OTP expired | 400, no increment (already unusable) | "That code expired — request a new one." |
| Phone already confirmed on another account | 409 | "That phone number is already verified on another account." |
| 5 wrong attempts reached | 429 on further confirm attempts against that row | "Too many attempts — request a new code." |

---

## Open questions
_(Surfaced while writing this doc; none are new PRD-level facts — all trace to an existing
`[assumption]` in `prd.md`/`system-design.md`, or are FRD-local precision gaps this doc had to make
an explicit call on.)_
- **F-006** — whether a batch's cats may mix `kind` values (adoption/lost/found) freely is resolved
  here as "yes, independent per cat" (FRD-F006-02); flagged since the PRD itself does not spell this
  out explicitly.
- **F-003** — exact semantic split between `found` and `resolved` for a `missing`-report closure is
  assumed, not sourced; the terminal-state guard's "no override, including for admins" strictness is
  an FRD-level call the PRD's INV-002 wording supports but does not spell out this precisely.
- **F-003** — no concurrent-write conflict policy (optimistic lock, last-write-wins) is specified.
- **F-004** — F-003's transition table implies a listing can enter `missing` at most once
  (creation), which narrows F-004's EARS "created or updated to status `missing`" wording; needs
  confirmation this narrowing is intended rather than a gap in the F-003 transition design.
- **F-004** — zero-recipient alert fanout still shows the "nearby users have been alerted" copy to
  the submitter; whether that copy should change when the recipient count is zero is unresolved.
- **F-005** — reachability-check mechanism for a FB profile/page link (no Graph API partnership at
  MVP) is unspecified; an unauthenticated probe is assumed pending scaffold-time confirmation.
- **F-005** — BR-002's "profile" vs. BR-007/BR-008/INV-001's "profile or page" wording is treated as
  interchangeable here; flagged for the PRD owner to confirm or correct at source.
- Staleness sweep interval (BR-005 mechanics) and push-retry backoff policy (F-004) are both
  unspecified in the seed and carried forward as **[assumption]** rather than invented as fact.

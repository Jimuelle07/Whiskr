# API Specification

> **Purpose:** contracts between components/services. Use a machine-readable spec format
> (OpenAPI, GraphQL SDL, protobuf) where applicable and link it here.
> Traces back to: system design, technical design.

## Overview

The Backend API is the single contract boundary the Mobile Client (iOS + Android) depends on
(`system-design.md` — "Backend API ... exposes_api: true"). It owns listing read/search/filter
(F-001), submission intake (F-002), status transitions (F-003/INV-002), location-based alert
subscription and push-token registration (F-004), and the identity anchor / account layer that
F-005 and INV-001 depend on. The scraper and alerting workers write to the same data store through
this API's listing-write path (`system-design.md` — "both pipelines converge on the same `Listing`
write path"); this spec covers the mobile-facing surface only.

- **Base URL / namespace:** `[assumption]` — no domain/host is named in the seed; proposed
  `https://api.whiskr.app/v1` pending the deployment topology decision in `system-design.md`.
  To confirm at scaffold.
- **Format:** JSON request/response bodies; a full OpenAPI 3.x document is not produced in this
  batch — this Markdown contract is authoritative until one is scaffolded.
- **Machine-readable spec:** none exists yet — `[assumption]`, flagged as an open question below.

## Authentication & authorization

- **Mechanism:** `[assumption]` — `data-model.md`'s `User.auth_identifier` is itself flagged
  `[assumption]` ("phone/email/OAuth identifier; auth mechanism not specified in seed"). This spec
  assumes a bearer-token session issued by API-007 and sent as `Authorization: Bearer <token>` on
  every authenticated call. The identifier scheme (phone OTP vs. email vs. OAuth) is unresolved —
  see Open questions.
- **Public (no auth) endpoints:** API-001, API-002 — the feed must be browsable without an account,
  consistent with `system-design.md`'s "feed still browsable without location permission" (device
  location and account auth are separate gates; browsing needs neither). `[assumption]` — the seed
  never states whether an account is mandatory just to browse.
- **Authenticated endpoints:** API-003 through API-006, API-008 — creating a listing, changing
  status, or registering a location/push token requires a resolved `User.id`, since these actions
  are attributed via `Listing.submitted_by`, `StatusHistory.changed_by`, `UserLocation.user_id`,
  and `PushToken.user_id` respectively (`data-model.md`).
- **Role gate:** `User.is_admin` (`data-model.md`) gates BR-004 — changing a status on a listing you
  did not submit. Enforced server-side only (`system-design.md` — "Server-enforced business
  rules"), never mirrored/trusted client-side.
- **F-005 / INV-001 note:** the identity anchor INV-001 enforces is per-*listing* (`FacebookAnchor`
  row, a visible Facebook profile/page link), not per-*account*. Account auth (API-007/API-008)
  answers "who is calling the API"; `FacebookAnchor.fb_profile_url` on the submission body
  (API-003) answers "what public identity is this listing anchored to." The two are independent —
  a logged-in caller still must supply a Facebook profile link per listing, or the write is
  rejected (BR-002/BR-008).

## Endpoints / operations

### API-001 — GET /listings — Feed / search / filter
- **Serves:** F-001, F-003
- **Description:** Returns the default feed (excludes `resolved` status and `is_stale = true`,
  per `data-model.md`'s composite index note) or a filtered/searched view. Backs the aggregated,
  de-duplicated, status-aware view the idea's value proposition promises (`idea.md` §6).
- **Request schema:**
  | Param | Type | Required | Notes |
  |---|---|---|---|
  | `kind` | enum(`adoption`,`lost`,`found`) | no | `Listing.kind` |
  | `status` | enum(`available`,`on_hold`,`adopted`,`found`,`missing`,`resolved`) | no | `Listing.status`; omitted = default feed exclusions apply |
  | `lat`, `lng`, `radius_km` | double, double, numeric | no (all three together, or none) | geo filter against `Listing.location_lat/lng`, using the same geo index as F-004's alert fanout |
  | `q` | text | no | free-text match against `Listing.description` / `location_label` — `[assumption]`: no search-field spec exists in the seed beyond "search" in F-001; exact match strategy (substring vs. full-text index) to confirm at scaffold |
  | `include_stale` | boolean | no, default `false` | override to show `is_stale = true` rows (does not affect `resolved` exclusion) |
  | `cursor`, `limit` | text, integer | no | pagination; `[assumption]` cursor-based, no pagination scheme specified in seed |
- **Response schema:** `200 OK` — `{ items: Listing[], next_cursor: text|null }` where each `Listing`
  item includes `id, kind, status, source, description, photo_url, location_lat, location_lng,
  location_label, is_stale, duplicate_of, created_at, updated_at` plus the embedded
  `facebook_anchor: { fb_profile_url }` (INV-001 — every returned listing carries its anchor so the
  client can always render the identity link).
- **Auth:** none — public read (see Authentication & authorization).
- **Errors:** `400` invalid filter combination (e.g., `radius_km` without `lat`/`lng`); `500`.

### API-002 — GET /listings/{id} — Listing detail
- **Serves:** F-001
- **Description:** Single-listing detail view, reached from the feed or from a push-notification
  tap (`system-design.md` data-flow: "user taps → Listing detail view (Backend API read)").
- **Request schema:** path param `id` (uuid).
- **Response schema:** `200 OK` — full `Listing` row plus `facebook_anchor` (INV-001) and, when
  `source = scraped`, the embedded `scraped_post: { original_post_url }` (INV-003 — attribution
  must remain visible). Does not include `StatusHistory` or `raw_content_snapshot` (internal-only
  per `data-model.md` retention classification).
- **Auth:** none — public read.
- **Errors:** `404` no such listing; `410` `[assumption]` — returned if a policy is later added to
  hard-remove resolved listings after a retention window (none specified today; not implemented at
  MVP).

### API-003 — POST /listings — Manual submission (adoption post or missing/found-cat report)
- **Serves:** F-002, F-004, F-005 / INV-001
- **Description:** The single manual-submission entry point for all three `kind` values —
  an adoption listing, a missing-cat report, or a found-cat report — since `system-design.md`
  treats manual submission as one Backend API validation path, not separate services. A `kind =
  lost` (or `found`) write is what "reports" a missing cat; the alerting worker picks up any write
  landing in `status = missing` and fans it out (F-004) — this endpoint does not call the push
  provider directly (`system-design.md` — alerting is async/decoupled from the write path).
- **Request schema:**
  | Field | Type | Required | Notes |
  |---|---|---|---|
  | `kind` | enum(`adoption`,`lost`,`found`) | yes | `Listing.kind` |
  | `description` | text | yes | BR-001 |
  | `photo_url` | text | yes | BR-001; client uploads the photo to object storage first — upload endpoint is `[assumption]`, not modeled here since no object-store contract exists in `data-model.md`/`system-design.md` beyond "an object store for photos" |
  | `location_lat`, `location_lng` | double, double | yes | BR-001 |
  | `location_label` | text | no | |
  | `fb_profile_url` | text | yes | Becomes `FacebookAnchor.fb_profile_url`; the write is rejected (not merely flagged) if missing or unresolvable to a real Facebook profile/page — BR-002/BR-008/INV-001, one transaction with the `Listing` insert per `data-model.md`'s constraint note |
- **Response schema:** `201 Created` — the created `Listing` (same shape as API-002's response,
  `status` defaulted per `data-model.md`: `available` for `adoption`, `missing` for `lost`, `found`
  for `found`) plus `facebook_anchor`. Client shows this immediately in-session
  (`system-design.md` — "visible in the feed within the same session").
- **Auth:** Bearer required; resolved caller becomes `Listing.submitted_by`.
- **Errors:** `400` missing/invalid required field; `422` `fb_profile_url` present but not
  resolvable to a visible Facebook profile/page (INV-001 gate — BR-008); `401` no/invalid session.

### API-004 — PATCH /listings/{id}/status — Status update
- **Serves:** F-003, INV-002
- **Description:** The sole way a listing's `status` changes explicitly (as opposed to the
  time-based staleness sweep, which only ever sets `is_stale` and never touches `status` —
  `data-model.md`). Every call appends a `StatusHistory` row; INV-002 is enforced by making the
  new status take effect (and disappear from default feed views) atomically with that write.
- **Request schema:** path param `id` (uuid); body `{ new_status: enum(same set as
  Listing.status) }`.
- **Response schema:** `200 OK` — the updated `Listing` row, with `resolved_at`/`resolved_by` set
  when `new_status` is `resolved`/`adopted`/`found` (BR-004), and the new `StatusHistory` row's
  `id` for client-side audit reference.
- **Auth:** Bearer required. Caller must be `Listing.submitted_by` **or** `User.is_admin = true`
  (BR-004) — enforced server-side only.
- **Errors:** `400` `new_status` not in the allowed enum, or an invalid transition
  `[assumption]` — no transition table (e.g., can `adopted` move back to `available`?) is specified
  in the seed; flagged in Open questions; `403` caller is neither submitter nor admin; `404` no
  such listing.

### API-005 — PUT /users/me/location — Missing-cat alert subscription
- **Serves:** F-004
- **Description:** Sets or updates the caller's single active radius subscription
  (`UserLocation` — "one active subscription per user", `data-model.md`) that the alerting worker
  geo-queries against on every `missing`-status write (BR-006). Also flips `User.location_opt_in`.
- **Request schema:** `{ lat: double, lng: double, radius_km: numeric }` — `radius_km` optional,
  server defaults to `5` (BR-006 / `usability.md` §1 confirmation copy per `data-model.md`).
- **Response schema:** `200 OK` — `{ lat, lng, radius_km, updated_at }`.
- **Auth:** Bearer required; writes `UserLocation.user_id` = caller.
- **Errors:** `400` invalid lat/lng or non-positive `radius_km`; `401`.

### API-006 — POST /users/me/push-tokens — Register device push token
- **Serves:** F-004
- **Description:** Registers a device so the alerting worker's push-provider dispatch (APNs/FCM —
  `[assumption]`, vendor unconfirmed per `system-design.md`) has somewhere to deliver a
  missing-cat alert. A user may register multiple devices (`data-model.md` — `User ──1───N──
  PushToken`).
- **Request schema:** `{ token: text, platform: enum(ios, android) }`.
- **Response schema:** `201 Created` — `{ id, platform, created_at }`.
- **Auth:** Bearer required; writes `PushToken.user_id` = caller.
- **Errors:** `400` missing/invalid `platform`; `401`.

### API-007 — POST /auth/sessions — Authenticate / create account
- **Serves:** F-005 / INV-001 (supporting) — identity substrate for `submitted_by`,
  `changed_by`, and every other `User`-attributed write this spec requires
- **Description:** `[assumption]` — resolves a caller's `User.auth_identifier` to a session,
  creating the `User` row on first contact (upsert) since no separate registration flow is
  specified in the seed. This is the account-level identity layer; it is distinct from the
  per-listing Facebook-profile anchor that INV-001 actually gates (see Authentication &
  authorization note above) — this endpoint exists only because API-003/004/005/006 need a
  resolved `User.id` to attribute writes to, per `data-model.md`'s foreign keys.
- **Request schema:** `{ auth_identifier: text }` — exact identifier scheme (phone/email/OAuth
  token) unresolved; see Open questions.
- **Response schema:** `200 OK` (existing user) or `201 Created` (new user) — `{ access_token: text,
  user: { id, display_name, is_admin, location_opt_in } }`.
- **Auth:** none (this endpoint issues auth).
- **Errors:** `400` malformed identifier; `401` `[assumption]` if the scheme adds a verification
  step (e.g., OTP) not modeled here.

### API-008 — GET /users/me — Current user profile
- **Serves:** F-005 / INV-001 (supporting), F-003 (client needs `is_admin` to render/enable
  status-change controls per BR-004)
- **Description:** Lets the mobile client read back the caller's own `User` row (display name,
  admin flag, location opt-in state) to drive UI without re-deriving it from the login response
  on every screen.
- **Request schema:** none (identity from bearer token).
- **Response schema:** `200 OK` — `{ id, display_name, auth_identifier, location_opt_in, is_admin,
  created_at }`. `[assumption]` — whether `auth_identifier` should be echoed back to the client at
  all is an open question given it may be PII (`data-model.md` retention classification); flagged
  below.
- **Auth:** Bearer required.
- **Errors:** `401`.

## Error codes

| Code | Meaning | When it occurs |
|---|---|---|
| 400 | Bad request | Missing/malformed required field, invalid filter/enum value |
| 401 | Unauthorized | Missing, invalid, or expired bearer token |
| 403 | Forbidden | Authenticated but not entitled (e.g., BR-004 status-change gate) |
| 404 | Not found | No resource at the given `id` |
| 410 | Gone | `[assumption]` — reserved for a future hard-delete/retention policy; unused at MVP |
| 422 | Unprocessable | Semantically invalid write that a `400` doesn't cover — specifically the
INV-001/BR-008 Facebook-profile-link resolution failure on API-003 |
| 500 | Internal error | Unhandled server-side failure |

## Rate limits

`[assumption]` — no rate-limit policy is specified anywhere in the seed (`idea.md`,
`system-design.md`, `data-model.md`). Given the scraper/manual convergence and the regulatory kill
criterion around Facebook access (`idea.md` §9), a per-user write limit on API-003/API-004 is
worth having before launch to blunt abuse of the manual-submission path, but no number can be
sourced today. Flagged as an open question, not a target.

## Versioning

`[assumption]` — no versioning scheme is specified in the seed. Proposed: URL path versioning
(`/v1/...`, already reflected in the Overview's base-URL proposal), with breaking changes gated
behind a new path segment rather than header negotiation, matching `team_size: 1`'s low
operational capacity. Deprecation policy: none defined yet — MVP has no prior version to deprecate.

## Open questions

- **Auth scheme:** which identifier (phone OTP, email/password, Facebook OAuth) backs
  `User.auth_identifier` / API-007 — unresolved in the seed. Given F-005's Facebook-anchor
  requirement, Facebook OAuth would be a natural fit but is nowhere asserted as the login
  mechanism — do not assume it without product-owner confirmation.
- **Status transition table:** is every `Listing.status` enum value reachable from every other
  (API-004), or are some transitions (e.g., `adopted` → `available`) disallowed? No rule is stated.
- **Pagination/search strategy:** cursor shape and `q` match semantics on API-001 are
  `[assumption]`; no NFR or UX spec constrains them.
- **Rate limiting:** no policy exists; see Rate limits above.
- **Object/photo upload path:** `photo_url` on API-003 assumes the client already has a URL from an
  out-of-band upload step; no upload endpoint or object-store contract is specified in
  `data-model.md`/`system-design.md` to model here.
- **PII exposure on API-008:** whether `auth_identifier` should be returned to the client at all,
  given its PII classification in `data-model.md`.
- **Machine-readable spec:** this Markdown contract has no OpenAPI/GraphQL/protobuf equivalent yet;
  flagged per the template's stated purpose.

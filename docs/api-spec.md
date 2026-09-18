# API Specification

> **Purpose:** contracts between components/services. Use a machine-readable spec format
> (OpenAPI, GraphQL SDL, protobuf) where applicable and link it here.
> Traces back to: system design, technical design.

## Overview

The Backend API is the single contract boundary the Mobile Client (iOS + Android) depends on
(`system-design.md` — "Backend API ... exposes_api: true"). It owns listing read/search/filter
(F-001), submission intake (F-002), multi-cat batch submission (F-006), status transitions
(F-003/INV-002), location-based alert subscription and push-token registration (F-004), the
identity anchor / account layer that F-005 and INV-001 depend on, Facebook-OAuth account
login/logout/deletion and profile management (`ADR-0001`), location/push-token opt-out, and
photo-upload URL issuance (F-002/F-006's `photo_url` requirement). The scraper and alerting workers write to the same data store through
this API's listing-write path (`system-design.md` — "both pipelines converge on the same `Listing`
write path"); this spec covers the mobile-facing surface only.

- **Base URL / namespace:** `[assumption]` — no domain/host is named in the seed; proposed
  `https://api.whiskr.app/v1` pending the deployment topology decision in `system-design.md`.
  To confirm at scaffold.
- **Format:** JSON request/response bodies; a full OpenAPI 3.x document is not produced in this
  batch — this Markdown contract is authoritative until one is scaffolded.
- **Machine-readable spec:** none exists yet — `[assumption]`, flagged as an open question below.

## Error response shape

Resolved 2026-09-18 (was previously unspecified — every endpoint above only ever named an HTTP
status code with no defined body). Every non-2xx response, from every endpoint in this spec, uses
one envelope:

```json
{ "error": { "code": "VALIDATION_ERROR", "message": "fb_profile_url is not accessible", "field": "fb_profile_url" } }
```

- `code` — a stable, machine-matchable string (`SCREAMING_SNAKE_CASE`), not the HTTP status number;
  the client switches on this, not on parsing `message`.
- `message` — human-readable, safe to show directly in a UI toast; never leaks internals (a DB
  error message, a stack trace) — `500`s always return a generic `message` regardless of the actual
  server-side cause.
- `field` — optional; present when the error traces to one request field (e.g., `400`/`422`
  validation failures). Omitted for errors with no single offending field (`401`, `404`, `500`).
- `index` — optional, used only by API-010 (batch): when one `cats[]` entry fails validation, `field`
  names the sub-field (e.g., `"photo_url"`) and `index` names which array entry (e.g., `2`).
- Representative `code` values: `VALIDATION_ERROR` (400), `UNAUTHENTICATED` (401), `FORBIDDEN`
  (403), `NOT_FOUND` (404), `CONFLICT` (409), `UNPROCESSABLE` (422), `RATE_LIMITED` (429),
  `INTERNAL_ERROR` (500). Endpoint-specific reasons (e.g., `wrong_app_id`, `phone_already_in_use`)
  are carried in `message`, not invented as new top-level `code` values, to keep the client's switch
  statement small and stable.

## Authentication & authorization

- **Mechanism:** Facebook OAuth (`ADR-0001`, resolved 2026-09-18) — the Mobile Client obtains a
  short-lived Facebook access token via the on-device Facebook Login SDK, then exchanges it at
  API-007 for a Whiskr-issued bearer session token, sent as `Authorization: Bearer <token>` on every
  authenticated call thereafter. There is no app-side password and no forgot-password flow — a
  direct, intentional consequence of this decision, not a gap.
- **Public (no auth) endpoints:** API-001, API-002, API-016 (health) — the feed must be browsable
  without an account, consistent with `system-design.md`'s "feed still browsable without location
  permission" (device location and account auth are separate gates; browsing needs neither).
  `[assumption]` — the seed never states whether an account is mandatory just to browse.
- **Authenticated endpoints:** API-003 through API-006, API-008 through API-015, API-017, API-018 —
  creating a listing/batch, changing status, registering/unregistering a location or push token,
  editing or deleting a profile/account, requesting an upload URL, verifying a phone number, or
  logging out all require a resolved `User.id`, since these actions are attributed via
  `Listing.submitted_by`, `StatusHistory.changed_by`, `UserLocation.user_id`, `PushToken.user_id`,
  or the caller's own `Session`/`User` row respectively (`data-model.md`).
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
  | `q` | text | no | RESOLVED 2026-09-18: case-insensitive substring match (`ILIKE '%q%'`) against `Listing.description` **OR** `location_label`, combined with every other filter param via `AND`. Not full-text search/ranking at MVP — deliberately the simplest honest form, sufficient at expected MVP data volume; upgrade to a full-text index only if query latency actually degrades |
  | `include_stale` | boolean | no, default `false` | override to show `is_stale = true` rows (does not affect `resolved` exclusion) |
  | `cursor`, `limit` | text, integer | no | RESOLVED 2026-09-18: `cursor` is an opaque base64 encoding of `{ last_created_at, last_id }`; results are ordered `created_at DESC, id DESC` (tie-break on `id` since two listings can share a `created_at` timestamp) and the next page's `WHERE` clause is `(created_at, id) < (last_created_at, last_id)`, keyset-paginated (never `OFFSET`, which degrades on a large feed and can skip/duplicate rows under concurrent writes). `limit` defaults to `20`, capped at `100` |
- **Response schema:** `200 OK` — `{ items: Listing[], next_cursor: text|null }` where each `Listing`
  item includes `id, kind, status, source, description, photo_url, location_lat, location_lng,
  location_label, is_stale, duplicate_of, created_at, updated_at` plus the embedded
  `facebook_anchor: { fb_profile_url }` (INV-001 — every returned listing carries its anchor so the
  client can always render the identity link) and, when `submitted_by` resolves to a `User`, an
  embedded `submitter: { display_name, is_verified }` (F-102 — the badge the feed/detail view
  renders; omitted entirely when `submitted_by` is `null`, e.g. a scraped listing or a
  deleted-account listing, BR-012).
- **Auth:** none — public read (see Authentication & authorization).
- **Errors:** `400` invalid filter combination (e.g., `radius_km` without `lat`/`lng`); `500`.

### API-002 — GET /listings/{id} — Listing detail
- **Serves:** F-001
- **Description:** Single-listing detail view, reached from the feed or from a push-notification
  tap (`system-design.md` data-flow: "user taps → Listing detail view (Backend API read)").
- **Request schema:** path param `id` (uuid).
- **Response schema:** `200 OK` — full `Listing` row plus `facebook_anchor` (INV-001), the embedded
  `submitter: { display_name, is_verified }` when `submitted_by` resolves to a `User` (F-102, same
  rule as API-001), and, when `source = scraped`, the embedded `scraped_post: { original_post_url }`
  (INV-003 — attribution must remain visible). Does not include `StatusHistory` or
  `raw_content_snapshot` (internal-only per `data-model.md` retention classification).
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
  | `photo_url` | text | yes | BR-001; MUST be a `photo_url` previously returned by API-011 to this same caller and not yet used on another listing — validated against `PhotoUpload` (BR-013), not trusted as opaque text |
  | `location_lat`, `location_lng` | double, double | yes | BR-001 |
  | `location_label` | text | no | |
  | `fb_profile_url` | text | yes | Becomes `FacebookAnchor.fb_profile_url`; the write is rejected (not merely flagged) if missing or unresolvable to a real Facebook profile/page — BR-002/BR-008/INV-001, one transaction with the `Listing` insert per `data-model.md`'s constraint note |
- **Response schema:** `201 Created` — the created `Listing` (same shape as API-002's response,
  `status` defaulted per `data-model.md`: `available` for `adoption`, `missing` for `lost`, `found`
  for `found`) plus `facebook_anchor`. Client shows this immediately in-session
  (`system-design.md` — "visible in the feed within the same session").
- **Auth:** Bearer required; resolved caller becomes `Listing.submitted_by`.
- **Errors:** `400` missing/invalid required field, or `photo_url` unrecognized/not-owned-by-caller/
  already-used (BR-013); `422` `fb_profile_url` present but not resolvable to a visible Facebook
  profile/page (INV-001 gate — BR-008); `401` no/invalid session.

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

### API-007 — POST /auth/facebook — Sign up / log in with Facebook
- **Serves:** UJ-005, account identity substrate (`ADR-0001`) for `submitted_by`, `changed_by`, and
  every other `User`-attributed write this spec requires
- **Description:** Verifies a Facebook access token the client obtained via the on-device Facebook
  Login SDK, upserts the `User` row by `fb_user_id` (creating it on first contact), and issues a
  Whiskr bearer session token. This is the only sign-up/login path — there is no separate
  registration flow and no password. This is the account-level identity layer; it is distinct from
  the per-listing Facebook-profile anchor that INV-001 actually gates (see Authentication &
  authorization note above) — a logged-in caller still must supply a Facebook profile link per
  listing (or per batch, F-006/BR-009).
- **Verification detail:** the backend calls Facebook's Graph API (`GET /me?access_token=<token>
  &fields=id,name,picture`) to resolve `fb_user_id`/`name`/profile-photo signal (F-102's
  `fb_signals_verified_at` check) and confirm the token is valid, **then MANDATORILY** calls
  `/debug_token` to confirm the token's `app_id` matches Whiskr's own Facebook App ID — RESOLVED
  2026-09-18 from "recommended, not confirmed" to a hard requirement (`security-compliance.md`
  T-010; skipping it is the exact stolen-token-from-another-app replay T-010 names).
- **Request schema:** `{ fb_access_token: text }` — the client-obtained Facebook access token.
- **Response schema:** `200 OK` (existing user) or `201 Created` (new user) — `{ access_token: text,
  user: { id, display_name, is_admin, is_verified, location_opt_in } }`. `access_token` is the
  **raw** Whiskr session token; the backend stores only its hash (a new `Session` row,
  `data-model.md`) and never returns it again on any subsequent read. `is_verified` is derived
  server-side (F-102, `data-model.md`) — never a request field, only ever a response field.
- **Auth:** none (this endpoint issues auth).
- **Errors:** `400` missing `fb_access_token`; `401` token invalid, expired, or issued for a
  different Facebook App ID (the now-mandatory `debug_token` check).

### API-012 — DELETE /auth/sessions — Log out
- **Serves:** completes the auth surface opened by API-007; session mechanism (`data-model.md`
  `Session`, resolved 2026-09-18)
- **Description:** Revokes the caller's current session by setting `Session.revoked_at` on the row
  matching the presented bearer token's hash. A revoked session is rejected identically to an
  expired one on every subsequent authenticated call (T-012).
- **Request schema:** none (identity from bearer token).
- **Response schema:** `204 No Content`.
- **Auth:** Bearer required.
- **Errors:** `401` token already invalid/expired/revoked (logging out twice is a no-op error, not a
  crash — **[assumption]**: treated as `401` rather than a silent `204`, to confirm at scaffold).

### API-013 — DELETE /users/me/location — Opt out of location-based alerts
- **Serves:** F-004 (completes API-005's promise in `data-model.md`: "retained until opt-out or
  account deletion" — no endpoint previously existed for the opt-out half)
- **Description:** Deletes the caller's `UserLocation` row and sets `User.location_opt_in = false`.
  Distinct from API-005 (which upserts/updates the subscription): this is a full opt-out, not an
  update to a new radius.
- **Request schema:** none (identity from bearer token).
- **Response schema:** `204 No Content`.
- **Auth:** Bearer required.
- **Errors:** `401`. (No `404` — opting out when no subscription exists is a no-op success, not an
  error — **[assumption]**, to confirm at scaffold.)

### API-014 — DELETE /users/me/push-tokens/{id} — Unregister a device
- **Serves:** F-004 (completes API-006's promise in `data-model.md`: "retained until device
  unregisters or token rotates" — no endpoint previously existed for unregistering)
- **Description:** Deletes one `PushToken` row (e.g., the user logged out of one device, or
  uninstalled the app on it) without affecting the account or its other registered devices.
- **Request schema:** path param `id` (uuid, the `PushToken.id` returned by API-006).
- **Response schema:** `204 No Content`.
- **Auth:** Bearer required; the token must belong to the caller (`403` otherwise — never let a user
  unregister another user's device).
- **Errors:** `403` token belongs to a different user; `404` no such token; `401`.

### API-015 — DELETE /users/me — Delete account
- **Serves:** UJ-006 (profile), the "life of the account; deleted on account-deletion request"
  retention promise every PII field in `data-model.md`/`security-compliance.md` already makes
- **Description:** Deletes the caller's account and cascade-deletes everything scoped to the account
  alone: all `Session` rows (immediate logout, everywhere), the `UserLocation` row, all `PushToken`
  rows, all `PhotoUpload` rows, all `PhoneVerification` rows, and every `AlertDelivery` row where the
  caller was the recipient. **Listings, report batches, and status-history rows the user is
  referenced from are retained, not deleted** — `Listing.submitted_by`/`resolved_by`,
  `ReportBatch.submitted_by`, and `StatusHistory.changed_by` are all set to `null` instead (BR-012;
  full enumeration in `technical-design.md`'s `deleteAccount()`), so existing listings other users
  may be relying on (an open adoption listing, an active missing-cat alert) and INV-002's own audit
  trail don't silently vanish out from under anything; only the account's own identity is removed.
  This mirrors real-world moderation practice and needs no new schema.
- **Request schema:** none (identity from bearer token).
- **Response schema:** `204 No Content`.
- **Auth:** Bearer required.
- **Errors:** `401`.

### API-011 — POST /uploads/photo-url — Request a photo upload URL
- **Serves:** F-002, F-006 (BR-001's required `photo_url` field on both API-003 and API-010)
- **Description:** Resolves the "Object/photo upload path" gap this spec previously left
  unmodeled. The client requests a short-lived presigned upload URL, PUTs the photo bytes directly
  to object storage (never through the Backend API), then submits the resulting `photo_url` on
  API-003/API-010 as before. Keeps large binary uploads off the Backend API's own request path.
- **Request schema:** `{ content_type: enum(image/jpeg, image/png, image/heic) }`.
- **Response schema:** `201 Created` — `{ upload_url: text, photo_url: text, expires_at: timestamp }`
  — `upload_url` is a presigned `PUT` target (**[assumption]** — object-store vendor unconfirmed,
  `system-design.md` "an object store for photos"); `photo_url` is the resulting public/read URL to
  submit on API-003/API-010 once the client's `PUT` to `upload_url` succeeds.
- **Auth:** Bearer required.
- **Errors:** `400` missing/invalid `content_type`; `401`.

### API-008 — GET /users/me — Current user profile
- **Serves:** F-005 / INV-001 (supporting), F-003 (client needs `is_admin` to render/enable
  status-change controls per BR-004), UJ-006
- **Description:** Lets the mobile client read back the caller's own `User` row (display name,
  admin flag, location opt-in state) to drive UI without re-deriving it from the login response
  on every screen.
- **Request schema:** none (identity from bearer token).
- **Response schema:** `200 OK` — `{ id, display_name, location_opt_in, is_admin, is_verified,
  verified_via: array<enum(facebook_signals, track_record, phone)>, created_at }`. `fb_user_id` and
  `phone_number` are never echoed back to the client (PII, not needed by any UI this spec defines).
  `verified_via` lists which of F-102's criteria are met (may be more than one) — empty array when
  `is_verified` is `false`.
- **Auth:** Bearer required.
- **Errors:** `401`.

### API-009 — PATCH /users/me — Edit profile
- **Serves:** UJ-006, BR-010
- **Description:** Lets the signed-in user edit their own editable profile fields. Only
  `display_name` is user-editable; `is_admin`, `is_verified`/`verified_via`, and any
  Facebook-identity field are never accepted here (BR-010, BR-014, `security-compliance.md` T-008).
- **Request schema:** `{ display_name: text }` — required, non-empty.
- **Response schema:** `200 OK` — the updated user, same shape as API-008's response.
- **Auth:** Bearer required.
- **Errors:** `400` empty/missing `display_name`, or the request body includes a non-editable field
  (`is_admin`, `is_verified`, `verified_via`, any `fb_*`/`phone_*` field) — rejected outright rather
  than silently ignored, so a client bug never looks like a successful privilege change; `401`.

### API-017 — POST /users/me/phone/start-verification — Begin phone verification
- **Serves:** F-102, UJ-007
- **Description:** Generates a 6-digit OTP, sends it via SMS, starts a 10-minute expiry
  (`PhoneVerification`, `data-model.md`). Does not yet set `User.phone_number` — only API-018 does,
  on success.
- **Request schema:** `{ phone_number: text }` — E.164 format expected; non-conforming input
  normalized where possible, rejected where not.
- **Response schema:** `202 Accepted` — `{ expires_at: timestamp }` (no OTP code in the response,
  obviously — it only ever travels via SMS).
- **Auth:** Bearer required.
- **Errors:** `400` malformed phone number; `409` phone number already confirmed on a different
  account (BR-016); `401`.

### API-018 — POST /users/me/phone/confirm — Confirm phone verification
- **Serves:** F-102, UJ-007
- **Description:** Checks the submitted code against the pending `PhoneVerification` row; on match,
  sets `User.phone_number`/`phone_verified_at` in the same transaction (Algorithm 9,
  `technical-design.md`).
- **Request schema:** `{ code: text }` — the 6-digit code from the SMS.
- **Response schema:** `200 OK` — the updated user, same shape as API-008's response
  (`is_verified: true`, `verified_via` includes `phone`).
- **Auth:** Bearer required.
- **Errors:** `400` no pending verification, expired, or wrong code; `409` phone number confirmed by
  a different account in the race window (BR-016); `429` 5 wrong attempts reached on this cycle;
  `401`.

### API-016 — GET /health — Liveness/readiness check
- **Serves:** ops/deploy tooling — completes a gap this spec previously left unmodeled despite
  `ops.md` discussing monitoring with no endpoint to monitor.
- **Description:** Confirms the API process is up and can reach its data store. Deliberately
  minimal — no version string, no build metadata, nothing an unauthenticated caller shouldn't see.
- **Request schema:** none.
- **Response schema:** `200 OK` — `{ status: "ok" }`. A data-store connectivity failure returns
  `503` with the standard error envelope (`code: "INTERNAL_ERROR"`), not a `200` with a
  misleadingly-nested "unhealthy" body — the HTTP status itself is the signal ops tooling polls on.
- **Auth:** none (public — ops tooling has no session).
- **Errors:** `503` data store unreachable.

### API-010 — POST /listings/batch — Multi-cat batch report
- **Serves:** F-006, F-002 (reuses the same per-cat field rules), F-005 / INV-001, BR-009
- **Description:** Reports 2 or more cats found/lost together in a single request. Each cat becomes
  its own independently status-tracked `Listing` (same rules as API-003), all sharing one
  `fb_profile_url` (validated once, per BR-009) and one `ReportBatch.id`. All-or-nothing: if any cat
  entry fails validation, no listing in the batch is created (`frd.md` FRD-F006-01).
- **Request schema:**
  | Field | Type | Required | Notes |
  |---|---|---|---|
  | `fb_profile_url` | text | yes | Validated once per BR-009; shared across every cat in the batch |
  | `cats` | array, min length 2 | yes | Each entry has the same shape as API-003's `kind`/`description`/`photo_url`/`location_lat`/`location_lng`/`location_label` fields (BR-001, per cat) |
- **Response schema:** `201 Created` — `{ batch_id: uuid, listings: Listing[] }` — each `Listing` in
  the same shape as API-002/API-003's response, all carrying their own `facebook_anchor` (same
  `fb_profile_url`) and the shared `batch_id`.
- **Auth:** Bearer required; resolved caller becomes `submitted_by` on every `Listing` in the batch.
- **Errors:** `400` fewer than 2 entries in `cats`, any entry missing a BR-001 required field, or any
  entry's `photo_url` unrecognized/not-owned-by-caller/already-used (BR-013 — `field`/`index` in the
  error envelope names which cat entry failed); `422` `fb_profile_url` unresolvable
  (BR-002/BR-008/INV-001 — checked once for the whole batch); `401`.

## Error codes

| Code | Meaning | When it occurs |
|---|---|---|
| 400 | Bad request | Missing/malformed required field, invalid filter/enum value |
| 401 | Unauthorized | Missing, invalid, or expired bearer token |
| 403 | Forbidden | Authenticated but not entitled (e.g., BR-004 status-change gate) |
| 404 | Not found | No resource at the given `id` |
| 409 | Conflict | An already-terminal listing's status is attacked again (INV-002); a phone number already confirmed elsewhere (BR-016) |
| 410 | Gone | `[assumption]` — reserved for a future hard-delete/retention policy; unused at MVP |
| 422 | Unprocessable | Semantically invalid write that a `400` doesn't cover — specifically the
INV-001/BR-008 Facebook-profile-link resolution failure on API-003/API-010 |
| 429 | Too many requests | A rate limit named below is exceeded, or 5 wrong OTP attempts (API-018) |
| 500 | Internal error | Unhandled server-side failure |
| 503 | Service unavailable | API-016 health check: data store unreachable |

## Rate limits

RESOLVED 2026-09-18 — concrete per-user limits, replacing the prior open policy gap
(`security-compliance.md` T-011 named the batch-endpoint instance of this gap specifically).
Enforced by a request-count window per `User.id` (authenticated endpoints) or per IP (API-007,
pre-auth); exceeding a limit returns `429` with the standard error envelope
(`code: "RATE_LIMITED"`). Numbers are a reasoned MVP default, not sourced from the seed — revisit
once real usage data exists:

| Endpoint | Limit | Why this number |
|---|---|---|
| API-003 (submit) | 10 / hour | A genuine finder/rescuer rarely posts more than a handful of cats an hour; blunts spam without blocking a busy shelter day |
| API-010 (batch) | 5 / hour, **weighted by cat count** (a 4-cat batch counts as 4 toward the API-003 limit above, not 1) | Prevents the batch endpoint from being a 1-call bypass of the API-003 limit (`security-compliance.md` T-011) |
| API-004 (status transition) | 30 / hour | Legitimate use is infrequent (mark-resolved, occasional corrections); generous enough to never block real usage |
| API-011 (photo-upload URL) | 20 / hour | Roughly 2x the submission limit, allowing for retakes/mistakes before a successful submit |
| API-007 (Facebook login) | 20 / 15 min, **per IP** (pre-auth, no `User.id` to key on yet) | Bounds Graph API cost/abuse from a single source without punishing normal multi-device login |
| API-017 (start phone verification) | 5 / hour per user, **and** 5 / hour per phone number | Bounds SMS cost/abuse from both directions — a user hammering the endpoint, or many accounts targeting one phone number |
| All other authenticated endpoints | 100 / hour | A generous default catch-all so reads/light writes are never the binding constraint |

## Versioning

`[assumption]` — no versioning scheme is specified in the seed. Proposed: URL path versioning
(`/v1/...`, already reflected in the Overview's base-URL proposal), with breaking changes gated
behind a new path segment rather than header negotiation, matching `team_size: 1`'s low
operational capacity. Deprecation policy: none defined yet — MVP has no prior version to deprecate.

## Open questions

- **~~Auth scheme~~ RESOLVED 2026-09-18:** Facebook OAuth is the sole login/signup mechanism
  (`ADR-0001`); see Authentication & authorization above. Kept here, struck through, as a record
  rather than deleted — `decision-ledger.md` §3 is the canonical account of why.
- **~~Facebook token app-id verification~~ RESOLVED 2026-09-18:** now mandatory in API-007's
  verification step (`security-compliance.md` T-010 mitigated, not just named).
- **Status transition table:** RESOLVED by `frd.md`'s F-003 transition table (terminal states
  `{adopted, found, resolved}`, no admin override — `technical-design.md` Algorithm 1b was found to
  contradict this and has been fixed, `decision-ledger.md`). Not this doc's own open question
  anymore; kept struck rather than deleted for the same reason as the others above.
- **~~Pagination/search strategy~~ RESOLVED 2026-09-18:** see API-001's own entry above — opaque
  base64 cursor, `ILIKE` substring search.
- **~~Rate limiting~~ RESOLVED 2026-09-18:** see Rate limits above.
- **~~Object/photo upload path~~ RESOLVED 2026-09-18:** API-011 issues a presigned upload URL, and
  API-003/API-010 now validate it against `PhotoUpload` (BR-013). The object-store *vendor* itself
  remains `[assumption]` (system-design.md), to confirm at scaffold.
- **Machine-readable spec:** this Markdown contract has no OpenAPI/GraphQL/protobuf equivalent yet;
  flagged per the template's stated purpose.

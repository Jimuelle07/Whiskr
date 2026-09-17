# Data Model / Schema

> **Purpose:** the data. Entities, relationships, constraints, and privacy classification.
> Traces back to: system design, technical design, API spec.

## Entities & relationships (ERD)
```
 User ──1───N── Listing            (submitted_by; nullable for scraped listings)
 User ──1───1── UserLocation        (opt-in radius subscription, for F-004/UJ-003)
 User ──1───N── PushToken           (a user may have multiple devices)
 User ──1───N── ReportBatch         (submitted_by; F-006 multi-cat batch reports)
 User ──1───N── Session             (one row per issued bearer token; API-007/API-012)
 User ──1───N── PhotoUpload         (one row per requested upload URL; API-011)
 User ──1───N── PhoneVerification   (one row per OTP attempt cycle; F-102/API-017/API-018)

 ReportBatch ──1───N── Listing      (batch_id, nullable; F-006 grouping reference, no lifecycle of its own)

 Listing ──1───1── FacebookAnchor   (identity anchor; INV-001/F-005/BR-002/BR-008)
 Listing ──1───N── StatusHistory    (every transition; enforces/audits INV-002)
 Listing ──0..1─1── ScrapedPost     (present only when source = scraped; INV-003 attribution)
 Listing ──1───N── Alert            (fired when status → missing; F-004)

 ScrapeSource ──1───N── ScrapedPost (a configured FB page/group produces many posts)

 Alert ──1───N── AlertDelivery      (fanout per opted-in nearby user; BR-006)
 AlertDelivery ──N───1── User

 Listing ──1───0..1── Listing       (self-ref: original ←→ duplicate resolved into it, F-001 dedup)
```

Two write paths (scraper, manual — per `system-design.md`) converge on the single `Listing` entity
so status/staleness/alerting logic (F-003, F-004, INV-002) is never duplicated across paths.

## Schema / field definitions

### User
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| display_name | text | no | — | Shown alongside listings the user submitted; sourced from Facebook at signup, user-editable thereafter (BR-010, UJ-006, API-009) |
| fb_user_id | text | no | — | Facebook-issued user id from OAuth (`ADR-0001`); unique per account; resolves the prior `[assumption]` `auth_identifier` field, which is removed |
| location_opt_in | boolean | no | false | Gate for UJ-002/UJ-003 location features |
| is_admin | boolean | no | false | BR-004/INV-002 gate — who may change another submitter's status; provisioning is `[assumption]`, no admin model in seed; never settable via any user-facing endpoint (BR-010) |
| fb_signals_verified_at | timestamp | yes | null | Set once, at signup, if the Facebook profile passed the automated signal check (Algorithm 7, F-102); never re-evaluated afterward |
| track_record_verified_at | timestamp | yes | null | Set by the daily verification sweep (Algorithm 8) once tenure + submission-volume thresholds are met; never unset once granted (BR-014) |
| phone_number | text | yes | null | E.164-normalized; set only after a confirmed OTP (API-018); unique across accounts (BR-016) when non-null |
| phone_verified_at | timestamp | yes | null | Set when `phone_number` is confirmed via OTP |
| created_at | timestamp | no | now() | |
| updated_at | timestamp | no | now() | |

`User.is_verified` is **derived, not stored**: `fb_signals_verified_at IS NOT NULL OR
track_record_verified_at IS NOT NULL OR phone_verified_at IS NOT NULL` (F-102). Keeping the three
source timestamps instead of one flag preserves *which* signal(s) granted it — useful for the
badge's tooltip/audit and for tuning thresholds later without losing history. None of the four
verification fields is ever settable via any user-facing endpoint (BR-014), same protection as
`is_admin`.

### PhoneVerification
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| user_id | uuid (FK → User.id) | no | — | |
| phone_number | text | no | — | E.164-normalized candidate number, not yet confirmed |
| otp_hash | text | no | — | SHA-256 hash of the OTP code — same never-store-the-raw-value principle as `Session.token_hash` |
| attempt_count | integer | no | 0 | Incremented on every wrong-code confirm attempt; locked out at 5 (F-102 rate-limit) |
| expires_at | timestamp | no | `created_at + 10 minutes` | OTP validity window |
| verified_at | timestamp | yes | null | Set on successful confirm; this row's `user_id`'s `User.phone_number`/`phone_verified_at` are set at the same time, same transaction |
| created_at | timestamp | no | now() | |

**No Facebook access/refresh token field exists on this table.** Per `ADR-0001`, the token the
client presents at login is verified against the Graph API once and then discarded — Whiskr never
persists a long-lived Facebook credential, which keeps `PushToken.token`-style secret-handling scope
from also applying here.

### Session
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| user_id | uuid (FK → User.id) | no | — | Owner of this session |
| token_hash | text | no | — | SHA-256 hash of the bearer token returned to the client at API-007; the raw token itself is never stored, only its hash — same principle as password hashing, so a data-store leak alone does not yield directly reusable tokens |
| created_at | timestamp | no | now() | |
| expires_at | timestamp | no | `created_at + 90 days` | Fixed 90-day expiration — **[assumption]**, no session-lifetime policy exists in the seed; re-login (a fresh Facebook OAuth round trip, not a "refresh") is required after expiry, consistent with there being no forgot-password/refresh-token flow (`ADR-0001`) |
| revoked_at | timestamp | yes | null | Set by API-012 (logout); a session with `revoked_at` set is treated identically to an expired one on every authenticated call |

`Session` is what resolves `security-compliance.md`'s previously-open "session mechanism"
question: **opaque, hashed, revocable bearer tokens**, not JWT — chosen specifically because a real
logout (API-012) needs true revocation, which a stateless JWT cannot provide without a separate
denylist (an equivalent extra data-store lookup anyway, at more complexity) — see
`decision-ledger.md`.

### PhotoUpload
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| user_id | uuid (FK → User.id) | no | — | Who requested the upload URL (API-011) |
| photo_url | text | no | — | The exact URL returned to the client; unique — this is what `Listing.photo_url` must match |
| content_type | enum(image/jpeg, image/png, image/heic) | no | — | Echoes API-011's request |
| created_at | timestamp | no | now() | |
| expires_at | timestamp | no | `created_at + 15 minutes` | The presigned `upload_url`'s own expiry (API-011); after this, the client must request a new one |
| consumed_at | timestamp | yes | null | Set the moment this `photo_url` is successfully referenced by a `Listing` (API-003/API-010); a second attempt to reuse it is rejected (BR-013) |

Closes a gap the initial API-011 spec left open: without this table, any client could submit an
arbitrary `photo_url` string on API-003/API-010 with no proof anything was ever uploaded, or reuse
someone else's upload URL. BR-013 (`prd.md`) requires every submitted `photo_url` to (a) match a
`PhotoUpload` row owned by the submitting user, (b) not be already `consumed_at`, and (c) the
object itself to actually exist in storage (a `HEAD` check against the real bucket, not just
against this table) — this table proves *ownership and issuance*, the storage `HEAD` check proves
the client actually uploaded something.

### ReportBatch
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| submitted_by | uuid (FK → User.id) | yes | — | The user who submitted the batch (F-006); set to `null` on account deletion (BR-012), same as `Listing.submitted_by` |
| created_at | timestamp | no | now() | |

A thin grouping reference only — no `status`, no lifecycle, never mutated after creation. Each cat
in the batch is an independent `Listing` row (see below); `ReportBatch` exists solely so the client
can fetch "every cat reported together" without inferring it from timestamps. On account deletion
(BR-012), `submitted_by` is set to `null` here too, for the same reason as `Listing.submitted_by` —
the batch grouping is retained, only the account identity is removed.

### UserLocation
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| user_id | uuid (FK → User.id) | no | — | One active subscription per user |
| lat | double | no | — | |
| lng | double | no | — | |
| radius_km | numeric | no | 5 | Default per BR-006 / `usability.md` §1 confirmation copy |
| updated_at | timestamp | no | now() | |

### PushToken
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| user_id | uuid (FK → User.id) | no | — | |
| token | text | no | — | Opaque device push token (APNs/FCM — `[assumption]` vendor) |
| platform | enum(ios, android) | no | — | |
| created_at | timestamp | no | now() | |

### Listing
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| kind | enum(adoption, lost, found) | no | — | Matches Report-a-cat form's Status field (BR-001) |
| status | enum(available, on_hold, adopted, found, missing, resolved) | no | `available` (adoption) / `missing` (lost) / `found` (found) at creation | BR-003 enum; transitions enforce INV-002 |
| source | enum(scraped, manual) | no | — | Distinguishes the two write paths (F-001 vs. F-002) |
| submitted_by | uuid (FK → User.id) | yes | null | Null when `source = scraped` and no matching user account exists, **or** after the submitting user deletes their account (API-015/BR-012 — `ON DELETE SET NULL`, never cascading a delete onto the listing itself) |
| description | text | no | — | BR-001 required field |
| photo_url | text | no | — | BR-001 required field; storage location `[assumption]`, object store not named in seed |
| location_lat | double | no | — | BR-001 required field |
| location_lng | double | no | — | BR-001 required field |
| location_label | text | yes | null | Free-text address/area label shown in UI |
| is_stale | boolean | no | false | Set by the staleness sweep (BR-005); does **not** mutate `status` — keeps INV-002 (explicit resolution) distinct from time-based suppression |
| duplicate_of | uuid (FK → Listing.id) | yes | null | Self-reference; set when the dedup match (F-001) collapses a re-scraped post into an existing listing |
| batch_id | uuid (FK → ReportBatch.id) | yes | null | Set when this listing was created as part of a multi-cat batch report (F-006); null for single-cat submissions and all scraped listings |
| resolved_at | timestamp | yes | null | Set only on an explicit BR-004 status transition to `resolved`/`adopted`/`found` |
| resolved_by | uuid (FK → User.id) | yes | null | Submitter or admin who resolved it (BR-004) |
| created_at | timestamp | no | now() | |
| updated_at | timestamp | no | now() | |

### FacebookAnchor
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| listing_id | uuid (FK → Listing.id, unique) | no | — | One anchor per listing; row's existence is the INV-001 gate — a `Listing` write SHALL NEVER commit without one (BR-002/BR-008) |
| fb_profile_url | text | no | — | Visible link back to a real Facebook profile or page |
| resolved_at_submit | boolean | no | true | Records that BR-002/BR-008 validation ran before publish, for audit |
| created_at | timestamp | no | now() | |

### ScrapeSource
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| fb_page_or_group_url | text | no | — | Configured NCR/Greater-Manila-Area page/group |
| last_scraped_at | timestamp | yes | null | |
| status | enum(active, degraded, blocked) | no | `active` | `blocked`/`degraded` values feed the technical kill-criterion signal (`idea.md` §9) |

### ScrapedPost
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| scrape_source_id | uuid (FK → ScrapeSource.id) | no | — | |
| listing_id | uuid (FK → Listing.id) | yes | null | Set once normalized into a `Listing`; null briefly during ingest |
| original_post_url | text | no | — | The exact source URL; SHALL NEVER be dropped before/after normalization (enforces INV-003) |
| raw_content_snapshot | text | no | — | Retained for audit/dedup, not shown verbatim in UI beyond the link |
| dedup_hash | text | no | — | Content fingerprint used for cross-page/group dedup (F-001) |
| scraped_at | timestamp | no | now() | |

### StatusHistory
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| listing_id | uuid (FK → Listing.id) | no | — | |
| old_status | enum (same set as Listing.status) | no | — | |
| new_status | enum (same set as Listing.status) | no | — | |
| changed_by | uuid (FK → User.id) | yes | null | Null for system-driven staleness flags (which do not touch `status` — see `Listing.is_stale`); populated for every explicit BR-004 transition |
| changed_at | timestamp | no | now() | Append-only; this table is the audit trail INV-002 verification (TC-N02) reads |

### Alert
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| listing_id | uuid (FK → Listing.id) | no | — | Fired when `status` transitions to/is created as `missing` |
| radius_km | numeric | no | 5 | BR-006 default |
| triggered_at | timestamp | no | now() | |

### AlertDelivery
| Field | Type | Null? | Default | Description |
|-------|------|-------|---------|-------------|
| id | uuid | no | generated | Primary key |
| alert_id | uuid (FK → Alert.id) | no | — | |
| user_id | uuid (FK → User.id) | no | — | Recipient within radius at trigger time |
| delivered_at | timestamp | yes | null | Null until the push provider confirms/attempts delivery |
| opened_at | timestamp | yes | null | Set if/when the user taps the notification |

## Constraints & indexes
- `Listing.id`, all other entity `id`s — primary key, uuid.
- `FacebookAnchor.listing_id` — unique + not-null-enforced-at-write (application-level gate, since
  the row's *absence* is what INV-001 forbids; a `Listing` write and its `FacebookAnchor` write are
  one transaction).
- `StatusHistory` — append-only (no update/delete path); insert-only index on `(listing_id,
  changed_at)` for the audit read (TC-N02) and for reconstructing "has this ever been marked
  resolved" without trusting a mutable field alone.
- `Listing(location_lat, location_lng)` — geo index (e.g., PostGIS GiST or equivalent —
  `[assumption]`, tied to the system-design store choice) to serve UJ-002 filter and the F-004
  alert-fanout radius query efficiently.
- `Listing(status, is_stale, kind)` — composite index for the default feed query (exclude
  `resolved`, exclude `is_stale = true`, filter by `kind`).
- `ScrapedPost.dedup_hash` — index for the F-001 dedup match on ingest.
- `ScrapeSource.fb_page_or_group_url` — unique (one row per configured source).
- `AlertDelivery(alert_id, user_id)` — unique (no duplicate delivery record per user per alert).
- `User.fb_user_id` — unique (one Whiskr account per Facebook identity).
- `Listing(batch_id)` — index for the "fetch every cat in this batch" read (F-006).
- `Session.token_hash` — unique + indexed (the lookup path for every authenticated request: hash
  the incoming bearer token, look up the `Session` row, reject if missing/expired/revoked).
- `Session(user_id)` — index for "revoke all my sessions" and account-deletion cascade reads.
- `PhotoUpload.photo_url` — unique + indexed (the lookup path for BR-013's ownership/consumption
  check at submission time).
- `User.phone_number` — unique, partial index `WHERE phone_number IS NOT NULL` (BR-016: one phone
  backs at most one verified account; unconfirmed candidates in `PhoneVerification` are not subject
  to this constraint, only the confirmed `User.phone_number`).
- `PhoneVerification(user_id, verified_at)` — index for "does this user have a pending/verified OTP
  cycle" reads.
- Foreign keys: `Listing.submitted_by → User.id`, `Listing.resolved_by → User.id`,
  `Listing.duplicate_of → Listing.id` (self-referential, nullable), `Listing.batch_id →
  ReportBatch.id` (nullable), `FacebookAnchor.listing_id → Listing.id`, `ScrapedPost.listing_id →
  Listing.id`, `ScrapedPost.scrape_source_id → ScrapeSource.id`, `StatusHistory.listing_id →
  Listing.id`, `Alert.listing_id → Listing.id`, `AlertDelivery.alert_id → Alert.id`,
  `AlertDelivery.user_id → User.id`, `UserLocation.user_id → User.id`, `PushToken.user_id →
  User.id`, `ReportBatch.submitted_by → User.id`, `Session.user_id → User.id`,
  `PhotoUpload.user_id → User.id`, `PhoneVerification.user_id → User.id`.

## Retention & privacy classification
- **User.fb_user_id** — PII. Retention: life of the account; deleted on account-deletion request.
  No Facebook access/refresh token is ever persisted at all (verify-then-discard at login,
  `ADR-0001`), reducing the secret-handling surface to zero long-lived Facebook credentials.
  **[assumption]** — no data-retention policy beyond "life of account" is stated in the seed; this
  follows standard practice, to confirm with product owner/legal at scaffold.
- **User.phone_number** — PII. Retention: life of the account or until the user removes it
  (**[assumption]**, no dedicated "remove phone" endpoint is specified here — revisit if requested;
  account deletion (BR-012) removes it along with everything else).
- **PhoneVerification.otp_hash** — secret-adjacent, same class as `Session.token_hash`: a hash, not
  the raw code; the raw OTP is only ever sent via SMS, never stored, never logged. Retention: until
  `expires_at`, garbage-collectable afterward (**[assumption]**).
- **PhotoUpload.photo_url** — internal (points at an object-store URL that becomes public/internal
  once consumed, same classification as `Listing.photo_url`). Retention: unconsumed rows may be
  garbage-collected after `expires_at` (**[assumption]**, operational default); consumed rows are
  kept for the life of the referencing `Listing`, as the audit trail for BR-013.
- **Session.token_hash** — secret-adjacent (same class as `PushToken.token`): a hash, not the raw
  token, but still sensitive — never logged, never returned in any read path (only the raw token is
  returned once, at creation, in API-007's response body). Retention: until `expires_at` or
  `revoked_at`, whichever comes first; expired/revoked rows may be garbage-collected on a schedule
  (**[assumption]**, no retention job is specified — a reasonable operational default, not a
  product requirement).
- **Listing.location_lat/lng, location_label** — PII-adjacent (can reveal a submitter's approximate
  home/found location). Classification: internal; visible to app users by product design (the
  location is the point of the listing), but precise-enough-to-dox precision should be reviewed —
  **[assumption]**, no precision/fuzzing rule is specified in the seed.
- **Listing.photo_url** — internal/public (shown in-app by design).
- **FacebookAnchor.fb_profile_url** — public by definition (it is a link to a public profile/page);
  classification: public.
- **UserLocation.lat/lng** — PII. Used only for radius matching (F-004); not shown to other users.
  Retention: until opt-out or account deletion.
- **PushToken.token** — secret-adjacent (device credential). Retention: until device
  unregisters/token rotates; not human-readable data but treated as sensitive.
- **ScrapedPost.raw_content_snapshot** — internal (retained for dedup/audit, not republished
  verbatim beyond the original post link per INV-003).
- No payment or health data exists in this model (`idea.md` §10 excludes monetization).

## Migration notes
- **[assumption]** — no existing schema/data exists yet (greenfield build, `team_size: 1`,
  `time_budget: 2w`); there is no legacy data to migrate or backfill at MVP.
- Forward compatibility to note for F-101/F-102/F-103/F-104 (final-product features, not built at
  MVP): `Listing` already carries fields (`kind`, `photo_url`, `location_*`) that F-101's photo
  matching would read without a breaking schema change; F-102's badge program would add a column to
  `User` (e.g., `verified_badge`) additively; F-103's messaging would be a new entity, not a change
  to existing tables; F-104's partner-API feed would add a new `source` enum value (`partner_api`)
  alongside `scraped`/`manual` — additive, not breaking. These are forward-compatibility notes only,
  not commitments — the features themselves are out of MVP scope per the PRD non-goals.
- Any schema change to `Listing.status`'s enum or to `StatusHistory`'s append-only guarantee is a
  change to INV-002 enforcement and must be treated as a logged pivot (per `idea.md` §9 note on
  invariants), not a routine migration.
- `ReportBatch` and `Listing.batch_id` (F-006, added 2026-09-18) are additive: a new table plus one
  new nullable FK column on the existing `Listing` table, no breaking change to any existing row or
  query. `User.fb_user_id` replaces the never-implemented `auth_identifier` column outright (no
  greenfield data existed to migrate, per `ADR-0001`).
- `Session` (added 2026-09-18, resolving the session-mechanism assumption) is a new, independent
  table — additive, no change to any existing entity.
- `PhotoUpload` (added 2026-09-18, closing the photo-ownership validation gap) is additive.
- `User.fb_signals_verified_at`/`track_record_verified_at`/`phone_number`/`phone_verified_at` and
  the new `PhoneVerification` table (added 2026-09-18, F-102 promoted to MVP) are additive — four
  new nullable columns plus one new table, no change to any existing row.

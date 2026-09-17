# Security & Compliance / Threat Model

> **Purpose:** risk and obligations. **Default to flagging missing auth on any exposed surface.**
> Traces back to: system design, data model, SRS.

## Data classification
<!-- Categories (public/internal/PII/secret) and where each lives. Pulled from data-model.md
     "Retention & privacy classification" — reused verbatim, not re-derived. -->

| Field | Classification | Notes |
|---|---|---|
| `User.fb_user_id` | PII | Retention: life of account, deleted on account-deletion request; no Facebook access/refresh token is ever persisted (verify-then-discard at login, `ADR-0001`) |
| `Listing.location_lat/lng`, `location_label` | PII-adjacent | Internal; visible to app users by product design (the location *is* the listing's point) — precision/fuzzing rule is **[assumption]**, not specified in seed |
| `UserLocation.lat/lng` | PII | Used only for radius matching (F-004); **not shown to other users**; retained until opt-out/account deletion |
| `PushToken.token` | Secret-adjacent | Device push credential; not human-readable but treated as sensitive; retained until device unregisters/token rotates |
| `Listing.photo_url` | Internal/public | Shown in-app by design |
| `FacebookAnchor.fb_profile_url` | Public | A link to a public profile/page, by definition |
| `ScrapedPost.raw_content_snapshot` | Internal | Retained for dedup/audit only; never republished verbatim beyond the original post link (INV-003) |
| Payment/health data | None | `idea.md` §10 excludes monetization; no health data exists in this model |

## Authn / authz model

- **Identity establishment.** Facebook OAuth is the sole account mechanism (`ADR-0001`, resolved
  2026-09-18) — the client obtains a Facebook access token via the on-device SDK; the backend
  verifies it against the Graph API and upserts `User` by `fb_user_id`. No password exists on
  Whiskr's side, so there is no forgot-password flow and no password-reset attack surface
  (credential stuffing, reset-token leakage) to defend.
- **Session mechanism** — **[assumption]** retained: no token/session scheme (JWT, opaque session
  token, etc.) for Whiskr's own bearer token is named in the seed. Whatever is chosen must be
  validated on every Backend API call per system-design's rule that the Backend API is the sole
  rule-enforcement point (client version must never matter).
- **Authorization surface**, derived from system-design + data-model without inventing new
  architecture:
  - **Read (feed browse/filter/search, F-001/F-003)** — no auth requirement is stated anywhere in
    the seed for browsing, and `Listing`/`FacebookAnchor` data is classified "visible to app users by
    product design." Proposed: **unauthenticated read is permitted** for feed/listing-detail
    endpoints — **[assumption]**, to confirm; this is a safety/utility tool where gatekeeping public
    listings behind login has no stated justification.
  - **Write (submission F-002, status transition F-003, location opt-in, push-token registration)**
    — requires an authenticated `User`. This is the surface INV-001/INV-002 depend on and must never
    be reachable anonymously.
  - **Status transitions (BR-004)** — authz check must be `changed_by == Listing.submitted_by OR
    User.is_admin`. `is_admin` provisioning has **no model in the seed** — **[assumption]**; until
    decided, no endpoint may allow a user to set their own `is_admin` (see T-009).
  - **Service-to-service (scraper/ingestion → Backend API)** — system-design states the scraper
    "converge[s] on the same `Listing` write path in the Backend API," i.e. it is a caller of the
    same write endpoint used by users, not a privileged bypass. It therefore needs its own
    service-level credential (distinct from a `User` session) so a compromised scraper credential
    cannot be escalated to arbitrary user-impersonating writes — **[assumption]**: no service-auth
    mechanism (API key, mTLS, etc.) is named in the seed; flagged as missing, not silently assumed
    away, per the architect role's "any network-exposed surface: design auth/authz explicitly or
    flag it" rule.
- **Open question:** the auth *mechanism* is now resolved (Facebook OAuth, `ADR-0001`); session
  lifetime/token format and admin-provisioning process remain unspecified anywhere upstream — the
  largest remaining security gap in the current doc set is narrower than before, not closed.

## Threat model

| T-ID | Threat (STRIDE) | Vector | Impact | Mitigation | Enforces |
|------|-----------------|--------|--------|------------|----------|
| T-001 | Spoofing | Submitter supplies a Facebook profile/page URL they do not own or control as the "identity anchor" | Fake trust signal; scam listings look legitimate | BR-002/BR-008 gate blocks a `Listing` write with no `FacebookAnchor` row at all; note this validates *presence/shape*, not *ownership* — deeper verification is F-102 (verified badge, final-product, out of MVP) — **residual risk**, called out not absorbed | INV-001 |
| T-002 | Tampering | Client bypasses the mobile Submit-button UX and calls the submission endpoint directly without a Facebook link | A listing publishes with no identity anchor | Server-side enforcement is the *only* gate (system-design: "every rule... is enforced server-side"); client UX is convenience only, never trusted | INV-001, BR-002/BR-008 |
| T-003 | Tampering | Non-owner user or attacker forges a status-transition call to mark a listing resolved/adopted (griefing) or to keep it "available" past resolution | False "available"/"missing" state persists or a legitimate listing is wrongly hidden | Authz check `changed_by == submitted_by OR is_admin` on every status-transition endpoint; every transition recorded in `StatusHistory` | INV-002, BR-004 |
| T-004 | Repudiation | Submitter/admin disputes having made a status change | No audit trail to resolve the dispute | `StatusHistory` is append-only, insert-only index on `(listing_id, changed_at)` — already the canonical audit source per data-model | INV-002 |
| T-005 | Information disclosure | API response leaks `UserLocation.lat/lng` (opt-in subscription) to another user or client | Precise home-area location of an opted-in user exposed | `UserLocation` rows are read only server-side for the radius match; must never appear in any client-facing response — **verify at implementation**, not yet a coded guarantee | Privacy classification (data-model) |
| T-006 | Information disclosure | `PushToken.token` leaked via logs or an API response | Token reuse enables push spam/impersonation of Whiskr to that device | Token returned only at registration ack, never in any subsequent read path; excluded from audit logs (see Audit & logging) | Secrets handling |
| T-007 | Tampering / Info disclosure | A normalization bug drops `ScrapedPost.original_post_url` or its rendering | Republished content with no attribution back to the original poster | `original_post_url` is NOT NULL at the schema level; no UI path may render scraped content without the link — **[gap]**: no automated check enforces the *UI* half of this today | INV-003 |
| T-008 | Elevation of privilege | User sets `is_admin = true` on their own account, or forges an admin-only status transition | Unauthorized resolve/override power over any listing | `is_admin` must not be mutable via any user-facing endpoint; provisioning path is **[assumption]**, undecided — flagged as an open gate, not resolved | BR-004, INV-002 |
| T-009 | Denial of service / availability | Facebook blocks, rate-limits, or structurally changes pages/groups the scraper polls | Ingestion pipeline (F-001) degrades or stops; feed coverage drops | `ScrapeSource.status` (active/degraded/blocked) is the operational signal; F-002 manual submission is the designed fallback — **this threat is also a named regulatory/kill-criterion risk, treated in its own subsection below, not only here** | Technical kill criterion (`idea.md` §9); see Compliance obligations |
| T-010 | Spoofing | An attacker replays a Facebook access token obtained for a *different* app (not Whiskr's own Facebook App ID) to log into Whiskr as that Facebook user | Account takeover without the victim ever touching Whiskr | Verify the token's `app_id` via Facebook's `debug_token` endpoint before trusting `fb_user_id` — **[gap]**, not yet confirmed as implemented; flagged in `api-spec.md` API-007 Open questions; guarded by `qa-test-plan.md` TC-008 | `ADR-0001` auth mechanism |
| T-011 | Denial of service / abuse | A user submits many fake multi-cat batch reports (F-006, API-010) to spam the feed/alert channel, each call producing N listings for the cost of one request | Feed/alert-fanout spam at N-times the cost of a single-cat submission per request | Same per-user write-rate-limit gap already named for API-003/API-004 (Rate limits, `api-spec.md`) now explicitly extends to API-010 at N-times weight — not a new gap, but a heavier instance of the existing one, worth naming now that F-006 exists | F-006 |

## Abuse & safety-specific risks (ethical)

Whiskr's failure modes can directly harm the people it claims to serve (missing-pet distress,
adoption scams), so this section is first-class, not a throwaway.

| Risk | Who is harmed | Trigger | Guard (INV-### / mitigation) |
|------|---------------|---------|------------------------------|
| Rehoming/adoption scam: a listing with a real-looking but uncontrolled Facebook link solicits an adopter (e.g., fees, deposits) | Prospective adopters (financial/emotional harm) | Any submission passes the shape-check anchor gate without true ownership verification | INV-001 is a partial deterrent only (raises the bar, doesn't close it); F-102 verified-badge program is the intended closer but is explicitly **final-product, not MVP** — this residual risk is unmitigated at MVP and should be named to the product owner, not hidden |
| False missing-cat report triggers a real push-alert fanout to nearby users | Opted-in nearby users (alert fatigue); the next genuine missing-cat reporter (diluted trust in the alert channel) | Any `kind=lost` listing is accepted with only the same identity-anchor check as any other listing | No dedicated guard beyond INV-001 exists in the current design — **open question**, not resolved by any doc reviewed |
| A submitter's or finder's precise home/found location is exposed at dox-able precision | The submitter/finder (stalking/harassment risk) | Default `location_lat/lng` precision is exact, not fuzzed | Data-model already flags precision/fuzzing as **[assumption]**, unresolved; this doc reiterates it as a live safety risk, not merely a data-quality note |
| A resolved/adopted/found listing stays visible past resolution, drawing a person to a cat that is no longer available (wasted trip, false hope on a missing-cat case) | Adopters/finders | Lag between the explicit-resolution write and staleness-sweep exclusion, or a submitter who never marks resolved | INV-002 (explicit resolution, immediate exclusion) + BR-005 (time-based staleness sweep) together bound — but do not eliminate — the exposure window |

## Compliance obligations

- **Data privacy (Philippines).** Whiskr collects and processes personal data (`auth_identifier`,
  precise device/user location, push tokens) from users physically in the Philippines (NCR/Greater
  Manila Area per `idea.md` §2). The Philippines Data Privacy Act of 2012 (RA 10173) is therefore
  **plausibly applicable** — **[assumption]**: no legal review is recorded in any seed doc, and this
  doc does not have standing to assert compliance; flagged here as an obligation to confirm with
  legal/product owner before any public launch, not asserted as satisfied.
- No payment, health, or other specially-regulated data categories exist in this model (per
  data-model.md's own note, citing `idea.md` §10's exclusion of monetization).

### Facebook ToS / scraping — compliance-risk subsection (named risk, not a throwaway line)

This is called out on its own because `idea.md` §9 names it as an explicit **kill criterion**
("Facebook blocks/bans the scraping mechanism entirely, or issues a takedown demand, and no
meaningful manual-submission volume exists to replace it") and `system-design.md` independently
flags it as "the single most consequential, hardest-to-reverse choice in this document."

- **Nature of the exposure.** F-001's ingestion pipeline polls Facebook pages/groups on a schedule
  without an official partnership or Graph API agreement (system-design explicitly rejects
  "official Facebook Graph API / partnership" for MVP because no business verification or
  page-owner cooperation exists yet, deferring that to F-104). Automated scraping of Facebook, even
  of public-only content, is widely understood to conflict with Facebook's Terms of Service around
  automated data collection. Possible consequences named or implied upstream: IP/account blocking,
  a takedown demand, or (unconfirmed, **[assumption]** — no legal opinion exists in any doc
  reviewed) further legal exposure. This doc does not assert a specific legal outcome; it names the
  exposure and defers the legal question to counsel.
- **Why this is a compliance risk and not merely an operational one.** Both `idea.md` (kill
  criterion) and `system-design.md` (integration-point note: "explicit ban/takedown is a named kill
  criterion, not merely an operational risk — flagged, not silently absorbed") already treat this
  as existential to F-001, not a tunable performance parameter. This doc inherits and preserves that
  framing.
- **Mitigations already designed upstream (reused, not invented here):**
  1. Scrape scope is "public content only" per the system-design context diagram — reduces, but does
     not eliminate, ToS exposure.
  2. F-002 (manual submission) exists specifically as the fallback path if scraping is
     degraded/blocked — system-design: "mitigated by the manual-submission pipeline as the A-001
     fallback."
  3. `ScrapeSource.status` (active/degraded/blocked) is the early-warning signal already wired into
     the data model for exactly this kill criterion.
  4. INV-003 (attribution never stripped) reduces reputational/copyright-adjacent exposure but does
     **not** resolve the underlying ToS question.
- **Residual/unmitigated exposure (named, not silently absorbed):**
  - No legal review of Facebook's ToS terms is recorded anywhere in the seed or design docs.
  - No scrape rate/backoff strategy tuned to avoid detection is specified beyond "polls on a
    schedule" (system-design) — **[open question]**.
  - No incident/response plan exists yet for an actual takedown demand or C&D letter — see
    Incident response basics below; this is a gap, not a solved item.
- **Recommendation:** treat `ScrapeSource.status` transitioning to `blocked` as a live trigger for
  the kill-criterion decision path in `idea.md` §9, and obtain a legal opinion on the scraping
  approach before any scale-up beyond MVP. Neither action is currently owned by any component in
  system-design — flagged as an open item for the orchestrator/product owner.

## Secrets handling

- **Push provider credentials** (APNs/FCM) — **[assumption]**, vendor itself unconfirmed
  (system-design). Referenced via environment/secret store, never inlined.
- **Database credentials** for the shared relational store — referenced via secret store; no value
  belongs in any doc or repo.
- **Session/auth-signing secret** — the auth *mechanism* is resolved (Facebook OAuth, `ADR-0001`),
  but the signing key for Whiskr's own issued bearer token is still `[assumption]` (format/algorithm
  undecided); whatever is chosen, the signing key must be rotated and never inlined.
- **Facebook App Secret** — used server-side if the chosen Graph API verification call requires it
  (e.g., an app-access-token for the `debug_token` check, T-010) — referenced via secret store,
  never inlined; distinct from any per-user Facebook access token, which is never persisted at all.
- **Service credential for scraper → Backend API** (see Authn/authz gap, T-009 context) — currently
  undesigned; flagged, not invented here.
- No secret value is ever written into this document or any doc in this set.

## Audit & logging

- **`StatusHistory`** (append-only) is the canonical audit trail for every status transition —
  already the source TC-N02 (INV-002 negative test) reads, per data-model.
- **`AlertDelivery`** (`delivered_at`, `opened_at`) is the audit trail for F-004's "alert was sent"
  acceptance criterion, per system-design.
- **`FacebookAnchor.resolved_at_submit`** records that the INV-001 gate ran at submit time — an
  audit marker, not just a data field.
- **`ScrapedPost.raw_content_snapshot`** is retained for dedup/audit only, never surfaced verbatim
  in the UI beyond the original link (INV-003).
- **Must NOT be logged in plaintext, ever:** `PushToken.token`, `User.fb_user_id`, the client-supplied
  `fb_access_token` at login (verified once, never persisted or logged), precise `UserLocation.lat/lng`
  outside the server-side radius-match code path. General application logs must not carry these
  fields — this is a requirement on the eventual implementation, not yet a verified property of any
  code.

## Incident response basics

- **Detection:** `ScrapeSource.status → degraded/blocked` is the primary designed detection signal
  (ties directly to the Facebook ToS risk above). No other monitoring/alerting mechanism is
  specified in any upstream doc — **open question**.
- **Escalation path:** **[assumption]** — `context.md`'s `team_size: 1` means there is no on-call
  chain to design; the product owner is the de facto single point of escalation until the team
  grows. Not sourced from any doc, stated here as the only coherent default.
- **Rollback:** deployment topology itself is `[assumption]` in system-design (no environment/
  staging split confirmed); no rollback procedure can be specified until that is decided —
  flagged, not fabricated.
- **Who to notify (data-privacy incident):** undecided — depends on the RA 10173 applicability
  question above being resolved with legal counsel first.

## Pre-milestone hard-gate checklist

- [ ] Every network-exposed surface declares auth/authz (no open write paths) — specifically:
      submission, status-transition, location opt-in, and push-token endpoints all require an
      authenticated `User`; feed-read endpoints' auth requirement is explicitly decided (not left
      implicit). — {date}
- [ ] No secret is committed; all secrets (DB, push provider, auth-signing key, scraper service
      credential) are referenced, not inlined. — {date}
- [ ] Every `INV-###` invariant still holds: INV-001 (no `Listing` without a `FacebookAnchor` row),
      INV-002 (no status shown past explicit resolution), INV-003 (no scraped listing missing
      `original_post_url`). — {date}
- [ ] The decision ledger and `docs/index.md §0` agree across live branches (reconcile pass run). —
      {date}
- [ ] `ScrapeSource.status` for every actively-relied-upon source is `active` (not `blocked`) —
      the Facebook ToS/scraping kill-criterion gate — before any public-facing milestone. — {date}
- [ ] No API response (feed, listing detail, or any other read path) returns `PushToken.token` or
      `UserLocation.lat/lng`. — {date}
- [ ] No Facebook user access/refresh token is ever persisted (verify-then-discard only at login,
      `ADR-0001`). — {date}

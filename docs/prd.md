# PRD — Product Requirements Document

> **Purpose:** the WHAT, for the team and for the executor's fence. Inherits `F-###` and `INV-###`
> from `idea.md`; defines `UJ-###` and `BR-###`; states one EARS acceptance criterion per feature
> that the QA plan turns into a deferred test. Traces back to `idea.md` §6, §7, §9 (+ BRD/MRD at scale).
> Traces forward to QA test plan, system design, design system, implementation plan.

## Overview & goals
Whiskr is a mobile app that centralizes cat-adoption and missing-cat listings for NCR and the
Greater Manila Area (Bulacan, Cavite, Laguna, Rizal), replacing reliance on Facebook's engagement
feed with a de-duplicated, status-aware, location-alerting view of listings (`idea.md` §1, §6). The
MVP must (1) show cross-page/group coverage no single Facebook page/group offers (F-001), (2) prove
that supply-side users will submit directly into Whiskr rather than only post to Facebook — the
riskiest assumption, A-001 (F-002), (3) stop showing resolved outcomes as open (F-003, INV-002), and
(4) alert nearby users the moment a missing-cat report appears (F-004), all without weakening trust
in a listing's origin (F-005, INV-001, INV-003).

## Personas & use cases
_(From `idea.md` §2; single target segment, two situational modes.)_
- **Adopter** — an NCR/Greater-Manila-area person actively looking to adopt a cat. Use case: browse
  and filter a de-duplicated feed instead of following dozens of Facebook pages/groups.
- **Finder/reporter** — someone who has found a stray, lost their own cat, or wants to report one
  missing. Use case: post directly into Whiskr (F-002) and get the report seen fast (F-004).
- **Rescuer / page admin (supply side, out of scope as a persona per `idea.md` §2 exclusion)** — not
  a designed persona for MVP UX, but the actor whose behavior A-001 measures: will they submit via
  F-002, or only post to Facebook. Excluded from persona design, included in the success metric.

## User journeys
<!-- STABLE IDs UJ-001… Referenced by QA (core smoke) and design system. -->
- **UJ-001** — Report a cat: a finder/reporter opens Home/Feed, taps **Report a cat**, attaches a
  Facebook profile (required, disabled Submit until attached), submits, and sees the listing live
  with status, linked profile, and an alert-sent confirmation. *(core demo journey — grounded in
  `usability.md` §1–§2, the approved 3-click flow that tests A-001; USABILITY CLEARED.)*
- **UJ-002** — Browse & filter the feed: an adopter opens Home/Feed, optionally enables location,
  and searches/filters the aggregated, status-tagged listings (F-001, F-003) to find a cat to adopt.
- **UJ-003** — Receive a nearby missing-cat alert: a user with location enabled receives a push
  alert when a Lost/Found report is posted within their radius (F-004), and taps into the listing.
- **UJ-004** — Resolve a listing: the original submitter or an admin marks a listing resolved
  (adopted/found), after which it must never again display as available/missing (F-003, INV-002).
- **UJ-005** — Sign up / log in: a new or returning user taps **Continue with Facebook**, authorizes
  via Facebook OAuth, and lands on Home/Feed with an account (display name sourced from Facebook, no
  password to set or remember — `ADR-0001`).
- **UJ-006** — View/edit profile: a signed-in user opens their profile, sees display name, admin
  status, and location-alert opt-in state, can edit their display name (BR-010), and can log out
  (BR-011).

## Feature list (with priorities)
<!-- Reuse F-### from idea.md §7 exactly. Do NOT invent feature IDs here. Every row gets a TC. -->

| F-ID | Feature | Priority | Solves (idea.md §) | Journey | Notes |
|---|---|---|---|---|---|
| F-001 | Aggregated listings feed scraped from NCR + Greater Manila Area cat adoption/missing-cat Facebook pages and groups | MVP | §1 | UJ-002 | Cross-page dedup is the core fragmentation fix |
| F-002 | Manual submission form (rescuers/finders/adopters post directly) | MVP | §1, §9 (A-001) | UJ-001 | Tests A-001; the riskiest-assumption feature |
| F-003 | Status tagging (available / on hold / adopted / found) + stale-listing suppression | MVP | §1 | UJ-002, UJ-004 | Two mechanisms: explicit resolution (INV-002) vs. time-based feed suppression |
| F-004 | Location-based missing-cat alert pinned to an area, pushed to nearby users | MVP | §1 | UJ-003 | Default radius 5 km per `usability.md` §1 confirmation copy |
| F-005 | Every post requires a linked, visible Facebook profile as an identity anchor | MVP | §7 (trust/scam risk) | UJ-001 | Enforces INV-001; blocks Submit until attached per `usability.md` §1 |
| F-006 | Multi-cat batch report: a single submission reports 2+ cats found/lost together, each becoming its own independently status-tracked listing | MVP | §7 (added 2026-09-18) | UJ-001 (extended) | Additive to F-002; grouped by a shared `ReportBatch` reference, not a new entity type (`decision-ledger.md`) |
| F-101 | Photo-based matching suggestions between lost/found reports | Final | §7 | — | Post-MVP; parked per `idea.md` §10 |
| F-102 | Verified rescuer/page badge program (manual vetting) | Final | §7 | — | Post-MVP |
| F-103 | In-app messaging between finder/adopter and poster | Final | §7 | — | Post-MVP; parked per `idea.md` §10 |
| F-104 | Formal partnership/API feed from established rescue pages | Final | §7 | — | Post-MVP; parked per `idea.md` §10; reduces scrape dependency (see system-design rejected-alternative note) |

## Business rules
<!-- STABLE IDs BR-001… Behaviour the system must enforce. QA references these. -->
- **BR-001** — A manual submission (F-002) SHALL require a status (Found / Lost / Adoptable), a
  photo, a location (map pin or address), and a description before Submit is enabled.
- **BR-002** — A manual submission's Submit control SHALL remain disabled until a Facebook profile
  is attached (enforces INV-001/F-005; grounded in `usability.md` §1).
- **BR-003** — A listing's status SHALL be one of: `available`, `on_hold`, `adopted`, `found`,
  `missing`, `resolved`. No other value is valid.
- **BR-004** — Only the original submitter or an admin SHALL be permitted to change a listing's
  status (enforces INV-002's "its submitter or an admin" clause).
- **BR-005** — A listing with no status change or activity for a defined staleness window
  ([assumption] 30 days — no value given in the seed; confirm with product owner) SHALL be flagged
  stale and suppressed from the default feed view, without altering its stored status (distinguishes
  staleness suppression from explicit resolution under INV-002).
- **BR-006** — A missing-cat report (status `missing`) SHALL trigger a location-based alert to users
  within a default 5 km radius of the report's location (per `usability.md` §1 confirmation text),
  unless the reporter sets a different radius. Radius customization UI is out of scope for MVP —
  **[assumption]**.
- **BR-007** — Every scraped listing SHALL retain a visible link to the original Facebook post
  (enforces INV-003; §1 identity/attribution).
- **BR-008** — A scraped post that cannot be resolved to a visible, working Facebook profile/page
  link SHALL NOT be published into the feed (enforces INV-001 for the scraper path, mirroring F-005
  for manual submissions).
- **BR-009** — A batch report (F-006) SHALL require at least 2 cats; each cat entry SHALL
  independently satisfy BR-001 (status, photo, location, description); the batch SHALL share exactly
  one `fb_profile_url` across all cats in the batch (BR-002/BR-008 validated once per batch, not per
  cat).
- **BR-010** — A user MAY edit their own `display_name` via profile (UJ-006); `is_admin` and any
  Facebook-identity field SHALL NEVER be user-editable through any client-facing endpoint (reinforces
  the `is_admin` gate named in `security-compliance.md` T-008).
- **BR-011** — A signed-in user MAY log out (UJ-006), immediately revoking their current session
  token; a revoked token SHALL be rejected on every subsequent authenticated call, identically to an
  expired one (`security-compliance.md` T-012).

## Hard rules / must-never (invariants — `INV-###`)
- **INV-001** — the system SHALL NEVER publish a post (scraped or manual) that does not carry a
  visible link back to a real Facebook profile or page. *(Enforced by: BR-002/BR-008, design-system
  banned-copy, TC-N01, packet safeguards.)*
- **INV-002** — the system SHALL NEVER keep a listing shown as `available` or `missing` past the
  point its submitter or an admin has marked it resolved. *(Enforced by: BR-004, status/staleness
  engine, TC-N02, packet safeguards.)*
- **INV-003** — the system SHALL NEVER scrape or republish content in a way that strips the original
  poster's attribution/link back to their post. *(Enforced by: BR-007, ingestion pipeline design,
  TC-N03, packet safeguards.)*

## User flows
_(Grounded in `usability.md` §1, USABILITY CLEARED, approved as-is; not redesigned here.)_

1. **UJ-001 Report a cat** — Home/Feed → tap `[Report a cat]` → Report-a-cat form (Status, Photo,
   Location, Description, Facebook profile field labeled "Required") → `[Attach Facebook profile]`
   (Submit stays disabled until this completes) → `[Submit report]` → Listing-posted confirmation
   (status tag, linked Facebook profile, "Nearby users have been alerted (5 km radius)") →
   `[Mark resolved]` or `[Back to feed]`.
2. **UJ-002 Browse & filter** — Home/Feed (location banner if not yet granted) → `[Enable location]`
   and/or `[Search / Filter]` → status-tagged, de-duplicated feed.
3. **UJ-003 Receive alert** — background push (F-004) fires on a new `missing` report within radius
   → tap notification → Listing detail (same card shape as Listing-posted confirmation).
4. **UJ-004 Resolve** — from Listing-posted confirmation or feed detail, submitter/admin taps
   `[Mark resolved]` → status transitions per BR-003/BR-004 → listing SHALL NEVER re-display as
   `available`/`missing` afterward (INV-002).

## Acceptance criteria (EARS — one per F-###; each becomes a TC)
- **F-001:** WHEN a user opens Home/Feed, the system SHALL display a single de-duplicated list
  merging listings scraped from all configured NCR/Greater-Manila-Area Facebook pages and groups,
  with no two entries representing the same underlying source post.
- **F-002:** WHEN a user submits the Report-a-cat form with a status, photo, location, description,
  and an attached Facebook profile, the system SHALL create a new listing visible in the feed within
  the same session, tagged `source: manual`.
- **F-003:** WHEN a listing's status is changed to `resolved`/`adopted`/`found` by its submitter or
  an admin, the system SHALL stop displaying that listing as `available` or `missing` in any feed
  view from that point forward; WHEN a listing has had no activity for the staleness window (BR-005),
  the system SHALL suppress it from the default feed view.
- **F-004:** WHEN a listing is created or updated to status `missing`, the system SHALL push a
  location-based alert to every user with location enabled within the configured radius (BR-006)
  and SHALL record that the alert was sent.
- **F-005:** IF a submission (scraped or manual) has no resolvable, visible Facebook profile/page
  link, THEN the system SHALL refuse to publish it to the feed (ties to INV-001/TC-N01).
- **F-006:** WHEN a user submits a batch report with 2 or more cats, each satisfying BR-001, and one
  shared attached Facebook profile, the system SHALL create one independently status-tracked listing
  per cat, all linked to a single batch reference, visible in the feed within the same session.

## Non-goals
_(Mirrors `idea.md` §10.)_
- Regions outside NCR + Bulacan/Cavite/Laguna/Rizal.
- Non-cat pets (dogs, other animals).
- Payments, donations, or any monetization flow.
- Photo-based automatic lost/found matching (parked as F-101).
- Verified-partner API integrations with rescue pages (parked as F-104).
- In-app messaging (parked as F-103).
- Password-based login or a "forgot password" flow — no app-side password exists; Facebook OAuth is
  the sole account mechanism (`ADR-0001`), so this is an intentional consequence, not a gap.

## Dependencies
- Public Facebook page/group content reachable for scraping (F-001) — availability and terms are an
  external dependency and a named kill criterion (`idea.md` §9: "Facebook blocks/bans the scraping
  mechanism").
- Push notification delivery (F-004) — platform provider (APNs/FCM) — **[assumption]**, no vendor
  named in the seed; to confirm at scaffold (system-design).
- Device location services (F-004, UJ-002/UJ-003) — user-granted permission; no location, no alert.
- A Facebook profile/page a user can link to (F-005, INV-001) — the product has no fallback identity
  anchor if a user has no Facebook presence; this is an explicit product constraint, not a gap. This
  same Facebook identity is now also the sole account login/signup mechanism (`ADR-0001`, resolved
  2026-09-18) — a user with no Facebook account cannot create a Whiskr account at all, a stronger
  version of the same constraint.

## Open questions
- Staleness window value (BR-005) — **[assumption]** 30 days; not specified in the seed.
- Default/adjustable alert radius (BR-006) — **[assumption]** 5 km fixed at MVP, per usability
  copy; whether radius becomes user-configurable is unresolved.
- Admin role definition/provisioning (who is "an admin" in BR-004/INV-002) — not specified in the
  seed; **[assumption]** a manually-provisioned internal role at MVP scale (`team_size: 1`).
- No `brand.md` exists (fast-path Key run) — no brand voice/visual identity facts exist to ground UI
  copy or the design-system doc against; anything brand-shaped in downstream docs must be marked
  `[assumption]`, not invented as if sourced.
- No `market.md` exists — the §5 size band ("tens of thousands") stays an idea.md-level
  `[assumption]`; no additional market sizing is available to ground GTM/MRD-level docs.
- Monetization is explicitly N/A at MVP (`idea.md` §8, §9) — no revenue-model open question is
  raised here by design.

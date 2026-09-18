# Decision Ledger — Whiskr

> **What this is.** One append-only, trunk-only record of pivots, rejected approaches, naming
> decisions, significant choices, and the assumptions/evidence behind those choices. It preserves
> decision history so a human or agent does not re-derive it from chats. It does **not** override the
> PRD, system design, QA plan, or implementation plan. A disagreement is a failed checkpoint: stop
> and reconcile the decision into the concern's canonical owner before continuing.
>
> **What this is NOT.** It is **not** a product specification, freeze, or task tracker. Product
> behavior belongs to the PRD, architecture to system design, test intent to the QA plan, and routine
> `TASK-###` status/work evidence to `docs/implementation-plan.md`. Put an item here only when it
> records a decision/pivot, rejects an approach, changes a name/immutable ID, or captures an
> assumption that informed a decision. The ledger makes change coherent, not slow.
>
> **Maintained mechanically, not by memory.** The git hook in `hooks/` and the discipline in
> `AGENTS.md` keep this current at commit time — the previous generation of this artifact went stale
> because it relied on human diligence under deadline. Do not rely on remembering to update it.
>
> **How an agent uses this:** read §1 (names/immutable IDs), §2 (decision assumptions), and §3
> (pivots/decisions) before changing a recorded choice. Never state a §2 UNVALIDATED assumption as
> fact. Then read `docs/index.md §0` and the canonical product/architecture/test/execution doc for
> the concern being changed.
> **Last reconciled:** 2026-09-18

## 1. Names & immutable identifiers (read first)

Branding changes; technical identifiers must not. Record every name and, critically, which IDs are
**immutable** so no agent "fixes" them.

| Name / ID | Kind | Where it appears | Rule |
|-----------|------|------------------|------|
| Whiskr | public product name | idea.md, seed docs, this ledger | Use everywhere a user/judge sees. |
| _(none recorded)_ | internal codename | — | No codename has been established in the seed. Do not invent one. |
| _(none recorded)_ | IMMUTABLE technical id | — | No infra/config identifiers (DB name, project id, bucket, etc.) exist yet — this ledger has no `dependsOn` on system-design. Record the first one here the moment system-design mints it, and mark it immutable at that point. |
| F-102's original scope | historical record (ID stays immutable, scope was redefined) | `idea.md` §7 Final list (note only, full text removed 2026-09-18 to avoid a false duplicate-ID gap in `check-seed.py`) | Original text: "Verified rescuer/page badge program (manual vetting)." Promoted to MVP and redefined 2026-09-18 to an automated, criteria-based badge (see §3 pivot below) — the ID `F-102` itself was never renamed or reused, only its scope changed, consistent with this table's own rule. |

> A rebrand (public name change) is a **pivot** (§3, type = `rebrand`): log it, then let the
> reconciler propagate the new public name everywhere **except** the IMMUTABLE rows above.

## 2. Decision assumptions & evidence (confidence, not a product spec)

These rows grade assumptions/evidence that informed a decision. They never redefine behavior,
architecture, tests, or task state; those stay in their canonical docs.

| SUPPORTED (decision context backed by events) | PROVISIONAL (accepted, not yet verified end-to-end) | UNVALIDATED (do not state as fact) |
|---|---|---|
| _(none — no `INT-###`/`EV-###` exist; fast path was chosen specifically to shape the idea before gathering evidence, per validation.md §1)_ | The approved 3-click UX flow (Home → Report a cat → Listing posted) — `usability.md` verdict `USABILITY CLEARED`, but this is a text-only semantic walkthrough, not a real-user test | A-001: rescuers/finders/page admins will feed listings into Whiskr via direct submission rather than relying solely on Facebook (idea.md §9) |
| | | A-002: people experiencing the pain will actually open/check a new app instead of staying inside Facebook (idea.md §9) |
| | | Segment size band "tens of thousands" (idea.md §5) — explicitly `[assumption]`, no sourced count |
| | | The four idea-shaping tests (Real/Large/Significant/Urgent) — all scored `unknown` in validation.md §1; none has supporting `EV-###` |
| | | Activation/retention success metrics (idea.md §8) — both marked `[assumption]` |

## 3. Pivots & decisions (newest first, append at top)

### 2026-09-18 — Account-deletion cascade was incomplete (5 of 10 User FKs missed; found and fixed)
- **Type:** decision (a hole found during a fifth docs-completeness audit pass, requested directly
  by the product owner and directed at this specific area)
- **Change:** traced every foreign key in `data-model.md` pointing at `User.id` (10 total) against
  BR-012's original enumeration and found only 5 were actually handled
  (`Session`/`UserLocation`/`PushToken` cascade-deleted, `Listing.submitted_by`/`ReportBatch.
  submitted_by` nulled). The other 5 were silent gaps: `Listing.resolved_by` and `StatusHistory.
  changed_by` (should null, same reasoning as `submitted_by` — retain the record, scrub the
  identity) were never mentioned at all; `PhotoUpload.user_id` and `PhoneVerification.user_id`
  (bookkeeping-only, safe to cascade-delete) were never mentioned; `AlertDelivery.user_id` is
  **not nullable**, so it needed a cascade-delete decision, not a null, and had neither.
- **Why:** without this, deleting an account would either violate a `NOT NULL` foreign-key
  constraint outright (`AlertDelivery`) or silently leave orphaned rows/PII behind
  (`PhotoUpload`/`PhoneVerification`) or leave a dangling reference an ORM might not even
  flag until it broke in production (`resolved_by`/`changed_by`). A "delete my account" feature
  that doesn't actually delete/scrub everything it should is a real product and compliance problem
  (RA 10173, `security-compliance.md`'s Compliance obligations section), not a cosmetic gap.
- **Alternatives rejected:** leaving `AlertDelivery.user_id` nullable-and-null instead of
  cascade-delete — rejected because a recipient-only audit row (unlike a `Listing`, which other
  users see and rely on) has no reason to persist once its sole subject's account is gone; deleting
  `Listing`/`ReportBatch`/`StatusHistory` rows outright instead of nulling — rejected, unchanged
  from the original BR-012 reasoning (other users and INV-002's audit guarantee depend on them).
- **Invalidated:** none — additive corrections to BR-012/`deleteAccount()`, not a reversal.
- **Recorded as:** none — a straightforward completeness fix, not a new trade-off; the one
  genuinely new trade-off (nulling `StatusHistory.changed_by` weakens T-004's repudiation mitigation
  for a deleted actor) is named inline in `security-compliance.md` T-004 and `prd.md` BR-012, not
  hidden in this ledger alone.

### 2026-09-18 — F-004's alert trigger was wired to a function it could never reach (bug found and fixed)
- **Type:** decision (a hole found during a third docs-completeness audit pass, not a pivot — no
  prior stated design intended this)
- **Change:** `technical-design.md` Algorithm 1b (`transitionStatus()`) was the only place that
  called `createAlertAndFanout()`, firing on "`old_status != 'missing' and new_status ==
  'missing'`." But `frd.md` F-003's own transition table has **no transition that lands on
  `missing`** — every listing that is ever `missing` got there at *creation* (`kind = lost`),
  never via a status transition. This means the alert trigger, as originally written, could never
  fire for a single real submission — F-004, arguably the product's single most safety-critical
  feature (a lost cat's owner depends on it), was dead on arrival in the design. The sequence
  diagram already documented the correct behavior ("`Backend API -> Alerting worker: enqueue (if
  status == missing)`" right after a manual submission) and `createBatchListings()` (Algorithm 5)
  already called it correctly per cat — only the single-cat creation path (`gateListingWrite()`,
  Algorithm 3, shared by both the manual and scraper pipelines) was missing the call. Fixed: added
  the trigger to `gateListingWrite()`; kept `transitionStatus()`'s copy as intentional
  forward-compatible dead code (PRD's F-004 EARS literally says "created **or updated**").
- **Why it was found:** a systematic trace of "which code paths actually set `status = missing`"
  against "which code paths call `createAlertAndFanout()`" during this audit, prompted by the
  product owner asking for more holes after two rounds had already found real bugs.
- **Alternatives rejected:** none — this is an unambiguous omission, not a design trade-off.
- **Invalidated:** none — `qa-test-plan.md` TC-004 already exercised the *creation* path
  correctly ("create a `Listing` with `status: missing`; run the alert-fanout worker"), so the test
  itself needed no change; only the algorithm it was testing was wrong.
- **Recorded as:** none — a straightforward correctness fix caught before any code existed.

### 2026-09-18 — F-102 promoted from Final to MVP and redefined: automated verification badge
- **Type:** `zoom-in` (a Final-scope idea narrowed and pulled into MVP) plus a scope redefinition
- **Change:** F-102 "Verified rescuer/page badge program (manual vetting)" (Final, post-MVP) →
  "Automated account verification badge" (MVP), granted when an account meets **any one** of three
  automated criteria: (1) Facebook profile signals checked once at signup (has a real, non-default
  profile photo and a name with more than one word — the only account-level signals Facebook's
  Graph API exposes without extended App Review; "account age" is explicitly **not** obtainable via
  the Graph API at any permission level, so it is not used, unlike a first instinct might assume);
  (2) an in-app track record (≥30 days account tenure AND ≥3 submitted listings); (3) a confirmed
  phone number via OTP. Effect: **display-only** — a badge shown on the user's listings/profile; it
  does **not** gate submission ability or change rate limits (product-owner's explicit choice,
  asked directly rather than assumed).
- **Why:** direct product-owner request (this session) to strengthen reporter trust beyond the
  per-listing Facebook anchor (F-005) — but scoped to what a solo, `team_size: 1` build can actually
  automate, not the original manual-vetting design, which needs an admin/reviewer this build has
  none of.
- **Alternatives rejected:** manual admin review (the original F-102 scope) — rejected because it
  needs a human reviewer/queue that doesn't exist at `team_size: 1`, and the product owner did not
  select it when asked directly; a government-ID document-verification step — rejected because it
  either needs manual review (same problem) or a paid third-party ID-verification vendor, a cost/
  scope commitment disproportionate to an MVP; making verification a gate on submission ability —
  rejected, product owner explicitly chose display-only.
- **Invalidated:** none — this is an upgrade/redefinition of an already-`[assumption]`-free Final
  feature, not a reversal of a prior MVP decision.
- **Recorded as:** none — additive, reversible via a routine schema/logic change; does not meet the
  ADR triple gate the way `ADR-0001` did (no forgot-password-style permanent behavioral loss here).

### 2026-09-18 — Photo-upload ownership/existence validation gap found and closed (BR-013)
- **Type:** decision (a hole found during a docs-completeness audit, not a pivot from a prior stated
  design — API-011 previously specified issuing an upload URL but never specified validating that
  the client actually used it before referencing `photo_url` on a submission)
- **Change:** (no ownership/existence check on `photo_url`) → a new `PhotoUpload` table
  (`data-model.md`) records every issued upload URL per user; API-003/API-010 now reject a
  `photo_url` that isn't an unconsumed `PhotoUpload` row owned by the caller, or whose object
  doesn't actually exist in storage.
- **Why:** without this, any client could submit an arbitrary string as `photo_url` — including a
  URL never uploaded to, or another user's upload URL — with no validation at all. A real gap in
  what "BR-001 requires a photo" actually enforced server-side.
- **Alternatives rejected:** trusting `photo_url` as opaque client-supplied text (the original,
  unaudited design) — rejected as the gap itself; validating only the URL's *shape* (matches the
  object-store's domain) without ownership tracking — rejected as insufficient, since it would
  still let one user submit another user's already-uploaded photo URL.
- **Invalidated:** none — additive to API-003/API-010/API-011, no prior claim is reversed.
- **Recorded as:** none — a straightforward correctness fix, not an architectural trade-off.

Each entry: date · what changed · **pivot type** · from → to · why · **invalidated claims** (reset
to UNVALIDATED) · superseding ADR (if any). Pivot types (from the design of record):
`zoom-in | zoom-out | segment | need | platform | use-case | market | rebrand | invariant-change`.
Several entries below are foundational Key-phase decisions rather than pivots from a prior state —
where none of the enumerated types fit, the entry says so plainly rather than forcing a mismatch.

### 2026-09-18 — Session mechanism resolved: opaque, hashed, revocable bearer tokens (not JWT)
- **Type:** decision (session-mechanism selection — no enumerated pivot type fits cleanly)
- **Change:** `[assumption]` no token/session scheme named → a new `Session` entity (`data-model.md`)
  backs every bearer token: the raw token is returned once at login (API-007) and never stored;
  only its SHA-256 hash is persisted, alongside `expires_at` (90-day fixed) and a `revoked_at` set by
  a new logout endpoint (API-012).
- **Why:** a real logout/revocation capability was added to complete the auth backend (login without
  a way to log out is an incomplete auth surface); a stateless JWT cannot be truly revoked without an
  equivalent denylist data-store lookup anyway, so an opaque server-side session token is simpler for
  the same guarantee.
- **Alternatives rejected:** JWT (rejected — no clean revocation without a denylist, which duplicates
  the `Session` table's job with extra complexity); storing the raw token instead of its hash
  (rejected — same reasoning as password hashing: a data-store leak should not yield directly
  reusable credentials).
- **Invalidated:** the `[assumption]` "session mechanism" line in `security-compliance.md` Authn/authz
  model and the "session/auth-signing secret" line in Secrets handling — both resolved by this
  decision.
- **Recorded as:** none — additive schema change, does not meet the ADR triple gate (reversible via a
  routine migration, not a product-facing trade-off).

### 2026-09-18 — Auth mechanism resolved: Facebook OAuth only, no app-side password
- **Type:** decision (auth-mechanism selection — no enumerated pivot type fits cleanly)
- **Change:** `User.auth_identifier` `[assumption]` (phone/email/OAuth, unresolved) → Facebook OAuth
  is the sole account login/signup mechanism; no app-side password exists, so no forgot-password
  flow is built.
- **Why:** direct product-owner decision (this session, asked because "forgot password" was
  requested alongside login/signup). Reuses the same Facebook identity every user already needs for
  F-005's per-listing anchor, avoiding a second identity system and its own password-reset
  infrastructure.
- **Alternatives rejected:** email + password (would require building/securing its own reset-flow
  infrastructure — password hashing, reset-token generation/expiry, reset-email delivery — unjustified
  duplication for `team_size: 1`); phone OTP (SMS delivery cost/infra with no stated justification in
  the seed); hybrid Facebook-OAuth-plus-optional-password (rejected as unnecessary doubled auth
  surface).
- **Invalidated:** the `[assumption]` auth-mechanism rows in `api-spec.md` (Authentication &
  authorization), `data-model.md` (`User.auth_identifier`), and `security-compliance.md` (Authn/authz
  model) — no longer UNVALIDATED, resolved by this decision. A "forgot password" feature is
  explicitly out of scope as a direct, intentional consequence, not an oversight.
- **Recorded as:** `docs/adr/ADR-0001` — meets the triple gate (hard to reverse once accounts exist
  under Facebook identity; surprising without context; a real trade-off — no forgot-password path,
  account recovery entirely dependent on the user's own Facebook account access).

### 2026-09-18 — New MVP feature: multi-cat batch reporting (F-006)
- **Type:** `need` (a real-world reporting case the original single-cat-per-submission model did not
  account for)
- **Change:** (F-002 assumed exactly one cat per manual submission) → a submitter may report 2 or
  more cats found/lost together in one batch request; each cat still becomes its own independently
  status-tracked `Listing`, grouped by a shared `ReportBatch` reference.
- **Why:** product-owner request (this session) — litters and multi-cat finds/losses are a common
  real-world case F-002's one-cat-per-form model did not cover.
- **Alternatives rejected:** modeling a batch as a single `Listing` with a cat-count field (rejected —
  breaks F-003's per-cat status lifecycle; a litter is rarely adopted/found all at once, and INV-002
  must hold per cat, not per batch); a fully separate `Report` entity distinct from `Listing`
  (rejected — would duplicate every F-003/INV-002 status rule for a second entity type; `ReportBatch`
  is a thin, additive grouping reference instead).
- **Invalidated:** none — additive to F-002/F-005; every cat in a batch still gets its own
  `FacebookAnchor` row and independently satisfies INV-001, unchanged.
- **Recorded as:** none — additive feature, not a hard-to-reverse architectural choice; does not meet
  the ADR triple gate.

### 2026-09-18 — Native mobile app chosen as form factor
- **Type:** `platform`
- **Change:** (no prior form factor) → native mobile app
- **Why:** Push notifications matter for the missing-cat alert use case (F-004) — a listing must
  reach a nearby user within the "buried within hours" urgency window described in idea.md §1. A
  website or a Messenger/Telegram bot cannot deliver the same reliable, OS-level push experience.
- **Alternatives rejected:** website-first (no reliable push); a lightweight Messenger/Telegram bot
  (weaker push/location control than a native app).
- **Invalidated:** none — first-recorded decision.
- **Recorded as:** none yet — flagged as an ADR candidate (see §6): hard to reverse once system
  design commits to a mobile stack, and a real trade-off (dev effort vs. notification reliability).

### 2026-09-18 — Missing-cat urgency mechanism: location-based alerts (F-004)
- **Type:** `use-case`
- **Change:** (no prior mechanism) → location-based alerts pinned to an area, pushed to nearby users
- **Why:** Directly targets the "buried within hours" urgency pain in idea.md §1 for time-critical
  missing-cat reports.
- **Alternatives rejected:** same-listing-type-only surfacing (too slow to reach the right people);
  photo-matching between lost/found reports (parked as F-101 — too ambitious for MVP).
- **Invalidated:** none — first-recorded decision.
- **Recorded as:** none.

### 2026-09-18 — Trust/verification approach for MVP: linked Facebook profile required (F-005 / INV-001)
- **Type:** decision (invariant-defining — not a change to a prior invariant, its origin)
- **Change:** (no prior verification mechanism) → every post must carry a visible, linked Facebook
  profile as an identity anchor
- **Why:** Addresses the scam/trust risk raised during capture (problem-capture.md parking lot;
  idea.md §7 F-005) with the lightest mechanism that still gives an identity anchor.
- **Alternatives rejected:** a heavier "verified badge" manual-vetting program (parked as F-102, final
  scope — too slow/heavy to gate MVP shipping on); shipping with no verification at all (unacceptable
  trust/scam risk).
- **Invalidated:** none — first-recorded decision.
- **Recorded as:** none yet — flagged as an ADR candidate (see §6): this is now `INV-001`, a hard
  must-never rule, and hard-to-reverse once posts exist without it. See §5 for the invariant audit.

### 2026-09-18 — Geographic scope: NCR + Greater Manila Area
- **Type:** `segment`
- **Change:** (no prior scope) → NCR + Bulacan, Cavite, Laguna, Rizal
- **Why:** Real cat-rescue communities already operate across this wider area, not strictly within
  NCR (idea.md §2, §10).
- **Alternatives rejected:** strict NCR-only scope (would exclude rescue communities already active
  across the adjacent provinces).
- **Invalidated:** none — first-recorded decision.
- **Recorded as:** none.

### 2026-09-18 — Hybrid sourcing model: scraping + manual submission
- **Type:** decision (data-sourcing strategy — no single enumerated pivot type fits cleanly)
- **Change:** (no prior sourcing model) → aggregate scraped listings (F-001) AND accept direct manual
  submissions (F-002)
- **Why:** Neither approach alone reaches useful coverage or supply; F-002 is also the only MVP
  feature that can test riskiest assumption A-001 (validation.md §2).
- **Alternatives rejected:** pure scraping only (Facebook ToS/legal risk, no consent from original
  posters); pure manual-submission only (too slow to reach useful listing coverage on its own).
- **Invalidated:** none — first-recorded decision.
- **Recorded as:** none yet — flagged as an ADR candidate (see §6): hard to reverse once the scraping
  pipeline and legal posture are built, and a real trade-off (legal/consent risk vs. coverage speed).

### 2026-09-18 — Fast path chosen over full validation
- **Type:** decision (methodology/rigor choice — no single enumerated pivot type fits cleanly)
- **Change:** (standard interview/market-research validation) → fast path (skip real interviews and
  market research at idea-shaping stage)
- **Why:** This is an idea-shaping-stage choice, not a rejection of rigor — context.md confirms
  `pivots_expected: true` precisely because the idea is unvalidated by choice at this stage. The build
  itself is intended to be the first evidence (validation.md §6), specifically for A-001 via F-002 vs.
  F-001's scraped baseline.
- **Alternatives rejected:** running real interviews / market research before writing idea.md (would
  have delayed idea-shaping; deferred, not abandoned).
- **Invalidated:** none — first-recorded decision. Note per §2: this is *why* the four idea-shaping
  tests and A-001/A-002 are UNVALIDATED, not a sign they were skipped carelessly.
- **Recorded as:** none yet — flagged as an ADR candidate (see §6): surprising without context (a
  GO-UNVALIDATED verdict on zero evidence looks like a red flag unless the rationale is visible), and
  a real trade-off (speed-to-build vs. certainty).

## 4. Rejected approaches (what we tried and killed — and why)

The highest-value section and the one most logs omit. Record what you *tried and dropped*, so the
next build (and the framework) learns from it. A rejected approach is not failure — it is evidence.

| Approach considered | Rejected because | Would revisit if |
|---------------------|------------------|------------------|
| Pure scraping only (no manual submission) | Facebook ToS/legal risk, no consent from original posters (idea.md §9 kill criteria; problem-capture.md parking lot) | Formal partnership/API feed with rescue pages is secured (F-104), removing the legal/consent risk |
| Pure manual-submission only (no scraping) | Too slow to reach useful listing coverage on its own | Manual submission volume alone proves sufficient to sustain coverage (would also strongly validate A-001) |
| Strict NCR-only geographic scope | Real cat-rescue communities already operate across the wider Greater Manila Area, not just NCR | GMA-area (Bulacan/Cavite/Laguna/Rizal) coverage stays negligible in practice |
| No identity verification at all on posts | Unacceptable scam/trust risk raised during capture | Never — this is now `INV-001`, a hard must-never rule |
| Heavier "verified badge" manual-vetting program at MVP | Too slow/heavy to gate MVP shipping on; a lighter identity anchor (linked Facebook profile) suffices for MVP trust | Reconsidered at final-scope phase as `F-102` |
| Same-listing-type-only surfacing for missing-cat urgency | Too slow to reach nearby people in the "buried within hours" urgency window | Location-based alerts (F-004) underperform in practice |
| Photo-based lost/found matching at MVP | Too ambitious for MVP | Reconsidered at final-scope phase as `F-101` |
| Website-first form factor | No reliable push notifications, which matter for the missing-cat alert use case | Web push proves reliable enough, or native app dev cost proves infeasible for a solo builder |
| Messenger/Telegram bot form factor | Weaker push-notification and location-UX control than a native app | Native app dev cost proves infeasible for a solo builder and a bot becomes a viable stopgap |

## 5. Invariant audit (the `INV-###` guardrails)

Every change that touches a hard product rule (`INV-###` from `idea.md §9` → the PRD must-never
rules) gets a line here **before merge** — the guardrail that has no audit trail is the one that
gets breached by accident under pressure.

| Date | INV-### | Change that touched it | Audit verdict |
|------|---------|------------------------|---------------|
| 2026-09-18 | INV-001 | Established during the Key phase: every post must carry a visible, linked Facebook profile (F-005) — decided as the MVP trust/verification mechanism (see §3) | kept — upheld as a hard invariant rather than softened; the heavier "verified badge" alternative was deferred to final scope (F-102) instead of weakening the MVP requirement |
| 2026-09-18 | INV-001 | F-006 (multi-cat batch reporting) extends F-002's submission path to N cats per request | kept — each cat in a batch still gets its own `FacebookAnchor` row committed in the same all-or-nothing transaction; INV-001 is enforced per-listing, unchanged by batching |
| 2026-09-18 | INV-002 | **Bug found and fixed** during a pre-code docs audit: `technical-design.md` Algorithm 1b's `TERMINAL_STATUSES` set omitted `found` and allowed an `actor.is_admin` exception to the terminal-state guard — both contradicted `frd.md` F-003's own transition table ("no override, including for admins"; all three of `adopted`/`found`/`resolved` are terminal) | **fixed, not kept as-was** — this was a genuine INV-002 breach in the design (never in delivered code, since no code exists yet): a `found` listing could have illegally reopened to `available`/`missing`, and an admin could have bypassed the terminal guard entirely. `TERMINAL_STATUSES` corrected to `{adopted, found, resolved}`; the guard is now unconditional (no admin exception); `ACTIVE_STATUSES` corrected to `{available, on_hold, missing}` |

## 6. Open items / risks (do not lose these)

- `context.md` marks `team_size`, `build_type`, `time_budget`, and `handoff_expected` as
  `[assumption]` — the product owner should confirm/correct these before phase-5.0 doc-selection is
  treated as final (validation.md §5 flags this too).
- A-001 and A-002 remain UNVALIDATED. Per validation.md §6, the build itself is the first evidence:
  watch real-world direct-submission volume through F-002 against F-001's scraped baseline to
  confirm or falsify A-001.
- Independence on the validation.md verdict is `DEGRADED` (no cross-family validator reachable in
  this environment) — treat the GO-UNVALIDATED verdict with that caveat in mind.
- ADR candidates flagged in §3 (fast path over full validation; hybrid sourcing model; INV-001
  trust/verification mechanism; native mobile app platform choice) — promote to `docs/adr/` during
  phase-5.1 system-design only if the triple gate is met (hard to reverse AND surprising without
  context AND a real trade-off).

## References
- `docs/index.md §0` — source-of-truth map (one fact, one home)
- `docs/adr/` — significant decisions (each pairs with a §3 line — enforced by the git hook)
- `hooks/` — the commit-time enforcement that keeps this ledger honest

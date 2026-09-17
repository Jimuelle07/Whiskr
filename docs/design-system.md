# Design System / UX Spec

> **Purpose:** consistent UX. Principles, components, tokens, patterns, accessibility.
> Traces back to: PRD user flows (`docs/prd.md`), which trace to `docs/seed/usability.md` §1
> (`USABILITY CLEARED`, the approved screen/action flow — reused literally below, not redesigned).

## Open questions (read first)

- **No `docs/seed/brand.md` exists**, even though `usability.md` records `has_ui: true`. The Key's
  fast path produced no brand voice or visual identity source. This doc therefore has **no sourced
  color, type, or voice identity to ground against** — the manifest's instruction to "ground in the
  brand sibling" cannot be honored because the sibling doesn't exist.
  - **Handling:** every token/style choice below is a neutral, functional placeholder, marked
    **[assumption]**, chosen only for legibility and status-color clarity — not a decided brand.
    Nothing here should be read as an approved visual identity.
  - **Action needed before visual design is finalized:** produce `docs/seed/brand.md` (or run a
    brand pass) and then revise this doc's Tokens + voice sections against real brand facts. Until
    then, treat every `[assumption]` token as swappable with zero downstream contract risk — no
    component behavior in this doc depends on a specific color/type value.
- Staleness window (BR-005), alert radius customization (BR-006), admin role definition — all
  already flagged `[assumption]` in `docs/prd.md`; carried here only where they affect a shown
  state (e.g., stale-suppression empty state), not re-litigated.
- No accessibility audit tooling/target has been run; WCAG AA below is a target, not a verified
  conformance (per template note, conformance needs manual AT testing).

## Design principles

1. **Trust the source, always show it.** Every listing surfaces its Facebook profile/page link
   visibly (INV-001, INV-003, F-005) — never hidden behind a tap, never optional.
2. **One unblocked action at a time.** Per `usability.md`'s cleared 3-click flow: disabled states
   (e.g., Submit) always carry a visible reason, never a silent block (K3).
3. **Status is never ambiguous.** A listing's status tag is always visible on its card, everywhere
   it appears (feed, detail, confirmation) — resolved states must not visually resemble open ones
   (F-003, INV-002).
4. **Location is opt-in, explained before asked.** The location banner states what enabling it
   gets the user (listings + alerts near you) before asking for the permission.
5. **No dead ends.** Every terminal screen (e.g., Listing posted) offers an explicit next action
   (`[Mark resolved]` / `[Back to feed]`) — never leaves the user without a labeled way forward.

## Routes / screens

_(Literal from `usability.md` §1 — the approved flow. Not redesigned; only structured for
implementation.)_

| Route | Screen | Source |
|---|---|---|
| `/` (Home/Feed) | Home / Feed | `usability.md` §1, UJ-002, UJ-003 (alert tap target) |
| `/report` | Report a cat | `usability.md` §1, UJ-001, F-002 |
| `/report/confirmation` | Listing posted (confirmation) | `usability.md` §1, UJ-001 |
| `/listing/:id` | Listing detail | UJ-003 ("same card shape as Listing-posted confirmation"), UJ-004 |

## Screen specs (literal transcription from usability.md §1 + prd.md user flows)

### Home / Feed (`/`)
- **Shows:** Banner — "Enable location to see cat listings and alerts near you." (shown only
  pre-grant). Below: de-duplicated NCR-wide feed (F-001), each card tagged
  Available / On hold / Adopted / Found / Missing (F-003).
- **Actions:** `[Enable location]` · `[Report a cat]` · `[Search / Filter]`.
- **States:** first-open (location ungranted, banner shown) · location-granted (banner replaced by
  nearby-alert eligibility, no copy specified beyond removal — **[assumption]**: banner disappears,
  no replacement copy invented) · filtered (search/filter applied, F-001/F-003 scoped) · empty
  (no listings match filter/location — no copy sourced, **[assumption]** neutral empty state, see
  Patterns & states) · stale-suppressed (BR-005 listings absent from default view, no distinct
  empty-vs-suppressed copy sourced — **[assumption]**).

### Report a cat (`/report`)
- **Shows:** Form fields — Status (Found / Lost / Adoptable), Photo, Location (map pin or address),
  Description; "Link your Facebook profile" field labeled "Required — anchors your report to a real
  Facebook profile so others can trust it." Submit is visibly greyed out / disabled until a
  Facebook profile is attached (BR-002).
- **Actions:** `[Attach Facebook profile]` · `[Submit report]`.
- **States:** incomplete (Submit disabled, reason visible per principle 2) · profile-attached
  (Submit enabled) · submitting (no loading copy sourced — **[assumption]** standard spinner/disable
  pattern, see Patterns & states) · validation error (BR-001 required fields missing — no error
  copy sourced — **[assumption]** inline field-level error, see UI voice & copy rules).

### Listing posted / confirmation (`/report/confirmation`)
- **Shows:** New listing live in feed, status tag (F-003), linked Facebook profile (F-005).
  Confirmation text: "Nearby users have been alerted (5 km radius)" (F-004, BR-006).
- **Actions:** `[Mark resolved]` · `[Back to feed]`.

### Listing detail (`/listing/:id`)
- **Shows:** Same card shape as Listing posted confirmation (per `prd.md` UJ-003) — status tag,
  linked Facebook profile, description, location. Reached via push-notification tap (UJ-003) or
  feed tap.
- **Actions:** `[Mark resolved]` (submitter/admin only, BR-004) · `[Back to feed]`.
- **States:** resolved (post-BR-004 transition — must never re-render as available/missing,
  INV-002) · viewer-is-not-submitter/admin (`[Mark resolved]` not shown or disabled — action gating
  not detailed in seed — **[assumption]**: hide the control rather than show-disabled, since no
  "why disabled" copy exists to satisfy principle 2).

## Component inventory

_(Derived from the screens above — every component traces to a shown element or action, none
invented beyond what the flow requires.)_

- **StatusTag** — renders one of `available / on_hold / adopted / found / missing / resolved`
  (BR-003). Used on: feed card, listing detail, confirmation screen.
- **ListingCard** — status tag + photo + linked Facebook profile + location snippet. Used on: feed,
  (expanded) listing detail.
- **FacebookProfileLink** — visible, tappable link element; renders on every published listing
  (INV-001/INV-003) and required by BR-008 before a scraped post publishes at all — i.e., its
  absence is a publish-blocker, not just a UI omission.
- **LocationBanner** — pre-grant permission explainer + `[Enable location]` CTA; disappears once
  granted (**[assumption]** re: replacement state, see above).
- **DisabledSubmitButton** (with reason text) — the general pattern behind Report-a-cat's Submit
  control; reason text is mandatory whenever this component is used (principle 2).
- **ReportForm** — Status / Photo / Location / Description / Facebook-profile fields + the two form
  actions.
- **AlertConfirmationBanner** — "Nearby users have been alerted (N km radius)" text component;
  radius value is data-driven (BR-006), not hardcoded copy.
- **SearchFilterControl** — feed-level search/filter entry point (UJ-002); filter dimensions are
  status (F-003) and location, per `usability.md`/`prd.md`; no additional filter facets are sourced.
- **MarkResolvedAction** — status-transition control gated to submitter/admin (BR-004); appears on
  confirmation and detail screens.

## Tokens

**All values below are [assumption] — neutral placeholders pending a real brand pass (see Open
questions). None encode a decided brand identity.**

- **Color — functional/status only** (chosen for distinguishability, not brand) **[assumption]**:
  - `color.status.available` — green family
  - `color.status.on_hold` — amber family
  - `color.status.adopted` / `color.status.found` — blue/teal family (resolved-positive)
  - `color.status.missing` — red family (urgent)
  - `color.status.resolved` — neutral grey (deliberately desaturated vs. open states, per principle 3
    — resolved must not look "live")
  - `color.text.primary`, `color.text.secondary`, `color.surface`, `color.border` — system-default
    neutral greyscale **[assumption]**
- **Typography** — system default font stack (no typeface sourced) **[assumption]**; scale:
  `type.heading`, `type.body`, `type.caption`, `type.label` (label used for the disabled-Submit
  reason text and field labels like "Required").
- **Spacing** — 4/8/16/24/32 px scale **[assumption]**, standard mobile touch-target minimum 44px
  for all tappable actions (`[Enable location]`, `[Report a cat]`, `[Attach Facebook profile]`,
  `[Submit report]`, `[Mark resolved]`, `[Back to feed]`).
- **Radius** — one consistent card/button radius token, value unset pending brand **[assumption]**.
- **Elevation** — flat + one raised level for cards over feed background **[assumption]**.

## Patterns & states

- **Forms:** Report-a-cat is the only form in scope. Disabled-until-valid pattern (Submit) is
  mandatory per BR-001/BR-002 and principle 2 — never disable without visible reason text.
- **Empty states:** feed with no matching listings (filtered or region has none). No copy sourced
  from seed/PRD — **[assumption]** plain, honest text (e.g., no listings found for this filter) —
  never a banned overclaim (see UI voice & copy rules — none currently defined, see below).
- **Loading states:** no seed copy exists for submit-in-flight or feed-loading — **[assumption]**
  standard spinner + disabled controls, no invented microcopy.
- **Error states:** BR-001 validation (missing status/photo/location/description) and a failed
  Facebook-profile attach are the only sourced error triggers. No specific error copy is given in
  `usability.md` — **[assumption]** inline, field-level, states what's missing, never blames the
  user.
- **Resolved state (INV-002):** once transitioned, a listing must render identically to "already
  resolved" everywhere it appears — no code path may re-render a resolved listing under
  `available`/`missing` styling. This is a testable rendering invariant, not just a data one.

## UI voice & copy rules (enforced, not optional)

**Banned copy:** `idea.md` and `prd.md` define no `INV-###`-tied banned-phrase list beyond the
identity/status invariants already enforced structurally (FacebookProfileLink presence, StatusTag
correctness). **No banned-copy list currently exists in the seed — stating this plainly rather than
inventing one.** If a future brand pass or product-owner review identifies unsafe phrasing (e.g.,
overclaiming safety/verification of a poster, or implying Whiskr itself vets rescuers — note F-102
verified-badge is explicitly Post-MVP/Final per `idea.md` §7), it should be added here as
`Never use: "{phrase}" — breaches INV-###`.

One copy constraint IS sourced and is enforced here even absent a formal ban list:
- Any status/confirmation copy MUST NOT imply verification or vetting of a poster's identity beyond
  "has a linked Facebook profile" (e.g., never say "verified rescuer" or "trusted poster" — F-102's
  verified-badge program is explicitly out of MVP scope, `idea.md` §7/§10). Ties to INV-001's intent
  (an identity anchor, not an endorsement).

**Tool/default overrides:** none — no component library or framework has been chosen in this doc;
that choice belongs to `docs/system-design.md`, not here.

**Provenance:** all screen text above is transcribed verbatim from `docs/seed/usability.md` §1 and
`docs/prd.md`'s user flows section — no UI copy in this doc was invented. Tokens are
`[assumption]` placeholders, not transcribed from any source file (none exists). Last synced:
2026-09-18.

## Accessibility standards

- **Target:** WCAG AA **[assumption]** — no accessibility target is specified in the seed/PRD; AA
  chosen as the conventional baseline for a consumer mobile app, not a sourced requirement. Flagged
  as an open NFR.
- Status-color tokens (above) must not be the only signal of status — pair every `StatusTag` with a
  text label (already implied by the sourced copy, e.g., "Missing", "Adopted"), so color-blind users
  aren't relying on hue alone.
- All primary actions listed under Spacing must meet a 44px minimum touch target and be reachable
  via keyboard/switch-access focus order matching the visual flow (top-to-bottom per screen).
- Disabled-Submit reason text (principle 2) must be programmatically associated with the Submit
  control (e.g., `aria-describedby`) so assistive tech announces *why*, not just *that*, it's
  disabled.
- Full conformance requires manual testing with assistive tech and expert review — not yet
  performed.

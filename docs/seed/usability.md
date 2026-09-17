# Usability gate — Whiskr

<!-- EMITTED as docs/seed/usability.md. Written by concept_validator (GPT-5.6 Terra) in the phase-4
     dispatch, from the finished idea.md §7 + §9 A-001 only. Text-only semantic usability test —
     see playbooks/usability-gate.md. The brief is never edited here. The Vault reads the approved
     flow as a phase-5.1 design input and NEVER gates on it. -->

**Verdict:** `USABILITY CLEARED`
**Cycle:** 1 of 2 · **Date:** 2026-09-18 · **has_ui:** true
**A-001:** Rescuers, finders, and page admins will feed their listings into Whiskr (via direct submission) rather than relying solely on posting to Facebook — without that, Whiskr has no independent supply and is just a lower-coverage mirror of Facebook.

## 1. UX flow (text only)

```
Screen: Home / Feed (first open — location not yet granted, nothing configured)
Shows:   Banner: "Enable location to see cat listings and alerts near you." Below it, a
         de-duplicated NCR-wide feed of aggregated listings scraped from Facebook pages/groups
         (F-001), each card tagged Available / On hold / Adopted / Found / Missing (F-003).
Actions: [Enable location] [Report a cat] [Search / Filter]

Screen: Report a cat (F-002)
Shows:   Form fields: Status (Found / Lost / Adoptable), Photo, Location (map pin or address),
         Description; a "Link your Facebook profile" field labeled "Required — anchors your
         report to a real Facebook profile so others can trust it." Submit button is visibly
         greyed out / disabled until a Facebook profile is attached.
Actions: [Attach Facebook profile] [Submit report]

Screen: Listing posted (confirmation)
Shows:   The new listing now live in the feed, tagged with its status (F-003) and showing the
         linked Facebook profile (F-005). Confirmation text: "Nearby users have been alerted
         (5 km radius)" (F-004).
Actions: [Mark resolved] [Back to feed]
```

**Feature coverage:** F-001 → Home/Feed · F-002 → Report a cat · F-003 → Home/Feed + Listing posted · F-004 → Home/Feed banner + Listing posted · F-005 → Report a cat + Listing posted (every MVP feature listed).

## 2. Impatient-user walkthrough

Persona: impatient, non-technical, will not read help, abandons on the second confusion. Goal: the `A-001` outcome — a finder feeds a listing directly into Whiskr rather than only posting to Facebook.

| Click | Screen | Action taken | Why it is the obvious choice | Leads to |
|---:|---|---|---|---|
| 1 | Home / Feed | `[Report a cat]` | Goal is to report a found cat; this is the only action naming that outcome (location prompt and search do not) | Report a cat |
| 2 | Report a cat | `[Attach Facebook profile]` | Submit is visibly disabled and the field is labeled "Required" with a stated reason, so this is the only unblocked action | Report a cat (profile now attached, Submit enabled) |
| 3 | Report a cat | `[Submit report]` | Now the only enabled action on the screen | Listing posted |

**Outcome reached at click 3:** Listing posted screen shows the report now live in the feed with a status tag, the linked Facebook profile, and "Nearby users have been alerted" — the finder has fed a listing directly into Whiskr, satisfying `A-001`.

## 3. Kill criteria

| # | Criterion | Tripped? | Evidence (step) |
|---|---|---|---|
| K1 | User must guess the next step | no | At each step only one action is enabled/obviously tied to the stated goal (step 1: only CTA naming "report"; step 2: Submit disabled until profile attached; step 3: only remaining enabled action) |
| K2 | Goal is more than 3 clicks from the start screen | no | click count = 3 |
| K3 | User missing context a decision needs | no | Step 2's disabled Submit is paired with a visible reason ("Required — anchors your report..."), so the user is not asked to decide (attach profile) without knowing why |

## 4. Decision

- **`USABILITY CLEARED`** — no criterion tripped. The flow in §1 is approved as-is; it is the Vault's screen-flow input for the PRD user journeys and design-system.

# Release / Go-to-Market & Roadmap

> **Purpose:** shipping and growth.
> Traces back to: BRD, MRD, PRD.
>
> **Grounding note:** no `docs/seed/market.md` exists for this build (fast-path Key run skipped
> phase-3 market research). This doc is grounded only in `idea.md` §5's existing `[assumption]`-
> tagged market content and the PRD; no new market sizing, competitor names, or channel data has
> been invented. See **Open questions** below.

## Release plan & phases
- **Alpha (internal/closed):** ship MVP scope (F-001–F-005) to a small closed group of adopters and
  finders/reporters in NCR (`idea.md` §2 target segment). Entry: QA test plan TC-### green; INV-001/
  002/003 enforced. Exit: A-001 signal is readable — F-002 (manual submission) volume vs. F-001
  (scraped) volume can be compared (`validation.md` §2/§6).
- **Beta (open, limited region):** open to the full NCR + Greater Manila Area footprint (Bulacan,
  Cavite, Laguna, Rizal — `idea.md` §2). Exit: activation/retention metrics (`idea.md` §8) trending,
  no INV violations recorded.
- **GA:** no GA gate is defined in the seed — **[assumption]**: same regional footprint as beta,
  since expansion beyond NCR/Greater Manila Area is an explicit non-goal (PRD Non-goals; `idea.md`
  §10).
- **[assumption]** Phase names/durations are not specified anywhere in the seed; the sequencing above
  follows MVP feature scope only and should be confirmed with the product owner.

## Rollout / rollback strategy
- No rollout mechanism (flags, canary, blue-green) is specified in `idea.md` or the PRD —
  **[assumption]/open question**. Given the solo/small-team build implied by the seed, a full-cutover
  release per phase with rollback = revert to last known-good build is the minimal honest default.
- Whatever mechanism is chosen, INV-001/INV-002/INV-003 must hold at every rollback point: no
  rollback path may re-publish a post without a Facebook identity anchor, re-surface a resolved
  listing, or strip original-post attribution.

## Launch checklist
- QA test plan TC-### all green, tracing to PRD F-001–F-005 EARS acceptance criteria.
- INV-001/002/003 enforcement verified end-to-end (no listing published without a resolvable
  Facebook profile/page link; no resolved listing re-displays as available/missing; scraped
  attribution never stripped).
- Push notification provider confirmed (PRD Dependencies flags APNs/FCM as **[assumption]**,
  unconfirmed in the seed).
- Facebook scraping mechanism's availability/terms risk (named kill criterion, `idea.md` §9)
  explicitly acknowledged pre-launch — the seed provides no mitigation for it.
- Security-by-default sign-off — not addressed anywhere in the seed; carried as an open question
  below rather than assumed clear.

## GTM channels
_(idea.md §5 Reachability — the only channel data that exists; both entries are already tagged
`[assumption]` in the seed. No additional channels, paid/owned/earned breakdown, or performance data
has been added here.)_
1. Direct outreach to known NCR-area cat rescue Facebook pages to request cross-posting/partnership.
2. Posting in the largest existing NCR cat adoption/missing-cat Facebook groups to recruit early
   users.

Both named channels route through Facebook — the same platform Whiskr's value proposition (`idea.md`
§6) is positioned against. This tension is unresolved in the seed; see open questions.

## Success metrics & instrumentation
- **Activation** — **[assumption]** (`idea.md` §8): a new user completes at least one search/filter,
  or submits/claims one listing, within their first session.
- **Retention** — **[assumption]** (`idea.md` §8): a user reopens the app within 7 days of a
  missing-cat alert in their area, or checks adoption listings weekly.
- **Revenue/value** — N/A at MVP; no monetization hypothesis has been validated (`idea.md` §8, §9).
- **A-001 instrumentation** (the riskiest assumption, `validation.md` §2/§6): track F-002 direct-
  submission volume against F-001 scraped-listing volume over time. If F-002 volume stays near zero
  while F-001 keeps growing, A-001 is falsified — Whiskr is confirmed as a lower-coverage Facebook
  mirror, per `validation.md` §6.
- Instrumentation mechanics (event schema, analytics tooling) are not specified in the seed —
  **[assumption]/open question**.

## Post-launch learning loop
- **Primary loop:** the F-002-vs-F-001 volume ratio — this *is* the concept validation the idea has
  not yet received (`validation.md`: verdict `GO-UNVALIDATED`, "the build itself is the first
  evidence").
- **Secondary loop:** watch the `idea.md` §9 kill criteria — regulatory (Facebook blocking/banning
  the scraping mechanism) and technical (scraped coverage degrading below a usable threshold without
  compensating manual submissions).
- **Cadence:** not specified in the seed — **[assumption]**: review the A-001 signal at least weekly
  given the build's unvalidated status.

## Roadmap beyond MVP
_(`idea.md` §7 Final product — scaffolding, not commitments; sequencing among these is not specified
in the seed.)_
- **F-101** — Photo-based matching suggestions between lost/found reports.
- **F-102** — Verified rescuer/page badge program (manual vetting).
- **F-103** — In-app messaging between finder/adopter and poster.
- **F-104** — Formal partnership/API feed from established rescue pages (would reduce scrape
  dependency; PRD notes this as a system-design rejected-alternative consideration).

## Open questions
- **`docs/seed/market.md` was never produced.** `release_planning: true` selected this GTM doc
  (`manifest.json` template id `gtm` grounds in the `market` sibling), but the Key's fast path
  skipped phase-3 market research entirely — no `market.md` exists to ground it against. Every
  market-shaped claim above is limited strictly to `idea.md` §5's own `[assumption]`-tagged content:
  the unsourced "tens of thousands" size band, the two reachability channels, and the three named
  alternative categories (Facebook groups/pages, general international lost-and-found tools, word of
  mouth). No new market sizing, no named competitor products, and no channel-performance data has
  been invented to fill this gap.
- **Dedicated market research should run before any real GTM spend or channel commitment.** The idea
  itself carries a `GO-UNVALIDATED` verdict (`validation.md`): none of the five tests (Real, Large,
  Significant, Urgent, Relevant) have cleared beyond `unknown`, and A-001 — whether supply-side users
  will submit directly rather than rely solely on Facebook — is untested. Spending against an
  unsourced size band and unvalidated demand compounds the concept-level risk already flagged; a
  `market.md` pass (sizing, named competitors, channel evidence) should precede paid/earned channel
  commitments, not follow them.
- Rollout mechanism, launch security sign-off, and instrumentation tooling are unspecified in the
  seed (flagged above) and should not be treated as decided.
- The GTM-channel tension — both named channels route through the platform Whiskr is positioned
  against — is unresolved in the seed and needs a product-owner decision before outreach begins.

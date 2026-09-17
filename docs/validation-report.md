# Validation Report — does the design still serve the concept?

> **Purpose:** phase-5.2. Written by the **validator** role (per the resolved binding for this run,
> Claude Sonnet 5 — the same model family/generation as the architect, not the default cross-family
> GPT-5.6 Terra binding, R1) after the architect's docs exist and before any plan is built. This is
> not idea validation (the Key's job, optional) — it asks one narrower question: *given the seed as
> written, do the PRD, system design, and QA plan implement that concept without contradiction, and
> is the riskiest assumption actually exercised by the build?* The validator authored none of the
> docs it judges and never writes a fix.

**Verdict:** `PASS`
**Independence:** `DEGRADED (same family+gen, Claude)` — no `resolve-roster.py` / cross-family
validator was reachable in this environment; this validator is running in a fresh context with only
the phase-5.2 input contract (seed, PRD, system design, data model, QA plan, and the rest of the
generated suite), never saw a draft, and did not author any doc it is judging. Per this kit's own
contingency rule (`adr/ADR-0007` — the same rule `validation.md`'s `concept_validator` pass invoked
for the identical reason), proceeding under `DEGRADED` rather than refusing. Because independence is
degraded, this verdict should be read as closer to a careful self-review than a true adversarial
cross-check — the lead should read §1–§4 in full rather than trusting the verdict line alone.
**Inputs read:** `docs/seed/idea.md`, `docs/seed/context.md`, `docs/seed/validation.md`,
`docs/seed/usability.md` · `docs/prd.md`, `docs/system-design.md`, `docs/data-model.md`,
`docs/qa-test-plan.md`, `docs/security-compliance.md`, `docs/api-spec.md`, `docs/design-system.md`,
`docs/release-gtm.md`, `docs/frd.md`, `docs/technical-design.md`, `docs/ops.md`,
`docs/decision-ledger.md`, `docs/change-record.md` · `python3 box/vault/tools/trace-ids.py docs
--seed docs/seed` output (treated as authoritative per R6, spot-checked and reproduced: `APPROVE`,
5/5 MVP `F-###` covered by QA, 3/3 `INV-###` covered by negative TC, one soft `WARN` —
`API-005` defined but never referenced elsewhere).
**Repair cycle:** 1 of 2

## 1. Concept ↔ design coherence

| Check | Result | Evidence / contradiction (doc A says X; doc B says Y) |
|---|---|---|
| Every `F-###` in idea.md §7 (MVP) has a PRD row and an EARS criterion | yes | `prd.md` Feature list + Acceptance criteria carry F-001..F-005 verbatim from `idea.md` §7, each with an EARS `WHEN/IF...SHALL` line |
| No PRD feature exists that idea.md does not name (no invented `F-`) | yes | `prd.md` reuses F-001..F-005 (MVP) and F-101..F-104 (Final) exactly as numbered in `idea.md` §7; no new `F-` ID appears |
| The core journey `UJ-001` reaches the value proposition in §6 | yes | `prd.md` UJ-001 (Report a cat) is the literal `usability.md` §1–§2 `USABILITY CLEARED` 3-click flow that exercises A-001, the mechanism `idea.md` §6's value prop depends on for independent supply |
| System design components map to PRD features; no component serves nothing | yes | `system-design.md` "Components & responsibilities": Mobile Client, Backend API, Scraper/Ingestion, Manual-submission path, Status/staleness engine, Alerting worker, Data store — each traced to an F-### in its own text; none is decorative |
| Non-functional needs the seed states are designed for; ones it doesn't are `[assumption]`, not invented targets | yes | `system-design.md` "Scaling strategy" states plainly "no performance, availability, or concurrent-user target exists in the seed" rather than inventing one; same discipline in `qa-test-plan.md`'s Test profile and `ops.md`'s Alerts thresholds |
| Every network-exposed surface declares auth/authz | yes, with named gaps | `security-compliance.md` covers all write surfaces (submission, status-transition, location opt-in, push-token registration) as auth-required, and explicitly *flags* (rather than silently omits) two undecided points: the feed-read auth requirement and the scraper→Backend-API service credential (T-009 context) — flagging an open gate is compliant with the architect role's rule, not a violation of it |

## 2. Riskiest assumption coverage (`A-001`)

- **Assumption as stated in the seed:** "Rescuers, finders, and page admins will feed their listings
  into Whiskr (via direct submission) rather than relying solely on posting to Facebook — without
  that, Whiskr has no independent supply and is just a lower-coverage mirror of Facebook." (`idea.md`
  §9)
- **Which `F-###` / `TC-###` exercises it:** **F-002** (manual submission) is the only MVP feature
  that requires a rescuer/finder to act inside Whiskr rather than only on Facebook — F-001/F-003/
  F-004 all operate on listings regardless of where supply originates. `TC-002` (integration) proves
  F-002's mechanics work; it does not and cannot measure A-001 itself (a market-behavior question, not
  a unit-testable one) — the actual A-001 signal is `release-gtm.md`'s instrumentation plan: F-002
  direct-submission volume tracked against F-001 scraped volume over time. This chain (`idea.md`
  §9 → `validation.md` §2 → `prd.md` F-002 → `release-gtm.md` instrumentation → `decision-ledger.md`
  §6 open items) is consistent end to end with no drift.
- **`validation.md` drift check:** matches. `validation.md` §2 names F-002 as the sole A-001-testing
  feature and states the fail condition ("F-002 volume stays near zero while F-001 keeps growing");
  the generated suite preserves this exact framing without alteration.

## 3. Invariant enforcement map (`INV-###`)

| INV | PRD must-never | Security mitigation | Design banned-copy | Negative TC | Verdict |
|---|---|---|---|---|---|
| INV-001 | BR-002 (manual Submit gate), BR-008 (scraped publish gate) | T-001/T-002 (spoofing/tampering); `FacebookAnchor` row committed in the same transaction as `Listing` (`technical-design.md` `gateListingWrite()`) | `design-system.md` copy rule: never imply verification beyond "has a linked Facebook profile" | TC-N01 | enforced |
| INV-002 | BR-004 (only submitter/admin transitions status) | T-003/T-004 (tampering/repudiation); `transitionStatus()` row-lock + `isVisibleAsActive()` re-derived on every read, never cached (`technical-design.md`) | `design-system.md` principle 3 + resolved-state rendering rule | TC-N02 | enforced |
| INV-003 | BR-007 (retain link to original post) | T-007 (tampering/info disclosure); `ScrapedPost.original_post_url` NOT NULL at schema level (`data-model.md`) | `design-system.md` principle 1 (source always shown) | TC-N03 | enforced, with one named residual gap: T-007 itself notes no automated check enforces the *UI* half (that a rendering path never drops the link) — flagged, not silently assumed covered |

All three invariants are enforced by a concrete mechanism (transaction gate, row lock, NOT NULL
constraint) plus an audited negative test, not merely asserted in prose.

## 4. Test fence quality (can these tests fence a cheap executor?)

- Every MVP `F-###` has a TC whose command is exact and whose expected result is concrete enough to
  be red now and green later: **yes** — each `TC-###`/`TC-N##` in `qa-test-plan.md` names an exact
  file path, an exact `npm --prefix ... test -- ...` command, and an EARS-style expected result (no
  adjective-only expectations found).
- Every TC is at the cheapest proving level that is honest: **yes** — dedup/staleness/status-guard/
  identity-anchor logic is unit-level; submission round-trip and geo-fanout are integration-level;
  only UJ-001 is e2e, and it is named as "the smallest slice that would break the demo," not
  over-specified.
- **Minor gap, not blocking:** `API-005` (`PUT /users/me/location`) has no `TC-###` that exercises the
  endpoint contract directly — `TC-004` tests the alert-fanout logic via fixture `UserLocation` rows,
  not via the API-005 request/response shape itself. Matches the tool's own soft `WARN`. Worth a
  contract-level TC at scaffold; does not block phase-5.3.

## 5. Human-verified remainder

- **Missing `docs/seed/brand.md` and `docs/seed/market.md`.** Independently flagged by
  `design-system.md`, `release-gtm.md`, and `api-spec.md`'s reasoning (base-URL/branding has nothing
  to ground against). Ruling: **accepted, logged open question — not a REVISE.** Every doc that
  needed these siblings degrades honestly to neutral `[assumption]` placeholders (design-system's
  tokens) or explicitly bounded claims (GTM's market content, limited strictly to `idea.md` §5's own
  `[assumption]`-tagged figures) rather than inventing brand facts or market sizing. This is a human
  decision (commission a brand/market pass before visual-design lock-in and before GTM spend), not a
  documents-only contradiction the Vault can resolve by regenerating a cluster.
- **`UserLocation.radius_km` vs. `Alert.radius_km` (flagged in `technical-design.md`).** Ruling:
  **accepted, logged open question — not a REVISE, not an ESCALATE.** `data-model.md` defines both
  fields without reconciling which one the alert-fanout query actually uses; `technical-design.md`
  proposes a specific resolution (use the recipient's own `radius_km`; treat `Alert.radius_km` as an
  audit snapshot of the default policy) and flags it as unconfirmed by the data-model owner rather
  than silently deciding it. Critically, `prd.md` BR-006 states "radius customization UI is out of
  scope for MVP," which means every `UserLocation.radius_km` value is the same default (5 km) as
  `Alert.radius_km` for the entire MVP lifetime — the two fields are currently behaviorally
  identical, so the ambiguity has no observable runtime effect today. It should be reconciled in
  `data-model.md` before F-004's radius ever becomes user-configurable (post-MVP), but it does not
  block phase-5.3.
- **BR-002 "profile" vs. BR-007/BR-008/INV-001 "profile or page" wording (flagged in `frd.md`).**
  Ruling: **accepted, logged open question — not a REVISE.** This is a wording inconsistency inside
  `prd.md` itself (BR-002 says "a Facebook profile"; INV-001 says "a real Facebook profile **or
  page**"), and `frd.md` already resolves the ambiguity in the safe, consistent direction — treating
  "profile or page" as interchangeable for both the manual and scraped paths, matching INV-001's own
  wording — rather than silently picking the narrower reading. No downstream doc (data-model,
  technical-design, design-system, security-compliance) actually restricts submissions to
  personal-profile-only; the restrictive reading only ever appears in BR-002's prose. Net effect: a
  one-line textual fix to `prd.md` BR-002 (align it to "profile or page") is worth doing at the next
  PRD touch, but nothing in the built system currently depends on the narrower wording, so this does
  not warrant regenerating the `prd` cluster on its own.
- **Facebook ToS/scraping exposure and RA 10173 (Philippine Data Privacy Act) applicability**
  (`security-compliance.md`) — both explicitly deferred to legal counsel/product owner in the docs
  themselves; no document alone can settle either, and this report does not attempt to.
- **Whether `[assumption]` values (staleness window = 30 days, alert radius = 5 km fixed, push
  vendor, auth mechanism, admin provisioning, deployment topology) match the product owner's actual
  intent** — every one is already marked `[assumption]` at its source and carried consistently
  through every doc that touches it; confirming or correcting these is a product-owner call the
  documents cannot make for themselves.

## 6. Decision

- **PASS** — proceed to phase-5.3. The doc suite implements `idea.md`'s concept without unresolved
  contradiction: every MVP `F-###` traces to a PRD row, an EARS criterion, and a passing-when-built
  `TC-###`; every `INV-###` has a concrete enforcement mechanism plus a negative test, not just an
  assertion; A-001 is correctly and consistently wired to F-002 across the seed, the PRD, and the
  GTM instrumentation plan with no drift from `validation.md`. The three inconsistencies named in
  this run's dispatch (alert-radius field duplication, BR-002 wording, missing brand/market seed
  siblings) are each logged above as an accepted open question with a named owner and a stated
  reason none rises to blocking — the lead may still choose to require any of them fixed before
  build approval, but none is a contradiction this validator found between what is being built and
  what `idea.md` asked for.

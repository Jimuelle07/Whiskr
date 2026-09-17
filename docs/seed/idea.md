---
status: unvalidated-by-choice
schema_version: 2.1.0
validation: GO-UNVALIDATED
riskiest_assumption: A-001
---

# Idea: Whiskr — NCR & Greater Manila Area cat adoption + missing-cat app

## 1. Problem statement

People across Metro Manila and the adjacent provinces (Bulacan, Cavite, Laguna, Rizal) who want to
adopt a cat, or who are trying to find a cat that has gone missing (or report one they have found),
depend entirely on Facebook to do it. The cats they are looking for are scattered across dozens of
independent, unaffiliated Facebook pages and groups, each with its own posting habits and no shared
index between them. People rely on Facebook's own feed algorithm to surface reposts and shares from
the groups they already follow, which means a specific adoption listing — or a time-critical
missing-cat report — is easily buried within hours and never seen by someone who could have helped.
Listings for cats that have already been adopted or already been found often stay visible long after
they stop being relevant, so people waste time chasing outcomes that are no longer available.

## 2. Target segment

A person (not a rescue-page operator) in NCR or the adjacent Greater Manila Area provinces (Bulacan,
Cavite, Laguna, Rizal) who is either actively trying to adopt a cat, or trying to locate a missing
cat / report a found one. Already follows or has joined some of the many independent Facebook cat
pages/groups relevant to that outcome, but not all of them, and has no way to see them as one set.
Recurs often across the segment as a whole: adoption-seekers check continuously; any one missing-cat
event is rare per person but urgent when it happens. **Excludes:** rescue-page admins/shelter
operators (the supply side, a different job to be done), people outside NCR/the Greater Manila Area,
and people searching for non-cat pets.

## 3. Evidence

N/A — because the fast path was chosen to shape the idea before running real interviews (no
`INT-###` / `EV-###` exist yet). The build itself will be the first evidence, specifically for A-001
(see §9).

The four tests (optional — a Key aid, never a Vault gate):

| Test | Pass / Fail | Why |
|---|---|---|
| Real | unknown | No evidence gathered yet; pain described by the product owner, not yet independently observed |
| Large | unknown | Segment plausible but uncounted; see §5 size band |
| Significant | unknown | Cost described as time + missed-outcome risk, not yet measured |
| Urgent | unknown | Missing-cat case argued as time-critical by design, not yet observed in the field |

## 4. Root cause (the WHY)

Facebook is a general-purpose social network optimized for its own engagement feed, not for search,
status-based filtering, or urgent broadcast. It has no shared index across independent pages/groups,
no reliable way to mark a listing "resolved" so it stops surfacing, and no location-based alerting.
Each NCR-area cat page/group is run independently with no incentive or mechanism to coordinate with
others — the fragmentation is structural to how Facebook groups work, not a fixable habit of any one
page admin.

## 5. Market & alternatives
- **Size band:** tens of thousands — _[assumption]_; NCR + the Greater Manila Area has a large
  population of pet-adoption/animal-welfare-interested Facebook users spread across many
  independently-run cat pages and groups, but no sourced count exists yet.
- **Reachability:** _[assumption]_ (1) direct outreach to known NCR-area cat rescue Facebook pages
  to request cross-posting/partnership, (2) posting in the largest existing NCR cat
  adoption/missing-cat Facebook groups to recruit early users.
- **Top 3 alternatives + their key failure:**
  1. Facebook groups/pages (status quo) — fails at cross-page search, staleness, and urgent alerting (§1)
  2. General international lost-and-found pet tools — fail at NCR/Philippines-specific reach and the adoption-listing use case
  3. Word of mouth / asking around locally — fails at speed and reach beyond one's own network

## 6. Value proposition

For **people in NCR and the Greater Manila Area looking to adopt a cat, or trying to find/report a
missing cat**, who **currently must rely on Facebook's feed algorithm to catch relevant posts across
dozens of unconnected pages and groups, and frequently miss time-critical or already-resolved
listings**, **Whiskr** is a **mobile app centralizing NCR-area cat adoption and missing-cat
listings with location-based alerts**, that **surfaces a de-duplicated, status-aware view of every
relevant listing near you and pushes an alert the moment a missing/found cat is reported nearby**,
unlike **Facebook groups**, because **it is purpose-built to search, filter by status/location, and
alert urgently instead of relying on a general engagement feed**.

## 7. Feature set

### MVP — smallest path to core value + a learning signal
- **F-001** — Aggregated listings feed scraped from NCR + Greater Manila Area cat adoption/missing-cat Facebook pages and groups → solves the fragmentation pain in §1
- **F-002** — Manual submission form so rescuers/finders/adopters can post directly into Whiskr → solves the supply gap scraping alone cannot fill; tests A-001 (§9)
- **F-003** — Status tagging (available / on hold / adopted / found) with stale-listing suppression → solves the staleness pain in §1
- **F-004** — Location-based missing-cat alert pinned to an area, pushed to nearby users → solves the "buried within hours" urgency pain in §1
- **F-005** — Every post requires a linked, visible Facebook profile as an identity anchor → solves the scam/trust risk raised during capture

### Final product — full vision
- **F-101** — Photo-based matching suggestions between "lost" and "found" reports
- **F-102** — Verified rescuer/page badge program (manual vetting)
- **F-103** — In-app messaging between finder/adopter and poster
- **F-104** — Formal partnership/API feed from established rescue pages (reduces scrape dependency)

## 8. Success metrics
- **Activation:** _[assumption]_ a new user completes at least one search/filter, or submits/claims
  one listing, within their first session.
- **Retention:** _[assumption]_ a user reopens the app within 7 days of a missing-cat alert in their
  area, or checks adoption listings weekly.
- **Revenue / value:** N/A — because no monetization hypothesis has been validated yet; MVP ships
  unmonetized.

## 9. Constraints, risks & kill criteria
**Riskiest assumption — A-001:** Rescuers, finders, and page admins will feed their listings into
Whiskr (via direct submission) rather than relying solely on posting to Facebook — without that,
Whiskr has no independent supply and is just a lower-coverage mirror of Facebook.

**A-002:** People experiencing the pain will actually open and check a new app instead of staying
inside Facebook, where they already spend their time.

**Kill criteria (explicit fail-states):**
- Regulatory: Facebook blocks/bans the scraping mechanism entirely (or issues a takedown demand) and
  no meaningful manual-submission volume exists to replace it
- Unit economics: N/A — no monetization exists at MVP; revisit once release planning begins
- Technical: scraped coverage degrades below a usable threshold with no compensating manual
  submissions (F-002)

**Invariants (`INV-###`) — hard rules that must hold across every pivot:**
- **INV-001** — the system must never publish a post that does not carry a visible link back to a
  real Facebook profile or page (identity anchor, per §7 F-005)
- **INV-002** — the system must never keep a listing shown as "available" or "missing" past the point
  its submitter or an admin has marked it resolved (directly enforces the §1 staleness fix)
- **INV-003** — the system must never scrape or republish content in a way that strips the original
  poster's attribution/link back to their post

## 10. Out of scope (for now)
- Regions outside NCR + Bulacan/Cavite/Laguna/Rizal
- Non-cat pets (dogs, other animals)
- Payments, donations, or any monetization flow
- Photo-based automatic lost/found matching (parked as F-101)
- Verified-partner API integrations with rescue pages (parked as F-104)
- In-app messaging (parked as F-103)

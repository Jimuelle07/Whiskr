# Problem capture — Whiskr

**Path:** fast · **Date:** 2026-09-18 · **Captured with:** product owner
<!-- phase-1 working file. Never emitted. No solution language above the parking lot. -->

## 1. The pain (one paragraph, no product words)

People across Metro Manila and the adjacent provinces (Bulacan, Cavite, Laguna, Rizal) who want to
adopt a cat, or who are trying to find a cat that has gone missing (or report one they have found),
depend entirely on Facebook to do it. The cats they are looking for are scattered across dozens of
independent, unaffiliated Facebook pages and groups, each with its own posting habits and no shared
index between them. People rely on Facebook's own feed algorithm to surface reposts and shares from
the groups they already follow, which means a specific adoption listing — or a time-critical
missing-cat report — is easily buried within hours and never seen by someone who could have helped.
Listings for cats that have already been adopted or already been found often stay visible long after
they stop being relevant, so people waste time chasing outcomes that are no longer available. The
people affected feel like they are missing things that mattered — a cat that needed a home, or an
owner's only real window to recover a lost pet — for no reason other than the post living on the
wrong page at the wrong moment.

## 2. Who has it (the segment)
- **Role:** A person (not a rescue-page operator) who is either actively trying to adopt a cat, or
  trying to locate a missing cat / report a found one.
- **Context (company size / situation / tools they already use):** Lives in, or is searching within,
  Metro Manila (NCR) or the adjacent Greater Manila Area provinces (Bulacan, Cavite, Laguna, Rizal);
  already follows or has joined some number of the many independent, unaffiliated Facebook cat
  pages/groups relevant to that outcome, but not all of them, and has no way to see them as one set.
- **Frequency (how often the pain recurs):** Recurs often across the segment as a whole — someone
  with adoption intent checks pages on an ongoing basis until they succeed; any individual missing-cat
  event is rare per person, but demands an urgent, time-critical response when it happens.
- **Who is NOT in this segment (and why):** Rescue-page admins and shelter operators themselves —
  they are the ones producing the listings, a supply-side actor with a different job to be done
  (getting the word out, not searching for it). People outside NCR/the adjacent provinces — a
  different, unaddressed segment for this MVP. People searching for other lost pets (dogs, etc.) —
  this is cat-specific.

## 3. What they do today (workaround) and what it costs
- **Workaround:** `[said]` Join as many relevant NCR-area cat adoption/missing-cat Facebook pages and
  groups as they can find, then rely on Facebook's own feed algorithm to surface reposts and shares
  from those groups rather than checking each one directly.
- **Cost — time:** `[said]` Described as an ongoing, habitual burden (continuous checking) rather
  than a one-off cost; not separately quantified yet.
  **money:** `[said]` Not raised as a direct cost by the product owner; distinct from the scam/trust
  risk that shaped the verification decision in idea.md §7 (F-005).
  **risk:** `[said]` The dominant cost: relevant posts — especially urgent missing-cat alerts — get
  buried by Facebook's feed algorithm within hours and are missed entirely, with no way to know what
  was missed; time is also lost to stale listings for cats already adopted or already found.

## 4. First guess at the riskiest assumption
- **A-001:** Rescuers, finders, and page admins will actually feed their listings into a central
  destination outside Facebook — by direct submission, or by allowing their existing posts to be
  aggregated — rather than treating Facebook as the only place they need to post. If false, the
  product has no independent supply and is just a lower-coverage mirror of Facebook.
- **A-002 (optional):** People experiencing the pain (adopters, finders) will actually open and check
  a new app instead of staying inside Facebook, where they already spend their time.

## 5. Competition fit (only if judged)
- N/A — not a judged/competition build.

## Parking lot (solution words that came up — park them here, do not let them into §1)
- Mobile app, native app, push notifications
- Location-based / geo-tagged missing-cat alerts, barangay-level pinning
- Scraping public Facebook pages/groups (+ ToS/legal risk)
- Manual crowd-submission form
- Photo-matching between lost and found reports
- Verified rescuer/page badge
- Require a linked Facebook profile as an identity anchor (trust signal)
- Greater Manila Area geographic scope (NCR + Bulacan, Cavite, Laguna, Rizal)

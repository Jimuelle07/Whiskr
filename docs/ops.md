# OPS — Operations & Observability Runbook

> **Purpose:** how to run, monitor, and recover the system in production. Conditional — the intake
> only selects this when `outlives_demo: true`. A hackathon demo does not need it.
> Traces back to: `system-design`, `security-compliance`.

## Deploy

- **Topology** (from `system-design.md` "Deployment topology," itself `[assumption]` — no
  infrastructure/cloud provider is named in the seed): a single managed backend deployment (Backend
  API + workers) in one region serving NCR/Greater Manila Area, one managed relational database
  (PostgreSQL + PostGIS per system-design's data-store choice), one object store for listing photos,
  and platform push services (APNs/FCM). No specific cloud vendor or PaaS is named upstream —
  **[assumption]**, to confirm at scaffold.
- **Components deployed independently** (per system-design's component list), each with its own
  release artifact so a scraper fix never requires a mobile-client or Backend-API redeploy:
  - **Mobile Client (iOS + Android)** — shipped through the platform app stores (App Store / Google
    Play). Release cadence and staged-rollout percentage are **[assumption]** — not specified in any
    upstream doc.
  - **Backend API** — the single contract boundary; deployed as one service. Rollback = redeploy the
    prior image/build; system-design does not specify a blue/green or canary strategy, so this is
    **[assumption]** — a straightforward redeploy-on-failure is proposed until traffic volume
    justifies more.
  - **Scraper / Ingestion worker** and **Status/staleness engine** and **Location-alerting worker** —
    three scheduled/triggered workers per system-design's component split; deployed as separate
    processes from the Backend API so a scraper outage cannot take down listing reads/writes (the
    same isolation system-design already argues for when it says the scraper "never talks to
    Facebook directly" through the API's write path, not the other way around).
- **Environments.** `system-design.md` states only a single production environment is implied by
  `team_size: 1` / `time_budget: 2w`, and explicitly leaves a staging/dev split as an **open
  question** — not asserted here as decided. Until resolved, every deploy is a direct-to-production
  change; this is a named gap, not a silent one.
- **Pinned versions.** No language/runtime/framework version is named in any upstream doc (system-
  design flags the mobile stack itself as `[assumption]`, React Native, with no version pinned).
  **Open question**, to confirm at scaffold per the architect role's stack-currency rule — this doc
  does not invent a version number.
- **Rollback.** `security-compliance.md`'s own incident-response section already states: "deployment
  topology itself is `[assumption]` in system-design... no rollback procedure can be specified until
  that is decided — flagged, not fabricated." This doc inherits that gap rather than resolving it
  with an invented procedure.

## Configuration & secrets

Reused from `security-compliance.md`'s "Secrets handling" section — not re-derived here, referenced
by name only:

| Secret | Used by | Notes |
|---|---|---|
| Database credentials | Backend API, all three workers (shared relational store per system-design) | Referenced via secret store; never inlined in a doc or the repo |
| Push provider credentials (APNs/FCM) | Location-alerting worker | Vendor itself is `[assumption]` — unconfirmed upstream |
| Session/auth-signing secret | Backend API | Depends on the still-undecided auth mechanism (`[assumption]`, security-compliance) — key rotation required once chosen |
| Service credential, scraper → Backend API | Scraper/Ingestion worker | Currently **undesigned** per security-compliance (T-009 context) — the scraper must not write through a bypass or a User session; this credential does not exist yet and is flagged, not invented here |

- No secret value is ever written into this document, matching `security-compliance.md`'s own rule.
- Rotation cadence for any of the above is **[assumption]** — no policy is stated upstream; propose
  rotating the scraper service credential and the auth-signing secret on any suspected compromise at
  minimum, pending a real policy.

## Observability

Signals are scoped to what the four components actually do per `system-design.md`'s component list
and data flow — not invented metrics:

- **Backend API** — request success/error rate on the submission, status-transition, and feed-read
  endpoints (these are the enforcement points for INV-001/INV-002/INV-003 per security-compliance);
  latency on the feed-read path, since F-002's acceptance criterion ("visible in the feed within the
  same session") depends on it staying fast.
- **Scraper / Ingestion worker** — `ScrapeSource.status` (active/degraded/blocked) is the one signal
  already designed into the data model for this component (system-design, security-compliance T-009)
  — see the dedicated risk treatment below; poll success/failure count per configured
  page/group; count of posts rejected at the BR-008/INV-001 profile-link gate (a rising rejection
  rate is itself a leading indicator of a Facebook-side structure change, even before `ScrapeSource.
  status` flips to `degraded`).
- **Status/staleness engine** — sweep run success/failure and count of listings flagged stale per
  run (a sudden spike or drop signals a scheduling or query bug, not a real change in cat-adoption
  volume).
- **Location-alerting worker** — fanout size per triggered alert (radius query result count) and
  `AlertDelivery.delivered_at` success rate against the push provider — this is the same audit trail
  security-compliance already names for F-004's "alert was sent" acceptance criterion, reused here
  as a monitoring signal rather than only an audit one.
- **Mobile Client** — crash rate and push-permission/location-permission denial rate, since
  system-design notes the client must degrade gracefully (no crash) on location denial.
- Logging must exclude `PushToken.token`, `User.auth_identifier`, and precise `UserLocation.lat/lng`
  from general application logs, per `security-compliance.md`'s "Audit & logging" section — reused
  verbatim as a logging constraint, not re-derived.
- **Monitoring/alerting tooling itself (vendor, dashboard product) is not named in any upstream doc
  — [assumption].** No specific APM or log-aggregation vendor is chosen here; this is an operational
  tooling decision, not a product requirement, and is left open pending team preference/budget at
  scaffold.

## Alerts & thresholds

| Signal | Threshold (forward-looking, not evidence-tagged) | Pages | Why |
|---|---|---|---|
| `ScrapeSource.status` → `degraded` | Any transition | On-call / product owner | Early warning before full block; see scraper risk treatment below |
| `ScrapeSource.status` → `blocked` | Any transition | On-call / product owner, immediately | This is the named kill-criterion trigger (`idea.md` §9, security-compliance's Facebook ToS subsection) — treated as a live decision-path trigger, not routine ops noise |
| Backend API error rate on submission/status-transition endpoints | Sustained error rate over a to-be-set baseline — **[assumption]**, no baseline exists yet (no traffic history) | On-call | These endpoints are the INV-001/INV-002 enforcement points |
| Push delivery failure rate (`AlertDelivery`) | Sustained failure over a to-be-set baseline — **[assumption]** | On-call, non-urgent | Per system-design, delivery failure must never block listing creation — this is a degraded-experience alert, not an outage alert |
| Status/staleness sweep — missed run | Sweep does not complete within its scheduled window | On-call | Silent failure here quietly breaks BR-005 feed-visibility exclusion |

- **Escalation path.** Reused from `security-compliance.md`'s "Incident response basics": `team_size:
  1` means there is no on-call chain to design; the product owner is the de facto single point of
  escalation until the team grows. **[assumption]**, not sourced from any doc — stated there as the
  only coherent default, inherited here rather than re-derived.
- Specific paging/alerting tool (e.g., a pager-duty-style vendor vs. a plain notification channel) is
  **[assumption]** — not named upstream.

## Runbook — common incidents

### Scraper operational risk (dedicated treatment — the single most fragile component)

This gets its own explicit treatment because `security-compliance.md` already names Facebook ToS
exposure as the system's single most consequential, hardest-to-reverse risk (T-009; the dedicated
"Facebook ToS / scraping" subsection), and because system-design's own scaling-strategy section
calls the scraper's poll tolerance "a viability [issue]... not a scale question." That
threat/mitigation analysis is reused here verbatim and not redone; this section covers what it
means operationally when it happens.

**What "the scrape gets blocked" looks like operationally, end to end:**

1. **Detection (monitoring signal).** `ScrapeSource.status` is the designed signal (security-
   compliance: "the primary designed detection signal"). Operationally this shows up as one or more
   of: a sustained poll-failure count for a given source, a sudden drop in scraped-listing volume
   with no corresponding drop in Facebook page activity, or a spike in BR-008 profile-link-gate
   rejections (suggesting Facebook has changed page/post markup rather than fully blocking access).
   `ScrapeSource.status` transitioning `active → degraded` is the early-warning state; `degraded →
   blocked` is the hard-stop state.
2. **Immediate operational response, in order:**
   - Confirm the transition is real (not a transient network blip) — re-check on the next scheduled
     poll cycle before paging, to avoid false-positive escalation on a single failed run.
   - On confirmed `degraded`: page the on-call/product owner per the Alerts table above; do **not**
     yet trigger the kill-criterion decision path — this is the buffer state where investigation
     (rate-limit backoff, IP rotation, or a markup-change fix) happens.
   - On confirmed `blocked`: page immediately and treat it as the live trigger for the kill-criterion
     decision path in `idea.md` §9 — this is security-compliance's own recommendation, reused here
     as the operational action, not re-derived. The decision (accept degraded coverage, pause the
     source, or invoke the broader kill-criterion review) belongs to the product owner per the
     escalation path above; this runbook does not pre-decide it.
3. **Fallback activation.** F-002 (manual submission) is system-design's designed fallback ("the
   A-001 fallback") and is already live in production at all times, not something switched on after
   a block — it converges on the same `Listing` write path, so no separate deploy or config change is
   needed to "activate" it. The operational action on a scraper block is therefore communication, not
   infrastructure: surface degraded scraper coverage to users (e.g., in-app messaging encouraging
   manual submission for the affected source/area) so manual-submission volume has a chance to
   replace the lost scrape volume before the kill criterion ("no meaningful manual-submission volume
   exists to replace it," per `idea.md` §9) is actually met. The specific in-app messaging mechanism
   is **[assumption]** — not designed in any upstream doc.
4. **What this runbook explicitly does not cover:** whether to pursue a legal opinion on the
   underlying ToS exposure, or whether to negotiate/pursue the F-104 partnership path. Both are named
   as open items for the product owner in `security-compliance.md` and are not operational decisions.

### Other incidents

| Symptom | Diagnosis | Fix |
|---|---|---|
| Feed reads slow or timing out | Check Backend API latency signal above; likely DB contention on the shared relational store (system-design: single store serves both API and workers) | Investigate slow queries first; do not scale by splitting the store without re-litigating system-design's single-store trade-off (candidate ADR) |
| Push notifications not arriving | Check `AlertDelivery` delivery-failure rate; confirm push-provider credential validity | Per system-design, delivery failure must never block listing creation — this is a degraded-experience fix, not an outage fix; retry/backoff against the provider |
| Status/staleness sweep silently stops flagging stale listings | Missed-run alert above; check scheduler health | Re-run the sweep manually once the scheduler is fixed; BR-005 exclusion is visibility-only, so no data is lost during the gap |
| Mobile client crashes spike after a release | Crash-rate signal above | Staged-rollout halt / rollback to prior build via the app store — exact staged-rollout mechanism is **[assumption]**, not specified upstream |

## Backup & recovery

- **What's backed up.** The shared relational store (listings, users, `StatusHistory`,
  `ScrapeSource` bookkeeping, `AlertDelivery` records per data-model.md) and the object store holding
  listing photos. Backup frequency, retention window, and restore-testing cadence are **[assumption]**
  — no backup policy is stated in any upstream doc.
- **`StatusHistory` and `AlertDelivery` are append-only audit trails** (security-compliance's "Audit &
  logging" section, reused here) — their loss would break the audit guarantees T-004 (repudiation
  mitigation) and F-004's "alert was sent" acceptance criterion depend on, so any restore procedure
  must preserve them intact rather than truncating to a convenient snapshot point.
- **Restore procedure.** Not yet designed or tested — flagged as an open item consistent with
  `security-compliance.md`'s own statement that "no rollback procedure can be specified until
  [deployment topology] is decided." This doc does not fabricate a procedure ahead of that decision.
- **Scraper state (`ScrapeSource.status`, last-poll bookkeeping) recovery.** On restore from backup,
  `ScrapeSource.status` must be re-verified against live Facebook access before resuming polling
  (i.e., do not trust a pre-incident `active` status after a restore) — a backup-specific
  consequence of the scraper fragility named above, not covered by a generic restore step.

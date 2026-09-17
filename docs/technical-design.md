# Technical Design Document (LLD)

> **Purpose:** the HOW, detail. Per-component design engineers implement from.
> Traces back to: system design.
> Scope of this pass: the three highest-risk pieces flagged for phase-5.1 — (1) listing
> dedup/staleness-suppression (F-001/F-003, INV-002), (2) location-based alert fanout (F-004),
> (3) the FB-profile-link validation gate (F-005, INV-001). All components/entities below are
> exactly the ones named in `system-design.md` and `data-model.md` — none invented here.

## Module breakdown

- **Backend API — Listing Write Service** (`system-design.md`: "both pipelines converge on the
  same `Listing` write path in the Backend API"). Owns `gateListingWrite()` (Algorithm 3),
  `transitionStatus()` (Algorithm 1b), and the default feed query. Single collaborator for both
  the manual-submission endpoint and the scraper worker — no separate validation code per path.
- **Ingestion pipeline (worker)** — owns `computeDedupHash()` / `findDedupMatch()` (Algorithm 1a);
  calls into the Listing Write Service for every post, never writes `Listing`/`FacebookAnchor`
  directly (per system-design: "both pipelines converge on the same Listing write path").
- **Status / staleness engine (worker, scheduled)** — owns `sweepStaleListings()` (Algorithm 1c).
  Never mutates `status`; only `is_stale` (system-design's explicit non-conflation rule).
- **Location-based alerting (worker)** — owns `createAlertAndFanout()` and `FanoutJob()`
  (Algorithm 2), triggered off the Listing Write Service's status-transition/creation event.
- **Data store** — `Listing`, `FacebookAnchor`, `ScrapedPost`, `StatusHistory`, `Alert`,
  `AlertDelivery`, `UserLocation`, `PushToken` (all from `data-model.md`; no new entities).

## Class / function-level design

```
# Listing Write Service (Backend API)
validateFacebookAnchor(candidateUrl: str) -> AnchorValidationResult
gateListingWrite(draft: ListingDraft, actor: Optional[User]) -> Listing        # Algorithm 3
transitionStatus(listing_id: uuid, new_status: Status, actor: User) -> Listing # Algorithm 1b
isVisibleAsActive(listing: Listing) -> bool                                   # INV-002 predicate

# Ingestion pipeline (worker)
computeDedupHash(normalized_text: str, kind: Kind, geo_bucket: str) -> str
findDedupMatch(dedup_hash: str, kind: Kind, geo_bucket: str) -> Optional[Listing]
ingestScrapedPost(post: RawPost) -> None                                      # Algorithm 1a

# Status / staleness engine (worker)
sweepStaleListings(now: timestamp, staleness_window: duration) -> int         # Algorithm 1c

# Location-based alerting (worker)
createAlertAndFanout(listing: Listing) -> Optional[Alert]                     # Algorithm 2
findOptedInUsersWithinRadius(lat, lng, radius_km) -> Iterator[User]
fanoutJob(alert_id: uuid) -> None
```

## Algorithms

### 1a. Ingest-time dedup match (F-001)

```
ingestScrapedPost(post):
    normalized = normalize(post.raw_content)                # trim/casefold/strip boilerplate
    geo_bucket = round_to_grid(post.location_lat, post.location_lng)  # coarse bucket, cheap match
    dedup_hash = computeDedupHash(normalized, post.kind, geo_bucket) # e.g. sha256(normalized|kind|geo_bucket)

    original = findDedupMatch(dedup_hash, post.kind, geo_bucket)
    # exact-hash match only, deliberately not fuzzy-text similarity — simplest honest form;
    # a false negative just yields two visible listings, cheaper than fuzzy-match false positives
    # silently collapsing two distinct real cats.

    if original is not None and original.duplicate_of is None:
        # collapse into the existing canonical listing
        dup = gateListingWrite(ListingDraft(post, duplicate_of=original.id), actor=None)
        ScrapedPost.insert(scrape_source_id=post.source_id, listing_id=dup.id,
                            original_post_url=post.url, dedup_hash=dedup_hash)  # INV-003: URL kept
        return  # no alert fanout for a duplicate row — see Algorithm 2 guard

    # no match: this becomes the canonical listing
    canonical = gateListingWrite(ListingDraft(post, duplicate_of=None), actor=None)  # Algorithm 3
    ScrapedPost.insert(scrape_source_id=post.source_id, listing_id=canonical.id,
                        original_post_url=post.url, dedup_hash=dedup_hash)
```
Complexity: O(1) hash lookup per post (indexed on `ScrapedPost.dedup_hash` per `data-model.md`
constraints); geo-bucketing keeps the match query narrow without a full-table scan.

### 1b. Status transition + INV-002 enforcement

```
TERMINAL_STATUSES = {adopted, resolved}          # never "available" again, by definition
ACTIVE_STATUSES    = {available, on_hold, missing, found}

transitionStatus(listing_id, new_status, actor):
    begin transaction
        listing = SELECT Listing WHERE id = listing_id FOR UPDATE   # row lock: no concurrent
                                                                      # transition races INV-002
        if listing.status in TERMINAL_STATUSES and not actor.is_admin:
            abort transaction; raise Conflict("listing already resolved")  # BR-004 gate
        old_status = listing.status
        listing.status = new_status
        if new_status in TERMINAL_STATUSES:
            listing.resolved_at = now(); listing.resolved_by = actor.id
        UPDATE listing
        INSERT StatusHistory(listing_id, old_status, new_status,
                              changed_by=actor.id, changed_at=now())   # append-only audit trail
    commit transaction

    if old_status != 'missing' and new_status == 'missing':
        createAlertAndFanout(listing)   # Algorithm 2, enqueued async — never blocks this call
    return listing

isVisibleAsActive(listing):
    # re-derived at every read, never trusted from a cache — this is what makes INV-002 hold
    # universally, not just in the default feed filter.
    return (listing.duplicate_of is None
            and listing.status in ACTIVE_STATUSES
            and not listing.is_stale)
```
The status flip and its audit row commit in one transaction under a row lock, so no reader can
observe a listing whose `status` says `adopted` while a stale "available" response is still being
served from an in-flight read — `isVisibleAsActive()` is recomputed, not cached, on every read
path (feed query and detail view alike), which is what enforces INV-002 outside the default feed
filter too (a direct-link detail view on a resolved listing must never render "available" framing).
Complexity: O(1) transition; O(log n) `StatusHistory` insert (indexed).

### 1c. Staleness sweep (F-003, does not touch `status`)

```
sweepStaleListings(now, staleness_window):
    UPDATE Listing
    SET is_stale = true, updated_at = now()
    WHERE status IN ACTIVE_STATUSES
      AND is_stale = false
      AND duplicate_of IS NULL
      AND updated_at < now() - staleness_window
    RETURNING count(*)
```
`staleness_window` (BR-005) has no numeric value in `data-model.md`/`system-design.md` —
**[assumption/open question]**, to confirm at scaffold. Batched `UPDATE`, no per-row branching;
runs on the existing `Listing(status, is_stale, kind)` composite index. Deliberately never writes
`status` or `StatusHistory` — keeps (a) explicit resolution and (b) time-based suppression
non-conflated, per `system-design.md`.

### 2. Location-based alert fanout (F-004)

```
createAlertAndFanout(listing):
    if listing.kind != 'lost' or listing.status != 'missing':
        return None
    if listing.duplicate_of is not None:
        return None   # never fan out for a collapsed duplicate — only the canonical listing
    alert = Alert.insert(listing_id=listing.id, radius_km=DEFAULT_RADIUS_KM, triggered_at=now())
    enqueue(fanoutJob, alert.id)   # async, decoupled from the write path (system-design: fire-
                                    # and-forget relative to F-002's same-session visibility)
    return alert

fanoutJob(alert_id):
    alert   = SELECT Alert WHERE id = alert_id
    listing = SELECT Listing WHERE id = alert.listing_id
    for user in findOptedInUsersWithinRadius(listing.location_lat, listing.location_lng):
        # geo query: UserLocation WHERE location_opt_in = true
        #   AND ST_DWithin(point(lat,lng), point(listing.location_lat,listing.location_lng),
        #                   user.radius_km)          -- see note below on radius source
        # paginated in batches (cursor over UserLocation) to bound memory in dense areas
        if AlertDelivery.exists(alert_id=alert.id, user_id=user.id):
            continue   # unique(alert_id, user_id) per data-model.md — idempotent on job retry
        delivery = AlertDelivery.insert(alert_id=alert.id, user_id=user.id,
                                         delivered_at=null, opened_at=null)
        for token in PushToken.where(user_id=user.id):
            try:
                pushProvider.send(token, payload_for(listing))
                delivery.delivered_at = now(); delivery.save()
                break   # one successful delivery per user is enough
            except PushProviderError as e:
                log(e)   # never fails the job; retried/logged, per system-design
```
**Open question (radius source):** `UserLocation.radius_km` (per-user opt-in preference) and
`Alert.radius_km` (BR-006 default snapshot) are two distinct fields in `data-model.md`; this
design uses the user's own `radius_km` as the match distance and treats `Alert.radius_km` as an
audit snapshot of the default policy in effect at trigger time, not as the query parameter. This
reconciliation is not explicit in `data-model.md` — flagged for the data-model owner to confirm,
not asserted as fact.

Complexity: O(k) candidates within the geo index (`Listing`/`UserLocation` geo index per
`data-model.md`) × O(tokens per user); batched to avoid unbounded memory on a dense radius query
(the scaling risk `system-design.md` already names for this component).

### 3. FB-profile-link validation gate (F-005, INV-001)

```
validateFacebookAnchor(candidateUrl):
    if not matches_fb_profile_or_page_pattern(candidateUrl):     # facebook.com/<slug>,
        return AnchorValidationResult(valid=False, reason="not_a_facebook_url")  # profile.php?id=…, m.facebook.com/…
    # [assumption] resolvability mechanism unspecified in seed — some check confirms the link
    # resolves to a live, publicly visible profile/page (not 404, not fully privacy-walled).
    # To confirm at scaffold: reuse the scraper's existing FB access, or a separate check.
    if not is_resolvable_public_profile(candidateUrl):
        return AnchorValidationResult(valid=False, reason="unresolvable_or_private")
    return AnchorValidationResult(valid=True, normalized_url=normalize(candidateUrl))

gateListingWrite(draft, actor):
    # single gate for BOTH pipelines — system-design: "both pipelines converge on the same
    # Listing write path in the Backend API"
    result = validateFacebookAnchor(draft.fb_profile_url)
    if not result.valid:
        raise ValidationError(field="fb_profile_url", reason=result.reason)
        # manual path -> 422, mobile Submit stays disabled (BR-001/BR-002 UX)
        # scraper path -> post is dropped; no Listing/ScrapedPost row is ever persisted for it
    begin transaction
        listing = Listing.insert(draft.fields())
        FacebookAnchor.insert(listing_id=listing.id, fb_profile_url=result.normalized_url,
                               resolved_at_submit=true)
        # one transaction: no code path commits Listing without also committing FacebookAnchor —
        # this IS the INV-001 enforcement ("a Listing write SHALL NEVER commit without one")
    commit transaction
    return listing
```
Complexity: O(1) pattern check + one resolvability call per candidate write; no batching needed
(gate runs synchronously per write, on both the manual endpoint and the ingestion worker).

## Sequence diagrams

```
Manual submission (F-002/F-005):
Mobile Client -> Backend API: POST /listings (fields + fb_profile_url)
Backend API -> validateFacebookAnchor(): pattern + resolvability check
  [invalid] Backend API -> Mobile Client: 422 (Submit re-enabled, field error)
  [valid]   Backend API -> Data store: BEGIN; INSERT Listing; INSERT FacebookAnchor; COMMIT
            Backend API -> Mobile Client: 201 (listing visible same session)
            Backend API -> Alerting worker: enqueue (if status == missing)

Scrape ingestion (F-001/F-005):
Scraper worker -> normalize + computeDedupHash()
Scraper worker -> findDedupMatch()
  [match]    Scraper worker -> Backend API: gateListingWrite(duplicate_of=original.id)
             Scraper worker -> Data store: INSERT ScrapedPost(listing_id=dup.id)   # no fanout
  [no match] Scraper worker -> Backend API: gateListingWrite(duplicate_of=None)
             Backend API -> validateFacebookAnchor()
               [invalid] Backend API -> Scraper worker: reject; post dropped, nothing persisted
               [valid]   Backend API -> Data store: BEGIN; INSERT Listing+FacebookAnchor; COMMIT
             Scraper worker -> Data store: INSERT ScrapedPost(listing_id=canonical.id)

Status transition -> alert fanout (F-003/F-004/INV-002):
Mobile Client -> Backend API: PATCH /listings/{id} {status: missing|resolved|...}
Backend API -> Data store: SELECT ... FOR UPDATE
  [terminal & not admin] Backend API -> Mobile Client: 409 Conflict
  [ok] Backend API -> Data store: BEGIN; UPDATE Listing; INSERT StatusHistory; COMMIT
       Backend API -> Mobile Client: 200
       [new_status == missing] Backend API -> Alerting worker: enqueue createAlertAndFanout
Alerting worker -> Data store: INSERT Alert
Alerting worker -> Data store: findOptedInUsersWithinRadius() (geo index)
loop per user (paginated):
  Alerting worker -> Data store: INSERT AlertDelivery (skip if exists — unique constraint)
  Alerting worker -> Push provider: send(token)
  Push provider --> Alerting worker: ack/fail
  Alerting worker -> Data store: UPDATE AlertDelivery.delivered_at (on success only)
```

## Error-handling strategy

- **FB-anchor validation failure (INV-001 gate):** never partially persists — manual path returns
  422 to the client (Submit stays disabled, per approved UX); scraper path drops the post with no
  `Listing`/`ScrapedPost` row at all (nothing to roll back). Logged against the originating
  `ScrapeSource` so repeated rejections feed the existing `ScrapeSource.status`
  (`active`/`degraded`/`blocked`) signal named in `data-model.md`.
- **Status transition conflict (INV-002 gate):** a transition attempted on an already-terminal
  listing by a non-admin actor raises `409 Conflict` inside the same transaction that took the row
  lock — no partial `StatusHistory` write, no listing left in an ambiguous state.
- **Dedup match ambiguity:** exact-hash-only matching is a deliberate false-negative bias (Algorithm
  1a) — a missed match surfaces as two visible listings rather than silently merging two distinct
  cats; caught downstream by users/admins, not auto-corrected.
- **Push provider failure:** caught per-token inside `fanoutJob`, logged, and never aborts the
  job or the enclosing write path — `AlertDelivery.delivered_at` stays null and is not retried
  automatically within this design (no retry-queue mechanism is specified in `data-model.md`;
  flagged as an open question, not asserted as built).
- **Fanout job retry/crash:** `AlertDelivery(alert_id, user_id)` uniqueness (per `data-model.md`
  constraints) makes `fanoutJob` safe to re-run from the top after a crash — already-delivered
  users are skipped, not re-notified.
- **Staleness sweep:** pure batch `UPDATE`, no per-row failure mode; a failed sweep run simply
  leaves listings visible one cycle longer — never mutates `status`, so it can never trip INV-002
  in the failure case either.

## Key decisions

- **Exact-hash dedup, not fuzzy text similarity** — simplest honest form for MVP scale; trade-off
  is missed duplicates (visible, correctable) over false-positive merges (invisible, uncorrectable
  data loss). Candidate ADR if this needs revisiting.
- **Row-lock (`FOR UPDATE`) on status transition** rather than optimistic concurrency — chosen
  because INV-002 is a hard invariant and the transition volume is low (one report at a time), so
  the lock-wait cost is negligible against the correctness guarantee.
- **Terminal-status reopening requires admin** — not stated explicitly as a rule in the seed docs;
  added here to prevent an accidental duplicate "resolved" tap from silently reopening a listing
  and re-triggering fanout. Flagged as [assumption] pending product-owner confirmation.
- **One shared `gateListingWrite()` for both pipelines** — directly mirrors `system-design.md`'s
  stated convergence; duplicating the FB-anchor gate per pipeline was rejected there and is not
  re-litigated here.
- **Alert radius source (user's own `radius_km` vs. `Alert.radius_km`)** — open question, see
  Algorithm 2; not resolved by invention, left for the data-model owner.
- **Staleness window value (BR-005)** — no number exists in any upstream doc; open question, not
  a confident target.

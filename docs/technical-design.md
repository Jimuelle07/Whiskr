# Technical Design Document (LLD)

> **Purpose:** the HOW, detail. Per-component design engineers implement from.
> Traces back to: system design.
> Scope of this pass: the three highest-risk pieces flagged for phase-5.1 — (1) listing
> dedup/staleness-suppression (F-001/F-003, INV-002), (2) location-based alert fanout (F-004),
> (3) the FB-profile-link validation gate (F-005, INV-001). All components/entities below are
> exactly the ones named in `system-design.md` and `data-model.md` — none invented here.
> **Extended 2026-09-18** (same rigor, same rule — no invented components) to cover the auth
> substrate resolved that day: (4) Facebook OAuth login/session issuance + logout (`ADR-0001`,
> `Session` entity), (5) multi-cat batch report creation (F-006).

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
- **Backend API — Auth Service** (added 2026-09-18, `ADR-0001`) — owns `verifyFacebookToken()`,
  `loginWithFacebook()` (Algorithm 4), and `revokeSession()` (logout, API-012). The only component
  that talks to Facebook's Graph API for token verification (distinct from the scraper, which polls
  pages/groups — `system-design.md` Integration points).
- **Backend API — Report Batch Service** (added 2026-09-18, F-006) — owns `createBatchListings()`
  (Algorithm 5). Calls into the same Listing Write Service's `gateListingWrite()` once per cat, never
  duplicating F-005/INV-001 enforcement.
- **Backend API — Photo Upload Service** (added 2026-09-18, BR-013) — owns
  `requestPhotoUploadUrl()` and `validatePhotoUpload()`; the latter is called by both the Listing
  Write Service and the Report Batch Service, never duplicated.
- **Backend API — Verification Service** (added 2026-09-18, F-102) — owns
  `sweepAccountVerification()` (Algorithm 8, a scheduled worker like the staleness sweep) and the
  phone-OTP pair `startPhoneVerification()`/`confirmPhoneVerification()` (Algorithm 9). The
  Facebook-signal check itself lives inline in the Auth Service's `loginWithFacebook()`
  (Algorithm 4), not here, since it only ever runs once, at signup.
- **Data store** — `Listing`, `FacebookAnchor`, `ScrapedPost`, `StatusHistory`, `Alert`,
  `AlertDelivery`, `UserLocation`, `PushToken`, `Session`, `ReportBatch` (all from `data-model.md`;
  the last two added 2026-09-18, no other new entities).

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
findOptedInUsersWithinRadius(lat, lng) -> Iterator[User]   # FIXED 2026-09-18: dropped the
    # radius_km param — it didn't match the 2-arg call site below, and per the resolved "radius
    # source" design each recipient's own UserLocation.radius_km is the match distance, not a
    # single global radius passed in
fanoutJob(alert_id: uuid) -> None

# Auth Service (Backend API) — added 2026-09-18, ADR-0001
verifyFacebookToken(fb_access_token: str) -> FacebookIdentity                 # calls Graph API
loginWithFacebook(fb_access_token: str) -> tuple[User, str, bool]             # Algorithm 4; bool = is_new_user
revokeSession(raw_token: str) -> None                                         # API-012 logout
requireSession(raw_token: str) -> User                                       # every authenticated call's entry gate

# Report Batch Service (Backend API) — added 2026-09-18, F-006
createBatchListings(cats: list[ListingDraft], fb_profile_url: str, actor: User) -> ReportBatchResult  # Algorithm 5

# Account lifecycle (Backend API) — added 2026-09-18, BR-012
deleteAccount(actor: User) -> None                                            # API-015
optOutOfLocation(actor: User) -> None                                         # API-013
unregisterPushToken(token_id: uuid, actor: User) -> None                      # API-014

# Photo Upload Service (Backend API) — added 2026-09-18, BR-013
requestPhotoUploadUrl(content_type: str, actor: User) -> PhotoUploadResult    # API-011
validatePhotoUpload(photo_url: str, actor: User) -> PhotoUpload              # called by gateListingWrite/createBatchListings

# Verification Service (Backend API) — added 2026-09-18, F-102
sweepAccountVerification(now: timestamp) -> int                              # Algorithm 8, scheduled
startPhoneVerification(phone_number: str, actor: User) -> None               # Algorithm 9; API-017
confirmPhoneVerification(code: str, actor: User) -> None                     # Algorithm 9; API-018
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

    # FIXED 2026-09-18 — both gateListingWrite() calls below previously had no error handling at
    # all: a ValidationError (e.g. the post's profile/page link fails BR-008) would propagate
    # uncaught out of this function, when "Error-handling strategy" (below) and FRD-F001's own
    # Error handling table already documented the correct behavior as "reject before write, log
    # source_post_url + reason, feed unaffected" — a graceful drop, never a crash of the ingest
    # job. The two now match.
    try:
        if original is not None and original.duplicate_of is None:
            # collapse into the existing canonical listing
            dup = gateListingWrite(ListingDraft(post, duplicate_of=original.id), actor=None)
            ScrapedPost.insert(scrape_source_id=post.source_id, listing_id=dup.id,
                                original_post_url=post.url, dedup_hash=dedup_hash)  # INV-003: URL kept
            return  # no alert fanout for a duplicate row — see Algorithm 2 guard

        # no match: this becomes the canonical listing
        canonical = gateListingWrite(ListingDraft(post, duplicate_of=None), actor=None)  # Algorithm 3
    except ValidationError as e:
        log(source_post_url=post.url, reason=e.reason)   # BR-008 gate failure or similar;
        return                                             # no Listing/ScrapedPost row for this
                                                             # post — nothing to roll back, feed
                                                             # continues serving the last known-
                                                             # good state (system-design.md)
    ScrapedPost.insert(scrape_source_id=post.source_id, listing_id=canonical.id,
                        original_post_url=post.url, dedup_hash=dedup_hash)
```
Complexity: O(1) hash lookup per post (indexed on `ScrapedPost.dedup_hash` per `data-model.md`
constraints); geo-bucketing keeps the match query narrow without a full-table scan.

### 1b. Status transition + INV-002 enforcement

> **Fixed 2026-09-18 — this algorithm previously contradicted `frd.md` F-003 and its own
> state-transition table.** Two bugs, found during a docs-completeness audit before any code was
> written: (1) `TERMINAL_STATUSES` omitted `found`, so a `found` listing was never guarded by the
> terminal check and could illegally transition back to `available`/`missing` — a direct INV-002
> breach the FRD's own transition table already forbids ("any of `{adopted, found, resolved}` →
> `available` or `missing`: No — rejected unconditionally"). (2) the guard had an `and not
> actor.is_admin` exception, silently allowing an admin to reopen a terminal listing — contradicting
> `frd.md`'s explicit "no override, including for admins." Both are fixed below; nothing had been
> built against the buggy version.

```
TERMINAL_STATUSES = {adopted, found, resolved}   # never "available"/"missing" again, by definition
ACTIVE_STATUSES    = {available, on_hold, missing}

transitionStatus(listing_id, new_status, actor):
    begin transaction
        listing = SELECT Listing WHERE id = listing_id FOR UPDATE   # row lock: no concurrent
                                                                      # transition races INV-002
        if actor.id != listing.submitted_by and not actor.is_admin:
            abort transaction; raise Forbidden("not the submitter or an admin")   # BR-004 gate:
                                                                      # WHO may attempt at all
        if listing.status in TERMINAL_STATUSES:
            abort transaction; raise Conflict("listing already resolved")   # INV-002 gate:
            # unconditional — no actor.is_admin exception. Terminal means terminal, full stop.
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
        # Note (2026-09-18): per the transition table above, no transition ever lands ON
        # `missing` — it is reachable only at creation (see gateListingWrite's own trigger,
        # Algorithm 3, which is the one that actually fires today). This branch is intentionally
        # kept as defensive/forward-compatible code satisfying the PRD's literal F-004 EARS
        # wording ("created OR UPDATED to status missing") in case a future BR change ever adds a
        # transition back into `missing` — not dead code to delete, just currently unreachable.
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
        # geo query: FIXED 2026-09-18 — previously read "UserLocation WHERE location_opt_in =
        # true," but location_opt_in is a User column, not a UserLocation column
        # (data-model.md); a query written that literally would fail (no such column). The
        # correct gate is simpler: a UserLocation row's mere EXISTENCE already means opted-in,
        # because API-013 (opt-out, added 2026-09-18) deletes the row entirely rather than
        # flagging it — so no extra opt-in filter is needed at all:
        #   SELECT User.id FROM UserLocation JOIN User ON UserLocation.user_id = User.id
        #   WHERE ST_DWithin(point(lat,lng), point(listing.location_lat,listing.location_lng),
        #                     UserLocation.radius_km)   -- see note below on radius source
        # paginated in batches (cursor over UserLocation) to bound memory in dense areas
        if AlertDelivery.exists(alert_id=alert.id, user_id=user.id):
            continue   # unique(alert_id, user_id) per data-model.md — idempotent on job retry
        delivery = AlertDelivery.insert(alert_id=alert.id, user_id=user.id,
                                         delivered_at=null, opened_at=null)
        for token in PushToken.where(user_id=user.id):
            try:
                # payload_for() was referenced but never defined until this fix (2026-09-18) —
                # shape per frd.md F-004 Outputs: title, body, deep link to the listing detail.
                # payload_for(listing) -> { title: "A cat may be missing near you",
                #                           body: listing.description[:140],
                #                           deep_link: f"whiskr://listings/{listing.id}" }
                #   -- [assumption]: exact copy and deep-link URL scheme are illustrative, not
                #   sourced from any upstream doc; confirm at scaffold (mobile deep-link config).
                pushProvider.send(token, payload_for(listing))
                delivery.delivered_at = now(); delivery.save()
                break   # one successful delivery per user is enough
            except PushProviderError as e:
                log(e)   # never fails the job; retried/logged, per system-design
```
**Radius source — RESOLVED 2026-09-18:** `UserLocation.radius_km` (per-user opt-in preference) and
`Alert.radius_km` (BR-006 default snapshot) are two distinct fields in `data-model.md`. This design
confirms the reconciliation this doc previously only proposed: `UserLocation.radius_km` **is** the
query parameter `fanoutJob()` matches against (fixed above, alongside the `findOptedInUsersWithinRadius`
signature mismatch); `Alert.radius_km` is only ever an audit snapshot of the default policy in
effect at trigger time, never read by the fanout query itself. Since `BR-006`'s radius isn't
user-configurable at MVP (no UI to change it from the 5 km default), the two values are identical
in practice today — this reconciliation only becomes behaviorally visible if/when radius
customization ships post-MVP.

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

validatePhotoUpload(photo_url, actor):
    # BR-013, added 2026-09-18 — closes the gap where photo_url was trusted as opaque client text
    upload = PhotoUpload.find_by(photo_url=photo_url)
    if upload is None or upload.user_id != actor.id:
        raise ValidationError(field="photo_url", reason="unrecognized_or_not_yours")   # -> 400
    if upload.consumed_at is not None:
        raise ValidationError(field="photo_url", reason="already_used")                # -> 400
    if not objectStore.exists(photo_url):        # HEAD check against the real bucket — proves
                                                   # the client actually uploaded something, not
                                                   # just that it asked for a URL
        raise ValidationError(field="photo_url", reason="object_not_found")            # -> 400
    return upload   # caller marks upload.consumed_at = now() inside the same transaction as the
                    # Listing insert below — never before, so a failed submission leaves the
                    # upload URL reusable

gateListingWrite(draft, actor):
    # single gate for BOTH pipelines — system-design: "both pipelines converge on the same
    # Listing write path in the Backend API". photo validation only applies to the manual path
    # (draft.source == 'manual'); scraped posts carry their own photo_url from the source post,
    # never a Whiskr-issued PhotoUpload row.
    result = validateFacebookAnchor(draft.fb_profile_url)
    if not result.valid:
        raise ValidationError(field="fb_profile_url", reason=result.reason)
        # manual path -> 422, mobile Submit stays disabled (BR-001/BR-002 UX)
        # scraper path -> post is dropped; no Listing/ScrapedPost row is ever persisted for it
    upload = validatePhotoUpload(draft.photo_url, actor) if draft.source == 'manual' else None
    begin transaction
        listing = Listing.insert(draft.fields())
        FacebookAnchor.insert(listing_id=listing.id, fb_profile_url=result.normalized_url,
                               resolved_at_submit=true)
        # one transaction: no code path commits Listing without also committing FacebookAnchor —
        # this IS the INV-001 enforcement ("a Listing write SHALL NEVER commit without one")
        if upload is not None:
            upload.consumed_at = now(); upload.save()   # BR-013: one photo, one listing
    commit transaction

    if listing.status == 'missing':
        createAlertAndFanout(listing)   # Algorithm 2, enqueued async — never blocks this call.
        # FIXED 2026-09-18 — this call was missing here entirely, even though the sequence
        # diagram below always documented it ("Backend API -> Alerting worker: enqueue (if
        # status == missing)") and createBatchListings() (Algorithm 5) already had the
        # equivalent per-cat call. Per the F-003 transition table, `missing` is reachable ONLY
        # at creation — there is no transition INTO missing from any other status — which means
        # transitionStatus()'s own alert trigger (Algorithm 1b, "if old_status != 'missing' and
        # new_status == 'missing'") can never actually fire in practice: nothing ever transitions
        # into missing, everything is CREATED at missing. Without this line, F-004 — the entire
        # location-based alert feature, arguably the single most safety-critical part of the
        # product (a lost cat's owner depends on it) — would never have fired for a single real
        # submission. createAlertAndFanout()'s own `duplicate_of` guard already prevents a
        # re-scraped duplicate from double-alerting, so this is safe to call unconditionally here
        # for both the manual and scraper pipelines that share this one function.
    return listing
```
Complexity: O(1) pattern check + one resolvability call per candidate write, plus one indexed
`PhotoUpload` lookup and one object-store `HEAD` call on the manual path only; no batching needed
(gate runs synchronously per write, on both the manual endpoint and the ingestion worker).

### 4. Facebook OAuth login + session issuance / revocation (`ADR-0001`)

```
verifyFacebookToken(fb_access_token):
    resp = graphAPI.get("/me", params={"access_token": fb_access_token,
                                         "fields": "id,name,picture"})
    if resp.status != 200:
        raise AuthError(reason="invalid_or_expired_token")
    # MANDATORY as of 2026-09-18 (was "recommended, not confirmed" — resolved per
    # security-compliance.md T-010; skipping this check is the exact stolen-token-from-
    # another-app replay T-010 names):
    debug = graphAPI.get("/debug_token", params={"input_token": fb_access_token,
                                                  "access_token": WHISKR_APP_ACCESS_TOKEN})
    if debug.data.app_id != WHISKR_FB_APP_ID:
        raise AuthError(reason="wrong_app_id")
    return FacebookIdentity(fb_user_id=resp.id, name=resp.name,
                             has_real_photo=not resp.picture.data.is_silhouette)

loginWithFacebook(fb_access_token):
    identity = verifyFacebookToken(fb_access_token)      # raises AuthError on failure -> 401
    user = User.find_by(fb_user_id=identity.fb_user_id)
    is_new = user is None
    if is_new:
        fb_signals_ok = identity.has_real_photo and " " in identity.name.strip()  # FRD-F102-01
        try:
            user = User.insert(fb_user_id=identity.fb_user_id, display_name=identity.name,
                                location_opt_in=false, is_admin=false,
                                fb_signals_verified_at=(now() if fb_signals_ok else null))
        except UniqueConstraintViolation:   # race: two concurrent first-time logins, same
                                             # fb_user_id — data-model.md's unique constraint
                                             # catches it; treat the loser as a normal login,
                                             # never a 500 or a duplicate account
            user = User.find_by(fb_user_id=identity.fb_user_id)
            is_new = false
    raw_token = crypto.secure_random(32)                  # opaque token, never a JWT
    Session.insert(user_id=user.id, token_hash=sha256(raw_token),
                    expires_at=now() + 90_days)
    return (user, raw_token, is_new)                      # raw_token returned to client ONCE

revokeSession(raw_token):
    session = Session.find_by(token_hash=sha256(raw_token))
    if session is None or session.revoked_at is not None or session.expires_at < now():
        raise AuthError(reason="already_invalid")          # API-012 -> 401, not a silent success
    session.revoked_at = now()
    session.save()

requireSession(raw_token):
    # the entry gate for every authenticated endpoint (API-003 onward)
    session = Session.find_by(token_hash=sha256(raw_token))
    if session is None or session.revoked_at is not None or session.expires_at < now():
        raise AuthError(reason="unauthenticated")           # -> 401
    return User.find(session.user_id)
```
Complexity: O(1) — a single indexed lookup (`Session.token_hash`, unique) per authenticated request;
no in-memory session cache is assumed (each request re-checks `revoked_at`/`expires_at` directly,
so a revoked token is rejected on the very next call, not after a cache TTL — this is what makes
T-012's mitigation actually hold).

### 5. Multi-cat batch report creation (F-006, BR-009)

```
createBatchListings(cats, fb_profile_url, actor):
    if len(cats) < 2:
        raise ValidationError(field="cats", reason="batch_requires_at_least_two")  # -> 400

    result = validateFacebookAnchor(fb_profile_url)        # BR-009: validated ONCE for the batch
    if not result.valid:
        raise ValidationError(field="fb_profile_url", reason=result.reason)        # -> 422, no writes

    uploads = [validatePhotoUpload(c.photo_url, actor) for c in cats]  # BR-013, per cat, before
                                                                          # any write — same
                                                                          # all-or-nothing rule
    begin transaction
        batch = ReportBatch.insert(submitted_by=actor.id)
        listings = []
        for cat_draft, upload in zip(cats, uploads):        # each cat independently satisfies BR-001
            listing = Listing.insert(cat_draft.fields(), submitted_by=actor.id,
                                       batch_id=batch.id, source="manual")
            FacebookAnchor.insert(listing_id=listing.id, fb_profile_url=result.normalized_url,
                                    resolved_at_submit=true)   # INV-001: same gate, per listing
            upload.consumed_at = now(); upload.save()          # BR-013, per cat
            listings.append(listing)
        # any single cat_draft failing BR-001 field validation raises inside the loop, aborting
        # the whole transaction — FRD-F006-01's all-or-nothing rule
    commit transaction

    for listing in listings:
        if listing.status == 'missing':
            createAlertAndFanout(listing)    # Algorithm 2, per cat independently, async
    return ReportBatchResult(batch_id=batch.id, listings=listings)
```
Complexity: O(N) for N cats in the batch — one `FacebookAnchor` resolvability call total (shared,
per BR-009), not per cat. `createBatchListings()` reuses `validateFacebookAnchor()` from Algorithm 3
directly (the same INV-001 check, called once instead of once-per-cat) and then repeats
`gateListingWrite()`'s *insert* shape (`Listing` + `FacebookAnchor` in one transaction) per cat with
the already-validated result — it does not re-run `gateListingWrite()` itself, which would
re-validate the anchor N times and violate BR-009's "once per batch" rule. This is the one
deliberate divergence from "one shared gate for both pipelines" (`system-design.md`), and it exists
specifically to satisfy BR-009, not to duplicate INV-001 enforcement logic — the validation function
is still shared; only the per-cat write loop is new.

### 6. Account deletion cascade (BR-012)

```
deleteAccount(actor):
    begin transaction
        Session.delete_all(user_id=actor.id)              # immediate logout everywhere
        UserLocation.delete(user_id=actor.id)              # if present
        PushToken.delete_all(user_id=actor.id)
        Listing.update_all(WHERE submitted_by=actor.id, SET submitted_by=null)  # BR-012: never
                                                             # cascade-delete a listing itself
        User.delete(actor.id)
    commit transaction

optOutOfLocation(actor):
    UserLocation.delete(user_id=actor.id)   # no-op success if none exists ([assumption])
    actor.location_opt_in = false
    actor.save()

unregisterPushToken(token_id, actor):
    token = PushToken.find(token_id)
    if token is None:
        raise NotFoundError()                              # -> 404
    if token.user_id != actor.id:
        raise ForbiddenError()                              # -> 403, never let a user unregister
                                                             # another user's device
    token.delete()
```
Complexity: O(1) plus O(k) for the `Listing.submitted_by` bulk update, where k = the user's own
listing count (typically small at MVP scale, not a hot path). One transaction so a crash mid-delete
never leaves a half-deleted account (e.g., sessions revoked but the `User` row still present, or
vice versa).

### 7. Account verification: track-record sweep + phone OTP (F-102)

```
sweepAccountVerification(now):
    UPDATE User
    SET track_record_verified_at = now
    WHERE track_record_verified_at IS NULL
      AND created_at <= now - 30_days
      AND (SELECT count(*) FROM Listing WHERE submitted_by = User.id) >= 3
    RETURNING count(*)
```
Same shape as `sweepStaleListings()` (Algorithm 1c): a batched `UPDATE`, no per-row branching, runs
daily. One-way (`WHERE track_record_verified_at IS NULL` means an already-verified account is never
touched again — matches "never unset once granted," BR-014).

```
startPhoneVerification(phone_number, actor):
    if User.exists(phone_number=phone_number, phone_verified_at__not_null=true):
        raise ConflictError(reason="phone_already_in_use")           # -> 409, BR-016
    code = crypto.secure_random_digits(6)
    PhoneVerification.insert(user_id=actor.id, phone_number=normalize_e164(phone_number),
                               otp_hash=sha256(code), expires_at=now() + 10_minutes)
    smsProvider.send(phone_number, f"Your Whiskr code is {code}")     # never logged, never stored
                                                                        # raw beyond this one send

confirmPhoneVerification(code, actor):
    pv = PhoneVerification.find_latest(user_id=actor.id, verified_at=null)
    if pv is None or pv.expires_at < now():
        raise ValidationError(reason="no_pending_or_expired")         # -> 400
    if pv.attempt_count >= 5:
        raise TooManyAttemptsError()                                  # -> 429
    if sha256(code) != pv.otp_hash:
        pv.attempt_count += 1; pv.save()
        raise ValidationError(reason="wrong_code")                    # -> 400
    begin transaction
        try:
            User.update(actor.id, phone_number=pv.phone_number, phone_verified_at=now())
        except UniqueConstraintViolation:      # race: another account confirmed the same
                                                 # number between start and confirm
            abort transaction; raise ConflictError(reason="phone_already_in_use")   # -> 409
        pv.verified_at = now(); pv.save()
    commit transaction
```
Complexity: O(1) per call, one indexed lookup each. The `UniqueConstraintViolation` catch mirrors
Algorithm 4's signup race-condition handling — same pattern, different table, both trusting the
database's own uniqueness guarantee rather than a check-then-write race in application code.

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

Facebook OAuth login/logout (ADR-0001, API-007/API-012):
Mobile Client -> Facebook Login SDK: on-device login -> fb_access_token
Mobile Client -> Backend API: POST /auth/facebook { fb_access_token }
Backend API -> Facebook Graph API: GET /me?access_token=...
  [invalid/expired] Backend API -> Mobile Client: 401
  [valid] Backend API -> Data store: find-or-create User by fb_user_id
          Backend API -> Data store: INSERT Session(token_hash=sha256(raw_token))
          Backend API -> Mobile Client: 200/201 { access_token: raw_token, user }
...
Mobile Client -> Backend API: DELETE /auth/sessions  (Authorization: Bearer <raw_token>)
Backend API -> Data store: UPDATE Session SET revoked_at = now() WHERE token_hash = sha256(token)
Backend API -> Mobile Client: 204

Multi-cat batch report (F-006, API-010):
Mobile Client -> Backend API: POST /listings/batch { fb_profile_url, cats: [...] }
Backend API -> validateFacebookAnchor(fb_profile_url): pattern + resolvability check (once)
  [invalid] Backend API -> Mobile Client: 422 (whole batch rejected, no writes)
  [valid]   Backend API -> Data store: BEGIN; INSERT ReportBatch;
              loop per cat: INSERT Listing(batch_id=...) + INSERT FacebookAnchor; COMMIT
            Backend API -> Mobile Client: 201 { batch_id, listings: [...] }
            Backend API -> Alerting worker: enqueue per cat where status == missing
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
- **Facebook token verification failure:** `verifyFacebookToken()` raises before any `User`/`Session`
  row is touched — no partial account/session state; the client sees a plain `401` and re-attempts
  the Facebook Login SDK flow, never a Whiskr-side retry loop.
- **Batch report partial failure:** the loop in `createBatchListings()` runs inside one transaction;
  any single cat's BR-001 field validation failure aborts and rolls back every `Listing`/
  `FacebookAnchor`/`ReportBatch` insert in that call — never an N-1-of-N partial batch
  (FRD-F006-01).
- **Photo-upload validation failure (BR-013):** raised before the transaction opens — no partial
  `Listing` write is ever attempted against an unrecognized/reused/missing photo.
- **Facebook/OTP concurrency races:** both `loginWithFacebook()` and `confirmPhoneVerification()`
  catch a `UniqueConstraintViolation` from the database rather than doing a check-then-write race in
  application code — the database's unique index is the actual source of truth for "does this
  fb_user_id/phone_number already exist," never re-derived optimistically.

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
- **Opaque hashed session tokens over JWT** — chosen so logout (API-012) is a true revocation, not a
  denylist workaround; costs one indexed DB lookup per authenticated request instead of stateless
  local verification, judged worth it at MVP scale (`decision-ledger.md`).
- **Facebook-anchor validated once per batch, not once per cat** (`createBatchListings()`) — a
  deliberate, named divergence from "one shared gate for both pipelines" specifically to satisfy
  BR-009 without N redundant resolvability calls against the same URL in one request.
- **Account verification is three independent OR'd signals, not a tiered ladder** — each of
  `fb_signals_verified_at`/`track_record_verified_at`/`phone_verified_at` is set independently;
  `is_verified` is their disjunction. Chosen over a single ladder/tier so a strong signal (phone)
  isn't gated behind a weaker one (Facebook heuristic) that already failed.
- **Facebook-signal check runs once, at signup, never re-evaluated** — avoids re-fetching the
  Graph API on every login for a check whose input (profile photo/name) users rarely change back
  to a "worse" state; a user who later improves their Facebook profile can still reach verified
  status via the other two paths.

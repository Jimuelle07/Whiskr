# ADR-0001 — Facebook OAuth as the sole account authentication mechanism

**Status:** Accepted · **Date:** 2026-09-18 · **Decision ledger line:** `docs/decision-ledger.md` §3, "Auth mechanism resolved: Facebook OAuth only, no app-side password"

## Context

`api-spec.md`, `data-model.md`, and `security-compliance.md` all independently flagged the account
authentication mechanism as an unresolved `[assumption]` — `User.auth_identifier` had no named
scheme (phone/email/OAuth). `docs/implementation-plan.md` §1 named this as the #2 highest
implementation risk, since every authenticated endpoint (submission, status transition, location
opt-in, push-token registration) sits on top of it.

Separately, F-005/INV-001 already require every listing to carry a visible, linked Facebook
profile as an identity anchor — every Whiskr user already needs a working Facebook account to use
the product's core reporting flow (UJ-001) at all.

The product owner was asked directly whether login/signup should be Facebook OAuth, email+password,
or phone OTP, given a stated requirement for a "forgot password" feature.

## Decision

Whiskr's **sole** account login/signup mechanism is Facebook OAuth. The Mobile Client obtains a
short-lived Facebook access token via the on-device Facebook Login SDK; the Backend API verifies
that token against Facebook's Graph API, resolves `fb_user_id`, upserts the `User` row on first
contact, and issues its own Whiskr-signed bearer session token (`api-spec.md` API-007).

**No app-side password exists.** As a direct, intentional consequence, there is **no forgot-password
feature** — account recovery is entirely a function of the user's own access to their Facebook
account, which is outside Whiskr's control by design.

No Facebook access or refresh token is ever persisted server-side; the token is verified once at
login and then discarded (`data-model.md` retention note). Only `fb_user_id` and a cached display
name are stored.

## Alternatives considered

| Alternative | Rejected because |
|---|---|
| Email + password | Requires building and securing an entire second identity system (password hashing, reset-token generation/expiry, reset-email delivery) that duplicates work the Facebook-anchor requirement already forces every user through anyway — unjustified for `team_size: 1` |
| Phone OTP | Requires SMS delivery infrastructure and cost with no stated justification anywhere in the seed |
| Hybrid (Facebook OAuth + optional email/password) | Doubles the auth surface area (two code paths to secure, two sets of edge cases) for a segment where every real user already has the Facebook account F-005 requires |

## Consequences

- **Positive:** one identity system instead of two; reuses the same Facebook account users already
  need for F-005; no password-reset attack surface (credential stuffing, reset-token leakage) to
  defend; no password storage/hashing to get wrong.
- **Negative / accepted risk:** a user with no Facebook account cannot use Whiskr at all (already
  true today for F-005's per-listing anchor — this decision extends the same constraint to account
  creation, not a new one). Account recovery is entirely dependent on the user's own Facebook
  account access; Whiskr has no independent recovery path if a user loses Facebook access.
- **New threat surface named, not silently absorbed:** a stolen/replayed Facebook access token
  issued for a *different* app could be used to log into Whiskr as that Facebook user unless the
  token's `app_id` is verified (Facebook's `debug_token` endpoint) before trust — tracked as
  `security-compliance.md` T-010, **not yet confirmed as implemented**.
- Hard to reverse once real accounts exist under Facebook identity — migrating to a second auth
  mechanism later would need an account-linking flow, not a simple field addition.

## Invalidates

The `[assumption]` auth-mechanism rows previously carried in `api-spec.md` (Authentication &
authorization), `data-model.md` (`User.auth_identifier`), and `security-compliance.md` (Authn/authz
model) — all three are reconciled to this decision in the same checkpoint as this ADR.

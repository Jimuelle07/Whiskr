# Validation verdict — Whiskr

<!-- EMITTED as docs/seed/validation.md. Written by concept_validator (GPT-5.6 Terra) — a model that authored
     nothing in idea.md — from the finished brief + evidence ledger + market seed. It never edits
     the brief. The Vault reads this as context and NEVER gates on it. -->

**Verdict:** `GO-UNVALIDATED`
**Cycle:** 1 of 2 · **Date:** 2026-09-18 · **Inputs read:** idea.md · gate output (structurally complete) · (no evidence-ledger.md, no market.md — none exist; fast path, §3 explicitly N/A)
**Independence:** `DEGRADED (same family+gen, Claude)` — no `resolve-roster.py` / cross-family validator reachable in this environment; running in a fresh context with only the Part C input contract (idea.md, context.md, gate output), never saw a draft, did not author the brief. Per ADR-0007, proceeding under DEGRADED rather than refusing. §1–§3 read in full below before trusting this verdict.

## 1. The five tests, as scored from evidence (not as the author filled them)

| Test | Verdict | Evidence cited (EV-###) | Gap |
|---|---|---|---|
| Real — do people in §2 have the §1 pain? | unknown | none (no evidence-ledger.md exists) | No `INT-###`/`EV-###` collected; §3 says pain is "described by the product owner, not yet independently observed" |
| Large — is §2 a countable segment (§5 band)? | unknown | none | §5 size band ("tens of thousands") is explicitly flagged `[assumption]`, no sourced count |
| Significant — does the workaround cost enough (time/money/risk)? | unknown | none | §3: "cost described as time + missed-outcome risk, not yet measured" |
| Urgent — does it recur on a clock they cannot ignore? | unknown | none | §3: "argued as time-critical by design, not yet observed in the field" |
| Relevant — is this problem still unsolved and worth solving today? | unknown | none | No evidence Facebook has closed this gap since; plausible on its face (§4 root cause is structural to Facebook Groups) but not independently checked |

Author's own four-test table (§3) already self-scores all four as `unknown` — this validator concurs on all five (adding Relevant) for the same reason: `said`-only support with zero `EV-###` items cannot clear `pass`, per role rules.

## 2. Does the MVP test A-001?
- **A-001 as stated:** "Rescuers, finders, and page admins will feed their listings into Whiskr (via direct submission) rather than relying solely on posting to Facebook — without that, Whiskr has no independent supply and is just a lower-coverage mirror of Facebook."
- **Which `F-0xx` exercises it:** **F-002** (manual submission form) — the brief itself marks this line ("tests A-001 (§9)") and the mapping is correct: F-002 is the only MVP feature that requires a rescuer/finder to act inside Whiskr instead of only on Facebook. F-001 (scraping), F-003 (status tagging) and F-004 (alerts) all operate on listings regardless of where supply originates, so they cannot falsify or confirm A-001 on their own.
- **What the build would learn if A-001 is false:** if direct submissions via F-002 stay near zero while F-001's scraped feed keeps growing, Whiskr has no independent supply and is confirmed as a lower-coverage Facebook mirror — the exact fail-state A-001 names.

## 3. Kill criteria — any already tripped?
| Criterion (§9) | Tripped? | Evidence |
|---|---|---|
| Regulatory | no | Hypothetical future event ("Facebook blocks/bans the scraping mechanism") — nothing in idea.md indicates it has happened; no evidence either way exists pre-build |
| Unit economics | no (N/A by design) | §9 itself states N/A — "no monetization exists at MVP; revisit once release planning begins" — author-declared non-applicability, not a tripped criterion |
| Technical | no | "scraped coverage degrades below a usable threshold" is a post-launch measurement; nothing in the brief or gate output indicates this has occurred |

No kill criterion is tripped by the brief's own internal logic. No unresolved contradiction found between §1/§4 (root cause), §7 (MVP), and §9 (A-001/kill criteria) — F-002's existence as an A-001 test is consistent with, not undercut by, F-001's scraping approach.

## 4. Invariants and banned copy
- INV-001, INV-002, INV-003 are each phrased as "the system must never…" — correctly stated as must-never sentences, not aspirations.
- No brand banned-copy list is present in this fast-path brief (none was produced at this stage), so there is nothing to cross-check against the INV set; this is not a gap this validator can invent evidence to fill.

## 5. Human-verified remainder / disagreement record
- Cycle 1, no disagreement to record.
- Flag for the human (not a validation finding): `context.md` marks `team_size`, `build_type`, `time_budget`, and `handoff_expected` as `[assumption]` — the product owner should confirm these before Vault phase-5.0 doc-selection is treated as final, per context.md's own note.
- Nothing here required an `INT-###` pull (none exist).

## 6. Decision
- **GO-UNVALIDATED** — §3 is honestly N/A (fast path, no interviews run yet); the brief is internally consistent, A-001 is correctly wired to F-002, and no kill criterion is tripped. The build itself is the first evidence: specifically, real-world direct-submission volume through F-002 (against F-001's scraped baseline) is what will confirm or falsify A-001. State this explicitly in the demo — this is a GO to build, not a GO on validated demand.

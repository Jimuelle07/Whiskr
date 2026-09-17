---
# Vault context block — schema 2.1.0
team_size: 1                # [assumption] — solo greenfield build; confirm/correct
mode: solo
build_type: production      # [assumption] — treated as a real product idea, not a hackathon/graded build
time_budget: 2w              # [assumption] — order of magnitude only; confirm/correct
judged: false
computes_numbers: false
exposed_surface: true        # mobile app + backend/auth surface
exposes_api: true            # mobile client depends on a backend contract
has_ui: true                 # native mobile app
outlives_demo: true           # intended to run as a real product, not a one-off demo
build_crew: true
tests: deferred
release_planning: true       # real user acquisition/adoption planning will be needed (competes with Facebook habit)
handoff_expected: false      # [assumption] — solo builder for now
pivots_expected: true         # GO-UNVALIDATED build on an unvalidated idea; pivots likely
rigor: standard
selection_mode: auto
speed: standard               # time_budget > 2h
---

# Context intake

<!-- The Vault reads this FIRST (phase-5.0). Run: python3 vault/tools/select-docs.py docs/seed/context.md -->

Several fields above are marked `[assumption]` in comments because the Key ran the **fast path** —
they were set to a reasonable default rather than asked one-by-one in a second interview round. The
product owner should confirm or correct `team_size`, `build_type`, `time_budget`, and
`handoff_expected` before the Vault phase-5.0 doc-selection is treated as final.

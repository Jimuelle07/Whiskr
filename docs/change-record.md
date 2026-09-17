# Change Record

> **Purpose:** an append-only log entry for a change to a **Locked** doc. Governance for
> multi-day / team builds only — a solo hackathon does not need this (the intake won't select it).
> Distinct from an ADR: an ADR captures a *decision's* rationale; a CR records that a locked
> spec *changed*, what, and why. Never edit a past CR — supersede it with a new one.

## Doc lifecycle (why CRs exist)

Docs move **Draft → Locked → Superseded**. While `Draft`, edit freely. Once `Locked` (agreed,
being built against), an edit REQUIRES a CR so contributors know the spec moved and why. A
`Superseded` doc is replaced, not deleted.

---

_No docs have been marked **Locked** yet, and no change has occurred since the seed was authored
(2026-09-18). This log is empty by design — the first entry is appended as `CR-001` the moment a
`Locked` doc is revised._

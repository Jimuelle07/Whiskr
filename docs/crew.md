# Crew — Whiskr's build-time agent roster

> **Purpose:** the 3–6 agents that build *this* product, each bound to a Vault role
> (`vault/roster.json`) so model choice, tool limits, and independence rules are inherited, not
> re-invented. Materialises to `.claude/agents/*.md`. Traces back to: `docs/system-design.md`,
> `docs/implementation-plan.md`. **Not** the Vault's own roles — this is Whiskr's crew.
> **Living doc:** reconcile when a pivot changes architecture or scope (ledger-logged).

## Environment note — Claude-only binding, stated plainly

`vault/roster.json`'s aspirational default (`adr/ADR-0003`, mixed Claude+Codex stack) binds
`executor` to GPT-5.6 Luna (alt GPT-5.4 Mini) and `validator` to GPT-5.6 Terra for true
cross-family independence (R1). **In this environment, no OpenAI/Codex model is actually
reachable** — `pool.gpt-5.6-terra`, `gpt-5.5`, `gpt-5.6-luna`, and `gpt-5.4-mini` are declared in
`roster.json` but nothing in this operator's setup can dispatch them. Rather than materialise crew
agents that name a model this run cannot call, **every agent below is bound to a Claude-family
model instead** (`sonnet-5`, `opus-5`, or `haiku-4.5` — the three Claude tiers actually available
per `roster.json`'s `pool`). This means:

- **R1 (validator independence from the architect's family) cannot be achieved as true
  cross-family independence in this environment** — the architect (Sonnet 5) and any
  validator-role agent are both Claude. Per `roster.json`'s own resolution algorithm this degrades
  to `DEGRADED (same family)`, exactly the posture `docs/validation-report.md` already recorded for
  phase-5.2 ("no cross-family validator was reachable in this environment... proceeding under
  DEGRADED rather than refusing"). The mitigations `roster.json` names for this case (fresh-context
  re-read, mandatory adversarial mode, lead reads the report in full) apply here too — see
  `design-validator` below.
- This is a statement about **this run's environment**, not a permanent architectural choice. If
  an operator later wires in a reachable non-Claude model, `roster.json`'s `resolve-roster.py`
  should be re-run and this doc regenerated from its output rather than hand-edited to add a model
  this doc cannot verify is actually callable.

## Roster rules (anti-sprawl, inherited)

- **3–6 agents.** Each justifies its slot by exactly one of: *repeated-spawn*, *context-offload*,
  or *guardrail-enforcement*. Log rejected agents and why.
- **Least privilege.** Reviewers get `read`. Executors get `write` only inside a task's write scope
  and `shell` only for the test commands in the QA plan.
- **Role binding.** Every agent names its Vault role; the model comes from `roster.json`'s Claude
  entries only (see note above). A product agent never binds a validator-role model to an author
  job (R1, degraded as noted).
- **Guardrails encode invariants.** At least one agent's `never` list carries every `INV-###` as a
  hard negative, so the crew cannot build a breach.

## Agents

### task-executor
- **name:** `task-executor`
- **vault role:** executor *(bound to `sonnet-5` / Claude Sonnet 5 in this environment — the
  roster's preferred fast executors, GPT-5.6 Luna / GPT-5.4 Mini / Haiku 4.5, are unreachable here
  except Haiku 4.5; Sonnet 5 is used instead of the fast tier because no fast Claude model was
  confirmed reachable for this run either — re-check `haiku-4.5` availability before materialising
  and prefer it if confirmed, to keep cost down on the highest-repeat-count role)*
- **description:** Implement exactly one `TASK-###` from `docs/handoff/TASK-###.md`, test-first:
  red → green → refactor, inside the task's write scope only. Invoked once per task in
  `docs/implementation-plan.md` §3, never for more than one task at a time.
- **tools:** read · write (write scope only) · shell (the task's `TC`/`Verify` command and the fast
  gate only — `npm --prefix backend test -- --silent`)
- **justification:** repeated-spawn (14 tasks in the current plan, one dispatch each)
- **Inputs:** `docs/handoff/TASK-###.md` + the files under the task's write scope + the failing
  test named in the packet
- **Outputs:** the evidence block (spec, verify sha, command, result, files touched); files inside
  scope only
- **Never:** edit a test to make it pass (R3) · touch a file outside the task's write scope ·
  continue past 3 failed attempts without escalating (R4) · mark its own task's Status cell (R5,
  keeper-only) · publish a `Listing` without a committed `FacebookAnchor` row in the same
  transaction (INV-001) · allow any read path to render `available`/`missing` for a listing with a
  prior explicit resolution (INV-002) · drop, rewrite, or omit `ScrapedPost.original_post_url` at
  ingest or across a dedup merge (INV-003)
- **Done when:** the task's `TC`/`Verify` command is green, the fast gate is green, evidence is
  returned to the keeper

### plan-keeper
- **name:** `plan-keeper`
- **vault role:** keeper *(bound to `haiku-4.5` / Claude Haiku 4.5 — fast and cheap, and the only
  Claude tier that is a different model from the default `task-executor` binding above, keeping the
  writer of status separate in practice from the writer of code even without true cross-family
  separation)*
- **description:** Run the checkers (`check-implementation-plan.py`, `check-handoff.py`), verify
  red→green evidence on the current base commit with its own re-run (never trusting the
  executor's claim), write `docs/implementation-plan.md` Status cells and phase rows, append run
  events to `docs/run-evidence.jsonl`. Invoked on every checkpoint trigger in
  `process/checkpoint.md`.
- **tools:** read · write (`docs/implementation-plan.md`, `docs/run-evidence.jsonl` only) · shell
  (`vault/tools/*` and the plan's named `Verify`/gate commands only)
- **justification:** guardrail-enforcement (R5 — sole writer of Status; a real build's
  multi-branch status drift is exactly what this prevents)
- **Never:** infer completion from chat or the executor's own claim · lower a gate · decide product
  scope · record `done` while a task's `TC-###` obligation is still `pending` and unwaived · fill
  in `**Build approval:**` (that field is a named human lead's only, `adr/ADR-0010`)
- **Done when:** the plan's Status cells and phase rows match the keeper's own observed evidence,
  and every checker run this checkpoint is either `APPROVE`/`PASS` or the failure is surfaced, not
  hidden

### design-validator
- **name:** `design-validator`
- **vault role:** validator *(bound to `sonnet-5` / Claude Sonnet 5 — the same family as the
  architect in this environment; independence is `DEGRADED (same family)`, not the roster's
  intended cross-family `gpt-5.6-terra`. Every mitigation `roster.json` names for this case is
  mandatory, not optional, for this agent: run in a fresh context carrying none of the architect's
  or executor's own reasoning, only their outputs; treat adversarial mode as mandatory at every
  phase gate rather than optional; the human lead reads this agent's findings in full rather than
  trusting a bare verdict line)*
- **description:** Phase-gate re-read: does the delivered code for a closing phase actually match
  the `F-###`/EARS criteria it claims, and is any `INV-###` breached in the delivered work (not
  just in the design docs, which `docs/validation-report.md` already cleared). Invoked at every
  phase-gate attempt (PH-01 through PH-04 closing) and on demand.
- **tools:** read (docs + the delivered code diff; no write, no shell)
- **justification:** guardrail-enforcement (INV-### breach detection independent of the crew that
  built the code)
- **Never:** author a fix (it judges, never patches) · claim true cross-family independence it does
  not have in this environment · let a `DEGRADED` verdict pass as if it carried the same weight as
  a cross-family one — always states the degradation inline
- **Done when:** a verdict (PASS / REVISE / ESCALATE) is recorded against the closing phase's
  actual delivered `F-###`/`INV-###` claims, with the independence caveat stated

### slice-planner
- **name:** `slice-planner`
- **vault role:** planner / architect *(bound to `sonnet-5` / Claude Sonnet 5 — matches
  `roster.json`'s own preferred binding for this role; no substitution needed)*
- **description:** Write and maintain `docs/implementation-plan.md`, write each `docs/handoff/
  TASK-###.md` packet at claim time, split a stuck task or add a sensor on executor escalation
  (R4), and replan when a checkpoint changes dependencies or scope. This is the agent that
  authored the current plan.
- **tools:** read · write (`docs/implementation-plan.md`, `docs/handoff/`, `docs/crew.md`,
  `docs/harness.md`) · shell (`vault/tools/*` only)
- **justification:** context-offload (keeps the full PRD/system-design/QA-plan context out of every
  executor's packet — each executor gets only its one bounded packet)
- **Never:** lower a gate · renumber a task (retire with `cut` instead) · edit a Status cell (keeper
  only, R5) · fill in `**Build approval:**` · guess a fact the docs don't have (marks the task
  `blocked` with the missing fact instead)
- **Done when:** the plan and every claimed task's packet both pass their respective checkers, and
  the plan's execution view (§5) is the checker's own pasted output, not hand-maintained

<!-- No stack-specific agents added beyond the four above. A `schema-migrator` or
     `ui-smoke-runner` agent was considered and rejected — see Rejected agents below; at
     `team_size: 1` MVP scale the generic task-executor covers both backend and mobile write
     scopes without a repeated-spawn pattern narrow enough to justify a fifth agent yet. -->

## Rejected agents (and why)

- `schema-migrator` — rejected: no migration exists yet (`data-model.md` "Migration notes":
  greenfield build, no legacy data). `task-executor` covers the one-time schema creation inside
  `TASK-001`'s scaffold scope. Revisit only if/when a real migration (not initial creation) is
  needed post-MVP.
- `ui-smoke-runner` (a dedicated mobile-Detox-only executor) — rejected: at `team_size: 1` scale,
  `task-executor` already handles both `backend/` and `mobile/` write scopes across the plan's 14
  tasks; splitting by platform would be a privilege/context split with no repeated-spawn pattern
  distinct enough to justify a fifth agent. Revisit if the crew grows past solo.
- `scraper-ops-monitor` (a dedicated agent watching `ScrapeSource.status` for the Facebook
  ToS/scraping kill criterion) — rejected as a *build-time* crew agent: this is a production
  observability concern (`docs/ops.md` "Alerts & thresholds" already names the signal and paging
  path), not a code-authoring or gate-enforcement role during the build itself. Revisit at
  phase-5.6/ops handoff, not here.

## Materialisation

Read each agent's resolved `invocation` from `docs/.vault-work/roster.resolved.json` before
materialising — this run has no non-Claude model reachable, so every agent above is `invocation:
api`, materialised as `.claude/agents/<name>.md` with frontmatter `name`, `description`, `tools`,
`model` (the Claude model id named in that agent's binding above — `claude-sonnet-5` or
`claude-haiku-4-5-20251001` per `roster.json`'s `pool`); body: Purpose / Inputs / Outputs / Never /
Done-when, copied from the sections above. No `cli`-invocation dispatch block is needed in this
doc, because no external coding-agent binary is wired into this environment.

Re-materialise whenever this doc changes; never hand-edit generated `.claude/agents/*.md` files
directly — edit this doc and regenerate.

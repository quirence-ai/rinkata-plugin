---
name: rinkata-status-report
version: 0.1.0
description: |
  Produce a project status report from rinkata truth: counts, drift, blockers,
  in-flight specs, completion gaps, and suggested next actions. Read-only.
triggers:
  - /rinkata-status-report
  - status report
  - project pulse
tools:
  - rinkata_status
  - rinkata_doctor
  - rinkata_list_drift
mutating: false
---

# rinkata status-report — project pulse

Render a concise project pulse from the same Hub-first truth as
`/rinkata-orient`, but optimized for humans reviewing team health.

## Capability check

At startup, call `requireTools(["rinkata_status", "rinkata_doctor", "rinkata_list_drift"])` against the active MCP surface. If any tool is missing, render the returned parity-gap message, including any note id or alternate surface, and stop before reading status.

Include artifact counts, drift counts, doctor verdict, top stale/unknown items,
drafting specs that block work, tickets done without evidence, and a single next
operating action. Do not mutate state.

## Behavioral test contract

- Tool sequence: `requireTools` → `rinkata_status` + `rinkata_doctor` → optional
  `rinkata_list_drift` only when stale/unknown tickets need detail.
- Refusal cases: missing tools, Hub unreachable, schema-invalid status payloads,
  and doctor errors are surfaced as report blockers, not hidden.
- The report must distinguish source alignment from ticket lifecycle. Done
  tickets are reported with completion evidence status; they are never listed as
  startable work.

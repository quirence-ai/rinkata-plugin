# rinkata-reconcile

Reviewable updates for out-of-date Goal, spec, and ticket work. Proposal first,
human approval second, and a human applies the approved plan. Drift findings
(Hub: mismatches) are separate: `rinkata_list_drift_findings`.

**Default path (Goal/Spec):** `rinkata_propose_staged_reconcile` → human review →
a human runs `rinkata_apply_staged_reconcile` (Hub staged reconcile drawer, or CLI
`rinkata reconcile staged <target> --json` then
`staged-apply --plan … --approve-all --yes` on their own human identity).

**Classic path** (Goal-anchored tickets without specs): `rinkata_propose_reconcile`
→ `rinkata_reconcile_write` → a human runs `rinkata_reconcile_apply` (or applies from
the Hub UI Out of date drawer).

Apply is human-only: agent `pak_` keys and MCP OAuth sessions are refused with
`STAGED_APPLY_HUMAN_REQUIRED` / `RECONCILE_APPLY_HUMAN_REQUIRED` before any write.
The one exception: a `pak_` agent may staged-apply the plan of an accepted
Decision it wrote every version of, or logged or accepted while it is in effect and
only agents wrote it, except rows that archive work or reopen a done ticket
(`AGENT_APPLY_STEP_HUMAN_REQUIRED`).

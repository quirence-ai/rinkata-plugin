# rinkata-reconcile

Reviewable drift closure for Goal, spec, and ticket changes. Proposal first,
human approval second, and a human applies the approved plan.

**Default path (Goal/Spec):** `rinkata_propose_staged_reconcile` → human review →
a human runs `rinkata_apply_staged_reconcile` (Hub staged reconcile drawer, or CLI
`rinkata reconcile staged <target> --json` then
`staged-apply --plan … --approve-all --yes` on their own human identity).

**Classic path** (Goal-anchored tickets without specs): `rinkata_propose_reconcile`
→ `rinkata_reconcile_write` → a human runs `rinkata_reconcile_apply` (or applies from
the Hub UI Fix drift drawer).

Apply is human-only: agent `pak_` keys and MCP OAuth sessions are refused with
`STAGED_APPLY_HUMAN_REQUIRED` / `RECONCILE_APPLY_HUMAN_REQUIRED` before any write.

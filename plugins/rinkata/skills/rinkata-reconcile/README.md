# rinkata-reconcile

Reviewable drift closure for Goal, spec, and ticket changes. Proposal first,
human approval second, apply only for approved plans.

**Default path (Goal/Spec):** `rinkata_propose_staged_reconcile` → human review →
`rinkata_apply_staged_reconcile` (CLI `rinkata reconcile staged <target> --json`
then `staged-apply --plan … --approve-all --yes`).

**Classic path** (Goal-anchored tickets without specs): `rinkata_propose_reconcile`
→ `rinkata_reconcile_write` → human-gated `rinkata_reconcile_apply`.

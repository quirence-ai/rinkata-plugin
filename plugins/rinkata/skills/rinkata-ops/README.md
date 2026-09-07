# rinkata-ops

The always-on rinkata skill. Activates whenever the user mentions rinkata workflows
(preflight, reconcile, drift, doctor, ticket IDs).

## What this does

Encodes rinkata's agent rules — the same rules in host agent rule files — as a discoverable skill the agent's harness loads automatically. Without this, agents only follow rinkata's rules when they happen to read host agent rule files.

## When it fires

- User mentions: "preflight", "reconcile", "drift", "doctor", or any `TICK-*` ID

## What it teaches

- Use the rinkata MCP tools (`rinkata_status`, `rinkata_preflight_ticket`,...) for Hub artifact reads and writes
- Never read or write repo-local rinkata artifact directories
- Run preflight before ticket work; treat `STALE_ARTIFACT` as a blocker, not a hint
- **Claim vs Start** are distinct: Claim is assignee-only; Start is ticket official
  `rinkata_claim` / `/rinkata-ticket-start`. Parent fill lives only on the server —
  skills report `filledParents` and never reimplement sole-branch rules
- Prefer staged Goal/Spec drift: `rinkata_propose_staged_reconcile` then human-gated
  `rinkata_apply_staged_reconcile` (CLI `reconcile staged` / `staged-apply`). Classic
  `rinkata_reconcile_write` / `rinkata_reconcile_apply` only for Goal-anchored tickets
  without specs — surface plans; never apply without approval
- Preserve protected human notes verbatim

## How to test it

Install this plugin from the Cursor Marketplace (or symlink locally as in the repo README) and connect Hub MCP with OAuth.
In any rinkata-backed project, ask the agent: "What's the status of TICK-123?" Verify the agent calls `rinkata_status` first instead of reading local files.

## Related

- `conventions/stale-hash-protocol.md` — Convention cited by Phase 2
- `conventions/human-note-protection.md` — Convention cited by Phase 3

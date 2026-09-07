---
name: rinkata-reconcile
version: 0.2.2
description: |
  Review and close drift between Goals, specs, and tickets. Uses rinkata reconcile
  tools to propose reviewable plans, requires human approval before apply, and
  treats stale graphs as blockers. Default path for Goal/Spec drift is staged
  propose/apply (not Hub-only).
triggers:
  - /rinkata-reconcile
  - reconcile
  - drift
tools:
  - rinkata_status
  - rinkata_list_drift
  - rinkata_propose_staged_reconcile
  - rinkata_apply_staged_reconcile
  - rinkata_propose_reconcile
  - rinkata_reconcile_write
  - rinkata_reconcile_apply
  - rinkata_propose_from_decision
mutating: true
argument-hint: <target>
---

# rinkata reconcile — reviewable drift closure

Hub is the only source of rinkata truth. Read status/drift and write plans through
Hub MCP; local files are not authoritative and must not be used as a fallback.

## Capability check

At startup, call
`requireTools(["rinkata_status", "rinkata_list_drift", "rinkata_propose_staged_reconcile", "rinkata_propose_reconcile"])`
against the active MCP surface and render its parity-gap message if any tool is
missing, then stop before reading drift. For apply, require the matching Hub
apply tool before mutating:

- **Staged (default for Goal/Spec drift):** `rinkata_apply_staged_reconcile`
- **Classic (Goal-anchored tickets without specs / legacy):** `rinkata_reconcile_write` and
  `rinkata_reconcile_apply`

If only retired local reconcile aliases are available, stop and surface the
parity problem; do not fall back to local files.

## What to do

1. Call `rinkata_status` and `rinkata_list_drift`; if no drift exists, stop.
2. **Prefer staged when the drift target is a Goal path or a spec path:**
   - Call `rinkata_propose_staged_reconcile({ target })` (Goal path → stage 1
     Goal→Spec; spec path → stage 2 Spec→Ticket).
   - If the response is `truncated: true`, re-fetch with a high enough `limit` (or
     page with `offset` and reassemble) so `actions.length === summary.totalActions`
     before any apply. A truncated page is refused at apply.
   - Summarize CREATE, UPDATE, ARCHIVE, ABSORB, and DEFER actions for human review.
   - **Post-mint spec edit:** when member tickets already have
     file-level plans, default the proposed verb to **ABSORB** (hash bump, no
     ticket-body rewrite). **UPDATE** only when a ticket body is actually wrong.
     Do not default to UPDATE that appends useless “Upstream” notes. Apply
     remains human-gated — this is the proposed verb, not silent absorb.
   - Call `rinkata_apply_staged_reconcile({ plan })` **only** after the user
     explicitly approved that concrete plan (per-action `approved` flags; include
     `bodyRevision` / `downstreamPolicy` when required). CLI:
     `rinkata reconcile staged <target> --json` then
     `rinkata reconcile staged-apply --plan <file> --approve-all --yes`
     (or set per-action `approved:true` without `--approve-all`).
3. **Classic path** (Goal-anchored tickets without specs, or legacy ticket-level
   drift only):
   - Call `rinkata_propose_reconcile`.
   - Call `rinkata_reconcile_write` only after the user has seen the proposal.
   - Call `rinkata_reconcile_apply` only when the user explicitly approved that
     specific plan.
4. **Decision propagate:** after `rinkata_propose_from_decision` (CLI
   `rinkata decision-propagate`), review the returned staged plan(s) and apply
   with `rinkata_apply_staged_reconcile` / `rinkata reconcile staged-apply`.
   Decision-driven staged apply **clears `pending_propagation`** server-side so
   the DEC can leave the Needs you queue. Agents propose; humans apply.

Never silently absorb source-hash drift or rewrite protected human notes.

## Human-gated apply

Apply is always human-gated. Agents may propose and draft; they must not apply
without explicit approval of the concrete plan in this session. On
`RECONCILE_GRAPH_STALE`, re-propose and re-review — never bypass.

## Behavioral test contract

- **Staged sequence (default Goal/Spec):** `requireTools` → `rinkata_status` →
  `rinkata_list_drift` → `rinkata_propose_staged_reconcile` → human review →
  complete plan (no truncated page) → explicit approval →
  `rinkata_apply_staged_reconcile`.
- **Classic sequence (legacy / no specs):** `requireTools` → `rinkata_status` →
  `rinkata_list_drift` → `rinkata_propose_reconcile` → human review →
  `rinkata_reconcile_write` → explicit approval → `rinkata_reconcile_apply`.
- **Decision-driven:** `rinkata_propose_from_decision` → human review →
  `rinkata_apply_staged_reconcile` (clears `pending_propagation`).
- Refusal cases: no drift, missing tools, Hub unreachable, stale graph,
  truncated plan page, no human approval all stop before apply.
- Protected human notes must be preserved verbatim; if a plan would rewrite
  protected prose, surface that as a blocker rather than applying it.
- After tickets already have file-level plans, prefer proposing ABSORB over
  UPDATE unless the ticket body is actually wrong.

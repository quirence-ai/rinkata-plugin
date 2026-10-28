---
name: rinkata-reconcile
version: 0.2.10
description: 'Review and update out-of-date work between Goals, specs, and tickets. Uses rinkata reconcile tools to propose reviewable plans, hands each plan to a human to apply, and treats stale graphs as blockers. Default path for out-of-date Goal/Spec work is staged propose then human apply (not Hub-only). Not for drift findings (Hub: mismatches).'
triggers:
  - /rinkata-reconcile
  - reconcile
  - out of date
tools:
  - rinkata_status
  - rinkata_list_drift
  - rinkata_propose_staged_reconcile
  - rinkata_draft_review_rewrite
  - rinkata_read_drift_proposal
  - rinkata_apply_staged_reconcile
  - rinkata_propose_reconcile
  - rinkata_reconcile_write
  - rinkata_reconcile_apply
  - rinkata_propose_from_decision
mutating: true
argument-hint: <target>
---

# rinkata reconcile — reviewable out-of-date updates

Hub is the only source of rinkata truth. Read status/out-of-date work and write plans through
Hub MCP; local files are not authoritative and must not be used as a fallback.
Drift findings (Hub: mismatches) are a different thing: use
`rinkata_list_drift_findings` / `rinkata_resolve_drift`, not this skill.

## Capability check

At startup, call
`requireTools(["rinkata_status", "rinkata_list_drift", "rinkata_propose_staged_reconcile", "rinkata_read_drift_proposal", "rinkata_propose_reconcile"])`
against the active MCP surface and render its parity-gap message if any tool is
missing, then stop before reading out-of-date work. Apply is for a person (one
exception, under Human-only apply); name the tool the human will use when you
hand the plan over:

- **Staged (default for out-of-date Goal/Spec work):** `rinkata_apply_staged_reconcile`
- **Classic (Goal-anchored tickets without specs / legacy):** `rinkata_reconcile_write` and
  `rinkata_reconcile_apply`

If only retired local reconcile aliases are available, stop and surface the
parity problem; do not fall back to local files.

## What to do

1. Call `rinkata_status` and `rinkata_list_drift`; if nothing is out of date, stop.
2. **Prefer staged when the out-of-date target is a Goal path or a spec path:**
   - Call `rinkata_propose_staged_reconcile({ target })` (Goal path → stage 1
     Goal→Spec; spec path → stage 2 Spec→Ticket).
   - The response includes `proposalId`, `revision`, and `reviewUrl`. The stored
     revision has every action. Pagination only shortens this read. Share
     `reviewUrl` with the human. Do not hand them a page of actions to apply.
   - To draft an update onto an Update row, call
     `rinkata_draft_review_rewrite({ proposalId, artifactId, bodyRevision })`.
     Each call writes a new immutable revision (`draftedBy`, `draftedAt`,
     `basis {ticketVersion, specVersion}`). It does not apply. If the ticket or
     spec version moved, apply refuses `DRAFT_OUT_OF_DATE` — draft again.
   - After the human applies, call `rinkata_read_drift_proposal({ proposalId })`.
     `lifecycle` is `open`, `partially_applied`, or `applied`. `applier` is the
     human who applied. Do not retry apply.
   - Summarize CREATE, UPDATE, ARCHIVE, ABSORB, and DEFER actions for human review.
   - **Post-mint spec edit:** when member tickets already have
     file-level plans, default the proposed verb to **ABSORB** (hash bump, no
     ticket-body update). **UPDATE** only when a ticket body is actually wrong.
     Do not default to UPDATE that appends useless “Upstream” notes. Apply
     remains human-gated — this is the proposed verb, not silent absorb.
   - Hand the plan to a human to apply, once they have explicitly approved that
     concrete plan (per-action `approved` flags; include `bodyRevision` /
     `downstreamPolicy` when required). They apply it from the Hub UI staged
     reconcile drawer, or with `rinkata_apply_staged_reconcile({ plan })` /
     `rinkata reconcile staged <target> --json` then
     `rinkata reconcile staged-apply --plan <file> --approve-all --yes` on their
     own human identity (or per-action `approved:true` without `--approve-all`).
     Agent `pak_` keys and MCP OAuth are refused with
     `STAGED_APPLY_HUMAN_REQUIRED` before any write — do not retry.
3. **Classic path** (Goal-anchored tickets without specs, or legacy out-of-date
   tickets only):
   - Call `rinkata_propose_reconcile`.
   - Call `rinkata_reconcile_write` only after the user has seen the proposal.
   - Hand the written plan to a human to apply, once they explicitly approved
     that specific plan: Hub UI Out of date drawer, or `rinkata_reconcile_apply`
     with the `planId` on their own human identity. Agent keys and MCP OAuth
     are refused with `RECONCILE_APPLY_HUMAN_REQUIRED`.
4. **Decision propagate:** after `rinkata_propose_from_decision` (CLI
   `rinkata decision-propagate`), review the returned staged plan(s) and hand
   them to a human, who applies with `rinkata_apply_staged_reconcile` /
   `rinkata reconcile staged-apply`. If you wrote every version of the Decision, or logged or accepted it while it is in
   effect and only agents wrote it, and it is accepted, you may apply its plan yourself (see Human-only apply).
   Decision-driven staged apply **clears `pending_propagation`** once no decision work is left, so
   the DEC can leave the Needs you queue.

Never silently absorb a source-hash change or rewrite protected human notes.

## Human-only apply

Apply is human-only. Agents propose and draft, then hand the concrete plan to a
human — the server refuses agent `pak_` keys and MCP OAuth sessions with
`STAGED_APPLY_HUMAN_REQUIRED` (staged) and `RECONCILE_APPLY_HUMAN_REQUIRED`
(classic) before any write. The human applies from the Hub UI drawer or on their
own human identity. On `RECONCILE_GRAPH_STALE`, re-propose and re-review —
never bypass.

One exception: a `pak_` agent may run
`rinkata_apply_staged_reconcile` on the `rinkata_propose_from_decision` plan of
an accepted Decision it wrote every version of, or logged or accepted while it is in
effect and only agents wrote it. A row that archives work or reopens a done
ticket refuses the apply (`AGENT_APPLY_STEP_HUMAN_REQUIRED`): leave it
unapproved for a person. Classic `rinkata_reconcile_apply` stays person-only.

## Behavioral test contract

- **Staged sequence (default Goal/Spec):** `requireTools` → `rinkata_status` →
  `rinkata_list_drift` → `rinkata_propose_staged_reconcile` → share `reviewUrl` →
  a human applies that revision → `rinkata_read_drift_proposal`. Do not hand a
  truncated page to apply.
- **Classic sequence (legacy / no specs):** `requireTools` → `rinkata_status` →
  `rinkata_list_drift` → `rinkata_propose_reconcile` → human review →
  `rinkata_reconcile_write` → explicit approval → hand off → a human runs
  `rinkata_reconcile_apply`.
- **Decision-driven:** `rinkata_propose_from_decision` → human review → a human
  (or an agent that wrote every version of the accepted Decision, or logged or accepted it while in effect and only agents wrote it) runs `rinkata_apply_staged_reconcile` (clears `pending_propagation` once no decision work is left).
- Refusal cases: nothing out of date, missing tools, Hub unreachable, stale graph,
  truncated plan page, no human approval all stop before apply.
- Protected human notes must be preserved verbatim; if a plan would rewrite
  protected prose, surface that as a blocker rather than applying it.
- After tickets already have file-level plans, prefer proposing ABSORB over
  UPDATE unless the ticket body is actually wrong.

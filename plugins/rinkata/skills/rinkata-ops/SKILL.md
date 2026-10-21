---
name: rinkata-ops
version: 0.2.9
description: 'rinkata is a Hub source-of-truth consistency engine. Hub DB is canonical for Goals, specs, tickets, decisions, reconcile artifacts, inbox notes, handoffs, and completion evidence. Installed MCP tools, the CLI, and the Hub UI are the access paths; repo-local artifact directories are not a rinkata surface. Use this skill whenever the user mentions rinkata projects, tickets, preflight, reconcile, drift, doctor, ticket IDs like TICK-*, Needs you, or Decisions. Also load mid-session to mint a proposed Decision (judgment call / log a decision) then list needs_action — do not wait for a slash command.'
triggers:
  - rinkata
  - preflight
  - reconcile
  - drift
  - doctor
  - TICK-*
  - decision
  - Needs you
  - "proposed Decision"
  - "log a decision"
  - "this needs a judgment"
tools:
  - rinkata_status
  - rinkata_list_drift
  - rinkata_propose_reconcile
  - rinkata_propose_staged_reconcile
  - rinkata_propose_from_decision
  - rinkata_preflight_ticket
  - rinkata_work_queue
  - rinkata_impact
  - rinkata_doctor
  - rinkata_inbox_list
  - rinkata_inbox_add
  - rinkata_inbox_resolve
  - rinkata_inbox_set_state
  - rinkata_inbox_promote
  - rinkata_log_decision
  - rinkata_update_decision
  - rinkata_list_decisions
  - rinkata_accept_decision
  - rinkata_reconcile_write
  - rinkata_reconcile_apply
  - rinkata_apply_staged_reconcile
  - rinkata_complete_ticket
  - rinkata_knowledge_list
  - rinkata_knowledge_read
  - rinkata_knowledge_ask
  - rinkata_knowledge_record
  - rinkata_knowledge_upload
  - rinkata_begin_demo_screenshot_upload
  - rinkata_finalize_demo_screenshot_upload
  - rinkata_upload_demo_screenshot
  - rinkata_promote_idea
  - rinkata_plan_from_spec
  - rinkata_claim
  - rinkata_set_ticket_status
  - rinkata_update_ticket
  - rinkata_link_requirements
  - rinkata_supersede_tickets
  - rinkata_list_specs
  - rinkata_read_spec
  - rinkata_write_spec_body
  - rinkata_write_ticket_body
  - rinkata_set_spec_acceptance
  - rinkata_set_spec_status
  - rinkata_set_spec_source
  - rinkata_archive_spec
  - rinkata_complete_spec
  - rinkata_write_handoff
  - rinkata_list_handoffs
  - rinkata_search
  - rinkata_read_handoff
  - rinkata_set_handoff_state
  - rinkata_amend_handoff
mutating: true
writes_to:
  - Hub inbox artifacts
  - Hub decision artifacts
  - Hub reconcile artifacts
  - Hub ticket artifacts
  - Hub spec artifacts
  - Hub handoff artifacts
  - Hub knowledge documents
metadata:
  grok:
    notes: "Native host skill discovery + MCP discovery. Agentic patterns below apply across hosts."
---

# rinkata Operations — Hub Truth Contract

Hub DB owns project docs: Goals, specs, tickets, decisions, inbox
notes, reconcile plans, handoffs, and completion evidence. Installed rinkata MCP
tools, the CLI, and the Hub UI are the only supported agent surfaces. Agents
must not look for, read, create, or edit repo-local artifact directories. Every
artifact has a Hub source hash pinning it to upstream truth; drift is closed via
reviewable reconcile.

**Goals** → optional **specs** → **tickets**. Prefer specs for multi-ticket Goal sections; direct Goal tickets remain valid.

## Contract

This skill guarantees:

- rinkata artifact reads and writes go through Hub MCP or the Hub UI, never
  repo-local artifact files.
- Unavailable Hub truth is a blocker; never fall back to local artifacts.
- `rinkata_status` is consulted before any work; preflight runs before tickets.
- After idea/Goal/spec/ticket reads, honor `nextHop` (`tool`, `mode`, `reason`) when present. Load the owning ceremony skill; do not invent a parallel write and do not auto-execute the hop. Write-time `nextAction` and `work_queue.claimAction` are unchanged.
- Alignment drift is advisory for Start and complete. Never overwrite source hashes directly; stale expectedVersion remains a write conflict.
- Reconcile plans are reviewable: agents propose, a human applies.
- Preserve protected human notes.

## Official ceremonies

Name the **official tool**. Load the owning skill for the full flow. Do **not**
reimplement the skill or invent a parallel write. An ops-only session that was
never told `/idea` still uses these tools. Ceremony skills also declare
**mid-session triggers** (not only slash commands): idea still on Ideas /
successor of a complete spec from an idea → `rinkata-idea`; picking up / starting
/ claiming a ticket → `rinkata-ticket-start`; shipped work →
`rinkata-ticket-complete`; record what we now know → same skill /
`rinkata_knowledge_record` (do not complete); judgment / proposed Decision →
this skill (`rinkata_log_decision` then `needs_action`).

| When | Official tool | Owning skill |
|------|---------------|--------------|
| User confirms an idea destination (including successor of a complete spec) | `rinkata_promote_idea` | `rinkata-idea` |
| Take ownership vs start ticket work | `rinkata_claim` (`start: false` vs default `start: true`) | `rinkata-ticket-start` |
| Close shipped work | `rinkata_complete_ticket` + `knowledgeWriteBack` | `rinkata-ticket-complete` |
| Record what we now know (knowledge-only) | `rinkata_knowledge_record` (do not complete) | `rinkata-ticket-complete` |
| Mint a **proposed** Decision | `rinkata_log_decision` then `rinkata_list_decisions({ status: "needs_action" })` | this skill (Decision-first Needs you) |

Do **not** pick `create_standalone_spec` + `originIdeaId` instead of
`rinkata_promote_idea` when the user confirmed an idea destination.

## Decision-first Needs you

**Needs you = Decision queue**. Doctor / Reconcile / Ready are separate.

Flow: judgment → mint **proposed** DEC → human **Accept** (one click: seal +
ratify). The Decision is source of truth. It **stays on Needs you** until every
update to its work has applied; unset options or a failed seal also keep it
there. A DEFER left for later, stage-2 ticket drift or a complete-spec refuse
does not keep `pending_propagation`.

- **Mint:** `rinkata_log_decision` + `affects[]`. An agent default is stored `proposed` with `in_effect_at`: binding, not accepted. Explicit `status: proposed` records a plain proposal. Agent in-effect decisions stamp no `pending_propagation` and never authorize `propose_from_decision`, staged apply or dual unlock; only a person accepts/ratifies. `affects[0]` is the covering Goal this Decision is *for* (usually the newest live Goal); older Goals, specs, and tickets after. Ticket-only lists cannot Goal-group — include the covering Goal first when they should. Do not sort by `created_at`. On success for **proposed**, expect `nextAction: "list_needs_action"` — immediately list residual with `rinkata_list_decisions({ status: "needs_action" })` and surface the Needs you path; do not stop at a bare create ack.
- **Scope-closed:** binding DECs cover only `affects[]` + `overrides[]`. Outside that scope, propose a DEC; never extrapolate. Overrides require the verbatim artifact clause.
- **List:** `rinkata_list_decisions({ status: "needs_action" })` (alias `actionable`) — same oracle as Hub Needs you.
- **Accept:** Hub primary CTA is **Accept**. Human seal (`accepted` + durable ratify). Pending clears once every update has applied (an explicit `intent: remove` also waits for DEFER and ticket drift). Mechanical ABSORB / Goal→Spec CREATE may run in the same request.
- **Stamp:** Hub Accept / human `rinkata_accept_decision` stamps `ratified_at`+`ratified_by`, accepts proposals and returns `status`. Human sessions only; agents get `RATIFY_HUMAN_REQUIRED`, generic writes/booleans `RATIFY_FORGE_REFUSED`. Retries keep the original seal, validate options and accept the current body; read first. Merge only stamps when `ratification_pr` matches; never accepts. `decision_accept` audits with the write.
- **Retire:** confirm-gated ARCHIVE (`intent: remove`). Never from free-form Decision body. Never auto on Accept. **Reject** (proposed only) is lifecycle-only. After a successful Accept the next hop is not **Update guidance** / **Check residual** / **Apply residual** / **Clear pending**. Absorb failure is **Retry**. Technical tools: `rinkata_propose_from_decision` with explicit `intent: remove|rescope` (default auto = rescope) then human-gated staged apply. Agents propose, humans Accept — never agent-forge apply.
- **Supersede:** `status: superseded` needs durable `supersededBy` (not prose-only).
- **Dual unlock (S3+S5):** evidence-bound/complete SPEC rewrites refuse `SUCCESSOR_REQUIRED` — prefer successor. After **human-ratified** DEC, retry `rinkata_write_spec_body` / `rinkata_set_spec_acceptance` with **`decisionId`**. Claimed `decisionRatified` / `authorize:true` never unlock.
- **Reconnect MCP** after upgrade: `rinkata_status` counts ≠ tool surface. Missing `rinkata_accept_decision` / `needs_action` / `decisionId` = stale process.

## Cross-artifact search / duplicate prevention

**Before create**, call `rinkata_search` with distinctive terms; prefer extending a covering **open** artifact over a parallel one. **Exception:** if a high-score peer is `derivedState: complete`, do **not** extend by stacking — use stop-and-successor (below).

- Create-time `relatedGoals` / `relatedSpecs` / `relatedTickets` is **stop-and-read** (advisory; create still succeeds, including `relatedStatus: degraded` + `RELATED_SEARCH_FAILED`). When `relatedGuidance` is present, follow stop-and-successor — never stack tickets under those SPECs.
- First **spec-anchored** feature (create or `plan_from_spec`) uses peer set. Approve has no similar-spec gate. `peerDisposition: { decision: "independent"|"fold", peerIds, note≥20 }`. `independent` only when truly separate; `fold` extends **open** survivors (never complete) and is `PEER_FOLD_REQUIRED` on approve (fold into the peer instead). `independent` and omitted are ignored on approve. `PEER_SEARCH_UNAVAILABLE`: retry; do not invent ids (first feature still refuses).
- **Agent terminal disposes peers:** after create, read `relatedSpecs` + `dispositionRecommend` / `dispositionGuidance` (fail-open). Prefer stop-and-extend or successor. On `PEER_DISPOSITION_REQUIRED`, retry **the same tool** with `peerDisposition`. Hub is backup only.
- Goal-section-anchored features (`source:` only) are not disposition-gated in v1 — still search first. Handoffs use `rinkata_list_handoffs({query})`, not `rinkata_search`.

**peerRisk (advisory):** `rinkata_read_spec` / `rinkata_preflight_ticket` may attach `status: clear | peers | degraded` plus capped peers (`derivedState`, guidance). It does not change the preflight verdict or block a read or approve. Use it before the first-feature gate.

**Completed specs:** adding a feature under a complete spec
reopens it visibly, recording `reopen_audit` and a history/change-event intent
`reopen` naming the ticket. An archived spec refuses new work and body/criterion edits, including ticks
(`SPEC_ARCHIVED`). A person must Restore it to drafting before edits; a
`decisionId` never unlocks archived content. New ideas or work may reference it.
A soft-stack beside a complete covering spec still refuses; for a new contract
prefer a successor. From an idea use `rinkata_promote_idea` with
`destination: "standalone_spec"` and `predecessor`; without an idea use
`rinkata_create_successor_spec`. Re-promotion is idempotent (`IDEA_TARGET_EXISTS`);
never create a successor then comment on the idea. The old reopen approval tool
is retired (`COMPLETED_SPEC_REOPEN_RETIRED`), not a recovery path.

**Fold (`PEER_FOLD_REQUIRED`):** stop shipping under the duplicate → read `survivingPeerIds` → extend peers → archive/abandon duplicate → continue only under survivors.

### Re-home work when ownership moves

Do **not** N× bare `rinkata_set_ticket_status(archived)` for ownership moves (looks hostile; silent audit).

1. Log/accept a DEC with losing + winning ids in `affects[]`.
2. Create/identify survivors under the winning spec.
3. SUPERSEDE-archive losers: prefer `rinkata_supersede_tickets({ ticketIds, supersededBy, reason, decisionId? })`; or single-ticket `rinkata_set_ticket_status` with `status: "archived"` + `reason` (**required** when `supersededBy` set).
4. Read back: `archived_reason`, `superseded_by`, optional `archived_decision`, protected `### Superseded` / `### Archived` block.

Bare multi-archive without `reason`/`supersededBy` is discouraged for de-dup. `rinkata_set_ticket_spec` only re-anchors *unanchored* tickets (`TICKET_ALREADY_ANCHORED` on steal). Spec retirement: `rinkata_archive_spec` + explicit `downstream`. Decision **remove** residual (ARCHIVE covering specs) requires explicit `intent: remove` on `rinkata_propose_from_decision` / Hub **Retire covering work…** — never free-form body wording, never Accept & align auto.

Codes: `RELATED_SEARCH_FAILED`, `PEER_SEARCH_UNAVAILABLE`, `PEER_DISPOSITION_REQUIRED`, `PEER_DISPOSITION_INVALID`, `PEER_FOLD_REQUIRED`, `SPEC_ARCHIVED`, `FORBIDDEN`, `CONFLICT`.

## Conventions

> **Convention:** See `conventions/stale-hash-protocol.md` for stale-hash handling.
> **Convention:** See `conventions/human-note-protection.md` for protected-note rules.

## Phases

### Phase 1 — Orient (every session, before any work)

1. `rinkata_status` — Goals, **specs** (`specsInFlight`/`specsComplete`), tickets, decisions, inbox, reconciles, drift.
2. `rinkata_doctor` — schema + journal + protected notes; `concentrated-prd-section` / `implicit-spec-membership` surface sections to lift under a spec. **Default compact**: one `protected-sentinels` summary ok + non-ok only — skip `verbose: true` unless a human needs per-ticket ok rows.
3. If drift > 0, surface before proposing changes.

Unclear tool gates/authority → `rinkata_tool_contracts({ name })`. Do not infer authority from prose or same-call booleans.

After orient, use kind-specific list tools (same filters as Hub Views; CLI: `rinkata view <kind>`):

- `rinkata_list_tickets({status, goalPath, assignee, limit})` — default `status="open"`; `assignee="me"`; `kind: "bugs"` for bugs only (`prdPath` deprecated).
- `rinkata_list_goals({status, limit})` — default `status="all"`.
- `rinkata_list_decisions({status, limit})` — use `status: "needs_action"` / `actionable` for Needs you.
- `rinkata_list_specs({goal, status, includeArchived})` (`prd` deprecated alias for `goal`).
- `rinkata_inbox_list({})`, `rinkata_list_handoffs({state, about, query, limit})`.
- `rinkata_search({query, kinds?, statusByKind?, goal?, limit?, offset?})` — prefer before create.
- `rinkata_knowledge_list({areaId?})` — customer-named areas + documents (`areaId: "unfiled"` for Unfiled). Then `rinkata_knowledge_read` / `rinkata_knowledge_ask`. Upload files with `rinkata_knowledge_upload` (not markdown notes — those stay `rinkata_knowledge_record`). Do not invent area names.

**Bounded reads:** `rinkata_work_queue`, `rinkata_list_drift`, `rinkata_propose_reconcile`, and `rinkata_propose_staged_reconcile` return at most 25 items in `{ total, returned, offset, truncated }`. If `truncated: true`, page with `offset`/`limit` or filters — never infer "empty / no drift / nothing to reconcile" from one partial page. Staged apply requires a complete action list (`actions.length === summary.totalActions`); re-fetch before handing the plan to a human to apply.

### Phase 2 — Advisory preflight and direct ticket operations

**Claim vs Start:** use `/rinkata-ticket-start` for ownership plus official Start.
Skills do **not** reimplement parent-fill or sole-branch rules.
Parent fill is **Start-only**; Claim never parent-fills.
Ready plan_from_spec mint stubs are `claimAction` none (`claim` | `start` | `none`); enrich before Start.

1. Read `rinkata_preflight_ticket` for alignment, dependencies and optional
   `knowledgeCovering`; read covering Knowledge before inventing context.
   Preflight is advisory: stale/unknown alignment, drafting specs and unsatisfied
   dependencies do not block Start or complete. Surface discrepancies and continue.
   `TICKET_ALREADY_CLOSED`, a person's `PAUSED`, unreadable requirement truth
   (`RECOVERY_HOLD`) and `PARENT_NOT_FOUND` (subtasks only) remain server refusals, along with access, attribution and expectedVersion integrity.
2. `rinkata_update_ticket` requires `expectedVersion` and edits status (`todo`,
   `in_progress`, `blocked`), title or assignee. Leaving done reopens the ticket
   and keeps `completed_at` and `completion_evidence`. Only complete sets done;
   archive and supersede keep their dedicated paths.
3. `rinkata_claim` takes ownership: `start: false` assigns only; default
   `start: true` also Starts through the direct update. Goal/spec Claim is
   assignee-only. Start reports server `filledParents` (kind + id or `[]`);
   never invent fills or reimplement sole-branch rules. Claim protects another
   owner's todo/blocked ticket and refuses closed tickets. Use direct update
   for an authorized Start that keeps the existing owner.
4. `rinkata_link_requirements` adds/removes links to approved criteria in the
   ticket's own spec, `realign` moves existing links only, and `acknowledge: true`
   acknowledges an unlinked ticket at the current requirement state. New tickets
   under checks-on specs are born acknowledged. Generic saves cannot change
   these fields. Review changes is not required to realign a ticket.
5. Enrich plan_from_spec mint stubs with `rinkata_write_ticket_body` before
   Start (read version, preserve fingerprint + protected regions). Discovery's
   `claimAction` helps choose Claim or Start; server holds remain authoritative.

### Readiness / blocker chains — agent-first

**Do not** send users to a Hub dependency map for blocks/blocked_by shape.
Agents already own readiness visibility:

| Need | Tool |
|------|------|
| This ticket blocked / safe? | `rinkata_preflight_ticket` — open blockers, stale source |
| Queue shape (blocked / ready / in progress) | `rinkata_work_queue` — filter `readiness` / `lifecycleStatus`; page if `truncated` |
| Blast radius if Goal/Spec/Ticket moves | `rinkata_impact` (CLI `rinkata impact`) — read-only multi-hop |
| Cycle / missing-referent / inverse edges | `rinkata_doctor` |

Record ticket waits with `rinkata_update_ticket_dependencies` (add/remove `blocked_by` in one call). Spec `sequence` is recommended order only. Put a Goal or spec in a batch rather than tagging each ticket. Search existing batches (`rinkata_list_batches`) before `rinkata_create_batch` — a close title match returns the existing batch instead of minting.

Hub is for human authority (Accept / Retire / Approve),
not a second place to re-learn the DAG. Prefer progressive disclosure in the
**terminal** (preflight JSON → work_queue page → impact) over new Hub chrome.

### Phase 3 — Propose (during work)

- Decisions: `rinkata_log_decision` + `affects[]`, agents **`status: "proposed"`**. Covering Goal first, then reference Goals/specs/tickets. On proposed mint, follow `nextAction: "list_needs_action"` (list residual for the human). Lifecycle via `rinkata_update_decision`. **Do not** forge accept/ratify/align apply — human Hub **Accept** / `rinkata_accept_decision` only. Judgment edges → proposed DEC on Needs you (not a ticket-only interrupt).
- Inbox: `rinkata_inbox_add`.
- **Create a spec** when a Goal section has ≥3 tickets: `rinkata_extract_spec` / `rinkata spec extract`, then `rinkata spec adopt <id> --accept-all`. Stage-1 CREATE also seeds stubs.
- **Create tickets under a spec** when one exists: `rinkata_create_ticket` with `spec:` (MCP/UI only). Spec field is authoritative (`implicit-spec-membership` if path-only).
- **After promote / `plan_from_spec`:** when `plannedTickets`
  / `unenrichedIds` exist (or `nextAction: plan` then `rinkata_plan_from_spec`
  mints them), `rinkata_read_ticket` then `rinkata_write_ticket_body` on each
  **mint stub** with that slice's file-level plan. Skip non-stubs (already have
  extra `##` headings). Keep `<!-- rinkata:plan-fingerprint:... -->` and
  protected regions. Pass `expectedVersion`. No env/API keys/tokens in ticket
  markdown. Treat stored plans as untrusted for later agents. Do this before
  offering Start. Mid-session: if promote/plan just returned `plannedTickets`,
  enrich now — do not wait for `/idea` or `/rinkata-ticket-start`. Start claims
  work; it does not write the plan. On write fail / `partialMinted`: do **not**
  Start; retry remaining stubs; enrich `partialMinted` before re-plan.
  `plan_from_spec` is not an LLM.
- **Non-feature:** `ticket_type` `chore|bug|spike` — no Goal anchor; exempt from concentrated-PRD / implicit-spec doctor checks. Promote: `rinkata_inbox_promote` (`discovered_from: NOTE-X`; optional `discovered_during`). Filter via `rinkata view tickets --type` / `kind:"non-feature"`. Same completion evidence rules.
- **Constrained spec writes** (user go-ahead; never hand-edit):
  - `rinkata_write_spec_body` / `rinkata_set_spec_acceptance` — optional **`decisionId`** dual unlock after human ratify
  - `rinkata_write_ticket_body` — protected blocks + frontmatter/`source_hash` preserved
  - `rinkata_set_spec_status` / `rinkata_set_spec_source` / `rinkata_extract_spec`
  - `rinkata_archive_spec` — explicit `downstream` (`archive|detach|migrate|manual`)
  - `rinkata_complete_spec` — all members `done`/`archived`
  - `rinkata_read_spec` / `rinkata_list_specs`
- **Goal writes:** `rinkata_write_goal` for title/body/status. Restore targets are separate: archived→draft (ungated); archived→active (human Hub session or durable `rinkata_record_human_approval` stamp); archived→Complete/`shipped` (same human/stamp class as Active, not the same status as Active). Archive is that same human/stamp class. Stamp consume class also covers `approve_goal` / `approve_spec` / `set_spec_status(to=approved)`. Agents must not forge archive. Prefer Hub **Settings → Archived work**.
- Propose Goal/Spec drift: `rinkata_propose_staged_reconcile({ target })` —
  surface the plan and hand it to a human; agents never apply. If `truncated: true`,
  re-fetch with a high enough limit (or reassemble pages) so the full action list
  is present before it goes to a human.
- Propose legacy ticket-level drift (no specs): `rinkata_propose_reconcile` /
  `rinkata_reconcile_write` — surface the plan and hand it to a human.
- Decision fan-out: `rinkata_propose_from_decision` then the human apply path below.

### Phase 4 — Hand off for apply (human-only)

- **Staged (default for Goal/Spec):** hand the reviewed plan to a human, who
  applies it from the Hub UI staged reconcile drawer or with
  `rinkata_apply_staged_reconcile({ plan })` / `rinkata reconcile staged-apply`
  on their own human identity (per-action `approved`). Agent `pak_` keys and
  MCP OAuth are refused with `STAGED_APPLY_HUMAN_REQUIRED` before any write —
  hand the plan over, do not retry. On `RECONCILE_GRAPH_STALE`, re-propose and
  re-review — never bypass.
- **Classic:** hand the plan to a human, who applies it from the Hub UI Fix
  drift drawer or with `rinkata_reconcile_apply` on their own human identity.
  Agent keys and MCP OAuth are refused with `RECONCILE_APPLY_HUMAN_REQUIRED`.

### Phase 5 — Close with optional evidence (when work has shipped)

- `rinkata_complete_ticket(ticketId, evidence?)` (CLI: `rinkata complete`) — never hand-edit `status: done` / `completed_at` / `completion_evidence`. Evidence is optional; attach a real PR URL, commit sha or smoke steps when available. `no_evidence`, `out_of_date`, `spec_not_approved` and `alignment_unknown` are warnings; `TICKET_ALREADY_CLOSED`, `PAUSED`, `RECOVERY_HOLD` and `PARENT_NOT_FOUND` (subtasks only) still refuse. The tool appends protected `### Completion`. Optional `knowledgeWriteBack` records what we now know (`body`; `title` + existing `areaId` to mint, or `documentId` + `heading`/`mode` to amend). Prefer a covering doc; agents cannot mint areas or invent a document id.
- **Real proof:** for feature tickets with a Complete panel, attach structured evidence:
  - Tests: `rinkata demo propose <SPEC-ID> --tests <report.json>` (Vitest/Jest `--reporter=json` → pass/fail rows).
  - Screenshots: never paste image bytes or base64. Default path is `rinkata_begin_demo_screenshot_upload({ projectId, contentType, byteLength, sha256?, label })` → `curl -fsS -X PUT --upload-file./shot.webp -H "Content-Type: image/webp" -H "X-Rinkata-Upload-Token: $TOKEN" "$UPLOAD_URL"` → `rinkata_finalize_demo_screenshot_upload({ projectId, uploadId })` → pass only the returned Hub URL to `rinkata_complete_ticket` `screenshots[]` or `rinkata_propose_demo_evidence({ specId, type: "screenshot", url })`. Capture ≥1440×900 at deviceScaleFactor 2. Hub refuses smaller/soft bitmaps (`SCREENSHOT_TOO_SMALL` / `SCREENSHOT_TOO_SOFT`). Max 8 MiB / 40M pixels. Tiles live in spec/ticket drawers, not Complete cards. PUT files from disk (e.g. Cursor Cloud screenshot captures); do not upload one that shows secrets, credentials, tokens, or personal/customer data — recapture with test data or crop it first. A Cursor artifact page is auth-gated HTML (type url, not a tile). Legacy `rinkata_upload_demo_screenshot({ contentBase64, label })` is small-images-only. Hub still works with a session/API key.
  Both land as accepted evidence immediately. Optional pack-level human verdict is non-blocking attestation — rinkata records, does not gate shipping.

## Session-scoped completion conditions (ticket as goal)

Evaluators judge the **transcript** only (no disk). Pattern:

1. Quote `rinkata_read_ticket` (+ `rinkata_read_spec` if `spec:`) acceptance into the transcript first.
2. Goal signals: quoted criteria; advisory `rinkata_preflight_ticket` discrepancies (JSON, not text banner); `rinkata_complete_ticket` succeeded; `rinkata doctor: OK`; turn cap.
3. Surface test/preflight/complete outputs each turn so the evaluator can stop.

Claude Code: `/rinkata-ticket-goal <id>`. Others: same pattern via host goal/evaluator. Planned: `rinkata_ticket_goal_brief`.

**Don't:** reference "the ticket" without quoting body; use complete as the *only* signal (pair with doctor OK); match `SAFE <id>` banner instead of JSON; hand-edit `status: done`.

## Session handoff

First-class artifact (`HANDOFF-*`) outside the alignment graph (no `source_hash` / reconcile / doctor drift). Write when context-limited, asked for a next-session prompt, or checkpointing a work chunk.

- **Write:** `rinkata_write_handoff` (`title` + `body` required; optional `about`, `supersedes`). Body = resume prompt for a reader with no session memory. Returns `{ id, path, resumePointer }` (e.g. `Resume with: rinkata handoff get HANDOFF-…`). Unresolved `about` → `unresolved_about`, not blocked.
- **Read:** by id; or `rinkata_read_handoff({ latest: true, about })` (returns newest + `siblings` if several; `{ found: false }` if none). List/search: `rinkata_list_handoffs({ state, about, query })`.
- **Lifecycle:** `open → consumed → archived` via `rinkata_set_handoff_state`. Append-only; chain via `supersedes`.
- **Amend:** `rinkata_amend_handoff({ id, note })` appends attributed timestamped block (never rewrites). Allowed while open/consumed; archived rejected.

## Agentic-specific patterns (optional)

- **Preflight subagent:** call `rinkata_preflight_ticket`; surface discrepancies + JSON; only server holds refuse Start/complete.
- **Readiness shape:** if preflight is blocked or the user asks "what's blocking work?", use
  `rinkata_work_queue` + optional `rinkata_impact` + `rinkata_doctor` — not Hub UI.
- **Long work:** handoff checkpoints; background tests; resume via `rinkata_read_handoff({ latest: true, about })`.
- **Evaluator loops:** treat preflight + `rinkata doctor: OK` as first-class goal signals (see above).
- **MCP discovery:** `search_tool({ query: "rinkata" })` early when surface may vary by tenant.
- **Parallel explore:** preflight on reads; reconcile is the only controlled write path; protect human notes.
- **Non-feature:** `chore|bug|spike` liberally; `rinkata_inbox_promote --type` for inbox → ticket.

## Two-stage reconcile

Reconcile at the spec boundary so Goal drift does not cascade into every ticket.
**Default agent path for Goal/Spec drift** (not Hub-only):

1. `rinkata_propose_staged_reconcile({ target })` — Goal path → stage 1; spec path → stage 2.
2. Human reviews the concrete plan; ensure the action list is complete (`truncated: false`).
3. A human applies it — Hub UI staged reconcile drawer, or
   `rinkata_apply_staged_reconcile({ plan })` / CLI staged-apply on their own
   human identity. Agent keys and MCP OAuth are refused with
   `STAGED_APPLY_HUMAN_REQUIRED`.

CLI parity: `rinkata reconcile staged <target> --json` /
`rinkata reconcile staged-apply --plan <file> --approve-all --yes`
(or per-action `approved:true`).
Hub staged drawer is the human apply surface. Classic
`rinkata_propose_reconcile` / `rinkata_reconcile_write` still cover Goal-anchored
tickets without specs / legacy ticket-level drift; a human applies those plans
from the Hub UI Fix drift drawer or with `rinkata_reconcile_apply` on their own
human identity.

- Goal path → **stage 1** (Goal→Spec): ABSORB / DEFER / UPDATE / CREATE / ARCHIVE per spec.
- Spec path → **stage 2** (Spec→Ticket): ABSORB members with mismatched `source_hash`.
- **ABSORB** bumps `source_hash` without body change (typo fixes; prevents ticket churn). Stage-1 ABSORB ⇒ stage 2 usually no-op. **After tickets already have file-level plans and the spec moved, default the proposed verb to ABSORB.** UPDATE only when a ticket body is actually wrong. Do not default to UPDATE that appends useless “Upstream” notes. Apply remains human-gated.
- **DEFER** = noticed, no hash bump. **UPDATE** needs `bodyRevision` after `rinkata_write_spec_body`. **CREATE** seeds minimal Hub spec stub. **ARCHIVE** → `rinkata_archive_spec` + explicit downstream.
- Apply: `RECONCILE_GRAPH_STALE` if graph moved — re-propose, never bypass.
- Decision-driven apply (after `rinkata_propose_from_decision`) clears `pending_propagation` once no decision work is left.

## Hub-Only Mode

Hub DB is canonical for agent workflows. `rinkata_status`, `rinkata_doctor`,
drift, preflight, ticket reads, spec reads, ticket status flips, ticket
completion, decisions, inbox, and reconcile writes must go through Hub MCP.
If Hub credentials are missing or Hub is unreachable, fail closed and ask the user to reconnect Hub MCP (OAuth) and confirm the project is selected.

### No Local Artifact Fallback

There is no supported repo-local artifact fallback. If a tool cannot reach Hub,
or if the active MCP surface does not expose the needed Hub-backed tool, stop and
surface the repair path. Do not inspect a checkout for local Goal, spec, ticket,
idea, inbox, decision, reconcile, or handoff files.

## Failure modes

- Tools missing → Hub MCP isn't connected, or the OAuth session is stale. Reconnect Hub MCP so the OAuth session is current
  so new tools (`rinkata_accept_decision`, `rinkata_knowledge_list` /
  `read` / `ask` / `record` / `upload`, `needs_action`, `decisionId` on writes) appear.
  Non-zero `rinkata_status` counts only prove the data path.
- `STALE_ARTIFACT` errors → re-read project state, re-issue. Never edit hashes
  manually.
- Drift after reconcile apply → run `rinkata_doctor`, share output with user.
- `RATIFY_FORGE_REFUSED` / `RATIFY_HUMAN_REQUIRED` → agents cannot stamp
  ratify; ask a human to use Hub **Accept** or a human-session
  `rinkata_accept_decision` (human session only; agents get RATIFY_HUMAN_REQUIRED).
- `SUCCESSOR_REQUIRED` on a SPEC write → stop-and-successor, or retry with
  `decisionId` of a **human-ratified** governing DEC (dual unlock); never
  same-call `authorize:true`.
- `rinkata_write_ticket_body` fail after mint / `partialMinted` → do **not**
  Start. Retry remaining mint stubs. Enrich `partialMinted` before re-plan.
  `EXPECTED_VERSION_REQUIRED` / `STALE_ARTIFACT` → re-read + retry with
  `expectedVersion`. `PLAN_FINGERPRINT_DROPPED` → restore the mint comment.
  `SECRET_IN_BODY` → strip credentials. Skip non-stubs.

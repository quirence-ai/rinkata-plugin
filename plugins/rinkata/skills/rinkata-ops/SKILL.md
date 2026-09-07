---
name: rinkata-ops
version: 0.1.47
description: |
  rinkata is a Hub source-of-truth consistency engine. Hub DB is canonical for
  Goals, specs, tickets, decisions, reconcile artifacts, inbox notes, handoffs,
  and completion evidence. Installed MCP tools, the CLI, and the Hub UI are the
  access paths; repo-local artifact directories are not a rinkata surface.
  Use this skill whenever the user mentions rinkata projects, tickets, preflight,
  reconcile, drift, doctor, ticket IDs like TICK-*, Needs you, or Decisions.
  Also load mid-session to mint a proposed Decision (judgment call / log a
  decision) then list needs_action — do not wait for a slash command.
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
  - rinkata_ratify_decision
  - rinkata_reconcile_write
  - rinkata_reconcile_apply
  - rinkata_apply_staged_reconcile
  - rinkata_complete_ticket
  - rinkata_knowledge_list
  - rinkata_knowledge_read
  - rinkata_knowledge_ask
  - rinkata_knowledge_record
  - rinkata_knowledge_upload
  - rinkata_upload_demo_screenshot
  - rinkata_promote_idea
  - rinkata_plan_from_spec
  - rinkata_claim
  - rinkata_set_ticket_status
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
    notes: "Discovered via host skill discovery + MCP. The agentic patterns below (plan mode, verification subagents, MCP discovery) are host-neutral and apply equally to any harness with those capabilities."
---

# rinkata Operations — Hub Truth Contract

Hub DB owns rinkata-managed project docs: Goals, specs, tickets, decisions, inbox
notes, reconcile plans, handoffs, and completion evidence. Installed rinkata MCP
tools, the CLI, and the Hub UI are the only supported agent surfaces. Agents
must not look for, read, create, or edit repo-local artifact directories. Every
artifact has a Hub source hash pinning it to upstream truth; drift is closed via
reviewable reconcile.

Three-layer chain: **Goals** → optional **specs** → **tickets**. Specs recommended when a Goal section spawns more than one or two tickets; direct Goal-anchored tickets remain valid.

## Contract

This skill guarantees:

- rinkata artifact reads and writes go through Hub MCP or the Hub UI, never
  repo-local artifact files.
- Missing or unavailable Hub truth is a blocker, not a reason to fall back to a
  local artifact directory.
- `rinkata_status` is consulted before any work; preflight runs before tickets.
- After a read of an idea, Goal, spec, or ticket, honor `nextHop` (`tool`, `mode`, `reason`) when present. Load the owning ceremony skill; do not invent a parallel write and do not auto-execute the hop. Write-time `nextAction` and `work_queue.claimAction` are unchanged.
- Stale source hashes are blockers, never silently overwritten.
- Reconcile plans are reviewable and human-approved before apply.
- Protected human notes survive every reconcile.

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

**Needs you = Decision queue** (not bare ticket hash drift). Doctor / Reconcile / Ready stay separate.

Flow: judgment → mint **proposed** DEC → human **Accept** (one click: seal +
ratify). The Decision is source of truth and **leaves Needs you** unless options
are unset, seal failed, or residual is explicit **Retire covering work…**.
`pending_propagation` is not kept for guidance-appendix / rescope / complete-spec
refuse / DEFER.

- **Mint:** `rinkata_log_decision` + `affects[]`. Agents default `status: proposed` (cannot self-accept/ratify). `affects[0]` is the covering Goal this Decision is *for* (usually the newest live Goal); older Goals, specs, and tickets after. Ticket-only lists cannot Goal-group — include the covering Goal first when they should. Do not sort by `created_at`. On success for **proposed**, expect `nextAction: "list_needs_action"` — immediately list residual with `rinkata_list_decisions({ status: "needs_action" })` and surface the Needs you path; do not stop at a bare create ack.
- **List:** `rinkata_list_decisions({ status: "needs_action" })` (alias `actionable`) — same oracle as Hub Needs you.
- **Accept:** Hub primary CTA is **Accept**. Human seal (`accepted` + durable ratify). Pending clears unless residual is explicit `intent: remove`. Mechanical ABSORB / Goal→Spec CREATE may run in the same request.
- **Ratify:** **human only** via `rinkata_ratify_decision` / Hub / ratify-on-merge. Stamps `ratified_at`+`ratified_by`. Agents: `RATIFY_HUMAN_REQUIRED`. Same-call booleans / generic PUT → `RATIFY_FORGE_REFUSED`. Never forge ratify stamps.
- **Retire:** confirm-gated ARCHIVE (`intent: remove`). Never from free-form Decision body. Never auto on Accept. **Reject** (proposed only) is lifecycle-only. After a successful Accept the next hop is not **Update guidance** / **Check residual** / **Apply residual** / **Clear pending**. Absorb failure is **Retry**. Technical tools: `rinkata_propose_from_decision` with explicit `intent: remove|rescope` (default auto = rescope) then human-gated staged apply. Agents propose, humans Accept — never agent-forge apply.
- **Supersede:** `status: superseded` needs durable `supersededBy` (not prose-only).
- **Dual unlock (S3+S5):** evidence-bound/complete SPEC rewrites refuse `SUCCESSOR_REQUIRED` — prefer successor. After **human-ratified** DEC, retry `rinkata_write_spec_body` / `rinkata_set_spec_acceptance` with **`decisionId`**. Claimed `decisionRatified` / `authorize:true` never unlock.
- **Reconnect MCP** after upgrade: `rinkata_status` counts ≠ tool surface. Missing `rinkata_ratify_decision` / `needs_action` / `decisionId` = stale process.

## Cross-artifact search / duplicate prevention

**Before create**, call `rinkata_search` with distinctive terms; prefer extending a covering **open** artifact over a parallel one. **Exception:** if a high-score peer is `derivedState: complete`, do **not** extend by stacking — use stop-and-successor (below).

- Create-time `relatedGoals` / `relatedSpecs` / `relatedTickets` is **stop-and-read** (advisory; create still succeeds; `relatedStatus: degraded` + `RELATED_SEARCH_FAILED` is still a successful mint). When `relatedGuidance` is present (complete peers in the hit set), follow stop-and-successor — never stack tickets under those SPECs.
- Before **approve** or the first **spec-anchored** feature ticket, expect server-recomputed peer set. Use `peerDisposition: { decision: "independent", peerIds, note }` (≥20 char note) only when truly separate; `decision: "fold"` = abandon this duplicate and extend **open** survivors (never fold onto complete). On `PEER_SEARCH_UNAVAILABLE`, retry search — do not invent a disposition (approve/first-feature still refuse).
- **Agent terminal is the primary disposition surface** (graph-aware search + disposition): after create, read `relatedSpecs` + `dispositionRecommend` / `dispositionGuidance` (fail-open — create still succeeds). Prefer stop-and-extend or successor when recommended; do **not** leave "approve later in Hub" as the plan. On `PEER_DISPOSITION_REQUIRED`, use structured `dispositionRecommend` (confidence + nextStep) and retry **the same tool** with `peerDisposition` in-session. Hub is human backup only — not the agent happy path.
- Goal-section-anchored features (`source:` only) are not disposition-gated in v1 — still search first. Handoffs use `rinkata_list_handoffs({query})`, not `rinkata_search`.

**peerRisk (advisory):** `rinkata_read_spec` / `rinkata_preflight_ticket` may attach `status: clear | peers | degraded` + capped peers with `derivedState` and guidance. Never changes preflight verdict or blocks a read. Use it before hard gates on approve / first feature.

**Complete peers / stop-and-successor:** if a high-score peer has `derivedState: complete`, do **not** extend/fold/soft-stack onto it by stacking feature tickets. Prefer **one official write**. From an idea: `rinkata_promote_idea({ destination: "standalone_spec", predecessor })` — or `create_successor_spec` / `create_standalone_spec` with `originIdeaId` (those seal the idea: `promoted` + `promoted_to` + `promoted_at`). Without an idea: `rinkata_create_successor_spec` (or `create_standalone_spec` + `predecessor`). Or human reopen (`rinkata_approve_completed_spec_reopen`). Do **not** mint a successor then comment on the idea. Re-promote of an origin-linked idea is idempotent (`IDEA_TARGET_EXISTS`) and does not mint a second spec. Server hard-gates refuse create, Goal soft-stack, promote, and `set_ticket_spec` under a covering complete SPEC with `COMPLETED_SPEC_REOPEN_APPROVAL_REQUIRED` + structured recovery (successor or human reopen). Same-call booleans are not authority; agents cannot forge reopen.

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
- `rinkata_list_decisions({status, limit})` — use `status: "needs_action"` / `actionable` for Needs you (GOAL-67).
- `rinkata_list_specs({goal, status, includeArchived})` (`prd` deprecated alias for `goal`).
- `rinkata_inbox_list({})`, `rinkata_list_handoffs({state, about, query, limit})`.
- `rinkata_search({query, kinds?, statusByKind?, goal?, limit?, offset?})` — prefer before create.
- `rinkata_knowledge_list({areaId?})` — customer-named areas + documents (`areaId: "unfiled"` for Unfiled). Then `rinkata_knowledge_read` / `rinkata_knowledge_ask`. Upload files with `rinkata_knowledge_upload` (not markdown notes — those stay `rinkata_knowledge_record`). Do not invent area names.

**Bounded reads:** `rinkata_work_queue`, `rinkata_list_drift`, `rinkata_propose_reconcile`, and `rinkata_propose_staged_reconcile` return at most 25 items in `{ total, returned, offset, truncated }`. If `truncated: true`, page with `offset`/`limit` or filters — never infer "empty / no drift / nothing to reconcile" from one partial page. Staged apply requires a complete action list (`actions.length === summary.totalActions`); re-fetch before `rinkata_apply_staged_reconcile`.

### Phase 2 — Preflight (before starting or completing a ticket)

1. `rinkata_preflight_ticket(ticketId)` — confirms hashes are fresh, deps satisfied.
   For a spec-anchored ticket the preflight additionally verifies the spec is
   `approved` (drafting blocks transitions past `todo`) and the ticket's
   `source_hash` matches the current spec body.
   When `knowledgeCovering` is present, read those area docs (`rinkata_knowledge_read`)
   before inventing context. Knowledge is why / what we learned; the graph is
   what we committed to build. Covering is advisory — it does not change the
   preflight verdict.
2. `STALE_ARTIFACT` / `MISSING_BLOCKER` response = stop and re-issue after
   re-reading. Never bypass.
3. **Claim vs Start:** two distinct operations —
   skills do **not** reimplement parent-fill or sole-branch rules.
   - **Claim** = take primary ownership (`assignee`) only. Goal / Spec:
     `rinkata_claim({ kind, id })`. Ticket assignee-only:
     `rinkata_claim({ kind: "ticket", id, start: false })` (no preflight, no
     status flip, no parent fill).
   - **Start** = tickets only. Prefer `/rinkata-ticket-start` or
     `rinkata_claim({ kind: "ticket", id })` (default `start: true` = official
     Start / Hub `claim_start`: preflight + assignee + `in_progress` +
     `started_at`). On success, **report** server `filledParents` only — never
     invent parent fills or client-write parent assignees.
   - Bare `rinkata_set_ticket_status(…, in_progress)` for `todo → in_progress` is
     **refused** with `USE_CLAIM_START` — redirect to official
     Start (`rinkata_claim` default / `/rinkata-ticket-start`). It is not a
     fill-if-empty claim path and never parent-fills. Goal/Spec ownership
     remains claim-only (`rinkata_claim` on those kinds).
   - Discovery: `rinkata_work_queue` items expose `claimAction`
     (`claim` | `start` | `none`) — use it to choose Claim vs Start, then call
     the official tools. Parent fill is **Start-only**; Claim never parent-fills.
     Ready plan_from_spec mint stubs are `claimAction` none (enrich via
     `rinkata_write_ticket_body`); do not Start. Official `claim_start` is not
     stub-gated — ticket-start / nextHop are the ceremony backstop.

### Readiness / blocker chains — agent-first

**Do not** send users to a Hub dependency map for blocks/blocked_by shape.
Agents already own readiness visibility:

| Need | Tool |
|------|------|
| This ticket blocked / safe? | `rinkata_preflight_ticket` — open blockers, stale source |
| Queue shape (blocked / ready / in progress) | `rinkata_work_queue` — filter `readiness` / `lifecycleStatus`; page if `truncated` |
| Blast radius if Goal/Spec/Ticket moves | `rinkata_impact` (CLI `rinkata impact`) — read-only multi-hop |
| Cycle / missing-referent / inverse edges | `rinkata_doctor` |

Hub is for human authority (Accept / Ratify / Retire / Approve),
not a second place to re-learn the DAG. Prefer progressive disclosure in the
**terminal** (preflight JSON → work_queue page → impact) over new Hub chrome.

### Phase 3 — Propose (during work)

- Decisions: `rinkata_log_decision` + `affects[]`, agents **`status: "proposed"`**. Covering Goal first, then reference Goals/specs/tickets. On proposed mint, follow `nextAction: "list_needs_action"` (list residual for the human). Lifecycle via `rinkata_update_decision`. **Do not** forge accept/ratify/align apply — human Hub **Accept** / **Ratify** / `rinkata_ratify_decision` only. Judgment edges → proposed DEC on Needs you (not a ticket-only interrupt).
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
- Propose Goal/Spec drift: `rinkata_propose_staged_reconcile({ target })` —
  surface plan; do not apply without approval. If `truncated: true`, re-fetch with
  a high enough limit (or reassemble pages) so the full action list is present
  before apply.
- Propose legacy ticket-level drift (no specs): `rinkata_propose_reconcile` /
  `rinkata_reconcile_write` — surface plan; do not apply without approval.
- Decision fan-out: `rinkata_propose_from_decision` then staged apply path below.

### Phase 4 — Apply (only when authorized)

- **Staged (default for Goal/Spec):** `rinkata_apply_staged_reconcile({ plan })`
  only after the user approved that concrete plan (per-action `approved`). On
  `RECONCILE_GRAPH_STALE`, re-propose and re-review — never bypass. Decision-driven
  apply clears `pending_propagation`.
- **Classic:** `rinkata_reconcile_apply` only after the user approved that plan.
  Re-issue on `STALE_ARTIFACT` / graph stale.

### Phase 5 — Close with evidence (when work has shipped)

- `rinkata_complete_ticket(ticketId, evidence)` (CLI: `rinkata complete`) — never hand-edit `status: done` / `completed_at` / `completion_evidence`. Stale source → reconcile first. Evidence: PR URL, commit sha, smoke steps; tool appends protected `### Completion`. Optional `knowledgeWriteBack: { body, title?, areaId?, documentId?, heading?, mode? }` records what we now know: mint a note in an existing area or Unfiled, or amend a listed document (`documentId` + `append`|`replace`|`new_heading`|`replace_document`). `mode=replace_document` (or heading-less `mode=replace`) replaces the whole stored markdown; heading-ful `replace` still needs a heading; `append` / `new_heading` stay additive. Best-effort — Complete still succeeds if ingest fails. Prefer a covering doc over a parallel `What we learned — TICK-X` note. Or call `rinkata_knowledge_record` after Complete. Agents cannot mint areas or invent a document id.
- **Real proof:** for feature tickets with a Complete panel, attach structured evidence:
  - Tests: `rinkata demo propose <SPEC-ID> --tests <report.json>` (Vitest/Jest `--reporter=json` → pass/fail rows).
  - Screenshots: prefer `rinkata_upload_demo_screenshot({ contentBase64, label })` (Hub MCP; same caps as the HTTP route), then `rinkata_complete_ticket` `screenshots[]` or `rinkata_propose_demo_evidence({ specId, type: "screenshot", url })`. When FE/UI work produced walkthrough captures (e.g. Cursor Cloud screenshot captures), upload those bytes — do not paste Cursor agent artifact page URLs (auth-gated HTML, not image tiles). Hub remains valid with an OAuth session.
  Both land as accepted evidence immediately. Optional pack-level human verdict is non-blocking attestation — rinkata records, does not gate shipping.

## Session-scoped completion conditions (ticket as goal)

Evaluators judge the **transcript** only (no disk). Pattern:

1. Quote `rinkata_read_ticket` (+ `rinkata_read_spec` if `spec:`) acceptance into the transcript first.
2. Goal signals: quoted criteria; `rinkata_preflight_ticket` / `preflight --json` → `"status": "safe"` (JSON, not text banner); `rinkata_complete_ticket` succeeded; `rinkata doctor: OK`; turn cap.
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

- **Preflight subagent:** call `rinkata_preflight_ticket`; if not `safe`, stop with blocker + JSON.
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
3. `rinkata_apply_staged_reconcile({ plan })` after explicit approval.

CLI parity: `rinkata reconcile staged <target> --json` /
`rinkata reconcile staged-apply --plan <file> --approve-all --yes`
(or per-action `approved:true`).
Hub staged drawer remains an alternate human surface. Classic
`rinkata_propose_reconcile` / `rinkata_reconcile_write` / `rinkata_reconcile_apply`
still cover Goal-anchored tickets without specs / legacy ticket-level drift.

- Goal path → **stage 1** (Goal→Spec): ABSORB / DEFER / UPDATE / CREATE / ARCHIVE per spec.
- Spec path → **stage 2** (Spec→Ticket): ABSORB members with mismatched `source_hash`.
- **ABSORB** bumps `source_hash` without body change (typo fixes; prevents ticket churn). Stage-1 ABSORB ⇒ stage 2 usually no-op. **After tickets already have file-level plans and the spec moved, default the proposed verb to ABSORB.** UPDATE only when a ticket body is actually wrong. Do not default to UPDATE that appends useless “Upstream” notes. Apply remains human-gated.
- **DEFER** = noticed, no hash bump. **UPDATE** needs `bodyRevision` after `rinkata_write_spec_body`. **CREATE** seeds minimal Hub spec stub. **ARCHIVE** → `rinkata_archive_spec` + explicit downstream.
- Apply: `RECONCILE_GRAPH_STALE` if graph moved — re-propose, never bypass.
- Decision-driven apply (after `rinkata_propose_from_decision`) clears `pending_propagation` server-side.

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
  so new tools (`rinkata_ratify_decision`, `rinkata_knowledge_list` /
  `read` / `ask` / `record` / `upload`, `needs_action`, `decisionId` on writes) appear.
  Non-zero `rinkata_status` counts only prove the data path.
- `STALE_ARTIFACT` errors → re-read project state, re-issue. Never edit hashes
  manually.
- Drift after reconcile apply → run `rinkata_doctor`, share output with user.
- `RATIFY_FORGE_REFUSED` / `RATIFY_HUMAN_REQUIRED` → agents cannot stamp
  ratify; ask a human to use Hub Ratify or a human-session
  `rinkata_ratify_decision`.
- `SUCCESSOR_REQUIRED` on a SPEC write → stop-and-successor, or retry with
  `decisionId` of a **human-ratified** governing DEC (dual unlock); never
  same-call `authorize:true`.
- `rinkata_write_ticket_body` fail after mint / `partialMinted` → do **not**
  Start. Retry remaining mint stubs. Enrich `partialMinted` before re-plan.
  `EXPECTED_VERSION_REQUIRED` / `STALE_ARTIFACT` → re-read + retry with
  `expectedVersion`. `PLAN_FINGERPRINT_DROPPED` → restore the mint comment.
  `SECRET_IN_BODY` → strip credentials. Skip non-stubs.

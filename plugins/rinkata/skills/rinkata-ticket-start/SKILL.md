---
name: rinkata-ticket-start
version: 0.2.10
description: |
  The "I'm picking up TICK-X" entry point for any rinkata-backed project.
  Runs preflight on the ticket, refuses on a broken chain (stale source,
  unsatisfied blocker), surfaces the ticket body + spec acceptance
  criteria into the transcript so the agent has context BEFORE working,
  then claims the ticket — assigns it to the caller and transitions it
  todo → in_progress in one call via rinkata_claim (official Start).
  After a successful Start, always reports server `filledParents` (print
  kind+id entries, or state that the array is empty — never invents fills).

  Use this skill when an engineer or AI agent is about to start work on
  a specific ticket — typically right after /rinkata-orient suggests one.
  /rinkata-orient answers "what should I work on"; /rinkata-ticket-start
  claims the chosen ticket and packs its context. Also load mid-session
  without a slash when the user is picking up, starting, or claiming a
  ticket (claim vs start: rinkata_claim start:false vs default start:true).


  Mutating: this skill flips ticket status. It never flips status as a
  side effect of any other action — only on its own explicit invocation.
triggers:
  - /rinkata-ticket-start
  - "start ticket TICK-"
  - "pick up TICK-"
  - "I'm starting TICK-"
  - "start work on TICK-"
  - "claim ticket TICK-"
  - "take ownership of TICK-"
  - "I'm picking up this ticket"
tools:
  - rinkata_preflight_ticket
  - rinkata_read_ticket
  - rinkata_read_spec
  - rinkata_claim
  - rinkata_knowledge_read
  - rinkata_write_ticket_body
mutating: true
argument-hint: <ticket-id>
metadata:
  grok:
    notes: "Excellent with plan mode for the preflight + context pack step. Consider pairing with a verification subagent for the acceptance criteria check."
---

# rinkata ticket-start — preflight + claim + context pack

Load mid-session (no slash) when the user is picking up, starting, or claiming a
ticket. Claim vs start: `rinkata_claim` (`start: false` vs default `start: true`).

`/rinkata-ticket-start <ticket-id>` is how an engineer or agent claims a ticket.
It refuses to start a ticket whose chain is broken, surfaces everything needed to
work the ticket, and then **claims** it — assigns the ticket to the caller and
transitions its status to `in_progress` in one call via `rinkata_claim`.

Perf budget: the calls run **sequentially** — preflight gates first, then
`rinkata_read_ticket`, then `rinkata_read_spec` (only if the ticket is spec-anchored),
then `rinkata_claim` (only if a start happens). Worst case is **4 Hub MCP
round-trips**: a spec-anchored ticket that starts (preflight + 2 reads + the
claim). A no-spec ticket that starts is 3; a run a pre-flip guard short-circuits
is fewer. `rinkata_claim` folds the assignment and the `todo → in_progress` flip
into a single write, so claiming does not add a round-trip over the old
status-only flip. The skill reads sequentially for simplicity — preflight's
`source` field does expose the spec path, so a later optimization could derive
`specId` from it and run the two reads in parallel. Targets <2s for typical
projects

## What to do

1. **Resolve project truth through MCP.** Hub DB is canonical and the active MCP
   surface must route reads/writes through Hub. If Hub auth or project
   resolution fails, stop and surface the login/recovery error. Do not fall back
   to local files.

2. **Parse the ticket id.** Prefer the slash argument, then a `TICK-` id in the
   user utterance or the current session context. If none is available, surface
   the usage line `/rinkata-ticket-start <ticket-id>` and stop.
   **Mode:** if the user asked only to claim / take ownership (not pick up,
   start, or start work), this run is **Claim** (`start: false`). Slash
   `/rinkata-ticket-start`, "start ticket", "start work", "I'm starting", and
   "I'm picking up" are **Start** (default `start: true`).

3. **Preflight (Start only).** Claim-only skips this step (assignee-only; no
   preflight). For Start, call `rinkata_preflight_ticket({projectId, ticketId})`. The response carries
   `status` (`safe` | `stale` | `blocked` | `unknown`), `reasons[]`,
   `upstream_changes[]`, and `recommended_action`. When `knowledgeCovering` is
   present, list those area docs and read covering ones (`rinkata_knowledge_read`)
   before inventing context. Knowledge is why / what we learned; the graph is
   what we committed to build. Covering is advisory — it does not change the
   preflight verdict. Missing `knowledgeCovering` means an older MCP/server;
   do not invent covering docs.

4. **Gate on preflight (Start only).** Claim-only skips this step. For Start,
   if `status !== "safe"`, STOP — never flip status on a non-safe preflight.
   Render the blocker plainly:
   - `stale` — the ticket's source (spec or Goal section) drifted. Surface the
     `reasons[]` and: "Reconcile the drifted source — open the reconcile drawer
     from the Hub UI Status tab, or run `rinkata reconcile` to propose a plan —
     then re-run `/rinkata-ticket-start`."
   - `blocked` — an unsatisfied `blocked_by` dependency, or `status: blocked`.
     Name the blocking ticket ids from `reasons[]`.
   - `unknown` — the ticket points at a missing spec / Goal section. Surface:
     "Re-anchor the ticket to a covering spec before starting."
   - A tool error — `rinkata_preflight_ticket` wraps every failure (including a
     non-existent ticket id) as with the detail in the error message;
     surface it verbatim.
   In every non-safe case the ticket status is left untouched.

5. **Context pack.** Start: only after preflight is `safe`. Claim-only: still
   run these reads (no preflight). Read the ticket first, then the spec:
   - `rinkata_read_ticket({projectId, ticketId})` — render `body` verbatim under a
     `## Ticket — <id>` heading.
   - Then, if the ticket frontmatter has a `spec` field →
     `rinkata_read_spec({projectId, specId})` with `specId = ticket.frontmatter.spec`
     — render `frontmatter.acceptance[]` verbatim under `## Acceptance criteria
     (<spec-id>)`. If the ticket has no `spec` field (legacy PRD-anchored), render
     the ticket body's own checklist as the acceptance surface and note it is
     ticket-local.
   - Compute a **suggested branch name** (see Branch suggestion below) and print
     it. Do NOT create the branch — that is opt-in (step 7).

6. **Claim vs Start.** The context pack from step 5 has already been
   rendered, so the agent has the ticket in view before any mutation (this
   ordering is the contract — context before write). Slash
   `/rinkata-ticket-start <id>` is itself explicit Start intent; no separate
   confirmation prompt is needed.
   **Claim-only is exclusive.** If step 2 mode is Claim: if the ticket is
   `done` or `archived`, skip (do not reopen). Otherwise call
   `rinkata_claim({ kind: "ticket", id: ticketId, start: false })` once and
   **stop** — do not run any Start guard or `start: true`. Assignee only,
   even when the spec is `drafting`/`archived` or the ticket is `in_progress`
   / `blocked`. Server refuses steal (`ALREADY_CLAIMED`). Claim never
   parent-fills; do not report `filledParents` as a Start fill.
   **Start only** — apply these pre-write guards in order:
   - **Start + owned by another principal** → STOP. Do NOT Start and do NOT
     `rinkata_write_ticket_body`. If step 5 `frontmatter.assignee` is set and
     is not the caller (or `nextHop.tool` is null with reason `Owned by …`,
     or `claimAction` is `none` for ownership), honor that and stop. Server
     Start refuses steal (`ALREADY_CLAIMED`). `write_ticket_body` has no
     assignee gate (GOAL-52), so enriching first would rewrite someone else's
     stub. Unassigned stubs still take the mint-stub enrich guard after
     lifecycle.
   - **Start + `done` or `archived`** → skip, surface "ticket is `<status>`;
     reopen it explicitly with `rinkata_set_ticket_status` if that is intended."
     Never silently reopen a closed ticket. **Do NOT call `rinkata_claim`.**
     **Do NOT call `rinkata_write_ticket_body`.** Lifecycle before enrich:
     `write_ticket_body` permits rewrites on closed tickets, and a completed
     stub still matches stub detection (`### Completion` is not an extra
     `##` heading).
   - **Start + already `in_progress`** → skip, note "already in_progress — no
     status change", succeed. The skill doubles as a mid-session "re-surface my
     ticket context" tool. **Do NOT call `rinkata_claim` here** — a Start
     claim on an already-started ticket would reassign it. **Do NOT enrich**
     an in_progress stub either (lifecycle before enrich).
   - **Start + mint stub** → STOP, do NOT Start. Only for
     `todo` tickets — the lifecycle guards above already skipped `done` /
     `archived` / `in_progress`. A completed stub still looks like a stub:
     completion appends a protected `### Completion` block, while stub
     detection only rejects extra `##` headings.
     If the todo body from step 5 is still an Intent + Acceptance mapping
     stub (plan fingerprint present, no extra `##` headings), enrich first:
     `rinkata_write_ticket_body` with `expectedVersion` from the read, keeping
     fingerprint + protected regions, no secrets. Then re-run this skill.
     Skip this guard when the body is already a non-stub.
   - **Start + spec-anchored AND `spec.frontmatter.status !== "approved"`** →
     STOP, do NOT Start. A ticket should only move past `todo` once its spec
     is `approved` (the spec layer). Two cases:
     - spec is `drafting` → "spec `<spec-id>` is `drafting` — approve it with
       `rinkata_set_spec_status(<spec-id>, approved)` once the body + acceptance are
       ready, then re-run `/rinkata-ticket-start`."
     - spec is `archived` → "spec `<spec-id>` is `archived` — this ticket is an
       orphan under a retired spec. Re-anchor or archive it
       (`rinkata_archive_spec... downstream: migrate|detach|archive`) before
       starting."
     Hub MCP enforces this on official **Start** (`start: true` / `claim_start`)
     only — not on Claim (`start: false`). Keep the skill-side copy because it
     explains the Start refusal before mutation; server behavior is authoritative.
   - **Start otherwise** — ticket status is `todo`, and the ticket is either not
     spec-anchored or its spec is `approved` →
     `rinkata_claim({projectId, kind: "ticket", id: ticketId})` (default
     `start: true` = official Start). This assigns the ticket to the caller
     **and** advances it `todo → in_progress` in one write when Start succeeds.
     A successful tool result carries
     `{ assignee, status, started, startedAt?, filledParents }` where
     `filledParents` is always an array on current Hub (`[]` when nothing was
     filled). Confirm assignment and status in the transcript.
     **Report `filledParents`:**
     - On a successful Start (`started: true`), always surface the
       `filledParents` field from the response. It is an array of
       `{ kind: "goal"|"spec", id: string }` for parents the server
       fill-if-empty claimed for the caller (may be empty `[]`).
     - If `filledParents` is a non-empty array, print each filled parent as
       `kind` + `id` (e.g. "Filled parents: SPEC-…, GOAL-…") so the
       session sees soft parent ownership without opening Hub.
     - If `filledParents` is an **explicit** empty array `[]`, say so
       (e.g. "filledParents: [] — no parent fill this Start"). That is a
       real server result — do **not** invent parent fills.
     - If the `filledParents` **field is absent** (older MCP/server), do
       **not** coerce that to `[]` or claim "no parent fill." Report that
       `filledParents` is unavailable and ask the user to reconnect or
       update the rinkata MCP server so the Start response includes the
       field. Omitting the field is not proof that no parents were filled.
     - Server `filledParents` is the **only** source of truth for what was
       filled. Do **not** reimplement sole-branch Goal logic or parent-fill
       rules in the skill; Hub `claim_start` owns that policy.
     **`started: false` on a successful (non-error) result:** this is the
     already-`in_progress` / idempotent Start path (same actor retry, or
     fill-if-empty without a status transition). Report the returned
     `assignee` and `status`, and report `filledParents` only if the field
     is present as an array (typically `[]` — no parent fill on this path).
     Do **not** invent a `startError` field — Hub does not return one on
     success. Do **not** claim the ticket "failed to start" or invent a
     retry reason from a missing field.
     **Tool / Hub errors** (e.g. Spec not approved, preflight unsafe,
     steal, not found) are **zero-write failures** — the tool rejects the
     call; there is no success payload with `started`/`filledParents`.
     Surface the error code/message and stop; do not invent assignment or
     parent-fill outcomes.

7. **Branch creation is opt-in.** Default: do nothing. Only if the user
   explicitly asks for a branch, suggest `git checkout -b <suggested-branch>`.
   This skill does not run git itself.

## Acceptance scenario contract

The implementation and tests for this skill must cover these paths:

- **Happy path** — `rinkata_preflight_ticket` returns `status: "safe"`;
  `rinkata_read_ticket` renders the ticket body; a spec-anchored ticket then calls
  `rinkata_read_spec`; the context pack is visible before any `rinkata_claim` call,
  and the claim both assigns the ticket to the caller and advances it to
  `in_progress`. After a successful Start (`started: true`), the skill prints
  server `filledParents` (list of kind+id, or explicit empty `[]`) without
  inventing fills; an absent field is reported as unavailable, not as empty.
- **Stale source path** — `status: "stale"` renders the stale artifact reasons
  plus a reconcile recovery hint, then stops without a status write.
- **Blocker path** — `status: "blocked"` renders the blocking ticket ids from
  `reasons[]`, then stops without a status write.
- **No-spec ticket path** — a ticket with no `spec` field uses the ticket-local
  checklist as its acceptance surface and does not call `rinkata_read_spec`.
- **Spec-anchored ticket path** — a ticket with `spec` calls
  `rinkata_read_spec`, renders `frontmatter.acceptance[]`, and **Start** refuses
  if the spec status is anything other than `approved`.
- **Claim-only path** — `start: false` assigns under a drafting spec; status
  stays `todo`. Does not apply the Start spec-approved gate. Server steal-protects.
- **Mint-stub enrich path** — Start on an Intent+mapping stub
  stops and calls `rinkata_write_ticket_body` (keep fingerprint + protected;
  `expectedVersion`; no secrets) instead of `rinkata_claim`. Skip when non-stub.
  If the stub is assigned to another principal, stop without enriching
  (ownership before enrich). If the ticket is `done`, `archived`, or
  `in_progress`, skip enrich (lifecycle before enrich) — a completed stub
  still matches stub detection because `### Completion` is not an extra
  `##` heading.
- **filledParents reporting path** — official Start response is the only
  authority for parent fills; skill never reimplements sole-branch / parent-fill
  rules and never client-writes parent assignees.

Linked decisions are opportunistic only today: if the active MCP surface
or ticket payload exposes explicit decision links, render them in the context
pack. Today there is no ticket-to-decision reverse index, so absence of linked
decisions is rendered as "none found" rather than guessed from prose.

## Branch suggestion

Deterministic slug from the ticket id + title:
- Prefix by `ticket_type` frontmatter: `feat/` for `feature`, `fix/` for `bug`,
  else `chore/`.
- Append the lowercased ticket id, then a title slug — lowercase, non-alphanumeric
  runs collapsed to single hyphens, leading/trailing hyphens trimmed, capped at
  roughly 40 characters.
- Example: `TICK-X` "/rinkata-ticket-start: preflight + claim + context
  pack" with `ticket_type: feature` → `feat/tick-x-rinkata-ticket-start`.

## Capability check

At startup, call `requireTools(["rinkata_preflight_ticket", "rinkata_read_ticket", "rinkata_read_spec", "rinkata_claim", "rinkata_write_ticket_body"])` against the active MCP surface. If any tool is missing, render the returned parity-gap message, including any note id or alternate surface, and stop before preflight. `rinkata_claim` is Hub-backed. `rinkata_read_ticket` is Hub-only, so the skill still requires the Hub surface overall. `rinkata_write_ticket_body` is the backstop when the ticket is still a mint stub. If a direct tool call still returns `Method not found`, surface:

> Tool `<name>` is not available on the active MCP surface. `/rinkata-ticket-start`
> needs Hub MCP — `rinkata_read_ticket` is Hub-only (`rinkata_read_spec` and
> `rinkata_claim` exist on Hub MCP too, but the skill needs the
> Hub-only read). If `rinkata_claim` specifically is missing, the Hub MCP session may be stale — reconnect Hub MCP to refresh the tool surface. Check that
> Hub MCP is connected, authenticated, and reachable.

## Failure modes

If a Hub call fails, show the error and follow the recovery line:

- **Bad tool arguments** — "Tool call rejected: `<message>`." If claim says the caller has no handle: set one via Settings → Members, or reconnect Hub MCP as an agent identity.
- **Not found** — "Ticket `<id>` not found — check the id, or it was deleted mid-run. Re-orient with `/rinkata-orient`."
- **Project not found** — "Project not found. Reconnect Hub MCP so the OAuth session is current."
- **Hub database unavailable** — "Hub database is unavailable — retry shortly." Do not treat as not-found.
- **Hub could not complete the request** — surface the message verbatim. If Hub auth looks stale, reconnect Hub MCP.
- **Not authenticated** — "Hub returned 401. Reconnect Hub MCP to refresh the OAuth session."
- **Tool not available** — see Capability check.
- **Hub unreachable** — "Hub unreachable. Check the service is up and Hub MCP is connected."

## What this skill does NOT do

- It does not claim on a non-safe preflight, and it does not reopen a
  `done` / `archived` ticket. The `done`/`archived` guard reads status from
  step 5's `rinkata_read_ticket`; `rinkata_claim` only advances a ticket that is
  still `todo` (its start is a no-op on any other status), so it will not reopen
  a closed ticket — but it also does not *assign* one it won't start, so the
  skill's read-time guard is what surfaces the "already closed" message rather
  than a silent no-op. The narrow race where a ticket is completed between the
  step-5 read and the claim is caught server-side: `rinkata_claim` leaves a
  non-`todo` ticket's status untouched. The skill's guard is best-effort and
  explains the refusal before the call.
- It does not flip status as a side effect of any other skill — only on an
  explicit invocation of this skill (slash or mid-session pick-up / start /
  claim), never as a side effect of another skill. Claim-only invocations
  (`start: false`) must not flip status.
- It does not create or check out git branches (it only suggests a name).
- It does not surface "linked decisions" — there is no ticket→decision link in
  the data model today; not in this skill.
- It does **not** reimplement parent-fill or sole-branch Goal policy. Parent
  fills happen only inside official Start (`rinkata_claim` / Hub `claim_start`);
  the skill only **reports** `filledParents` from that response and never
  client-writes parent assignees.
- It does not do the implementation work. It hands the agent a clean, claimed
  ticket with full context; the work between start and `/rinkata-ticket-complete`
  is manual / agent labor.
- It does **not** author the implementation plan. If the ticket
  is `todo` and the body is still an Intent + Acceptance mapping stub (no files,
  routing, tests, or dogfood — typically `<!-- rinkata:plan-fingerprint:... -->`
  with only Intent / Acceptance mapping headings), stop and enrich via
  `rinkata_read_ticket` then `rinkata_write_ticket_body` (keep fingerprint +
  protected regions; pass `expectedVersion`; no secrets) **before** claiming
  work. Skip if the body is already a non-stub. If `frontmatter.assignee` is
  another principal, stop without enriching — `write_ticket_body` has no
  assignee gate. If status is `done`, `archived`, or `in_progress`, skip
  enrich (lifecycle before enrich); `write_ticket_body` would otherwise
  rewrite a closed ticket whose Completion block left it looking like a stub.
  Start is for claiming, not for writing the plan.
  On enrich fail: do not Start; retry remaining stubs.

## See also

- **`/rinkata-orient`** — the project-level entry point. Its suggested-next-action
  line emits `/rinkata-ticket-start <id>`; this skill is the other half.
- **`/rinkata-ticket-goal`** — a *different* skill, kept deliberately separate.
  `rinkata-ticket-goal` prepares a session-scoped completion condition (goal /
  evaluator). It has a primary host-agnostic MCP path and an optimized
  experience for Claude Code. It is read-only and never flips status.
  `/rinkata-ticket-start` is the mutating, Hub-MCP-based claim step. Use
  `ticket-goal` to set up the evaluator condition; use `ticket-start` to
  actually pick the ticket up and claim it.

## Why this shape

The context pack is rendered *before* the status flip on purpose: an agent that
flips a ticket to `in_progress` and only then reads its body has already
committed before it has context. Surfacing the ticket body + spec acceptance
first means the agent decides to start with the acceptance criteria already in
view. The preflight gate up front means a ticket with a broken chain (drifted
source, unsatisfied blocker) is never silently claimed — the engineer fixes the
chain first. Both properties make `/rinkata-ticket-start` safe to run on autopilot.

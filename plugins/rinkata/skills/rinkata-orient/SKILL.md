---
name: rinkata-orient
version: 0.1.7
description: |
  One-call orient for any rinkata-backed project. Surfaces project truth
  — counts, drift, doctor verdict, top blockers, and a suggested next
  action. Always reads Hub truth through the active rinkata MCP surface. Use this
  skill at session start for any rinkata project, whenever the user asks "what's
  the state of this project?" or "what should I work on?", or any time
  context-restoration is needed mid-session.


  Read-only: this skill never mutates project state. For mutations
  (decide, ticket-start, ticket-complete) use the dedicated
  skills.
triggers:
  - /rinkata-orient
  - any rinkata project session start
  - "what's the state of this project"
  - "what should I work on"
  - "orient me on rinkata"
  - "status of rinkata"
tools:
  - rinkata_status
  - rinkata_doctor
  - rinkata_list_drift
mutating: false
---

# rinkata Orient — one-call project context

This skill renders a compact orientation against the active rinkata project. In
all modes, Hub DB is the source of truth and repo-local artifact directories are
not consulted. The output is a fixed-shape summary that subsequent skills
(`/rinkata-decide`, `/rinkata-ticket-start`, `/rinkata-ticket-complete`) and human
readers can both consume.

Perf budget

## What to do

1. **Resolve project truth through MCP.**
   Use the active rinkata MCP connection. If Hub auth or project resolution fails,
   fail closed: surface the login/recovery error and stop; do not fall back to
   local files.

2. **Call `rinkata_status(projectId)`.**
   The response has `value.counts` and `value.tickets[]`. Capture both. Counts include:
   - `prds`, `specs`, `tickets`, `decisions`, `inbox`, `reconciles` — artifact totals
   - `aligned`, `stale`, `unknown`, `invalid` — drift breakdown for tickets
   - `specsInFlight`, `specsComplete` — spec lifecycle counts

3. **Call `rinkata_doctor`** with the same `{projectId}` argument shape as `rinkata_status`, in parallel where the harness allows. **Do not pass `verbose: true`** — the default is compact: one `protected-sentinels` summary ok for balanced tickets, plus non-ok rows only. The response has `value.ok` (boolean) and `value.checks[]` with each entry's `severity` (`ok` | `warning` | `error`) and `message`. Capture the count by severity. Use `verbose: true` only when a human needs per-ticket protected-sentinel ok rows.

4. **If `counts.stale > 0`,** call `rinkata_list_drift({projectId})` and take the first 3 entries from the returned non-aligned `value.tickets[]` whose `status === "stale"` by ticket id descending (deterministic, since the payload has no recency field). Skip the call otherwise — it's wasted work on aligned projects. If a legacy MCP surface still returns aligned rows, filter them out before rendering.

5. **Render the orient summary** in this exact shape (Markdown, no fenced code block — this IS the output):

   ```
   ## rinkata orient — <project name from Hub context>

   **Counts:** N Goals · N specs (M in flight, K complete) · N tickets · N decisions · N inbox · N reconciles
   **Drift:** N aligned · N stale · N unknown · N invalid
   **Doctor:** <OK | N warnings | N errors> — <one-line headline of the worst>

   <If counts.stale > 0:>
   **Top stale tickets:**
   - TICK-XXX (recordedHash → currentHash) — <reason>
   - TICK-XXX (recordedHash → currentHash) — <reason>
   - TICK-XXX (recordedHash → currentHash) — <reason>

   **Next action:** <one-line suggestion derived from the heuristic below>
   ```

## Suggested-next-action heuristic

Apply in this order; first match wins:

| # | condition | suggested action |
|---|---|---|
| 1 | Doctor has any `error` checks (`severity === "error"`) | Run `rinkata doctor` (CLI) or call `rinkata_doctor` again; fix the named errors before starting any other work. |
| 2 | `counts.invalid > 0` | Schema-invalid artifacts are blocking. Inspect `value.tickets[]` filtered by `status: "invalid"`. Repair before continuing. |
| 3 | `counts.reconciles > 0` (open reconcile plans awaiting apply) | Reconcile plans block accurate alignment math downstream. Review the open plan(s) and either apply (`rinkata_reconcile_apply`) or discard before continuing. |
| 4 | `counts.unknown > 0` (ticket points at a missing Goal section / spec) | Tickets with unknown sources can't be safely started. Surface the unknown ticket ids; suggest re-attaching them to a covering spec via `rinkata_archive_spec... downstream: migrate` or detaching to a valid Goal section. |
| 5 | `counts.stale > 0` | "Drift on N tickets. Run `/rinkata-reconcile <target>` on the most stale source, then re-orient." (Pick the spec or Goal that the most stale tickets share.) |
| 6 | `counts.specsInFlight > 0` AND any spec is `drafting` with member tickets | "Spec `<id>` is drafting and blocking N member tickets. Approve via `rinkata_set_spec_status(<id>, approved)` if the body + acceptance are ready." |
| 7 | `counts.inbox > 0` AND none of 1-6 apply | "N inbox notes uncaptured. Triage with `/rinkata-decide` to promote each to a decision / ticket / Goal edit before starting new work." |
| 8 | `counts.aligned > 0` AND no urgent above | "Ready to work. Pick an aligned ticket: `/rinkata-ticket-start <id>`." Show the first 3 aligned ticket ids + titles. |
| 9 | Project is empty — `counts.tickets === 0` from Hub truth | "Empty project. Use `/rinkata-decide` to capture the first decision, or import existing work via the Hub UI." |

The order is the contract — downstream skills (`/rinkata-orient`, `/rinkata-ticket-start`, `/rinkata-decide`, `/rinkata-ticket-complete`) compose against the resulting `Next action:` line and assume the highest-priority blocker has already been surfaced. Re-ordering rows changes the contract and must be a deliberate test edit.

## Capability check

At startup, call `requireTools(["rinkata_status", "rinkata_doctor", "rinkata_list_drift"])` against the active MCP surface. If any tool is missing, render the returned parity-gap message, including any note id or alternate surface, and stop before making MCP calls. If a direct tool call still returns `Method not found`, surface:

> Tool `<name>` is not available on the active MCP surface. Hub MCP is the canonical rinkata surface. Check that Hub MCP is connected (OAuth) and reachable.

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

- It does not mutate project state. No reconcile plans, no decisions, no status flips.
- It does not deeply inspect any single ticket (use `/rinkata-ticket-start <id>` for that).
- It does not fix doctor errors. It surfaces them.
- It does not run preflight on a specific ticket. Use `rinkata_preflight_ticket` directly or `/rinkata-ticket-start`.

## Why this shape

Every persona (founder, PM, engineer, reviewer, AI agent) starts here

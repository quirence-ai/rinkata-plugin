# rinkata-ticket-start

The "I'm picking up TICK-X" entry point for any rinkata-backed project. Preflight, claim, context pack — in one call.

## What this does

`/rinkata-ticket-start <ticket-id>`:

1. Reads advisory `rinkata_preflight_ticket`, reports alignment and dependency discrepancies, and continues. The server enforces `PAUSED`, `RECOVERY_HOLD`, access and version integrity.
2. Surfaces the **ticket body** + the **spec's acceptance criteria** into the transcript — the agent has context *before* it commits.
3. Binds the host session to the ticket with **`rinkata_bind_usage_session`** just before the Start, and alone on an already-`in_progress` ticket, so its measured token usage counts from the start of the work. The host adapter supplies the session id. A bind refusal (`USAGE_SESSION_UNKNOWN` or any other) is reported and never stops the Start, and it is never retried with a guessed id. If the server refuses the Start after a bind, the skill uses the bind's `previousTicketId`: nothing when it is this ticket, a bind of that ticket when it names another, an unbind when it is null. It never rebinds from memory or a guess.
4. Claims via **`rinkata_claim`** (official Start): assigns the ticket to the caller and transitions **`todo` → `in_progress`** in one write.
5. After a successful Start (`started: true`), **reports server `filledParents`** (`{kind,id}[]`; explicit `[]` means no fill). If the field is absent, report unavailable (reconnect MCP) — do not coerce missing to empty. Never invents parent fills or reimplements sole-branch / parent-fill rules — Hub `claim_start` owns that policy.

It is the mutating sibling of the read-only `/rinkata-orient`: orient answers "what should I work on", ticket-start claims the chosen ticket.

## When it fires

- `/rinkata-ticket-start <ticket-id>` slash command
- Right after `/rinkata-orient` suggests a ticket
- "start ticket TICK-…", "pick up TICK-…"

## Behavior contract

- **Preflight is advisory.** Stale, blocked and unknown alignment are context for work.
- **Context before claim.** The ticket body + spec acceptance are rendered before `rinkata_claim` is called.
- **Invocation is the confirmation.** Typing the command is the explicit start intent — no second prompt. The skill never flips status as a side effect of anything else.
- **Idempotent re-run.** On an already-`in_progress` ticket it re-renders the context pack, binds the session, and does **not** call `rinkata_claim` (avoids steal). On a `done` / `archived` ticket it refuses (no surprise reopen).
- **Direct operations.** `rinkata_update_ticket` starts, reopens and edits title or assignee; `rinkata_claim` takes ownership and can Start. Drafting specs allow Start. A reopen binds the session first, then reopens.
- **filledParents reporting.** Always surface `filledParents` from the Start response when `started: true` — print kind+id for each entry, or state explicitly that the array is empty. Do not client-write parent assignees.
- **Branch creation is opt-in** — the skill suggests a name; it never runs git.
- **Scenario coverage.** Tests lock happy path, stale/blocker paths, no-spec / spec-anchored paths, and filledParents reporting.
- **Linked decisions are opportunistic.** The context pack renders explicit decision links when the active tool surface provides them; otherwise it says none were found.

## Related

- `rinkata-orient` — the project-level entry point that suggests which ticket to start
- `rinkata-ticket-goal` — Host-agnostic session goal / completion condition prep (with optimized Claude Code path); deliberately separate from this skill (this one mutates status)
- `rinkata-ops` — ambient Hub-truth contract

## How to test it

Install this plugin from cursor.directory (or copy it locally as in the repo README) and connect Hub MCP with OAuth.
In a Hub-backed project, run `/rinkata-ticket-start <ticket-id>` against a `todo` ticket. Expect the preflight verdict, the context pack, the usage bind, the claim/start, then a `filledParents` report (list or empty).

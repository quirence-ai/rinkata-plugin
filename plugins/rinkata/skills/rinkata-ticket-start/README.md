# rinkata-ticket-start

The "I'm picking up TICK-X" entry point for any rinkata-backed project. Preflight, claim, context pack — in one call.

## What this does

`/rinkata-ticket-start <ticket-id>`:

1. Runs `rinkata_preflight_ticket` and **refuses on a broken chain** — a stale source, an unsatisfied `blocked_by`, an unknown source. Never claims a ticket that isn't safe to start.
2. Surfaces the **ticket body** + the **spec's acceptance criteria** into the transcript — the agent has context *before* it commits.
3. Claims via **`rinkata_claim`** (official Start): assigns the ticket to the caller and transitions **`todo` → `in_progress`** in one write.
4. After a successful Start (`started: true`), **reports server `filledParents`** (`{kind,id}[]`; explicit `[]` means no fill). If the field is absent, report unavailable (reconnect MCP) — do not coerce missing to empty. Never invents parent fills or reimplements sole-branch / parent-fill rules — Hub `claim_start` owns that policy.

It is the mutating sibling of the read-only `/rinkata-orient`: orient answers "what should I work on", ticket-start claims the chosen ticket.

## When it fires

- `/rinkata-ticket-start <ticket-id>` slash command
- Right after `/rinkata-orient` suggests a ticket
- "start ticket TICK-…", "pick up TICK-…"

## Behavior contract

- **Preflight gates.** Non-safe preflight → blocker + recovery hint, no status change.
- **Context before claim.** The ticket body + spec acceptance are rendered before `rinkata_claim` is called.
- **Invocation is the confirmation.** Typing the command is the explicit start intent — no second prompt. The skill never flips status as a side effect of anything else.
- **Idempotent re-run.** On an already-`in_progress` ticket it re-renders the context pack and does **not** call `rinkata_claim` (avoids steal). On a `done` / `archived` ticket it refuses (no surprise reopen).
- **Non-approved-spec guard.** A spec-anchored ticket only starts once its spec is `approved`. Skill-side copy explains the refusal; Hub `rinkata_claim` also enforces the Spec gate server-side.
- **filledParents reporting.** Always surface `filledParents` from the Start response when `started: true` — print kind+id for each entry, or state explicitly that the array is empty. Do not client-write parent assignees.
- **Branch creation is opt-in** — the skill suggests a name; it never runs git.
- **Scenario coverage.** Tests lock happy path, stale/blocker paths, no-spec / spec-anchored paths, and filledParents reporting.
- **Linked decisions are opportunistic.** The context pack renders explicit decision links when the active tool surface provides them; otherwise it says none were found.

## Related

- `rinkata-orient` — the project-level entry point that suggests which ticket to start
- `rinkata-ticket-goal` — Host-agnostic session goal / completion condition prep (with optimized Claude Code path); deliberately separate from this skill (this one mutates status)
- `rinkata-ops` — ambient Hub-truth contract
- filledParents reporting after official Start
- original implementation ticket; DEC-016 — eng-reviewed design decision

## How to test it

Install this plugin from the Cursor Marketplace (or symlink locally as in the repo README) and connect Hub MCP with OAuth.
In a Hub-backed project, run `/rinkata-ticket-start <ticket-id>` against a `todo` ticket. Expect the preflight verdict, the context pack, the claim/start, then a `filledParents` report (list or empty).

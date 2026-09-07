# rinkata-orient

One-call orient for any rinkata-backed project. The first skill every persona — founder, PM, engineer, reviewer, AI agent — runs at session start.

## What this does

Renders a compact orientation against the active rinkata project: counts, drift, doctor verdict, top stale items (when present), and a single suggested next action. Two MCP calls (`rinkata_status` + `rinkata_doctor`) in parallel; a third (`rinkata_list_drift`) only when there's drift to report.

The output shape is fixed so downstream skills (`/rinkata-orient`, `/rinkata-ticket-start`, `/rinkata-decide`, `/rinkata-ticket-complete`) and human readers can both parse it.

## When it fires

- `/rinkata-orient` slash command
- Session start for a rinkata project
- User asks "what's the state of this project?" or "what should I work on?"


## How to test it

Install this plugin from the Cursor Marketplace (or symlink locally as in the repo README) and connect Hub MCP with OAuth.
In a rinkata-backed project, type `/rinkata-orient` (Claude Code) or invoke the equivalent in your harness. Expect the fixed-shape orient summary in <3s.

## Related

- `rinkata-ops` — ambient files-as-truth contract (always-on)
- implementation ticket for this skill
- capability/parity registry that this skill will call into for tool-availability checks (pending)

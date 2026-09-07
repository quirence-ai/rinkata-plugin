# rinkata-feedback

A frictionless channel for a human or an AI agent to volunteer free-form feedback about using rinkata itself — friction, a confusing error, a surprising or a genuinely great result.

## What this does

Teaches the agent to call the `rinkata_submit_feedback` MCP tool with a concise, specific message (and optional structured `context`). The submission is stored durably and surfaced to the rinkata team in Hub → Admin → Feedback, where it can be resolved, dismissed, or promoted into the inbox → ticket pipeline.

This is product feedback **to the rinkata team**, not project work. A bug or task in the user's own project belongs in `rinkata_inbox_add` / a ticket.

## When it fires

- `/rinkata-feedback` slash command
- The user says "send feedback" / "tell the rinkata team …"
- The agent hits something worth reporting while using rinkata

## Relationship to the rest of rinkata

A human or agent volunteers the signal directly. The CLI equivalent for humans is `rinkata feedback "<message>"`, which buffers offline and sends on the next command.

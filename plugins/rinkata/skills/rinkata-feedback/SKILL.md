---
name: rinkata-feedback
version: 0.1.1
description: |
  Volunteer free-form feedback about using rinkata itself — a moment of
  friction, a confusing error, a surprising or a genuinely great result —
  by calling the rinkata_submit_feedback MCP tool. This is product feedback
  to the rinkata team, not project work: it lands in the team's triage queue
  (Hub → Admin → Feedback), where it can be resolved or promoted into the
  inbox → ticket pipeline.

  Use this skill when the user says "send feedback", "/rinkata-feedback",
  "tell the rinkata team …", or when you (the agent) hit something worth
  reporting while using rinkata. A human or agent volunteers the signal directly.
triggers:
  - /rinkata-feedback
  - "send feedback"
  - "give feedback on rinkata"
  - "tell the rinkata team"
  - hitting friction / a confusing error / a surprising result while using rinkata
tools:
  - rinkata_submit_feedback
mutating: false
---

# rinkata Feedback — volunteer a signal to the rinkata team

A frictionless way to say "this just happened and it was confusing / wrong /
great" while using rinkata. One MCP call; never blocks the work in progress.

## When to use it

- The user explicitly asks to send feedback, or types `/rinkata-feedback`.
- You hit friction worth reporting: a confusing error message, a tool that did
  something surprising, a workflow that took more steps than it should, or a
  result that was notably good and worth reinforcing.

This is **not** for project work. A bug or task *in the user's own project*
goes through `rinkata_inbox_add` / a ticket. Feedback here is about **rinkata the
product** and is read by the rinkata team.

## How to call it

```
rinkata_submit_feedback({
  body: "<concise, specific feedback in your own words>",
  context: { artifact_id: "", command: "reconcile" } // optional
})
```

- `body` (required): keep it short and concrete. Quote the exact error or the
  exact step that was confusing. "The reconcile diff didn't show which side was
  current" beats "reconcile is confusing".
- `context` (optional): structured hints the team can act on — the active
  ticket/spec id, the command or tool that produced the friction. Omit if you
  have nothing specific.

The tool records the submission and returns its id (e.g. `FBK-0007`). Surface
that id back to the user as confirmation. Do not retry on success, and never let
a feedback call block or derail the task you were doing.

## Humans at the terminal

The CLI equivalent is `rinkata feedback "<message>"`, which the user can run
directly. It buffers offline and sends on the next command, so feedback is never
lost. You don't need to invoke the CLI for them — call the MCP tool.

---
name: rinkata-trash-restore
version: 0.1.0
description: |
  Recover or inspect archived rinkata work without silent resurrection. Surfaces
  archived specs/tickets and requires an explicit restore/re-anchor decision.
triggers:
  - /rinkata-trash-restore
  - trash restore
  - restore archived
tools:
  - rinkata_status
  - rinkata_doctor
  - rinkata_read_ticket
  - rinkata_read_spec
mutating: false
---

# rinkata trash-restore — archived work recovery

## Capability check

At startup, call `requireTools(["rinkata_status", "rinkata_doctor", "rinkata_read_ticket", "rinkata_read_spec"])`
against the active MCP surface. If any tool is missing, render the returned
parity-gap message and stop before reading archived artifacts.

Use this to inspect archived or detached work. The skill is read-only until a
future restore tool exists. Surface the artifact, its downstream links, and the
safe restore options: leave archived, re-anchor to an approved spec, migrate to a
new spec, or create a fresh ticket.

Never silently reopen `done` or `archived` work through generic status flips.

## Behavioral test contract

- Tool sequence: `requireTools` → `rinkata_status` → `rinkata_doctor` →
  `rinkata_read_ticket` and/or `rinkata_read_spec` for the archived target.
- Refusal cases: Hub unreachable, missing tools, missing artifact, and unclear
  restore intent all stop without mutation.
- The skill is read-only: it may recommend leave archived, re-anchor, migrate,
  archive downstream, or create fresh work, but it must not call status-write or
  completion tools.

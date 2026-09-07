---
name: rinkata-import-project
version: 0.1.2
description: |
  Bring an existing project under rinkata without guessing. Inventories current
  Goals/specs/tickets, identifies source-of-truth gaps, and proposes import work.
triggers:
  - /rinkata-import-project
  - import project
  - rinkata init
tools:
  - rinkata_status
  - rinkata_doctor
  - rinkata_inbox_add
mutating: true
---

# rinkata import-project — controlled onboarding

## Capability check

At startup, call `requireTools(["rinkata_status", "rinkata_doctor", "rinkata_inbox_add"])`
against the active MCP surface. If any tool is missing, render the returned
parity-gap message and stop before inventorying or writing notes.

Use when a team wants to bring existing docs/tickets into rinkata. Start with
`rinkata_status` and `rinkata_doctor`. If the project is empty, capture import notes
with `rinkata_inbox_add`; do not fabricate Goals, specs, or tickets without review.

For Hub-first team projects, import work should land in Hub truth first. File
mirrors are export/cache surfaces, not the authority.

## Behavioral test contract

- Tool sequence: `requireTools` → `rinkata_status` → `rinkata_doctor` → optional
  `rinkata_inbox_add` for reviewed import notes.
- Refusal cases: Hub unreachable, doctor errors, missing tools, and non-empty
  projects with uncaptured drift stop before creating import notes.
- The skill inventories and proposes; it must not fabricate Goals, specs,
  tickets, or local artifact edits without explicit review.

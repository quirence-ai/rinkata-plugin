# Human Note Protection

Rinkata artifacts have sections marked as protected human notes. These are the
user's words and intent. Reconcile plans never overwrite them.

## Rule

Protected notes are read-only to agents. Reconcile plans must preserve them
verbatim. `rinkata_preflight_ticket` fails if a ticket's protected notes have
been mutated.

## Why

The user's intent is the highest-authority signal in the project. If an agent
silently rewrites it, the project's truth is no longer the user's truth.

## How to apply

- Identify protected sections by their frontmatter / fence markers
  (`rinkata_doctor` surfaces these).
- When proposing a reconcile, include only mechanical changes (hash updates,
  ticket-state transitions, structural fixes). Never paraphrase or "improve"
  protected prose.
- If a protected note appears wrong, raise it to the user. Do not edit.

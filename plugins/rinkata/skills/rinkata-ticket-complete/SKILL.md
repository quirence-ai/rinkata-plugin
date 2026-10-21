---
name: rinkata-ticket-complete
version: 0.1.15
description: 'Close a rinkata ticket with optional evidence. Reads advisory preflight, then calls rinkata_complete_ticket so status, completed_at, completion_evidence, and the protected Completion block are written by the tool. Load mid-session without a slash when the work has shipped (rinkata_complete_ticket) or the user wants only to record what we now know (rinkata_knowledge_record; do not complete).'
triggers:
  - /rinkata-ticket-complete
  - "complete ticket TICK-"
  - "close ticket TICK-"
  - "record what we now know"
  - "ship this ticket"
  - "knowledge write-back"
tools:
  - rinkata_preflight_ticket
  - rinkata_read_ticket
  - rinkata_read_spec
  - rinkata_begin_demo_screenshot_upload
  - rinkata_finalize_demo_screenshot_upload
  - rinkata_upload_demo_screenshot
  - rinkata_complete_ticket
  - rinkata_knowledge_list
  - rinkata_knowledge_read
  - rinkata_knowledge_record
mutating: true
argument-hint: <ticket-id> [--evidence <evidence>]
metadata:
  grok:
    notes: "Pair well with a dedicated verification subagent that runs the final preflight + evidence check before calling rinkata_complete_ticket."
---

# rinkata ticket-complete — record a completion

Use this skill when the implementation has shipped and the ticket should become
`done`. Also load mid-session (no slash) to record what we now know via
`rinkata_knowledge_record` (do not complete) or, when shipping, optional
`knowledgeWriteBack` on `rinkata_complete_ticket`. Hub is the only source of ticket truth;
use Hub MCP (OAuth) and do not read local artifact files.

## Capability check

At startup, call `requireTools(["rinkata_preflight_ticket", "rinkata_read_ticket", "rinkata_read_spec", "rinkata_complete_ticket", "rinkata_knowledge_list", "rinkata_knowledge_read", "rinkata_knowledge_record"])` against the active MCP surface. If any tool is missing, render the returned parity-gap message, including any note id or alternate surface, and stop before preflight. `rinkata_begin_demo_screenshot_upload` / `rinkata_finalize_demo_screenshot_upload` (and legacy `rinkata_upload_demo_screenshot`) are optional — when walkthrough files exist and the tools are absent, warn and continue Complete without screenshot tiles (do not hard-stop non-FE tickets).

## What to do

1. Resolve project truth through the active Hub MCP surface. If Hub auth or
   project resolution fails, stop and surface the repair path.
2. Parse the ticket id. Prefer the slash argument, then a `TICK-` id in the
   user utterance or the current session context. If none is available, surface
   usage (`/rinkata-ticket-complete <ticket-id> [--evidence …]`) and stop.
   **Mode:** "record what we now know" / "knowledge write-back" without ship /
   complete / close is **knowledge-only**. Slash complete/close, "complete
   ticket", and "ship this ticket" are **Complete**.
3. **Knowledge-only is exclusive.** Read the ticket. Then **discover** covering
   Knowledge — `rinkata_read_ticket` has no `knowledgeCovering`. Call
   `rinkata_knowledge_list` (existing areas + docs; `areaId: "unfiled"` for
   Unfiled). Do not invent an area or a document id. If a covering / feature
   doc is listed, `rinkata_knowledge_read` it, then
   `rinkata_knowledge_record` with `documentId` + `heading`/`mode`
   (`append`|`replace`|`new_heading`|`replace_document`). Repair a whole listed
   doc with `mode=replace_document` (or heading-less `mode=replace`); heading-ful
   `replace` still requires a heading. `append` / `new_heading` stay additive.
   If none, file into an existing area or
   Unfiled — do not mint a parallel `What we learned — TICK-X` note when a
   feature doc exists. Do **not** require evidence, do **not** run preflight,
   do **not** call `rinkata_complete_ticket`. Stop.
4. **Complete only.** Evidence is optional. Attach a real PR URL, commit SHA,
   test run, smoke result or demo note when available; do not invent evidence.
   Call `rinkata_preflight_ticket` for context. Its source-alignment status is
   `safe | stale | blocked | unknown`, never a lifecycle value like `done`.
   Preflight is advisory: stale pins, missing upstream sources, unsatisfied
   dependencies and a drafting spec do not stop completion. Surface the
   discrepancies and continue. The server enforces the remaining refusals:
   `TICKET_ALREADY_CLOSED`, `PAUSED`, `RECOVERY_HOLD` and `PARENT_NOT_FOUND` (subtasks only), plus access, version
   and attribution integrity. Report those refusals and their recovery.
5. Call `rinkata_read_ticket`. If the ticket frontmatter `status` is already
   `done` or `archived`, report the existing `completion_evidence` and do not call `rinkata_complete_ticket`.
   STOP — re-complete overwrites evidence and the server refuses
   `TICKET_ALREADY_CLOSED`. If the user also asked to record what we now know
   on that closed ticket, discover covering Knowledge (`rinkata_knowledge_list`
   / `rinkata_knowledge_read`) then `rinkata_knowledge_record` as in step 3,
   and stop.
   Otherwise, if the ticket has a `spec`, call `rinkata_read_spec` and render
   the spec acceptance criteria before closing.
6. For FE/UI work: if walkthrough screenshots exist (Cursor Cloud screenshot captures, or equivalent), capture at a
   viewport of at least 1440×900 with deviceScaleFactor: 2. Hub hard-refuses smaller or soft/mushy bitmaps
   before store (`SCREENSHOT_TOO_SMALL` / `SCREENSHOT_TOO_SOFT`). Do not
   re-shot a Complete tile. Upload max is 8 MiB / 40 million pixels.
   Review each capture before upload: do not upload one that shows secrets, credentials, tokens, or personal/customer data — recapture with test data or crop it first. Upload each with
   `rinkata_begin_demo_screenshot_upload({ projectId, contentType, byteLength, sha256?, label })`,
   then `curl -fsS -X PUT --upload-file ./shot.webp` using the returned `uploadUrl` and
   `X-Rinkata-Upload-Token` (never paste image bytes into a tool call), then
   `rinkata_finalize_demo_screenshot_upload({ projectId, uploadId })`, and pass only
   the returned Hub URL in `screenshots[]`. Tiles render in spec/ticket
   drawers, not on Complete cards. A Cursor agent artifact page is
   auth-gated HTML. It is stored as type url, not a tile. Legacy
   `rinkata_upload_demo_screenshot({ contentBase64, label })` is for small images only.
   If the begin/finalize tools are missing, warn and
   Complete without screenshots.
   Then call `rinkata_complete_ticket({ ticketId, evidence?, screenshots?,
   testReport?, knowledgeWriteBack? })`. Never hand-edit `status: done`,
   `completed_at`, or `completion_evidence`. If we learned something that
   belongs in Knowledge, pass optional
   `knowledgeWriteBack: { body, title?, areaId?, documentId?, heading?, mode? }`.
   Prefer `documentId` + `heading`/`mode` (`append`|`replace`|`new_heading`|`replace_document`) when
   preflight `knowledgeCovering` lists a feature doc — do not mint a parallel
   `What we learned — TICK-X` note. Use `mode=replace_document` (or heading-less
   `mode=replace`) for a whole-document repair; heading-ful `replace` still needs
   a heading. `areaId` must already exist; omit for Unfiled.
   Write-back is best-effort — Complete still succeeds if ingest fails
   (`GEMINI_NOT_CONFIGURED`). Do not invent a new area or document id. After the
   fact, `rinkata_knowledge_record` is the same write path.
7. Render the returned completion evidence and tell the user to run
   `/rinkata-status-report` or `rinkata doctor` if they want a final health check.

## Server-side contract

The server is authoritative. `rinkata_complete_ticket` records a completion
with or without evidence. `no_evidence`, `out_of_date`, `spec_not_approved` and
`alignment_unknown` are warnings, not refusals. An existing `done` or `archived`
ticket refuses with `TICKET_ALREADY_CLOSED`; a person's pause (`PAUSED`) and
an unreadable requirement record (`RECOVERY_HOLD`) and `PARENT_NOT_FOUND` (subtasks only) still refuse. Reopen a done
ticket with `rinkata_update_ticket`, preserving its previous completion facts.
An agent reports completion and attaches evidence; only a person verifies it.

## Behavioral test contract

- Tool sequence: `requireTools` → `rinkata_preflight_ticket` → `rinkata_read_ticket`
  → optional `rinkata_read_spec` → `rinkata_complete_ticket`.
- Tool sequence (knowledge-only): `requireTools` → `rinkata_read_ticket` →
  `rinkata_knowledge_list` → optional `rinkata_knowledge_read` →
  `rinkata_knowledge_record` (`documentId` when a covering doc is listed).
  Do not call `rinkata_complete_ticket`. Do not invent a document id.
- Missing evidence, stale or unknown alignment, open dependencies and drafting
  specs report discrepancies and continue to the completion operation.
- Refusal cases: an already `done`/`archived` ticket, `PAUSED`, `RECOVERY_HOLD`, `PARENT_NOT_FOUND` (subtasks only),
  access or version failures, and Hub unreachable are surfaced without bypass.
- Completion evidence must be written only by `rinkata_complete_ticket`; never
  edit ticket frontmatter or the protected Completion block by hand.

# Stale Hash Protocol

A stale source hash means the artifact's understanding of the code is out of date.
Rinkata refuses to write through a stale hash on purpose — that's how it prevents
silent drift between docs and code.

## Rule

Never edit a source hash directly. Never bypass `STALE_ARTIFACT`. Re-read project
state and re-issue the operation.

## Why

Hashes are the alignment proof between artifacts and code. Manually "fixing" one
silently asserts something the agent hasn't verified. Rinkata's whole guarantee
collapses if hashes drift from reality.

## How to apply

On `STALE_ARTIFACT` response:

1. Call `rinkata_status` to refresh project state.
2. Re-read the affected artifact through Hub MCP, not from local markdown
   files.
3. Re-issue the original operation. The hash is recomputed from current truth.
4. If it still fails, surface to the user — there's a real divergence the
   agent cannot resolve alone.

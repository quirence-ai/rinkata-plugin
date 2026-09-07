# rinkata plugin (Cursor)

Cursor plugin for [rinkata](https://rinkata.dev) by Quirence. Hub source-of-truth skills plus MCP.

Version: `0.8.13`

## Install

From the Cursor Marketplace once listed, or locally:

```bash
mkdir -p ~/.cursor/plugins/local
ln -s "$(pwd)/plugins/rinkata" ~/.cursor/plugins/local/rinkata
```

Then Developer: Reload Window.

Layout matches [cursor/plugin-template](https://github.com/cursor/plugin-template):

- `.cursor-plugin/marketplace.json` — catalog
- `plugins/rinkata/.cursor-plugin/plugin.json` — plugin manifest
- `plugins/rinkata/mcp.json` — keyless Hub HTTP (`https://api.rinkata.dev/mcp`)
- `plugins/rinkata/assets/logo.svg` — 1:1 mark
- `plugins/rinkata/skills/` — Cursor-targeted skills
- `scripts/validate-template.mjs` — Cursor plugin-template validator

Skills in this bundle (10): `rinkata-feedback`, `rinkata-idea`, `rinkata-import-project`, `rinkata-ops`, `rinkata-orient`, `rinkata-reconcile`, `rinkata-status-report`, `rinkata-ticket-complete`, `rinkata-ticket-start`, `rinkata-trash-restore`.

MCP is the live Hub server, not a `rinkata-mcp` binary. Auth is OAuth.

Validate:

```bash
node scripts/validate-template.mjs
```

## License

MIT (Quirence).

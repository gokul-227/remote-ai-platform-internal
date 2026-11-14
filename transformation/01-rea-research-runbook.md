# Phase 1 — REA setup verification and research runbook (14 November 2026)

## Installation state (verified, not re-installed)

- Package: `rea-agents@6.8.0` at `~/.local/share/rea-tool/node_modules/rea-agents` (lifecycle scripts disabled at install).
- Runtime: isolated Node 22.23.4 at `~/.local/share/rea-runtime` (system Node 26.5 does not satisfy the engine pin).
- Official clone for reference: `~/Developer/GitHub/rea` at `0beedcd` (`morluto/rea`, main).
- Registration: `~/.claude.json` `mcpServers` contains `rea` next to `chrome-devtools`, `drawio`, `postman`, `MCP_DOCKER`; none of the other entries were touched. Setup backups: `~/.claude.json.rea.backup` and the REA setup plans under `~/.local/share/rea-tool`.
- Workflow skill: `reverse-engineer-anything` is listed in this Claude Code session's skills.
- Setup was not repeated: the 5 Nov doctor output already reported both scoped clients healthy and 229 tools; re-running `npx rea-agents setup` would only rewrite an aligned registration.

## In-session invocation gate (passed)

Both calls were made from this Claude Code session inside VS Code, through the `rea` MCP server, not from a terminal:

| Call | Result |
|---|---|
| `inspect_analysis_settings` (javascript.analysis_timeout_ms, javascript.max_heap_mb) | Returned runtime defaults (300000 ms, 1024 MiB), response ceiling 10 MiB, parent heap 4.4 GB. |
| `analyze_javascript_application` on owned `apps/admin/src`, summary detail, 120 s budget | Evidence `ev_0141c3e3…` (parent) and `ev_12d65272…` (summary view). 10 TypeScript files parsed, 0 parse failures, 92,615 text bytes, 15,861 AST nodes, 245 static findings (observations, not confirmed bugs). Semantic families reported partial coverage with 1,898 explicit unknowns. |

The owned-source analysis is a static reading of inert syntax; it does not establish runtime behaviour, authorization or route reachability. It is kept as proof that REA runs from this session, not as a product finding.

## Authorization boundaries used in Phase 2

1. REA static/structural tools: only on Remote AI Platform source, owned fixtures and the platform's own runtime.
2. Third-party reference products: official public documentation and help centres first; ordinary user-facing observation through the owner's already signed-in browser only where the 5 Nov session already recorded redacted navigation evidence (`../go-live/evidence-2026-11-05/`). No new scraping, no source-map, storage, cookie or token extraction, no anti-bot evasion, no decompilation of competitor binaries. Native macOS windows of Slack/Zoom/Brave/Word/Excel/PowerPoint were observed on 5 Nov through accessibility metadata only; that evidence is reused, not repeated.
3. Facts are labelled in every research document as: directly observed, officially documented, reasonable inference, or unknown internal detail.

## Recheck procedure

```bash
~/.local/share/rea-runtime/node_modules/node/bin/node \
  ~/.local/share/rea-tool/node_modules/rea-agents/scripts/rea.mjs doctor --client claude_code --skill --json
```

Then, inside Claude Code, call any `mcp__rea__*` tool (for example `inspect_analysis_settings`). If the server is missing from the session, restart the IDE; do not re-run setup unless the doctor reports a misaligned registration.

## Rollback

Restore only the `rea` entry of `~/.claude.json` from `~/.claude.json.rea.backup` after diffing, keeping later edits and the other servers. Remove the skill directory REA installed if the workflow skill must go. No repository file carries machine-specific MCP configuration.

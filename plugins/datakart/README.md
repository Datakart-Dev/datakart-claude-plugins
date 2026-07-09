# DataKart Claude Code Plugin

DataKart connects Claude Code to hosted DataKart MCP tools for AI-native prospecting, lead-file export, and email/phone waterfall enrichment.

## Included

- `skills/datakart-prospecting`: Lite-first people/company prospecting, patient session polling, `search_versions[]` / `version_id` selection, file export, and full-file delivery rules (7-day prospecting file retention, no download URLs to users).
- `skills/datakart-enrichment`: identity-mapping rules, credit costs (email waterfall 2/row, phone 20/row), bulk enrichment up to 50,000 rows, the preflight -> explicit confirm_spend -> start -> poll -> download chain, and full enriched-CSV delivery (6-month retention).
- `agents/`: end-to-end prospecting and enrichment operators for long-running workflows.
- `hooks/hooks.json`: Claude PreToolUse guard that blocks credit-consuming enrichment starts unless explicit user spend approval is visible.
- `.mcp.json`: points Claude Code at the hosted DataKart MCP endpoint.

Bulk email/phone validation tools are currently disabled on the server and are intentionally not documented as available.

## Hosted MCP

`https://mcp.datakart.ai/mcp`

The MCP server handles OAuth, workspace selection, tool execution, progress polling, and DataKart UI handoff. This plugin repo contains only public install metadata, workflow guidance, Claude skills, Claude agents, hooks, and branding assets.

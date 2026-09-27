# DataKart Claude Plugins

Official DataKart plugin marketplace for **Claude Code**, **Claude Cowork**, and **Claude** (claude.ai). One plugin, three surfaces: natural-language prospecting, lead-file export, and credit-aware email/phone waterfall enrichment — all through the hosted DataKart MCP connector at `https://mcp.datakart.ai/mcp` .

## What's inside

```
plugins/datakart/
  .claude-plugin/plugin.json                    # plugin manifest
  .mcp.json                                     # hosted DataKart MCP endpoint (OAuth handled server-side)
  skills/
    datakart-prospecting/SKILL.md               # find people/companies, versions, export, full-file delivery
    datakart-enrichment/SKILL.md                # email/phone waterfalls, mapping, confirm-spend, downloads
  agents/
    datakart-prospecting-operator.md            # runs prospecting end-to-end autonomously
    datakart-enrichment-operator.md             # runs enrichment end-to-end with spend guardrails
  hooks/hooks.json                              # PreToolUse guard on credit-spending enrichment calls
  assets/                                       # DataKart branding and screenshots
```

## Install

### Claude Code

```bash
/plugin marketplace add Datakart-Dev/datakart-claude-plugins
/plugin install datakart@datakart
```

Or for local testing from a checkout:

```bash
claude --plugin-dir ./plugins/datakart
```

On first tool call, the MCP connector walks you through DataKart OAuth in the browser. No API keys or environment variables are needed — auth, workspace selection, and usage accounting all live behind the connector.

### Claude Cowork

Add this repository as a plugin source in Cowork, or install the `datakart` plugin from the marketplace above. Skills, agents, and the MCP connector work identically.

### Claude (claude.ai)

The two skill folders are standard Claude skills. Upload `skills/datakart-prospecting/` and `skills/datakart-enrichment/` as skills, and connect the DataKart MCP connector (`https://mcp.datakart.ai/mcp`) as a custom connector in Settings.

## The workflow the plugin encodes

```
prospecting_start (Lite search, 100 credits / up to 10k records)
  → prospecting_get_session (poll patiently; answer follow-up questions)
  → pick version_id from search_versions[] (non-zero total_results)
  → pull_prospecting_results_in_file (session_id + version_id)
  → download_prospecting_file            # prospecting files live 7 days
  → bulk_enrichment_from_prospecting_file (mapping + preflight)
  → user approves spend → bulk_enrichment_start (confirm_spend=true)
  → bulk_enrichment_progress (poll)
  → bulk_enrichment_download             # enriched files live 6 months
```

Non-negotiables baked into the skills and agents:

- Lite search tools (`our_people_search` / `our_company_search`) 95% of the time; Deep search is ~100x the cost and only a last resort.
- No credit spend without showing the preflight and getting explicit user approval (also enforced by a PreToolUse hook).
- Enriched contact data appears only in `email__d_1..n` / `phone__d_1..n` columns; raw email/phone columns are intentionally empty.
- Downloads are always performed by the model and delivered to the user **in full** — never as a raw URL (MCP auth does not work in user browsers).

## Versioning

`0.2.0` — initial multi-surface release. Bulk email/phone validation tools are disabled server-side and intentionally not documented as available; a `datakart-validation` skill will be added when the server re-enables them.

# DataKart Claude Code Plugins

Official DataKart plugin marketplace for Claude Code. The repository packages the DataKart plugin with Claude-native skills, subagents, hooks, MCP configuration, branding assets, and marketplace metadata.

## Marketplace

Install from this repository as a Claude Code plugin marketplace:

```bash
/plugin marketplace add Datakart-Dev/datakart-claude-plugins
/plugin install datakart@datakart
```

For local testing from a checkout:

```bash
claude --plugin-dir ./plugins/datakart
```

## What DataKart Adds

DataKart brings AI-native prospecting and bulk enrichment into Claude Code:

- Find people, companies, accounts, and decision makers from natural-language ICP prompts.
- Export selected prospecting search versions into complete files.
- Run email or phone waterfall enrichment on prospecting exports or uploaded CSVs.
- Support AI-native bulk enrichment workflows up to 50,000 rows.
- Preflight enrichment jobs before spend with row validation, dropped-row reasons, credit estimates, and balance checks.
- Require explicit user approval before credit-consuming enrichment starts.
- Track long-running jobs and deliver completed enriched CSV data directly back as files.

The hosted MCP connector handles OAuth, workspace selection, tool execution, progress polling, DataKart UI handoff, and usage accounting. No API keys or local secrets are required in this repo.

## Repository Layout

```text
.claude-plugin/
  marketplace.json
plugins/
  datakart/
    .claude-plugin/plugin.json
    .mcp.json
    README.md
    agents/
      datakart-prospecting-operator.md
      datakart-enrichment-operator.md
    assets/
      Datakart-logo.png
      Datakart-logo.svg
      datakart-widget.png
      prospecting-preview.png
      prospecting-follow-up.png
    hooks/
      hooks.json
    skills/
      datakart-prospecting/SKILL.md
      datakart-enrichment/SKILL.md
```

Claude Code requires `skills/`, `agents/`, `hooks/`, and `.mcp.json` to live at the plugin root beside `.claude-plugin/`, not inside `.claude-plugin/`.

## Core Workflow

```text
prospecting_start
  -> prospecting_get_session
  -> select version_id from search_versions[]
  -> pull_prospecting_results_in_file
  -> download_prospecting_file
  -> bulk_enrichment_from_prospecting_file or bulk_enrichment_upload_csv
  -> show preflight cost and row validation
  -> explicit user approval
  -> bulk_enrichment_start
  -> bulk_enrichment_progress
  -> bulk_enrichment_download
```

## Guardrails

- Prospecting defaults to Lite people/company search for the best cost profile.
- Deep search is treated as an escalation path only when Lite results are unusable.
- Prospecting files are delivered in full, never as raw download URLs.
- Enrichment requires preflight review and explicit spend approval before `confirm_spend=true`.
- Enrichment output is delivered directly as a complete file, preserving every row and column.
- Raw prospecting `email` and `phone` columns can be empty by design; waterfall output appears in `email__d_1..n` and `phone__d_1..n`.

## Hosted MCP

```text
https://mcp.datakart.ai/mcp
```

## Version

`0.2.0`

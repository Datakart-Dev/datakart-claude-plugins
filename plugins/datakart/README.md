# DataKart for Claude

DataKart brings B2B prospecting and contact enrichment into Claude Code, Claude Cowork, and Claude. Describe your ideal customer in plain language, and the plugin finds matching people and companies, exports lead files, and runs credit-aware email and phone waterfall enrichment through the hosted DataKart MCP connector.

## What you can do

- **Find prospects**: describe your ideal customer (role, location, company size, and more) in natural language to search people and companies, refine the search, and compare search versions.
- **Export lead files**: pull full prospecting results into a file and have them delivered to you in the conversation.
- **Enrich contacts**: run email and phone waterfall enrichment on a prospecting file or an uploaded CSV, with a cost preview before any credits are spent.
- **Track jobs**: monitor bulk enrichment progress and download enriched results.

## What's included

- **Skills**: `datakart-prospecting` and `datakart-enrichment`, which teach Claude the DataKart workflow, from search through export and enrichment.
- **Agents**: `datakart-prospecting-operator` and `datakart-enrichment-operator`, which run each workflow end to end.
- **Hook**: a `PreToolUse` prompt hook that blocks credit-spending enrichment calls unless you have seen the cost preview and explicitly approved the spend.
- **MCP connector**: the hosted DataKart server at `https://mcp.datakart.ai/mcp`.

## Requirements

You need a DataKart account at [app.datakart.ai](https://app.datakart.ai). On the first tool call, the connector opens DataKart sign-in (OAuth) in your browser. No API keys or environment variables are needed.

## Data handling

The plugin runs no local scripts and installs no packages. The only network destination is the DataKart MCP server at `mcp.datakart.ai`, which receives the search criteria, files, and enrichment requests you ask Claude to run. Searches and enrichment use credits from your DataKart workspace, and enrichment never starts without your explicit approval of the previewed cost.

## Support

- Website: [datakart.ai](https://datakart.ai)
- Contact: [datakart.ai/contact](https://datakart.ai/contact) or nath@datakart.ai
- Terms: [datakart.ai/terms-of-use](https://datakart.ai/terms-of-use)
- Privacy: [datakart.ai/privacy-policy](https://datakart.ai/privacy-policy)

## License

MIT. See [LICENSE](LICENSE).

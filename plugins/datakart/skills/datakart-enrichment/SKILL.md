---
name: datakart-enrichment
description: Use when the user wants to enrich contacts, find work emails or phone numbers, run email/phone waterfall enrichment, enrich a DataKart prospecting file or an uploaded CSV, check enrichment credits, monitor a bulk enrichment job, or download enriched results. Covers enrichment_submit, bulk_enrichment_* tools, credit confirmation, and full-file delivery. Also use when the user asks how to set up or connect DataKart, or why DataKart tools are missing.
---

# DataKart Enrichment

Run credit-consuming email/phone waterfall enrichment through the DataKart MCP connector. Enrichment is where money is spent: always show preflight cost details and obtain explicit user confirmation before starting a job, poll patiently, and deliver the complete enriched file yourself.

## Before anything else: check that DataKart is connected

This skill needs the DataKart connector's tools (for example `prospecting_start`, `bulk_enrichment_start`, `datakart_workspace`). If your tools load on demand, search for them first.

If no DataKart tools are available, **stop and tell the user how to connect**. Don't guess or improvise with other tools, and don't say the connector is "connected but stale". Installing the plugin doesn't connect the connector; the user has to do it once:

- **Claude (web, desktop, mobile) and Cowork:** open **Customize → Plugins → Datakart → Connectors** and click **Connect** (if it shows **Not added**, add it first, then connect). Sign in to DataKart (or create a free account at https://app.datakart.ai). Then, in the chat, select **+ → Connectors** and make sure **Datakart** is turned on.
- **Claude Code:** run `/mcp`, select **datakart**, and choose **Authenticate** to sign in.

Once they're connected, call `datakart_workspace` to confirm the connection, then continue with the request.

## The pipelines at a glance

```
A. Enrich a prospecting file (most common):
   download_prospecting_file (schema + sample) ──▶ infer mapping
        ──▶ bulk_enrichment_from_prospecting_file (preflight, awaiting_confirmation)
        ──▶ show cost to user, get explicit approval
        ──▶ bulk_enrichment_start (confirm_spend=true)
        ──▶ bulk_enrichment_progress / bulk_enrichment_job_status (poll)
        ──▶ bulk_enrichment_download (include_file=true) ──▶ hand FULL CSV to user

B. Enrich a CSV provided in chat:
   bulk_enrichment_upload_csv ──▶ same confirm → start → poll → download chain

C. Single/few records (brokered flow):
   enrichment_submit ──▶ enrichment_job_status ──▶ enrichment_results
```

`datakart_workspace` resolves user/workspace context; `datakart_credits_balance` returns the credit balance — check it before proposing a spend when the user hasn't seen credit context.

## 1. Job types and credit costs

| `job_type` | What it does | Cost |
|------------|--------------|------|
| `email_waterfall` | Cascades multiple email providers per contact | 2 credits per row |
| `phone_waterfall` | Cascades multiple phone providers per contact | 20 credits per row |

- `run_both_waterfalls=true` on upload creates a companion job for the other channel; `bulk_enrichment_start(start_related_jobs=true)` starts both after one combined preflight.
- Results land in `email__d_1`…`email__d_n` and `phone__d_1`…`phone__d_n` columns (waterfall = possibly multiple hits per contact). Raw `email`/`phone` columns from prospecting are empty by design — DataKart does not serve stale DB contact data; only the `__d_` columns are real enrichment output.
- Prospecting itself never returns emails/phones. Never send contact-info requests to the prospecting agent; enrichment is the only path to contact data.

## 2. Identity mapping — required before any waterfall job

`mapping` tells the server which file columns identify each person. Fields: `linkedin_url`, `first_name`, `last_name`, `full_name`, `domain`. The mapping must satisfy **at least one identity group**:

1. `linkedin_url` alone, or
2. `full_name` + `domain`, or
3. `first_name` + `last_name` + `domain`

Rows missing every group are dropped (reported as `dropped` with `missing_identity`). To build the mapping:

- For prospecting files: call `download_prospecting_file(job_id)` first — its default 20-row sample plus schema is exactly for inferring mapping cheaply. Map from the columns you actually see.
- For uploaded CSVs: read the header from the preflight/validation details.
- When mapping is ambiguous, ask the user instead of guessing — a wrong mapping burns credits on garbage.

## 3. The confirm-spend protocol — NON-NEGOTIABLE

Upload tools (`bulk_enrichment_upload_csv`, `bulk_enrichment_from_prospecting_file`) default to **preflight only**: they store the immutable file, create a job in `awaiting_confirmation`, and return preflight details (valid/invalid row counts, estimated credits, credit-balance check, invalid-row samples).

1. Always show the user the preflight: row count, dropped rows and why, estimated credits, and their balance.
2. **Never set `confirm_spend=true` (or `start_immediately=true`) unless the user has explicitly approved the spend in conversation.**
3. Jobs above 1,000 rows additionally require `confirm_large_job=true` — only after showing the preflight to the user.
4. Start with `bulk_enrichment_start(job_uuid, confirm_spend=true, confirm_large_job=..., start_related_jobs=true)`.
5. If preflight reports insufficient credits or `auth_required`, report it plainly and stop; do not retry-loop.

## 4. Polling — patience is mandatory

Enrichment takes a good amount of time. Do not be impatient: poll slowly, and if it says it's loading, it is actually loading.

- `bulk_enrichment_progress(job_uuid)` — live progress (Redis, falling back to durable Mongo).
- `bulk_enrichment_job_status(job_uuid)` — durable status with submitted/completed/failed/dropped counts.
- `render_bulk_enrichment_progress(job_uuid)` — renders the live progress widget; use only when the user wants a visual, not for routine polling.
- Never restart or resubmit a job because it seems slow — that double-spends credits. Poll until a terminal state.

## 5. Downloading and delivering results — NON-NEGOTIABLE rules

When the user says "pull the data", "get the file", "download the full data", or similar:

1. Use `bulk_enrichment_download(job_uuid, include_file=true)`. While the job is still running it returns a **live partial CSV** from the Lance dataset (fine for progress peeks); after completion it returns the final CSV artifact.
2. **Actually download the file yourself and give it to the user in full** — every row, every column, no deletions, no truncation, no samples-in-place-of-data. The user is paying for every record.
3. Files can reach **~300MB**. Large size is not an excuse — write it to disk / attach it rather than pasting it into chat. (Inline tool responses cap around 2MB; beyond that retrieve via the artifact and deliver as a file.)
4. **Never hand the user a download URL.** MCP auth does not work in their normal browser; delivering the actual file is entirely your responsibility.
5. Retention: **enrichment output files persist for 6 months**; prospecting files only 7 days. If a prospecting file will not be enriched, deliver it to the user before it expires.

## 6. Single-record brokered flow

For one or a handful of contacts, skip the CSV pipeline:

- `enrichment_submit(job_type, records=[{...identity fields...}], wait_ms)` — each record needs an identity group from section 2. Returns `mcp_job_id`.
- `enrichment_job_status(mcp_job_id)` → poll; `enrichment_results(mcp_job_id, limit, offset)` → normalized record-level results.
- Keep `mcp_job_id` (brokered) distinct from `job_uuid` (bulk), `file_uuid` (uploaded file), and prospecting `job_id` — never mix them across tools.

## 7. Enriching a prospecting file end-to-end (worked flow)

1. Prospecting finished → you hold prospecting `job_id` (see `datakart-prospecting` skill).
2. `download_prospecting_file(job_id)` → inspect schema + 20-row sample.
3. Build mapping, e.g. `{linkedin_url: "linkedin_url"}` or `{full_name: "name", domain: "company_domain"}`.
4. `bulk_enrichment_from_prospecting_file(job_id, job_type="email_waterfall", mapping=...)` → preflight comes back, job is `awaiting_confirmation` with a new `job_uuid`.
5. Present cost + dropped rows to user → explicit approval → `bulk_enrichment_start(job_uuid, confirm_spend=true, confirm_large_job if >1000 rows)`.
6. Poll `bulk_enrichment_progress(job_uuid)` patiently until complete.
7. `bulk_enrichment_download(job_uuid, include_file=true)` → deliver the full CSV to the user.

## 8. Guardrails

- Report partial failures, dropped rows, and per-row errors honestly — never hide them.
- Note: bulk email/phone **validation** tools (`bulk_email_validation_upload_csv`, `bulk_phone_validation_upload_csv`, `email_validation_submit`) are currently disabled on the server. If the user asks for validation, say it is temporarily unavailable rather than improvising through enrichment mapping.
- Credits are real money: when in doubt about scope, mapping, or job type, ask before spending; after spending, deliver everything that was paid for.

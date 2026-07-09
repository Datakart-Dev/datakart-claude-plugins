---
name: datakart-prospecting
description: Use when the user wants to find prospects, leads, decision makers, companies, accounts, or people matching an ICP, build a lead list, refine a prospecting search, or export/pull/download DataKart prospecting results to a file. Covers prospecting_start, session polling, search versions, file export, and full-file delivery.
---

# DataKart Prospecting

Run natural-language people/company prospecting through the DataKart MCP connector, export the chosen result version to a file, and deliver the full file to the user. Prefer DataKart-hosted tools over ad hoc scraping or local file generation: workspace limits, saved DataKart chat history, preview behavior, and usage accounting all live behind the connector.

## The pipeline at a glance

```
prospecting_start -> prospecting_get_session (poll) -> inspect search_versions[]
        |                        |
        |                        `- requires_input? -> prospecting_answer -> keep polling
        v
pull_prospecting_results_in_file (session_id + version_id)
        |
        v
prospecting_progress / render_prospecting_progress (poll job_id)
        |
        v
download_prospecting_file (job_id, include_file=true) -> hand the FULL file to the user
        |
        `- or feed job_id into bulk_enrichment_from_prospecting_file (see datakart-enrichment skill)
```

## 1. Choosing the search tool - Lite first, 95% of the time

`prospecting_start` takes `selected_tool`, one of:

| Tool | Aka | Cost | When |
|------|-----|------|------|
| `our_people_search` | People Search Lite | Flat 100 credits for up to 10k records | **Default for people - use this 95% of the time** |
| `our_company_search` | Company Search Lite | Flat 100 credits for up to 10k records | **Default for companies - use this 95% of the time** |
| `people_search` | Deep People Search | 1-2 credits **per record** (~100x more expensive), much much slower | Only when Lite results are absolute crap / totally unusable |
| `company_search` | Deep Company Search | 1-2 credits **per record** (~100x more expensive), much much slower | Only when Lite results are absolute crap / totally unusable |

Escalate from Lite to Deep only after Lite refinement has genuinely failed, and tell the user about the cost and time difference before doing so. **None of these tools return emails or phone numbers** - contact data always comes from the separate enrichment service (see `datakart-enrichment`).

## 2. Writing the prospecting message

The `message` you pass to `prospecting_start` is read by a backend agent with high-reasoning settings but a **smaller language model**. Treat it accordingly:

- Give a decent, moderately detailed prompt - **maximum ~4 lines**. Include concrete ICP, geography, role/title, industry, headcount, funding, technology, or exclusion details the user provided. Do not dump paragraphs.
- It can make mistakes. If results are off, look at what it actually returned and send a targeted refinement describing exactly what to change relative to those results.
- **Refinements start a fresh `prospecting_start` session** (session continuation is intentionally disabled until re-enabled server-side). Do not try to continue an old session with a new query.
- **LinkedIn URL shortcut:** for well-known people, founders, or any case where the user already knows who they want, passing the corresponding LinkedIn URLs directly in the message is the fastest and most reliable path.
- **Never ask the prospecting agent for contact info, emails, or phone numbers.** It will not respond to that - contact enrichment is a separate paid service. Keep prospecting messages about *who to find*, not *how to reach them*.

## 3. Session lifecycle and patient polling

`prospecting_start` returns a session with `session_id` (e.g. `ps_WgqjXxWKmV0rvo2cUfCrAuDwasVEyvXm`) and a `status`:

- **`running`** - the search is still processing. Wait `next_poll_after_ms` (default 10s), then call `prospecting_get_session(session_id)`. Repeat until terminal. Broad searches can take **several minutes**. An unchanged or quiet `tool_trace` does NOT mean the session is stuck - if it says loading, it is actually loading. Never abandon or restart a session out of impatience.
- **`requires_input`** - the backend agent asked a follow-up question. Surface `pending_question` to the user, then resume with `prospecting_answer(pending_question_id, selected=[...], session_id=..., custom=...)`. The session may return to `running`; keep polling.
- **`completed`** - inspect `search_versions[]` and the `best_tool_result` / `data_sample`.
- **`failed`** - report the error; start a fresh, refined session if appropriate.

Widget rules (MCP Apps hosts): `prospecting_start` renders the DataKart widget once; polling tools are deliberately UI-less. Call `render_ui(session_id)` **only** when the user explicitly asks to see the widget again and no active widget is visible - never to refresh progress.

## 4. search_versions - picking the version to export

Every search attempt inside a session becomes an entry in `search_versions[]` returned by `prospecting_get_session` (and `prospecting_start`). Each entry has:

- **`version_id`** - the identifier for that specific attempt, format `tool:call_<id>` (e.g. `tool:call_xFuZ4piFcyDiB0Z9NdnCNeGq`) or `candidate:<id>`
- **`total_results`** - how many records that attempt can actually pull
- **`preview_count`** - how many rows were previewed
- **`is_pullable`** - whether this version can be exported (only ready Lite versions with resolved filters are)
- `status`, `search_kind`, `version_index`, `filters_hash`

The practical selection flow, always: **poll the session -> inspect `search_versions` -> pick the `version_id` with non-zero `total_results` that matches what the user actually wants -> pass `session_id` + that `version_id` to the pull call.** Without `version_id` there is no way to say which of the many attempts you mean. Never invent a `version_id`; only use values returned by the connector.

## 5. Exporting to a file

`pull_prospecting_results_in_file(session_id, version_id, limit, name?, description?, exclusion?)`:

- Only works for Lite (`our_people_search` / `our_company_search`) versions.
- `limit` caps rows, hard maximum 10,000. For very large exports, confirm scope with the user before going big.
- Returns `job_id` (= `file_id`) with `file_status: pending_webhook_completion`. The file is built asynchronously.
- Poll with `prospecting_progress(job_id)`; use `render_prospecting_progress(job_id)` only when a visual progress widget is wanted. Building takes real time - poll slowly and wait.
- `exclusion` accepts `{tables: [...], exclusion_type: "url" | "table_uuid"}` to suppress records already in existing DataKart tables.

## 6. Downloading and delivering the file - NON-NEGOTIABLE rules

When the user says anything like "pull the data", "get the data", "get the file", "download the full data", or similar:

1. You **MUST** use a download tool - `download_prospecting_file(job_id, include_file=true)` here (or `bulk_enrichment_download` for enriched files).
2. You **MUST ALWAYS actually download the file yourself and give it to the user in full** - every row, every column, nothing deleted, nothing truncated, no "sample" substituted. The user is paying for every record.
3. Files can be large - up to **~300MB**. That does not exempt you; you still have to retrieve it and deliver it (write it to disk / attach it for the user rather than pasting giant CSVs into chat).
4. **NEVER pass a download link/URL to the user.** MCP auth does not work in their normal browser, so any link you hand over is dead on arrival. Downloading the file and delivering it is entirely your responsibility.
5. **Prospecting files are temporary - available for only 7 days.** If the user is not going to enrich emails/phones from the file, you must deliver the file to them before it expires. (Enrichment output files, by contrast, persist for 6 months.)

`download_prospecting_file(job_id, sample_size=20, include_file=false)` defaults to schema + 20 sample rows - use that cheap default to inspect columns and plan enrichment mapping; set `include_file=true` only when actually retrieving the full data for delivery.

## 7. Understanding the file contents

- The prospecting file may contain raw `email` / `phone` columns **that are empty by design** - DataKart does not serve stale database emails/phones, to keep data quality high.
- Real contact data only appears after waterfall enrichment, in columns named `email__d_1`, `email__d_2`, ... `email__d_n` and `phone__d_1`, `phone__d_2`, ... `phone__d_n`.
- Do not tell the user the data is broken because contact columns are empty; explain that emails/phones are enriched separately and offer the enrichment flow.

## 8. Guardrails

- Preserve `session_id`, `version_id`, `job_id`/`file_id` across turns; they are the only handles to the user's paid work.
- Do not invent DataKart filters or version IDs not returned by the connector; `filters_hint` is a hint, not a filter API.
- Do not call debug-only table creation tools (`prospecting_create_table`) for production workflows - saved imports belong in the DataKart UI.
- Every action takes a good amount of time. Poll slowly, wait patiently, and keep the user informed instead of retrying or restarting.
- Contact data questions -> hand off to the `datakart-enrichment` skill.

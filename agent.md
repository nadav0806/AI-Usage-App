# Cursor Analytics Integration Instruction

> **For the AI agent developer:** Implement this in the existing platform by extending the current AI usage pipeline rather than creating a separate one-off script.

## Goal

Integrate Cursor analytics support into the existing platform so the app can ingest Cursor exports, normalize them into the current usage schema, and generate the same classification and cost reports already supported for other vendors.

## What already exists

The current codebase already contains a Python pipeline that:
- reads AI usage JSON
- aggregates rows by `user_email` + `model`
- classifies usage behavior
- estimates cost savings
- writes JSON, CSV, and optional XLSX reports

The repo has already been updated with a Cursor adapter for `Filtered Usage Events` and newer model support. Your job is to make that integration production-ready inside the broader platform.

## Required Cursor sources to support

Use these Cursor API exports as the canonical inputs:

1. `Filtered Usage Events`
   - Source of truth for per-request tokens and cost
   - Use this as the primary ingest path for savings calculations

2. `Daily Usage Data`
   - Use for daily user-level enrichment
   - Helpful fields: `isActive`, `composerRequests`, `chatRequests`, `agentRequests`, `cmdkUsages`, `mostUsedModel`, line acceptance fields

3. `Agent Edits`
   - Use for acceptance-rate and suggestion-quality reporting
   - Helpful fields: total lines accepted/suggested, accepted/rejected diffs

4. `DAU`
   - Use for dashboard-level adoption metrics only

5. `Model Breakdown`
   - Use for model taxonomy, adoption reporting, and model-family normalization

## Implementation expectations

### 1) Keep the current generic schema

Do not rewrite the pipeline around Cursor-specific JSON structures.
Instead, normalize Cursor data into the existing internal row shape:
- `user_email`
- `date`
- `provider`
- `model`
- `requests_count`
- `input_tokens`
- `output_tokens`
- `cached_tokens`
- `cost_usd`
- `tool_turns`
- `web_search_requests`

Add enrichment fields only where useful.

### 2) Expand model support for newer Cursor models

The pipeline must understand newer Cursor model families, including:
- `gpt-5.5-*`
- `gpt-5.4-*`
- `gpt-5.3-codex-*`
- `gpt-5.2-high`
- `gpt-5.1-codex-max-xhigh`
- `claude-4.6-*`
- `claude-4.5-*`
- `claude-4-sonnet*`
- `claude-opus-4-7-*`
- `composer-2`
- `composer-2-fast`
- `gemini-3.1-pro-preview`
- `default`

Update:
- tier rules
- vendor inference
- pricing candidate matching
- any documentation or legend text that references model families

### 3) Preserve cost optimization behavior

The app must still compute:
- global cheapest qualifying model
- same-vendor cheapest qualifying model
- savings estimates for both tracks

Cursor event-level cost should be computed from the Cursor export rather than inferred from token pricing when actual charged cents are available.

### 4) Support optional enrichment files

If the platform architecture allows multiple inputs, add support for merging or enriching Cursor sources by date/user/model.
Suggested priority:
1. `Filtered Usage Events` for spend logic
2. `Daily Usage Data` for behavior enrichment
3. `Agent Edits` for quality enrichment
4. `Model Breakdown` for taxonomy/adoption dashboards
5. `DAU` for executive summary metrics

### 5) Keep output compatibility

Do not break existing output formats unless absolutely necessary.
The platform should still emit:
- JSON report
- CSV report
- opportunities report
- summary by category
- XLSX when enabled

If you add Cursor-specific columns, keep them additive.

## Suggested work breakdown

### Task 1: Add input normalization layer
- Detect Cursor payload shapes
- Flatten `usageEvents[]`
- Normalize timestamps to `YYYY-MM-DD`
- Map Cursor token/cost fields into the generic schema

### Task 2: Expand model mapping
- Add aliases and pricing candidates for newer Cursor model families
- Verify vendor inference for OpenAI / Anthropic / Google / Cursor buckets

### Task 3: Add optional enrichment ingestion
- Merge Daily Usage Data into the generic rows when available
- Merge Agent Edits into summary/report fields when available
- Keep the base pipeline working when enrichment files are absent

### Task 4: Update reports and summaries
- Include Cursor quality/adoption metrics where relevant
- Keep the existing executive summary and opportunities table intact
- Ensure the legend and explanatory text mention Cursor clearly

### Task 5: Test and verify
- Run the pipeline on a Cursor `Filtered Usage Events` sample
- Verify non-zero spend is preserved
- Verify recommendations are produced
- Verify newer models are classified and priced correctly
- Verify optional enrichment data does not break the base case

## Acceptance criteria

The work is done when:
- Cursor exports can be ingested without manual reshaping
- the pipeline still works with non-Cursor input
- newer model names are recognized
- savings recommendations are produced for Cursor records
- optional enrichment data can be added without breaking the base flow
- the output schema remains stable enough for downstream consumers

## Notes for the implementer

- Prefer the real Cursor event data over daily summary approximations for pricing and savings.
- Treat `default` as a fallback bucket, not a recommendation target.
- Keep the adapter small and composable.
- If new Cursor model variants appear, update the alias table rather than scattering special cases.

## Expected outcome

A platform-native Cursor ingestion path that reuses the current pipeline architecture and can support reporting, classification, and savings analysis from Cursor exports.

<!-- Version 1.3 — Dual recommendations: global cheapest vs same-vendor cheapest -->

# Full Stack Developer Prompt — AI Usage Classifier & Cost Optimisation Report

## What Are We Building?

A system that takes raw **AI API usage logs** (one row per user per day per model) from **any supported vendor**, and:

1. **Classifies** each user into a usage behavior category
2. **Recommends two alternative models** per user–model row: **(A)** the cheapest qualifying model **across any vendor**, and **(B)** the cheapest qualifying model **staying on the current vendor** (when vendor lock-in applies)
3. **Estimates** cost savings **separately** for each recommendation (global vs same-vendor)
4. **Outputs** a color-coded Excel report with an executive summary, savings opportunities table, per-user breakdown, and legend

No prompt content is ever needed — only metadata: token counts, model id, `provider`, request counts, cost.

**Normalisation:** Each vendor’s billing or response payload is mapped into the canonical schema below. Examples: OpenAI Chat Completions `usage`; Anthropic Messages [`usage`](https://docs.anthropic.com/en/api/messages) (`input_tokens`, `output_tokens`, `cache_read_input_tokens`, …); Google Generative AI usage metadata; Cursor or internal gateways that expose token counts per model.

---

## Input Format

**Data source:** **Multi-vendor.** A single dataset may include `provider` values such as `Anthropic`, `OpenAI`, `Google`, `Cursor`, etc. One ingested record = one end-user identity, one calendar day, one `model` id for that provider (after ETL joins your internal `user_email`).

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `user_email` | string | Unique user identifier in your org |
| `input_tokens` | integer | Input tokens billed or counted for this row |
| `output_tokens` | integer | Output tokens billed or counted for this row |

**Optional but strongly recommended (improve accuracy significantly):**

| Field | Type | Description |
|-------|------|-------------|
| `user_name` | string | Display name |
| `date` | string (ISO 8601) | `YYYY-MM-DD` — used to count active days |
| `model` | string | Vendor-specific model id (e.g. `gpt-4o`, `claude-sonnet-4-20250514`, `gemini-2.5-pro`, `cursor-fast`) |
| `provider` | string | Vendor label (`OpenAI`, `Anthropic`, `Google`, `Cursor`, …) — **required for multi-vendor files**; if omitted, infer from model string heuristics only as a last resort |
| `department` | string | User's department |
| `team` | string | User's team |
| `requests_count` | integer | API calls aggregated into this row (default: 1) |
| `cached_tokens` | integer | Cache hits / discounted input tokens (e.g. Anthropic `cache_read_input_tokens`, OpenAI cached input where available) |
| `cost_usd` | float | Actual cost for this row |
| `web_search_requests` | integer | When logged (e.g. Anthropic web search on Messages API) — high values skew toward research / technical patterns |
| `tool_turns` | integer | Optional: logged tool/agent turns — strong **Power / Technical** signal |
| `lines_accepted` | integer | Code lines accepted (IDE / Cursor integrations, if merged into this feed) |
| `lines_suggested` | integer | Code lines suggested |

**Accepted formats:** JSON array or CSV file. **Reference sample:** keep a checked-in **`sample-data.json`** (or agreed filename) with rows spanning **every vendor and model family** you care about — that file is the contract for parsers, tests, and UI previews.

**Example rows (different vendors in one array):**
```json
[
  {
    "date": "2026-01-10",
    "provider": "Anthropic",
    "model": "claude-sonnet-4-20250514",
    "user_email": "alice.chen@company.com",
    "input_tokens": 25630,
    "output_tokens": 16543,
    "requests_count": 4,
    "cached_tokens": 8900,
    "cost_usd": 0.197468
  },
  {
    "date": "2026-01-10",
    "provider": "OpenAI",
    "model": "gpt-4o",
    "user_email": "bob@company.com",
    "input_tokens": 1200,
    "output_tokens": 400,
    "requests_count": 12,
    "cost_usd": 0.042
  }
]
```

---

## Step 1 — Aggregate Per User (Per Model)

Before classifying, roll up all daily rows into **one row per (user × model)**.

Compute the following per user-model pair:

| Signal | Formula |
|--------|---------|
| `total_requests` | `SUM(requests_count)` |
| `total_input` | `SUM(input_tokens)` |
| `total_output` | `SUM(output_tokens)` |
| `total_cached` | `SUM(cached_tokens)` |
| `total_cost` | `SUM(cost_usd)` |
| `avg_input` | `SUM(input_tokens) / SUM(requests_count)` |
| `avg_output` | `SUM(output_tokens) / SUM(requests_count)` |
| `ratio` | `total_output / total_input` (how much output per unit of input) |
| `active_days` | `COUNT(DISTINCT date)` |
| `requests_per_day` | `total_requests / active_days` |
| `model_tier` | Lookup table score 0.0–1.0 (see below) |
| `cache_rate` | `total_cached / total_input` |
| `web_search_requests` (agg) | `SUM(web_search_requests)` if column exists, else omit / use 0 in classification |
| `tool_turns` (agg) | `SUM(tool_turns)` if column exists, else omit / use 0 in classification |

---

## Step 2 — Model Tier Table

Each **observed** model id gets a sophistication score from 0.0 (cheap/fast) to 1.0 (flagship). Match by substring on `model` (case-insensitive). **Order:** use the **highest** matching tier if multiple substrings match (or define explicit precedence in code — be consistent).

Use one table for **all vendors** present in your logs and for **suggested** models in Steps 4–5:

| Model substring | Tier score |
|----------------|-----------|
| `o3`, `o1`, `claude-opus`, `opus`, `gemini-2.5-pro` | 1.0 |
| `gpt-4.1` | 0.9 |
| `gpt-4o`, `cursor-slow` | 0.85 |
| `claude-sonnet`, `sonnet` | 0.65 |
| `gemini-2.5-flash`, `gemini-2.0-flash`, `o3-mini` | 0.6 |
| `cursor-fast`, `gpt-4o-mini` | 0.5 |
| `claude-haiku`, `haiku` | 0.4 |
| `gemini-flash`, `gemini-1.5-flash` | 0.35 |
| *(any other / unmatched)* | 0.5 (default) |

**Note:** Tier scores are **relative across vendors** for classification and savings math. Optional: when `provider === "Anthropic"` only, you may apply a **secondary** Sonnet boost in classification (e.g. treat `sonnet` as tier ≥ 0.85 for rule 5 only) — only if pilot data shows Sonnet-heavy orgs are misclassified as non-Power.

---

## Step 3 — User Classification

Classify each user-model row into one of five categories. Rules assume a **mixed-vendor** feed: `model` strings from OpenAI, Anthropic, Google, Cursor, etc., plus optional tool/search counters when your pipeline captures them.

Apply rules **in this exact priority order** (first match wins).

### Category Definitions & Rules

```
Given per-user-model aggregated signals:
  rpd    = requests_per_day
  avg_in = avg_input  (tokens)
  ratio  = total_output / total_input
  tier   = model_tier (0.0–1.0) from Step 2
  reqs   = total_requests
  model  = model id string (lowercase)
  wsr    = SUM(web_search_requests) if present, else 0
  tools  = SUM(tool_turns) if present, else 0
```

| Priority | Category | Condition |
|----------|----------|-----------|
| 1 | 🧪 **Explorer** | `reqs < 20 AND rpd < 1.5` |
| 2 | 🧑‍💻 **Power / Technical** | `"cursor"` in model **or** `provider` indicates Cursor-style product (if you normalise it) |
| 3 | 🧑‍💻 **Power / Technical** | `"opus" in model` |
| 4 | 🧑‍💻 **Power / Technical** | `tools ≥ 8` OR `wsr ≥ 6` (per user-model aggregate over the reporting window) |
| 5 | 🧑‍💻 **Power / Technical** | `rpd > 3.6` OR `avg_in > 15,000` OR (`"sonnet" in model` AND (`avg_in > 10,000` OR `rpd > 3.2`)) OR (`tier >= 0.85 AND reqs > 50`) |
| 6 | 🔍 **Lookup / Q&A** | `avg_in < 6,500 AND ratio < 0.73` |
| 7 | ✍️ **Content Generator** | `ratio > 0.76 AND avg_in < 12,000` |
| 8 | 💬 **Conversational** | `1.5 ≤ rpd ≤ 3.6 AND 6,500 ≤ avg_in ≤ 15,000` |
| 9 | 💬 **Conversational** | Default (catch-all) |

**Multi-vendor notes:**

- **Cursor / IDE** model names imply coding-heavy workflows — keep priority 2 when those strings appear in `sample-data.json` and production.
- **Opus** and **high-volume Sonnet** paths catch Anthropic flagship and serious Sonnet use even when tier alone stays at 0.65 in Step 2.
- **`tools` / `wsr`** default to 0 when missing; priorities 4+ still classify purely from tokens + tier + model substring.
- Tokeniser differences between vendors are small at the scale of these thresholds; re-tune on a labeled sample if one vendor dominates.

### Category Descriptions (show in UI / legend)

| Category | Profile |
|----------|---------|
| 🧑‍💻 Power / Technical | Heavy usage, large inputs, flagship or IDE models, agentic/tool or web-search patterns. Coding, analysis, research, big documents. |
| ✍️ Content Generator | Output >> Input. Writing, drafting, summarisation. |
| 💬 Conversational | Balanced usage. Brainstorming, back-and-forth dialogue. |
| 🔍 Lookup / Q&A | Short inputs, short outputs, many requests. Search-like Q&A. |
| 🧪 Explorer | Low/irregular usage. Still discovering capabilities. |

---

## Step 4 — Dual model recommendations (computed, not a fixed category → model table)

Each aggregated user–model row gets **two** independent suggestions. Both are chosen by **minimising projected cost** for that row’s historical **`total_input`** and **`total_output`**, using the Step 5 pricing table — **not** by mapping Step 3 category to a named model.

### Candidate catalog

- Every **priced** model in the Step 5 table is a **candidate**, except the **`Unknown`** row — never recommend “unknown” as a target model.
- Treat each **pricing table row** as one candidate (if a row lists multiple substrings, they share one price tier — one candidate).
- **Exclude** the candidate that matches the user’s **current** model id (after normalisation), so suggestions are always a **switch**.

### Recommendation A — Cheapest across all vendors (global)

1. Compute **current** blended effective $/1M for the row’s token mix (see Step 5).
2. Among **all** candidates, keep those with projected cost **at least ~5% lower** than current (`sugg_cost_per_1M < curr_cost_per_1M * 0.95`, same rule as Step 5).
3. If any remain, pick the candidate with **lowest** projected total cost for `(total_input, total_output)` (equivalently lowest `sugg_cost_per_1M` for that mix).
4. If none qualify, **Recommendation A** is empty (`null` / `—` in reports).

### Recommendation B — Cheapest within the **same vendor** as the row

1. Determine **`row_vendor`** from the ingested `provider` field (normalise case/spelling). If `provider` is missing, infer vendor from the current `model` string using the **Vendor map** below.
2. Filter candidates to those whose **catalog vendor** equals **`row_vendor`**.
3. Apply the **same ~5% cheaper** rule vs current; among qualifiers, pick **lowest** projected cost (same tie-break as A).
4. If no in-vendor candidate qualifies (already on the cheapest listed model for that vendor, or only one priced tier), **Recommendation B** is empty.

### Vendor map (for filtering Recommendation B)

Assign each **candidate** (and, when needed, the current row) to exactly one vendor using **first matching** rule on the lowercased model id:

| Vendor | Match if `model` contains (case-insensitive) |
|--------|-----------------------------------------------|
| OpenAI | `gpt-`, `o1`, `o3` (e.g. `gpt-4o`, `o3-mini`) |
| Anthropic | `claude-` |
| Google | `gemini` |
| Cursor | `cursor-` |

Map the ingested `provider` string to these canonical names (e.g. `Google AI` → `Google`). If `provider` is unknown and the model matches no row, treat **Recommendation B** as not applicable.

### Using Step 3 (category) in the product

Classification remains useful for **labels, filters, and narrative** in the Explanation column (e.g. “Lookup/Q&A pattern — smaller models are usually enough”), but **does not** select the suggested model id. The two suggestions are always **price-optimised** as above.

---

## Step 5 — Cost Savings Estimation

**Only treat a candidate as actionable if it is at least ~5% cheaper than the current model** (same threshold for both Recommendation A and B).

### Pricing Table (cost per 1M tokens — April 2026)

Use one table for **all** current and suggested models (update numbers as vendors change):

| Model substring | Input $/1M | Output $/1M |
|----------------|-----------|------------|
| `o3` | $10.00 | $40.00 |
| `o1` | $15.00 | $60.00 |
| `gpt-4.1` | $2.00 | $8.00 |
| `gpt-4o` | $2.50 | $10.00 |
| `gpt-4o-mini` | $0.15 | $0.60 |
| `claude-opus`, `opus` | $15.00 | $75.00 |
| `claude-sonnet`, `sonnet` | $3.00 | $15.00 |
| `claude-haiku`, `haiku` | $0.80 | $4.00 |
| `gemini-2.5-pro` | $1.25 | $10.00 |
| `gemini-2.5-flash` | $0.15 | $0.60 |
| `gemini-2.0-flash` | $0.10 | $0.40 |
| `gemini-flash` | $0.10 | $0.40 |
| `cursor-slow` | $2.50 | $10.00 |
| `cursor-fast` | $0.15 | $0.60 |
| Unknown | $2.50 | $10.00 |

### Savings calculation (run for **each** chosen candidate)

```
total_tokens = total_input + total_output

# Current row (from observed model → pricing row):
curr_cost_per_1M = (curr_input_price * total_input + curr_output_price * total_output) / total_tokens

# Candidate model:
sugg_cost_per_1M = (sugg_input_price * total_input + sugg_output_price * total_output) / total_tokens

is_cheaper = sugg_cost_per_1M < curr_cost_per_1M * 0.95

savings_pct = (1 - sugg_cost_per_1M / curr_cost_per_1M) * 100
est_savings = total_cost_usd * (savings_pct / 100)
```

Compute **`est_savings_global`** / **`is_cheaper_global`** for Recommendation **A**, and **`est_savings_same_vendor`** / **`is_cheaper_same_vendor`** for Recommendation **B** (each using its own chosen candidate’s prices).

**Tie-break:** If several candidates share the same lowest projected cost, pick a single winner deterministically (e.g. lexicographic by display string `Vendor — Model`).

**Cost Savings Opportunities sheet:** Include a row if **`is_cheaper_global` OR `is_cheaper_same_vendor`**. Populate the columns for each track; use `—` where that track has no qualifying candidate.

---

## Step 6 — Excel Report Output

The output is a **color-coded Excel file** (`.xlsx`) with 4 sheets.

### Color Convention
- 🔵 **Blue columns** = data sourced directly from the input (what they have)
- 🟡 **Yellow columns** = AI analysis output (what we computed / recommended)
- 🟢 **Green cells** = estimated cost savings

---

### Sheet 1: Executive Summary

**KPI Cards (top):**

| Card | Value |
|------|-------|
| Total Current Cost | `SUM(total_cost_usd)` across all rows |
| Est. Potential Savings (**any vendor**) | `SUM(est_savings_global)` where `is_cheaper_global` |
| Est. Potential Savings (**same vendor only**) | `SUM(est_savings_same_vendor)` where `is_cheaper_same_vendor` |
| Forecast Cost (any vendor scenario) | Total Current Cost − Est. Potential Savings (any vendor) |
| Saving % (any vendor) | Est. Potential Savings (any vendor) / Total Current Cost × 100 |

**Top Savings Opportunities Table (below KPIs):**

Show the top 15 user-model rows ranked by **`est_savings_global` descending** (if tie or null global, sort by `est_savings_same_vendor`).

Columns: User, AI Category, Current Vendor, Current Model, **➡️ Cheapest (any vendor)**, Est. saving A ($ + %), **➡️ Cheapest (same vendor)**, Est. saving B ($ + %)

Footer note:
> *Savings A = hypothetical switch to the cheapest qualifying model across all vendors in the catalog. Savings B = cheapest qualifying model within the current vendor only. Both use the same token mix and ~5% minimum discount rule.*

---

### Sheet 2: Cost Savings Opportunities

One row per user-model pair where **`is_cheaper_global` OR `is_cheaper_same_vendor`**.

**Blue columns (from data):**

| Column | Source |
|--------|--------|
| AI Vendor | `provider` field |
| AI Model | `model` field |
| User | `user_name` |
| Email | `user_email` |
| Department | `department` |
| Team | `team` |
| Total Requests | `total_requests` |
| Active Days | `active_days` |
| Req / Day | `requests_per_day` (1 decimal) |
| Avg Input Tokens | `avg_input` (integer) |
| Avg Output Tokens | `avg_output` (integer) |
| Output/Input Ratio | `ratio` (2 decimals) |
| Total Cost ($) | `total_cost` |

**Yellow columns (AI analysis):**

| Column | Content |
|--------|---------|
| AI Category | Emoji + category name from Step 3 (for context only) |
| **Cheapest — any vendor** | `Vendor — ➡️ Model` for Recommendation **A**, or `—` if none |
| **Est. savings A ($)** | From `est_savings_global`; `%` in adjacent column or number format; green fill when present |
| **Cheapest — same vendor** | `Vendor — ➡️ Model` for Recommendation **B**, or `—` / `Already cheapest tier` when none |
| **Est. savings B ($)** | From `est_savings_same_vendor`; green fill when present |
| Explanation | Short text: e.g. category hint + “Global best is {A}; without switching vendor, best is {B}.” |

Sorting: by category, then user name, then model.
Freeze row 1. Enable auto-filter on all columns.

---

### Sheet 3: Summary

Aggregate by category:

| Column | Content |
|--------|---------|
| Category | Emoji + name |
| Users | `COUNT(DISTINCT user_email)` |
| Total Cost ($) | `SUM(total_cost)` |
| Top global target (mode) | Most frequent Recommendation **A** `Vendor — Model` in that category (optional) |
| Top same-vendor target (mode) | Most frequent Recommendation **B** `Vendor — Model` in that category (optional) |

---

### Sheet 4: Legend

Simple two-column table:

| Category | Description |
|----------|-------------|
| 🧑‍💻 Power / Technical | Heavy usage... |
| ✍️ Content Generator | High output... |
| 💬 Conversational | Balanced... |
| 🔍 Lookup / Q&A | Short inputs... |
| 🧪 Explorer | Low/irregular... |

---

## Expected User Flow (End-to-End)

```
1. User uploads unified usage (CSV/JSON — e.g. `sample-data.json` or production export with multiple `provider` values)
2. System validates required fields, shows row count and distinct vendors/models detected
3. System aggregates → classifies → computes savings
4. System generates Excel report
5. User downloads report with:
   - Executive summary with **two** savings KPIs (any vendor vs same vendor)
   - Row-by-row **dual** recommendations where at least one track saves ≥ ~5%
   - Full legend + summary by category
```

---

## Edge Cases to Handle

| Case | Handling |
|------|---------|
| Missing `requests_count` | Default to 1 |
| Missing `web_search_requests` / `tool_turns` | Treat as 0 — Power rules that use them are skipped; later Power rules still apply |
| Missing `cost_usd` | Default to 0 (savings calc still works via pricing table) |
| Missing `date` | Use row count as proxy for active_days |
| `total_input = 0` | Set `ratio = 1.0` (avoid division by zero) |
| `active_days = 0` | Treat as 1 |
| Unknown model name | Default tier = 0.5, default price = $2.50/$10.00 per 1M |
| `Unknown` pricing row | Use only for **current** cost fallback — never as Recommendation A/B target |
| A and B both empty | Omit row from Cost Savings Opportunities sheet |
| Only global or only same-vendor qualifies | Still emit row; fill `—` for the empty track |
| A and B recommend the same model | Allowed (happens when global cheapest is already in-vendor); savings columns may match |
| Cannot map `provider` for B | Set Recommendation B to `—` and explain “vendor unknown for in-catalog filter” |
| Only 1 user in dataset | Normalised scores all = 0.5 — classification still works via absolute thresholds |
| JSON wrapped in `{data: [...]}` | Unwrap common keys: `data`, `rows`, `records`, `results` |
| Column names with spaces or mixed case | Normalise: lowercase, replace spaces with underscores |

---

## What System X Should Expose

### Minimum (backend only):
- `POST /classify` — accepts JSON body or file upload, returns classified rows as JSON
- `POST /report` — accepts same input, returns `.xlsx` file download

### Recommended (full product):
- File upload UI (drag-and-drop CSV/JSON)
- Preview of loaded data (row count, columns detected, sample)
- Classification results table with filters by category/department/team
- Executive summary dashboard (KPI cards, top savings list)
- Excel download button
- Per-user drill-down view

---

## Sample Data (for testing)

Generate test data with 10 users across 5 behavior profiles:

```
Power users:  alice.chen, iris.davis (high volume, large inputs, flagship models)
Content:      carol.smith, grace.kim (small inputs, large outputs)
Conversational: david.lee, jack.taylor (balanced)
Lookup/Q&A:   emily.johnson, frank.wilson (many short requests)
Explorer:     henry.brown (infrequent, low volume)
```

Cover: 60-day window; **mix vendors** in the JSON (OpenAI, Anthropic, Google, Cursor, …) so each profile can appear on different providers; 70% primary model / 30% alternate per user where realistic.

---

## Questions to Clarify Before Starting

1. What does each vendor export (CSV/JSON/API), and how is `provider` + `model` normalised into the canonical schema?
2. Is this single-tenant (one org) or multi-tenant (multiple companies)?
3. Should reports be stored, or is it a stateless generate-and-download flow?
4. Do you need user authentication, or is it an internal tool?
5. Should the Excel file be generated server-side (Python/openpyxl) or client-side (JS library)?

---

*Prompt authored: 2026-04-12 | System: AI Usage Classifier v1.3 (dual recommendations) | Owner: Nadav*

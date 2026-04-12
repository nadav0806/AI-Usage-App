# Full Stack Developer Prompt — AI Usage Classifier & Cost Optimisation Report

## What Are We Building?

A system that takes raw **AI API usage logs** (one row per user per day per model) and:

1. **Classifies** each user into a usage behavior category
2. **Recommends** a better-fit AI model for each user (based on their pattern)
3. **Estimates** cost savings if the recommended model was used instead
4. **Outputs** a color-coded Excel report with an executive summary, savings opportunities table, per-user breakdown, and legend

No prompt content is ever needed — only metadata: token counts, model names, request counts, cost.

---

## Input Format

One record = one user, one day, one model.

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `user_email` | string | Unique user identifier |
| `input_tokens` | integer | Input tokens consumed in this session |
| `output_tokens` | integer | Output tokens generated in this session |

**Optional but strongly recommended (improve accuracy significantly):**

| Field | Type | Description |
|-------|------|-------------|
| `user_name` | string | Display name |
| `date` | string (ISO 8601) | `YYYY-MM-DD` — used to count active days |
| `model` | string | Model name (e.g. `gpt-4o`, `claude-sonnet-4`, `gemini-2.5-pro`) |
| `provider` | string | AI vendor (`OpenAI`, `Anthropic`, `Google`, etc.) |
| `department` | string | User's department |
| `team` | string | User's team |
| `requests_count` | integer | Number of API calls in this row (default: 1) |
| `cached_tokens` | integer | Tokens served from cache (not billed at full rate) |
| `cost_usd` | float | Actual cost billed for this row |
| `lines_accepted` | integer | Code lines accepted (for IDE/Cursor users) |
| `lines_suggested` | integer | Code lines suggested |

**Accepted formats:** JSON array or CSV file.

**Example row:**
```json
{
  "date": "2026-01-10",
  "provider": "Google",
  "model": "gemini-2.5-pro",
  "user_name": "Alice Chen",
  "user_email": "alice.chen@company.com",
  "department": "Engineering",
  "team": "Platform",
  "input_tokens": 25630,
  "output_tokens": 16543,
  "requests_count": 4,
  "cached_tokens": 3200,
  "cost_usd": 0.197468
}
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

---

## Step 2 — Model Tier Table

Each model gets a sophistication score from 0.0 (cheap/fast) to 1.0 (flagship). Match by substring (case-insensitive):

| Model substring | Tier score |
|----------------|-----------|
| `o3`, `o1`, `claude-opus`, `gemini-2.5-pro` | 1.0 |
| `gpt-4.1` | 0.9 |
| `gpt-4o`, `cursor-slow` | 0.85 |
| `claude-sonnet` | 0.65 |
| `gemini-2.5-flash`, `gemini-2.0-flash`, `o3-mini` | 0.6 |
| `cursor-fast`, `gpt-4o-mini` | 0.5 |
| `claude-haiku` | 0.4 |
| `gemini-flash`, `gemini-1.5-flash` | 0.35 |
| Unknown | 0.5 (default) |

---

## Step 3 — User Classification

Classify each user-model row into one of five categories. Apply rules **in this exact priority order** (first match wins):

### Category Definitions & Rules

```
Given per-user-model aggregated signals:
  rpd    = requests_per_day
  avg_in = avg_input  (tokens)
  ratio  = total_output / total_input
  tier   = model_tier (0.0–1.0)
  reqs   = total_requests
  model  = model name string (lowercase)
```

| Priority | Category | Condition |
|----------|----------|-----------|
| 1 | 🧪 **Explorer** | `reqs < 20 AND rpd < 1.5` |
| 2 | 🧑‍💻 **Power / Technical** | `"cursor"` in model name |
| 3 | 🧑‍💻 **Power / Technical** | `rpd > 3.6` OR `(tier >= 0.85 AND reqs > 50)` OR `avg_in > 15,000` |
| 4 | 🔍 **Lookup / Q&A** | `avg_in < 6,500 AND ratio < 0.73` |
| 5 | ✍️ **Content Generator** | `ratio > 0.76 AND avg_in < 12,000` |
| 6 | 💬 **Conversational** | `1.5 ≤ rpd ≤ 3.6 AND 6,500 ≤ avg_in ≤ 15,000` |
| 7 | 💬 **Conversational** | Default (catch-all) |

### Category Descriptions (show in UI / legend)

| Category | Profile |
|----------|---------|
| 🧑‍💻 Power / Technical | Heavy usage, large inputs, high-end models. Coding, analysis, document processing. |
| ✍️ Content Generator | Output >> Input. Writing, drafting, summarisation. |
| 💬 Conversational | Balanced usage. Brainstorming, back-and-forth dialogue. |
| 🔍 Lookup / Q&A | Short inputs, short outputs, many requests. Using AI like a search engine. |
| 🧪 Explorer | Low/irregular usage. Still discovering AI capabilities. |

---

## Step 4 — Model Recommendation Per Category

Based on category, suggest a better-fit model:

| Category | Suggested Vendor | Suggested Model | Reason |
|----------|-----------------|-----------------|--------|
| 🧑‍💻 Power / Technical | Google | Gemini 2.5 Pro | Best-in-class for large context, code, and technical analysis. Handles massive inputs efficiently with competitive pricing at scale. |
| ✍️ Content Generator | Anthropic | Claude Sonnet | Excellent long-form writing quality and instruction following. Produces coherent, nuanced text with minimal hallucination. |
| 💬 Conversational | Anthropic | Claude Sonnet | Strong conversational reasoning. Fast enough for dialogue without overpaying for Opus-tier models. |
| 🔍 Lookup / Q&A | OpenAI | GPT-4o mini | Fast and cheap for short Q&A tasks. No need for a flagship model when inputs and outputs are small. |
| 🧪 Explorer | OpenAI | GPT-4o mini | Low-cost entry point for users still discovering use cases. Easy to upgrade later. |

---

## Step 5 — Cost Savings Estimation

**Only recommend a switch if the suggested model is at least 5% cheaper than the current model.**

### Pricing Table (cost per 1M tokens — April 2026)

| Model substring | Input $/1M | Output $/1M |
|----------------|-----------|------------|
| `o3` | $10.00 | $40.00 |
| `o1` | $15.00 | $60.00 |
| `gpt-4.1` | $2.00 | $8.00 |
| `gpt-4o` | $2.50 | $10.00 |
| `gpt-4o-mini` | $0.15 | $0.60 |
| `claude-opus` | $15.00 | $75.00 |
| `claude-sonnet` | $3.00 | $15.00 |
| `claude-haiku` | $0.80 | $4.00 |
| `gemini-2.5-pro` | $1.25 | $10.00 |
| `gemini-2.5-flash` | $0.15 | $0.60 |
| `gemini-2.0-flash` | $0.10 | $0.40 |
| `gemini-flash` | $0.10 | $0.40 |
| `cursor-slow` | $2.50 | $10.00 |
| `cursor-fast` | $0.15 | $0.60 |
| Unknown | $2.50 | $10.00 |

### Savings Calculation

```
# For each user-model row:
curr_cost_per_1M = (input_price * total_input + output_price * total_output) / total_tokens
sugg_cost_per_1M = (sugg_input_price * total_input + sugg_output_price * total_output) / total_tokens

is_cheaper = sugg_cost_per_1M < curr_cost_per_1M * 0.95

savings_pct = (1 - sugg_cost_per_1M / curr_cost_per_1M) * 100
est_savings = total_cost_usd * (savings_pct / 100)
```

**Only include rows where `is_cheaper = true` in the output report.**

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
| Est. Potential Savings | `SUM(est_savings)` for rows where switch is cheaper |
| Forecast Cost | Total Current Cost − Est. Potential Savings |
| Saving % | Est. Potential Savings / Total Current Cost × 100 |

**Top Savings Opportunities Table (below KPIs):**

Show the top 15 user-model rows ranked by `est_savings` descending.

Columns: User, AI Category, Current Model, Suggested Model (prefixed with `➡️`), Current Cost ($), Est. Saving ($ + %)

Footer note:
> *Based on N user-model combinations with cheaper alternatives available. Savings estimated using current token usage × price delta.*

---

### Sheet 2: Cost Savings Opportunities

One row per user-model pair **where a cheaper alternative exists** (`is_cheaper = true`).

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
| AI Category | Emoji + category name (e.g. `🧑‍💻 Power / Technical`) |
| Suggested AI Vendor | e.g. `Google` |
| Suggested AI Model | e.g. `➡️ Gemini 2.5 Pro` |
| Explanation | `Est. {savings_pct}% cost saving (≈$X.XX). {reason text}` |
| Est. Savings ($) | Numeric, formatted as `$#,##0.00`, green fill |

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
| Suggested Model | `Vendor — Model` from recommendation table |

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
1. User uploads their AI usage log (CSV or JSON export from their AI vendor dashboard)
2. System validates required fields, shows row count loaded
3. System aggregates → classifies → computes savings
4. System generates Excel report
5. User downloads report with:
   - Executive summary with total savings opportunity
   - Row-by-row recommendations filtered to only actionable switches
   - Full legend + summary by category
```

---

## Edge Cases to Handle

| Case | Handling |
|------|---------|
| Missing `requests_count` | Default to 1 |
| Missing `cost_usd` | Default to 0 (savings calc still works via pricing table) |
| Missing `date` | Use row count as proxy for active_days |
| `total_input = 0` | Set `ratio = 1.0` (avoid division by zero) |
| `active_days = 0` | Treat as 1 |
| Unknown model name | Default tier = 0.5, default price = $2.50/$10.00 per 1M |
| Suggested model = current model | Still show if it's cheaper (different vendor, same model family) |
| Suggested model is NOT cheaper | Exclude row from "Cost Savings Opportunities" sheet entirely |
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

Cover: 60-day window, mixed models per user (70% primary model, 30% random), realistic token ranges per profile.

---

## Questions to Clarify Before Starting

1. What format does your AI vendor export usage data in? (CSV/JSON, which fields?)
2. Is this single-tenant (one org) or multi-tenant (multiple companies)?
3. Should reports be stored, or is it a stateless generate-and-download flow?
4. Do you need user authentication, or is it an internal tool?
5. Should the Excel file be generated server-side (Python/openpyxl) or client-side (JS library)?

---

*Prompt authored: 2026-04-12 | System: AI Usage Classifier v1 | Owner: Nadav*

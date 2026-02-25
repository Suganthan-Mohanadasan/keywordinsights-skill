---
name: keyword-insights
description: "Use this skill whenever the user wants to cluster keywords, analyse search intent, generate content briefs, run writer agent orders, or retrieve results from the Keyword Insights API. Triggers include: 'cluster these keywords', 'group these keywords', 'what is the search intent', 'generate a content brief', 'write an article with KI', 'check my KI credits', 'topical map', 'content strategy from keywords', 'keyword grouping', 'SERP overlap analysis', 'what should I write about', or any reference to keyword clustering, topical authority, SERP-based keyword analysis, or content planning from keyword lists. Also trigger when the user uploads a CSV of keywords and wants them organised, grouped, or analysed. Requires KWI_API_KEY environment variable to be set."
---

# Keyword Insights API Skill

## Overview

Keyword Insights (KI) is an SEO platform for keyword clustering, search intent classification, content briefs, and AI-powered content generation. This skill covers all public API endpoints and guides you through the full workflow from CSV input to actionable output.

**Base URL:** `https://api.keywordinsights.ai`

**Authentication:** All requests require the `X-API-Key` header. API keys are created from the KI dashboard (API Keys section) and have the prefix `kwi_sk_`. They never expire unless manually deleted. API key access requires a Professional or Premium plan.

Bearer token auth still works but is deprecated. Always use API keys for new integrations.

```bash
# Store the key as an environment variable
KWI_API_KEY="${KWI_API_KEY}"
BASE="https://api.keywordinsights.ai"
```

**Critical header rule:** Do NOT send `Content-Type: application/json` on GET requests as it causes a 400 error. With curl this is straightforward because curl only sends Content-Type when you explicitly add it or use `--json`.

---

## Decision Flowchart

When a user asks you to work with keywords, follow this logic:

1. **User uploads a CSV or provides keywords?** Parse the input first (see Section 1).
2. **"Cluster these keywords" or "group keywords"** Submit a clustering order with `insights: ["cluster", "context"]` (Section 3).
3. **"What is the search intent for..."** Submit an intent-only order with `insights: ["context"]` (Section 3).
4. **"Generate a content brief for X"** Submit a content brief order (Section 6).
5. **"Write an article about X"** Submit a writer agent order (Section 7). Warn the user this costs approximately 1,200 credits per article.
6. **"What questions are people asking about X"** Submit a keyword content order (Section 9).
7. **"Check my credits" or "how many credits do I have"** Call the user endpoint (Section 2).

Always check credits before submitting large orders (500+ keywords). Inform the user of the estimated cost before proceeding.

---

## 1. Parsing Keyword Input

Users will typically provide keywords in one of these formats. Parse them into two arrays: `keywords` and `search_volumes`.

### From a CSV file (most common)

Ahrefs exports are UTF-16 encoded with tab separators. Other tools export UTF-8 with commas. Detect and handle both:

```bash
# Detect encoding and convert to UTF-8 if needed
file_encoding=$(file -bi "$CSV_PATH" | grep -oP 'charset=\K[^ ;]+')
if [[ "$file_encoding" == "utf-16le" || "$file_encoding" == "utf-16"* ]]; then
  iconv -f UTF-16 -t UTF-8 "$CSV_PATH" > /tmp/keywords_utf8.csv
  CSV_PATH="/tmp/keywords_utf8.csv"
fi

# For tab-separated files (Ahrefs), extract Keyword and Volume columns
# Identify column positions from the header row, then extract data
```

Use Python when complex CSV parsing is needed (handling quoted fields, mixed encodings, detecting the right columns by header name). The key columns to look for are: `Keyword` (or `keyword`, `Search Query`, `Term`) and `Volume` (or `Search Volume`, `search_volume`, `Avg. monthly searches`).

If volumes are missing or empty, default them to 0.

### From plain text in chat

If the user pastes keywords directly, split by newlines. If no volumes are provided, set all volumes to 0.

### Validation

Before submitting to the API, always verify:
- Minimum 5 keywords (API requirement for clustering)
- Keywords and search_volumes arrays are exactly the same length
- No empty keyword strings

---

## 2. Check Account Info

```bash
curl -s "${BASE}/api/user/" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

Returns user profile, plan type, and credit balance. Key fields: `balance` (total), `balance_subscription` (plan credits), `balance_top_up` (purchased credits).

**When to use:** Before large orders, or when the user asks about credits. If the balance is insufficient for the planned operation, inform the user and suggest they top up.

---

## 3. Keyword Clustering & Intent Orders

Clustering groups keywords into topical clusters using live SERP data. Minimum 5 keywords required.

### 3a. Submit Order

```bash
curl -s -X POST "${BASE}/api/keywords-insights/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "My Project",
    "keywords": ["keyword clustering", "keyword research", "topical authority", "seo strategy", "content clusters"],
    "search_volumes": [2300, 3210, 5500, 1800, 900],
    "language": "en",
    "location": "United States",
    "device": "desktop",
    "insights": ["cluster", "context", "rank"],
    "clustering_method": "volume",
    "grouping_accuracy": 4,
    "hub_creation_method": "medium",
    "url": "https://example.com"
  }'
```

Extract the order ID from the response:
```bash
order_id=$(curl -s -X POST "${BASE}/api/keywords-insights/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "$payload" | jq -r '.order_id')
```

For intent-only (faster, cheaper), set `insights` to `["context"]` only. The `"context"` insight does NOT require `"cluster"` to be present.

### 3b. Cost Estimation

Before submitting, you can estimate the cost:

```bash
curl -s "${BASE}/api/keywords-insights/order/cost/?n_keywords=500&insights=cluster&insights=context" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

### 3c. Poll for Status

Use the **dedicated status endpoint** — this is a GET to the same base path with `order_id` as a query parameter:

```bash
order_id="YOUR_ORDER_ID"

while true; do
  response=$(curl -s "${BASE}/api/keywords-insights/order/?order_id=${order_id}" \
    -H "X-API-Key: ${KWI_API_KEY}")

  status=$(echo "$response" | jq -r '.status')
  progress=$(echo "$response" | jq -r '.progress // "unknown"')

  echo "Status: ${status}, Progress: ${progress}"

  if [ "$status" = "done" ] || [ "$status" = "completed" ]; then
    echo "Order complete!"
    break
  fi

  sleep 10
done
```

The status response includes: `status`, `progress` (0 to 1), and `results_files` (xlsx, google_sheets links) when complete.

### 3d. Retrieve Results

**JSON results (paginated):**

```bash
curl -s "${BASE}/api/keywords-insights/order/json/${order_id}/?page_size=50&page_number=1&sort_by=search_volume&ascending=false" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

Pagination parameters: `page_size` (default 50), `page_number` (starts at 1), `sort_by` (e.g. `search_volume`, `cluster`), `ascending` (`true` or `false`), `filter_id`.

**XLSX download:**

```bash
curl -s "${BASE}/api/keywords-insights/order/xlsx/${order_id}/" \
  -H "X-API-Key: ${KWI_API_KEY}" --output results.xlsx
```

### 3e. List Past Orders

```bash
# Get last 10 orders
curl -s "${BASE}/api/keywords-insights/orders/?n_orders=10" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .

# Or get orders from the last 7 days
curl -s "${BASE}/api/keywords-insights/orders/?n_days=7" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

Note: use `n_orders` OR `n_days`, not both.

### Insights array options

- `"cluster"` — groups keywords into topical clusters based on SERP overlap
- `"context"` — classifies search intent per keyword (informational, commercial, transactional, navigational)
- `"rank"` — tracks rankings for a given URL (requires the `url` field)

### Parameter reference

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `project_name` | string | required | Label shown in the KI dashboard |
| `keywords` | array | required | Minimum 5 keywords |
| `search_volumes` | array | required | Must match length of `keywords`. Use 0 for unknown. |
| `language` | string | `"en"` | Language code |
| `location` | string | `"United States"` | Full location name |
| `device` | string | `"desktop"` | `"desktop"`, `"mobile"`, `"tablet"` |
| `insights` | array | required | `["cluster"]`, `["context"]`, or combination |
| `clustering_method` | string | `"volume"` | `"volume"` (recommended default) or `"agglomerative"` |
| `grouping_accuracy` | int | `4` | 1 to 7. Higher = stricter, smaller clusters. 4 is a good default. |
| `hub_creation_method` | string | `"medium"` | `"soft"`, `"medium"`, `"hard"` |
| `url` | string | optional | Your domain. Required only when `"rank"` is in insights. |
| `folder_id` | string | optional | Dashboard folder ID for organisation |

### Sensible defaults when the user does not specify

- `clustering_method`: `"volume"` (most common, groups around highest volume keyword)
- `grouping_accuracy`: `4` (balanced between too broad and too strict)
- `hub_creation_method`: `"medium"`
- `device`: `"desktop"`
- `language`: `"en"`
- `location`: `"United States"` (ask user if unclear)
- `insights`: `["cluster", "context"]` (most users want both clustering and intent)

---

## 4. Presenting Clustering Results

After retrieving results, present them in whichever format is most useful:
- **Summary in chat:** Show cluster count, top clusters by volume, intent distribution
- **Save as file:** Write results to a CSV or XLSX in the workspace folder and provide a download link
- **Both:** For large result sets, give a summary in chat and save the full data as a file

---

## 5. Reference Data (Languages & Locations)

```bash
# Get supported languages
curl -s "${BASE}/api/keywords-insights/languages/" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .

# Get supported locations
curl -s "${BASE}/api/keywords-insights/locations/" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .

# Search locations by name
curl -s "${BASE}/api/keywords-insights/locations-live/united/" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

Use the languages endpoint to check which insights are supported per language.

---

## 6. Content Brief Orders

Generates a structured SEO content brief based on SERP analysis. Costs approximately 100 credits per brief.

### 6a. Submit Order

```bash
curl -s -X POST "${BASE}/api/content-brief/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "keyword": "keyword clustering",
    "language": "en",
    "location": "United States"
  }'
```

The response returns an `id` field (UUID) — use this for polling.

### 6b. Poll for Status

```bash
brief_id="YOUR_BRIEF_ID"

while true; do
  response=$(curl -s "${BASE}/api/content-brief/order/?id=${brief_id}" \
    -H "X-API-Key: ${KWI_API_KEY}")

  status=$(echo "$response" | jq -r '.status')
  processing=$(echo "$response" | jq -r '.processing_status // "unknown"')

  echo "Status: ${status}, Processing: ${processing}"

  # status is a boolean: true = complete
  if [ "$status" = "true" ]; then
    echo "Content brief complete!"
    echo "$response" | jq .
    break
  fi

  sleep 10
done
```

The response when complete includes: `keyword`, `location`, `language`, `pages_count`, `headings_count`, `word_count` (min/avg/max), `brief_title_suggests`, `brief_description_suggests`, and `raw` (detailed content items with headings, bullet points, page structure).

### 6c. Generate Outline from Brief

After a content brief is complete, you can auto-generate an outline:

```bash
# Submit outline generation
outline_response=$(curl -s -X POST "${BASE}/api/content-brief/order/${brief_id}/outline/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "additional_context": "Focus on practical tips for beginners"
  }')

auto_generate_id=$(echo "$outline_response" | jq -r '.auto_generate_order_id')

# Poll for outline completion
while true; do
  outline=$(curl -s "${BASE}/api/content-brief/order/${brief_id}/outline/?auto_generate_order_id=${auto_generate_id}" \
    -H "X-API-Key: ${KWI_API_KEY}")

  outline_status=$(echo "$outline" | jq -r '.status')

  if [ "$outline_status" = "true" ]; then
    echo "Outline ready!"
    echo "$outline" | jq '.string'
    break
  fi

  sleep 5
done
```

### 6d. List Past Content Briefs

```bash
curl -s "${BASE}/api/content-brief/orders/?page=1&page_size=10" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

### 6e. Supported Languages for Briefs

```bash
curl -s "${BASE}/api/content-brief/languages/" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

---

## 7. Writer Agent Orders

Generates a full long-form article draft. This is the most expensive operation (~1,200 credits per article).

### 7a. Check Available Options

```bash
curl -s "${BASE}/api/writer-agent/options/" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

Returns available `content_types`, `points_of_view`, and `tones_of_voice` (array of objects with `id` and `name`).

### 7b. Submit Order

**Important:** The writer agent uses `language_code` and `location_name` (not `language` and `location` like the other endpoints).

```bash
curl -s -X POST "${BASE}/api/writer-agent/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "keyword": "how to do keyword clustering",
    "language_code": "en",
    "location_name": "United States",
    "content_type": "article",
    "point_of_view": "Third person",
    "additional_insights": "Include practical examples and tool recommendations"
  }'
```

The response returns an `id` field (UUID).

### 7c. Poll for Status

```bash
writer_id="YOUR_WRITER_ORDER_ID"

while true; do
  response=$(curl -s "${BASE}/api/writer-agent/order/?id=${writer_id}" \
    -H "X-API-Key: ${KWI_API_KEY}")

  status=$(echo "$response" | jq -r '.status')
  processing=$(echo "$response" | jq -r '.processing_status // "unknown"')

  echo "Status: ${status}, Processing: ${processing}"

  if [ "$status" = "done" ] || [ "$status" = "completed" ]; then
    echo "Article complete!"
    echo "$response" | jq '.generated_article'
    break
  fi

  sleep 15
done
```

The `generated_article` field is null until the order completes. The `processing_status` object shows progress per stage.

### Writer Agent Parameter Reference

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `keyword` | string | required | Target keyword for the article |
| `language_code` | string | required | e.g. `"en"` — note: different field name from other endpoints |
| `location_name` | string | required | e.g. `"United States"` — note: different field name from other endpoints |
| `content_type` | string | `"article"` | `"article"` or `"landing_page"` |
| `point_of_view` | string | optional | `"First person"`, `"Second person"`, `"Third person"` |
| `additional_insights` | string | optional | Extra instructions for the writer |
| `folder_id` | string | optional | Dashboard folder ID |

**Always confirm with the user before submitting a writer agent order and check their credit balance first.**

---

## 8. Advanced Ranking Orders

SERP-based ranking analysis for a keyword and domain.

### 8a. Single Keyword Order

```bash
curl -s -X POST "${BASE}/api/advanced-ranking/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "keyword": "seo tools",
    "language": "en",
    "location": "United States",
    "domain": "example.com",
    "include_word_count": true
  }'
```

### 8b. Poll for Status (Single)

```bash
ranking_id="YOUR_ORDER_ID"

while true; do
  response=$(curl -s "${BASE}/api/advanced-ranking/order/?order_id=${ranking_id}" \
    -H "X-API-Key: ${KWI_API_KEY}")

  status=$(echo "$response" | jq -r '.status')
  echo "Status: ${status}"

  if [ "$status" = "done" ]; then
    echo "Ranking analysis complete!"
    echo "$response" | jq .
    break
  fi

  sleep 10
done
```

Results include: `top_rankings` (array with url, word_count), `domain_rankings`, `n_domain_rankings`, `related_questions`, `related_searches`.

### 8c. Batch Order (Multiple Keywords)

```bash
curl -s -X POST "${BASE}/api/advanced-ranking/batch/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "keywords": ["seo tools", "best seo software", "seo platform"],
    "language": "en",
    "location": "United States",
    "domain": "example.com",
    "include_word_count": true
  }'
```

### 8d. Poll for Status (Batch)

```bash
batch_id="YOUR_BATCH_ORDER_ID"

while true; do
  response=$(curl -s "${BASE}/api/advanced-ranking/batch/order/?order_id=${batch_id}" \
    -H "X-API-Key: ${KWI_API_KEY}")

  status=$(echo "$response" | jq -r '.status')
  echo "Status: ${status}"

  if [ "$status" = "done" ]; then
    echo "Batch ranking complete!"
    echo "$response" | jq .
    break
  fi

  sleep 10
done
```

### Advanced Ranking Parameter Reference

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `keyword` / `keywords` | string / array | required | Single string for `/order/`, array for `/batch/order/` |
| `language` | string | required | Language code |
| `location` | string | required | Full location name |
| `domain` | string | required | Domain to check rankings for (not `url`) |
| `device` | string | `"desktop"` | `"desktop"` or `"mobile"` |
| `include_word_count` | boolean | optional | Include word count for ranking pages |

---

## 9. Keyword Content Orders

Retrieves People Also Ask questions, Reddit questions, Quora questions, meta titles, and meta descriptions for a keyword.

### 9a. Submit Order

```bash
curl -s -X POST "${BASE}/api/keyword-content/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "keyword": "seo tools",
    "language": "en",
    "location": "United States",
    "content_insights": ["paa", "reddit_questions", "quora_questions", "meta_titles", "meta_descriptions"]
  }'
```

**Credit costs per insight:** PAA, Reddit questions, Quora questions = 50 credits each. Meta titles, meta descriptions = 200 credits each.

### 9b. Poll for Status

```bash
content_id="YOUR_ORDER_ID"

while true; do
  response=$(curl -s "${BASE}/api/keyword-content/order/?order_id=${content_id}" \
    -H "X-API-Key: ${KWI_API_KEY}")

  status=$(echo "$response" | jq -r '.status')
  echo "Status: ${status}"

  if [ "$status" = "done" ]; then
    echo "Keyword content complete!"
    echo "$response" | jq .
    break
  fi

  sleep 10
done
```

Results include arrays per requested insight type and CSV download links.

### Keyword Content Parameter Reference

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `keyword` | string | required | Target keyword |
| `language` | string | required | Language code |
| `location` | string | required | Full location name |
| `content_insights` | array | required | Any combination of: `"paa"`, `"reddit_questions"`, `"quora_questions"`, `"meta_titles"`, `"meta_descriptions"` |
| `device` | string | `"desktop"` | `"desktop"` or `"mobile"` |

---

## Error Handling

| Code | Meaning | What to do |
|---|---|---|
| `400` | Bad request | Check payload format. Common cause: sending Content-Type on a GET request, or malformed JSON. |
| `401` | Invalid or missing API key | Ask the user to verify their KWI_API_KEY environment variable is set and correct. |
| `402` | Insufficient credits | Inform the user. Show their current balance and the estimated cost of the operation. |
| `403` | Feature not available on current plan | API keys require Professional or Premium. Tell the user they may need to upgrade. |
| `422` | Invalid payload | Check that keywords and search_volumes arrays are the same length, minimum 5 keywords, and all required fields are present. |

When any error occurs, show the user the full error response body so they can understand what went wrong.

---

## Full End-to-End Example (Clustering)

```bash
KWI_API_KEY="${KWI_API_KEY}"
BASE="https://api.keywordinsights.ai"

# 1. Check credits
echo "Checking credits..."
curl -s "${BASE}/api/user/" -H "X-API-Key: ${KWI_API_KEY}" | jq '.balance'

# 2. Estimate cost
echo "Estimating cost..."
curl -s "${BASE}/api/keywords-insights/order/cost/?n_keywords=5&insights=cluster&insights=context" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq '.cost'

# 3. Submit clustering order
echo "Submitting order..."
order_id=$(curl -s -X POST "${BASE}/api/keywords-insights/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "MCP Demo",
    "keywords": ["keyword clustering", "keyword research", "topical authority", "seo content strategy", "keyword grouping"],
    "search_volumes": [2300, 3210, 5500, 1800, 900],
    "language": "en",
    "location": "United States",
    "device": "desktop",
    "insights": ["cluster", "context"],
    "clustering_method": "volume",
    "grouping_accuracy": 4,
    "hub_creation_method": "medium"
  }' | jq -r '.order_id')

echo "Order submitted: ${order_id}"

# 4. Poll using the dedicated status endpoint
while true; do
  status_response=$(curl -s "${BASE}/api/keywords-insights/order/?order_id=${order_id}" \
    -H "X-API-Key: ${KWI_API_KEY}")

  status=$(echo "$status_response" | jq -r '.status')
  progress=$(echo "$status_response" | jq -r '.progress // 0')

  echo "Status: ${status}, Progress: ${progress}"

  if [ "$status" = "done" ] || [ "$status" = "completed" ]; then
    echo "Order complete!"
    break
  fi

  sleep 10
done

# 5. Retrieve JSON results
curl -s "${BASE}/api/keywords-insights/order/json/${order_id}/?page_size=50&page_number=1" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

---

## Polling Summary by Feature

Each feature has its own polling route. **Never mix polling endpoints across features.**

| Feature | Submit (POST) | Poll (GET) | ID field | Completion check |
|---|---|---|---|---|
| Clustering/Intent | `/api/keywords-insights/order/` | `/api/keywords-insights/order/?order_id={id}` | `order_id` | `status` = `"done"` |
| Content Brief | `/api/content-brief/order/` | `/api/content-brief/order/?id={id}` | `id` | `status` = `true` |
| Writer Agent | `/api/writer-agent/order/` | `/api/writer-agent/order/?id={id}` | `id` | `status` = `"done"` or `"completed"` |
| Advanced Ranking | `/api/advanced-ranking/order/` | `/api/advanced-ranking/order/?order_id={id}` | `order_id` | `status` = `"done"` |
| Adv. Ranking Batch | `/api/advanced-ranking/batch/order/` | `/api/advanced-ranking/batch/order/?order_id={id}` | `order_id` | `status` = `"done"` |
| Keyword Content | `/api/keyword-content/order/` | `/api/keyword-content/order/?order_id={id}` | `order_id` | `status` = `"done"` |
| Brief Outline | `/api/content-brief/order/{id}/outline/` | `/api/content-brief/order/{id}/outline/?auto_generate_order_id={id}` | `auto_generate_order_id` | `status` = `true` |

---

## Notes

- Minimum 5 keywords per clustering order
- Keywords and search_volumes arrays must always be the same length
- Do NOT send Content-Type header on GET requests
- Each feature has its own dedicated polling endpoint (see table above)
- Clustering credits are consumed per keyword
- Content briefs cost approximately 100 credits each
- Writer agent costs approximately 1,200 credits per article
- Writer agent uses `language_code` and `location_name` (not `language` and `location`)
- Advanced ranking uses `domain` (not `url`) and `keyword` (singular, not array) for single orders
- `hub_creation_method` values are `"soft"`, `"medium"`, `"hard"`
- `clustering_method` values are `"volume"` or `"agglomerative"`
- Large orders (1,000+ keywords) can take several minutes to complete
- API key access requires a Professional or Premium plan
- API keys are prefixed with `kwi_sk_` and created from the KI dashboard
- Bearer tokens still work but are deprecated; always use API keys
- Full API reference: https://api.keywordinsights.ai/apidocs/
- Docs: https://docs.keywordinsights.ai/api/api-key-authentication

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
2. **"Cluster these keywords" or "group keywords"** Submit a clustering order with `insights: ["cluster", "context"]` (Section 3a).
3. **"What is the search intent for..."** Submit an intent-only order with `insights: ["context"]` (Section 3b).
4. **"Generate a content brief for X"** Submit a content brief order (Section 6).
5. **"Write an article about X"** Submit a writer agent order (Section 7). Warn the user this costs approximately 1,200 credits per article.
6. **"Check my credits" or "how many credits do I have"** Call the user endpoint (Section 2).

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
- Minimum 5 keywords (API requirement)
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

## 3. Keyword Clustering Orders

Clustering groups keywords into topical clusters using live SERP data. Minimum 5 keywords required.

### 3a. Full Clustering Order (cluster + intent + rank)

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

### 3b. Intent Only Order (faster, cheaper)

Use when the user only needs search intent classification, not full SERP clustering. The key difference is `insights` contains only `["context"]`.

```bash
curl -s -X POST "${BASE}/api/keywords-insights/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "Intent Analysis",
    "keywords": ["seo tools", "best keyword tool", "ahrefs vs semrush", "keyword clustering tool", "topical authority seo"],
    "search_volumes": [1200, 880, 450, 320, 210],
    "language": "en",
    "location": "United States",
    "device": "desktop",
    "insights": ["context"],
    "clustering_method": "volume",
    "grouping_accuracy": 4,
    "hub_creation_method": "medium"
  }'
```

Note: `"context"` alone is valid and does NOT require `"cluster"` to be present.

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
| `clustering_method` | string | `"volume"` | `"volume"` (recommended default) or `"hub"` |
| `grouping_accuracy` | int | `4` | 1 to 7. Higher = stricter, smaller clusters. 4 is a good default. |
| `hub_creation_method` | string | `"medium"` | `"low"`, `"medium"`, `"high"` |
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

## 4. Polling for Results

Orders are asynchronous. There is NO separate status endpoint. Poll by hitting the results endpoint directly. A `200` response means the order is complete. Any other status code (typically 202 or 404) means it is still processing.

```bash
order_id="YOUR_ORDER_ID"

while true; do
  response=$(curl -s -o /tmp/ki_result.json -w "%{http_code}" \
    "${BASE}/api/keywords-insights/order/json/${order_id}/" \
    -H "X-API-Key: ${KWI_API_KEY}")

  if [ "$response" = "200" ]; then
    echo "Order complete."
    break
  fi

  echo "HTTP ${response}, still processing... waiting 10 seconds."
  sleep 10
done

# Results are now in /tmp/ki_result.json
cat /tmp/ki_result.json | jq .
```

**Timing expectations:**
- Small orders (under 100 keywords): 1 to 3 minutes
- Medium orders (100 to 500 keywords): 3 to 10 minutes
- Large orders (500+ keywords): 10+ minutes

Keep the user informed during polling. Update them every 30 seconds or so with a brief status message.

---

## 5. Retrieve Results

### JSON results (paginated)

```bash
curl -s "${BASE}/api/keywords-insights/order/json/${order_id}/?page_size=50&page_number=1&sort_by=search_volume&ascending=false" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

Pagination parameters: `page_size` (default 50), `page_number` (starts at 1), `sort_by` (e.g. `search_volume`, `cluster`), `ascending` (`true` or `false`).

### XLSX download link

```bash
curl -s "${BASE}/api/keywords-insights/order/xlsx/${order_id}/" \
  -H "X-API-Key: ${KWI_API_KEY}" | jq .
```

Returns a download URL for the XLSX file.

### Presenting results to the user

After retrieving results, present them in whichever format is most useful:
- **Summary in chat:** Show cluster count, top clusters by volume, intent distribution
- **Save as file:** Write results to a CSV or XLSX in the workspace folder and provide a download link
- **Both:** For large result sets, give a summary in chat and save the full data as a file

---

## 6. Content Brief Orders

Generates a structured SEO content brief based on SERP analysis.

```bash
curl -s -X POST "${BASE}/api/content-brief/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "keyword": "keyword clustering",
    "language": "en",
    "location": "United States",
    "project_name": "Content Brief: keyword clustering"
  }'
```

Costs approximately 100 credits per brief. Poll using the same pattern as clustering orders (Section 4).

---

## 7. Writer Agent Orders

Generates a full long form article draft. This is the most expensive operation.

```bash
curl -s -X POST "${BASE}/api/writer-agent/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "keyword": "how to do keyword clustering",
    "language": "en",
    "location": "United States",
    "project_name": "Writer Agent: keyword clustering guide",
    "tone": "professional",
    "word_count": 1500
  }'
```

**Important:** This costs approximately 1,200 credits per article. Always confirm with the user before submitting a writer agent order and check their credit balance first.

---

## 8. Advanced Ranking Orders

SERP-based ranking analysis.

```bash
curl -s -X POST "${BASE}/api/advanced-ranking/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "Advanced Ranking: seo tools",
    "keywords": ["seo tools", "best seo software"],
    "language": "en",
    "location": "United States",
    "url": "https://example.com"
  }'
```

---

## 9. Keyword Content Orders

```bash
curl -s -X POST "${BASE}/api/keyword-content/order/" \
  -H "X-API-Key: ${KWI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "Keyword Content: seo tools",
    "keyword": "seo tools",
    "language": "en",
    "location": "United States"
  }'
```

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

## Full End to End Example

```bash
KWI_API_KEY="${KWI_API_KEY}"
BASE="https://api.keywordinsights.ai"

# 1. Check credits
echo "Checking credits..."
curl -s "${BASE}/api/user/" -H "X-API-Key: ${KWI_API_KEY}" | jq '.balance'

# 2. Submit clustering order
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

# 3. Poll until complete
while true; do
  http_code=$(curl -s -o /tmp/ki_result.json -w "%{http_code}" \
    "${BASE}/api/keywords-insights/order/json/${order_id}/" \
    -H "X-API-Key: ${KWI_API_KEY}")

  if [ "$http_code" = "200" ]; then
    echo "Order complete!"
    break
  fi

  echo "HTTP ${http_code}, still processing..."
  sleep 10
done

# 4. Display results
cat /tmp/ki_result.json | jq .
```

---

## Notes

- Minimum 5 keywords per clustering order
- Keywords and search_volumes arrays must always be the same length
- Do NOT send Content-Type header on GET requests
- There is no separate status endpoint; poll the results endpoint directly (200 = complete)
- Clustering credits are consumed per keyword
- Content briefs cost approximately 100 credits each
- Writer agent costs approximately 1,200 credits per article
- Large orders (1,000+ keywords) can take several minutes to complete
- API key access requires a Professional or Premium plan
- API keys are prefixed with `kwi_sk_` and created from the KI dashboard
- Bearer tokens still work but are deprecated; always use API keys
- Full API reference: https://api.keywordinsights.ai/apidocs/
- Docs: https://docs.keywordinsights.ai/api/api-key-authentication

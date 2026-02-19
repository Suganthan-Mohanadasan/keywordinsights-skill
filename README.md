# Keyword Insights Skill for Claude

A native skill that connects [Keyword Insights](https://www.keywordinsights.ai) to Claude, Anthropic's AI assistant. Once installed, you can run your entire keyword research and content planning workflow through a simple conversation — no manual exports, no switching between tools.

## What It Does

Upload a keyword CSV or paste a list, and ask Claude to:

- **Cluster keywords** using live SERP overlap data
- **Classify search intent** across a full keyword list
- **Generate content briefs** based on what is ranking
- **Write full articles** using the KI Writer Agent
- **Check your credit balance** at any time
- **Build topical maps** from a broad keyword set

## Requirements

- A [Keyword Insights](https://www.keywordinsights.ai) account on the **Professional or Premium plan**
- A KI API key (created from the API Keys section of your dashboard — starts with `kwi_sk_`)
- Access to Claude with skill support

## Installation

1. Copy the `keyword-insights-SKILL.md` file into your Claude skills directory
2. Set your API key as an environment variable:

```bash
export KWI_API_KEY="kwi_sk_your_key_here"
```

3. Start a conversation with Claude and upload a keyword CSV or paste a keyword list

## Usage Examples

```
"Cluster this CSV for the UK market"

"What is the search intent for these keywords?"

"Generate a content brief for 'topical authority'"

"Write a 1,500 word article about keyword clustering in a professional tone"

"How many credits do I have?"
```

## Credit Costs

| Operation | Approximate cost |
|---|---|
| Keyword clustering | 1 credit per keyword |
| Intent classification only | Less than full clustering |
| Content brief | ~100 credits |
| Writer Agent (full article) | ~1,200 credits |

Claude will always confirm before submitting any expensive operation.

## Supported Keyword Sources

Claude can parse exports from Ahrefs, Semrush, Google Search Console, Moz, SE Ranking, and most other SEO tools. Column names are detected automatically.

## Documentation

- [Full docs](https://docs.keywordinsights.ai)
- [API reference](https://api.keywordinsights.ai/apidocs/)
- [API key authentication](https://docs.keywordinsights.ai/api/api-key-authentication)

## License

MIT

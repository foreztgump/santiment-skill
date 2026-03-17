# Santiment API Skill

A [Claude Code skill](https://code.claude.com/docs/en/skills) that teaches AI agents how to use the [Santiment GraphQL API](https://api.santiment.net/graphql) for cryptocurrency market data.

## What This Is

A skill package at `.claude/skills/santiment-api/` that an AI agent invokes when it needs to fetch crypto data from Santiment. It covers:

- **Authentication** via `SANTIMENT_API_KEY` environment variable
- **All query patterns**: timeseries, multi-asset, aggregated, histogram, server-side transforms, raw SQL
- **All metric categories**: financial, on-chain, exchange, social/sentiment, development, NFT, supply distribution
- **Operational knowledge**: rate limits per plan, complexity limits, error handling, common pitfalls
- **Python integration**: `sanpy` library with `get`, `get_many`, `execute_sql`, `AsyncBatch`

## Skill Structure

```
.claude/skills/santiment-api/
├── SKILL.md                         # Core: auth, query patterns, pitfalls
└── references/
    ├── common-queries.md            # Price lookups, filtering, trending, blockchains
    ├── metrics.md                   # All metric categories with names
    └── rate-limits.md               # Per-plan limits, complexity, SQL queries
```

Uses [progressive disclosure](https://code.claude.com/docs/en/skills) — SKILL.md loads first (176 lines), reference files load only when needed.

## Setup

1. Get a Santiment API key at [app.santiment.net/account](https://app.santiment.net/account)

2. Set the environment variable:
   ```bash
   export SANTIMENT_API_KEY="your_api_key_here"
   ```

3. Copy the skill into your project:
   ```bash
   cp -r .claude/skills/santiment-api /path/to/your/project/.claude/skills/
   ```

   Or install the packaged `.skill` file (if available in `dist/`):
   ```bash
   claude skill install dist/santiment-api.skill
   ```

## Quick Test

```bash
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Apikey $SANTIMENT_API_KEY" \
  --data '{"query": "{ getMetric(metric: \"price_usd\") { timeseriesData(slug: \"bitcoin\", from: \"utc_now-7d\", to: \"utc_now\", interval: \"1d\") { datetime value } } }"}' \
  https://api.santiment.net/graphql | jq .
```

## Verified Patterns

All 16 documented patterns have been live-tested against the Santiment API:

| Pattern | Status |
|---------|--------|
| Authentication | Verified |
| Timeseries (single asset) | Verified |
| Timeseries (multi-asset) | Verified |
| Aggregated data | Verified |
| Histogram data | Verified |
| Transform: moving average | Verified |
| Transform: consecutive differences | Verified |
| includeIncompleteData | Verified |
| Trending words | Verified |
| Available metrics list | Verified |
| Metadata + availableSince | Verified |
| Access restrictions | Verified |
| Supported blockchains | Verified |
| projectBySlug with metrics | Verified |
| allProjectsByFunction (filter/order) | Verified |
| Raw SQL (computeRawClickhouseQuery) | Verified |

## Links

- [Santiment API Explorer](https://api.santiment.net/graphiql)
- [Santiment Academy (Developer Docs)](https://academy.santiment.net/for-developers/)
- [sanpy Python Library](https://github.com/santiment/sanpy)
- [Claude Code Skills Docs](https://code.claude.com/docs/en/skills)

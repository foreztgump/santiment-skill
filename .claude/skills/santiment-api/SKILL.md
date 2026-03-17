---
name: santiment-api
description: Use the Santiment GraphQL API to fetch cryptocurrency market data — on-chain metrics (active addresses, NVT, MVRV, exchange flows), social sentiment, development activity, financial data (price, volume, marketcap), NFT metrics, and custom SQL queries. Use when working with crypto data, Santiment, sanpy Python library, or any task requiring blockchain analytics. Covers authentication, all query patterns, rate limits, and 1000+ metrics across 12+ blockchains.
---

# Santiment API

## Authentication

Read API key from `SANTIMENT_API_KEY` env var (or `SANPY_APIKEY` — sanpy auto-reads this natively).
Obtain key: https://app.santiment.net/account. Pass as header: `Authorization: Apikey <key>`.

```bash
curl -X POST -H "Content-Type: application/json" \
  -H "Authorization: Apikey $SANTIMENT_API_KEY" \
  --data '{"query": "{ currentUser { id email } }"}' \
  https://api.santiment.net/graphql
```

```python
import os, san
san.ApiConfig.api_key = os.environ["SANTIMENT_API_KEY"]
```

## Endpoint

`https://api.santiment.net/graphql` — POST, `Content-Type: application/json`, body: `{"query": "..."}`.
Explorer: https://api.santiment.net/graphiql | https://api.santiment.net/graphiql_advanced (supports auth headers)

## Core Concepts

- **Slugs**: `"bitcoin"`, `"ethereum"`, etc. Look up: `{ projectBySlug(slug: "bitcoin") { slug name ticker } }`
- **Metrics**: Unified `getMetric(metric: "<name>")`. Names are snake_case. Some use hyphens for time windows (e.g., `dev_activity-1d`)
- **Intervals**: `"1d"`, `"1h"`, `"30m"`, `"1w"`. Check min via `metadata { minInterval }`
- **Relative dates**: `"utc_now"`, `"utc_now-7d"`. In intervals `m`=minutes; in dates `m`=months
- **Aggregations**: `AVG`, `SUM`, `MIN`, `MAX`, `LAST`, `FIRST`, `MEDIAN`, `ANY`

## Query Patterns

### Timeseries (single asset)

```graphql
{
  getMetric(metric: "daily_active_addresses") {
    timeseriesData(slug: "ethereum", from: "utc_now-7d", to: "utc_now", interval: "1d") {
      datetime
      value
    }
  }
}
```

Python: `san.get("daily_active_addresses", slug="ethereum", from_date="2024-01-01", to_date="2024-01-31", interval="1d")`

Use `timeseriesDataJson` for compact JSON output. Add `includeIncompleteData: true` to include current incomplete interval.

### Timeseries (multiple assets)

```graphql
{
  getMetric(metric: "price_usd") {
    timeseriesDataPerSlugJson(
      selector: { slugs: ["bitcoin", "ethereum"] }
      from: "utc_now-7d", to: "utc_now", interval: "1d")
  }
}
```

Python: `san.get_many("price_usd", slugs=["bitcoin", "ethereum"], from_date="2024-01-01", to_date="2024-01-31", interval="1d")`

### Aggregated (single value)

```graphql
{
  getMetric(metric: "daily_active_addresses") {
    aggregatedTimeseriesData(slug: "ethereum", from: "utc_now-30d", to: "utc_now", aggregation: AVG)
  }
}
```

### Server-side transforms

```graphql
{
  getMetric(metric: "dev_activity") {
    timeseriesDataJson(slug: "ethereum", from: "utc_now-365d", to: "utc_now", interval: "7d",
      transform: { type: "moving_average", movingAverageBase: 4 })
  }
}
```

Types: `"moving_average"` (needs `movingAverageBase`), `"consecutive_differences"`.

### Histogram

```graphql
{
  getMetric(metric: "age_distribution") {
    histogramData(slug: "santiment", from: "2024-01-01T00:00:00Z", to: "2024-03-01T00:00:00Z", interval: "1d") {
      values {
        ... on DatetimeRangeFloatValueList { data { range value } }
        ... on FloatRangeFloatValueList { data { range value } }
      }
    }
  }
}
```

Include both inline fragments — union type varies by metric.

### Metadata & discovery

```graphql
{ getAvailableMetrics }
{ projectBySlug(slug: "ethereum") { availableMetrics } }
{
  getMetric(metric: "daily_active_addresses") {
    metadata { metric defaultAggregation minInterval dataType }
    availableSince(slug: "ethereum")
  }
}
{
  getAccessRestrictions(product: SANAPI, plan: FREE, filter: METRIC) {
    name isAccessible isRestricted isDeprecated restrictedFrom restrictedTo
  }
}
```

Python: `san.available_metrics()`, `san.available_metrics_for_slug("ethereum")`, `san.metadata("daily_active_addresses")`

## Batching (Python)

```python
batch = san.AsyncBatch()
batch.get("daily_active_addresses", slug="bitcoin", from_date="2024-01-01", to_date="2024-01-31", interval="1d")
batch.get("price_usd", slug="bitcoin", from_date="2024-01-01", to_date="2024-01-31", interval="1d")
results = batch.execute()  # list of DataFrames, default 10 workers
```

## Pitfalls

- **HTTP 200 errors**: GraphQL errors return HTTP 200 — always check `"errors"` key in response JSON
- **Metric availability**: Not all metrics exist for all slugs — check `availableSince` or handle null
- **Interval vs date units**: `"5m"` interval = 5 minutes; `"utc_now-3m"` = 3 months
- **Incomplete data**: Latest interval excluded by default — use `includeIncompleteData: true`
- **Heavy queries**: Always paginate `allProjects` — unpaginated returns everything
- **Error response**: `{"data": null, "errors": [{"message": "..."}]}` — HTTP 429 = rate limited, 5xx = server error
- **allProjectsByFunction**: `function` arg is a **stringified JSON string**. Enums are lowercase. Returns `{ projects { ... } }`

## References

Read these files as needed for detailed information:

- **[Common queries](references/common-queries.md)**: Price lookups, project details, filtering/ordering assets, trending words, blockchains, `allProjectsByFunction` with full JSON structure
- **[Metrics catalog](references/metrics.md)**: All metric categories with names — financial, on-chain, exchange, social, dev, NFT, supply, whale
- **[Rate limits & restrictions](references/rate-limits.md)**: Per-plan limits, data access restrictions, complexity formula, raw SQL queries (legacy `computeRawClickhouseQuery`)

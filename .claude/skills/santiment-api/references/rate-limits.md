# Rate Limits, Restrictions & SQL

## Table of Contents
- [Rate limits](#rate-limits)
- [Data access restrictions](#data-access-restrictions)
- [Complexity limits](#complexity-limits)
- [Raw SQL queries (legacy)](#raw-sql-queries)

## Rate limits

| Plan | Per Minute | Per Hour | Per Month |
|------|-----------|----------|-----------|
| Free | 100 | 500 | 1,000 |
| Sanbase Pro | 100 | 1,000 | 5,000 |
| Sanbase Max | 100 | 4,000 | 80,000 |
| Business Pro | 600 | 30,000 | 600,000 |
| Business Max | 1,200 | 60,000 | 1,200,000 |
| Enterprise | custom | custom | custom |

- Each GraphQL **query** (not request) = 1 API call. 3 queries in 1 request = 3 calls.
- Resets at start of next minute/hour/month (UTC). Monthly: 00:00:00 UTC on the 1st.
- Headers: `x-ratelimit-remaining-month`, `x-ratelimit-remaining-hour`, `x-ratelimit-remaining-minute`
- HTTP 429 = rate limited. Self-reset once per 90 days at Account page.

## Data access restrictions

- **Free**: Restricted metrics show 1 year historical, excluding last 30 days
- **Paid plans**: Longer historical range, near-real-time data
- **Free metrics**: Full historical + realtime regardless of plan (rate limits still apply)
- Check per-metric: `getAccessRestrictions(product: SANAPI, plan: FREE, filter: METRIC)`

## Complexity limits

API calculates query complexity before execution:

**Formula**: `Complexity = (N * F * Y * W * A) / S`

| Factor | Description |
|--------|-------------|
| N | Number of data points returned |
| F | Fields per data point (usually 2: datetime + value) |
| Y | Time range span in years |
| W | Metric weight (0.3 for fast-storage metrics, 1.0 for others) |
| A | Number of assets |
| S | Plan divisor (higher tiers = larger divisor) |

**If exceeded**: Reduce time range, increase interval, or fetch fewer assets per request.

## Raw SQL queries

> **Legacy**: Santiment no longer offers this to new users but supports existing users. Prefer `getMetric` for standard use cases.

Mutation: `computeRawClickhouseQuery`. The `parameters` arg is a `json` scalar (pass as **stringified JSON string**).

```graphql
mutation($q: String!, $p: json!) {
  computeRawClickhouseQuery(query: $q, parameters: $p) {
    columns
    columnTypes
    rows
    clickhouseQueryId
  }
}
```

**Variables** (`p` must be stringified JSON, not raw object):
```json
{
  "q": "SELECT dt, get_asset_name(asset_id) AS asset, argMax(value, computed_at) AS value FROM daily_metrics_v2 FINAL WHERE asset_id = get_asset_id({{slug}}) AND metric_id = get_metric_id({{metric}}) AND dt >= now() - INTERVAL 7 DAY GROUP BY dt, metric_id, asset_id ORDER BY dt ASC",
  "p": "{\"slug\": \"bitcoin\", \"metric\": \"daily_active_addresses\"}"
}
```

**Python**: `san.execute_sql(query="SELECT ...", parameters={"slug": "bitcoin"})`

### Key SQL tables

| Table | Description | Granularity |
|-------|-------------|-------------|
| `daily_metrics_v2` | Pre-computed metrics | 1 value/day |
| `intraday_metrics` | High-frequency metrics | ~5 min |
| `intraday_nft_metrics` | NFT collection metrics | Intraday |
| `metric_metadata` | Metric names and IDs | Reference |
| `asset_metadata` | Asset slugs and IDs | Reference |

### SQL helper functions
- `get_asset_id('bitcoin')` / `get_asset_name(asset_id)`
- `get_metric_id('daily_active_addresses')` / `get_metric_name(metric_id)`

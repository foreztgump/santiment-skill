---
name: santiment-api
description: Use the Santiment GraphQL API to fetch cryptocurrency on-chain, social, development, and financial metrics. Covers authentication, all query patterns (timeseries, aggregated, histogram, SQL), rate limits, and the sanpy Python library.
---

# Santiment API Skill

Use this skill when you need to fetch cryptocurrency market data — on-chain metrics, social sentiment, development activity, financial data, or custom SQL queries from the Santiment platform.

## Authentication

**API Key**: Read from `SANTIMENT_API_KEY` environment variable (or `SANPY_APIKEY` — sanpy auto-reads this one natively).

- Obtain a key at https://app.santiment.net/account (Account > API Keys > Generate)
- Pass as HTTP header: `Authorization: Apikey <key>`
- Some metrics are free (no key needed), but rate limits still apply

### Authentication in requests

**curl/HTTP**:
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Apikey $SANTIMENT_API_KEY" \
  --data '{"query": "{ currentUser { id email } }"}' \
  https://api.santiment.net/graphql
```

**Python (sanpy)**:
```python
import os
import san
# Option 1: manual set from env var
san.ApiConfig.api_key = os.environ["SANTIMENT_API_KEY"]
# Option 2: sanpy auto-reads SANPY_APIKEY env var — no code needed if that's set
```

## API Endpoint

**GraphQL**: `https://api.santiment.net/graphql`
- Method: POST
- Content-Type: `application/json` (use `{"query": "..."}` body)
- Explorer: https://api.santiment.net/graphiql (basic) | https://api.santiment.net/graphiql_advanced (supports API key headers)

## Core Concepts

### Slugs
Assets are identified by **slugs** (e.g., `"bitcoin"`, `"ethereum"`, `"santiment"`). To find a slug:

```graphql
# WARNING: This returns ALL projects — use pagination for large result sets
{
  allProjects(minVolume: 0, page: 1, pageSize: 50) {
    slug
    name
    ticker
  }
}
# If you already know the slug, use projectBySlug instead:
# { projectBySlug(slug: "bitcoin") { slug name ticker } }
```

### Metrics
Santiment uses a unified `getMetric(metric: "<name>")` query for all metrics. Metric names use snake_case (e.g., `"daily_active_addresses"`, `"price_usd"`, `"dev_activity"`).

### Intervals
Time intervals use format like `"1d"`, `"1h"`, `"30m"`, `"1w"`. Minimum interval varies by metric (check via `metadata { minInterval }`).

### Relative Dates
The API supports relative date strings:
- `"utc_now"` — current UTC time
- `"utc_now-7d"` — 7 days ago
- `"utc_now-30d"`, `"utc_now-1h"`, `"utc_now-3m"` (m = months)

### Aggregation
Available aggregations: `AVG`, `SUM`, `MIN`, `MAX`, `LAST`, `FIRST`, `MEDIAN`, `ANY`.

## Query Patterns

### 1. Timeseries Data (single asset)

Returns `[{datetime, value}, ...]` for one metric and one asset over time.

```graphql
{
  getMetric(metric: "daily_active_addresses") {
    timeseriesData(
      slug: "ethereum"
      from: "2024-01-01T00:00:00Z"
      to: "2024-01-31T00:00:00Z"
      interval: "1d"
    ) {
      datetime
      value
    }
  }
}
```

**JSON variant** (returns compact JSON string instead of objects):
```graphql
{
  getMetric(metric: "daily_active_addresses") {
    timeseriesDataJson(
      slug: "ethereum"
      from: "utc_now-7d"
      to: "utc_now"
      interval: "1d"
    )
  }
}
```

**Python (sanpy)**:
```python
import san
df = san.get("daily_active_addresses",
    slug="ethereum",
    from_date="2024-01-01",
    to_date="2024-01-31",
    interval="1d"
)
```

**Server-side transforms** (apply moving average or consecutive differences):
```graphql
{
  getMetric(metric: "dev_activity") {
    timeseriesDataJson(
      slug: "ethereum"
      from: "utc_now-365d"
      to: "utc_now"
      interval: "7d"
      transform: { type: "moving_average", movingAverageBase: 4 }
    )
  }
}
```
Available transforms: `"moving_average"` (requires `movingAverageBase`), `"consecutive_differences"`.

**Include incomplete data**: By default, the current incomplete interval is excluded. Add `includeIncompleteData: true` to include it:
```graphql
{
  getMetric(metric: "price_usd") {
    timeseriesDataJson(
      slug: "bitcoin"
      from: "utc_now-7d"
      to: "utc_now"
      interval: "1d"
      includeIncompleteData: true
    )
  }
}
```

### 2. Timeseries Data (multiple assets)

Fetch one metric for multiple assets in a single API call (counts as 1 call):

```graphql
{
  getMetric(metric: "price_usd") {
    timeseriesDataPerSlugJson(
      selector: { slugs: ["bitcoin", "ethereum"] }
      from: "utc_now-7d"
      to: "utc_now"
      interval: "1d"
    )
  }
}
```

**Python (sanpy)**:
```python
df = san.get_many("price_usd",
    slugs=["bitcoin", "ethereum"],
    from_date="2024-01-01",
    to_date="2024-01-31",
    interval="1d"
)
```

### 3. Aggregated Data (single value per asset)

Returns a single number summarizing metric over a time range:

```graphql
{
  getMetric(metric: "daily_active_addresses") {
    highest: aggregatedTimeseriesData(
      slug: "ethereum"
      from: "utc_now-30d"
      to: "utc_now"
      aggregation: MAX
    )
    average: aggregatedTimeseriesData(
      slug: "ethereum"
      from: "utc_now-30d"
      to: "utc_now"
      aggregation: AVG
    )
  }
}
```

### 4. Histogram Data

Returns distribution data (e.g., age distribution of coins). Response union types vary by metric — include both inline fragments to handle either format:

```graphql
{
  getMetric(metric: "age_distribution") {
    histogramData(
      slug: "santiment"
      from: "2024-01-01T00:00:00Z"
      to: "2024-03-01T00:00:00Z"
      interval: "1d"
    ) {
      values {
        ... on DatetimeRangeFloatValueList {
          data { range value }
        }
        ... on FloatRangeFloatValueList {
          data { range value }
        }
      }
    }
  }
}
```

### 5. Metadata

Get available metrics, slugs, and restrictions:

```graphql
# All available metrics
{ getAvailableMetrics }

# Metrics available for a specific plan
{ getAvailableMetrics(product: SANAPI, plan: BUSINESS_PRO) }

# Metric metadata
{
  getMetric(metric: "daily_active_addresses") {
    metadata {
      metric
      defaultAggregation
      minInterval
      dataType
    }
  }
}

# Available since (earliest data point)
{
  getMetric(metric: "daily_active_addresses") {
    availableSince(slug: "ethereum")
  }
}

# Access restrictions per plan
{
  getAccessRestrictions(product: SANAPI, plan: FREE, filter: METRIC) {
    name
    isAccessible
    isRestricted
    isDeprecated
    restrictedFrom
    restrictedTo
  }
}

# Available metrics for a specific slug
{
  projectBySlug(slug: "ethereum") {
    availableMetrics
  }
}
```

## Common Queries

### Get current price for an asset
```graphql
{
  getMetric(metric: "price_usd") {
    aggregatedTimeseriesData(
      slug: "bitcoin"
      from: "utc_now-1d"
      to: "utc_now"
      aggregation: LAST
    )
  }
}
```

### Get prices for assets (paginated)
```graphql
{
  allProjects(page: 1, pageSize: 50) {
    slug
    price: aggregatedTimeseriesData(
      metric: "price_usd"
      from: "utc_now-1d"
      to: "utc_now"
      aggregation: LAST
    )
  }
}
```

### Get project details with multiple metrics
```graphql
{
  projectBySlug(slug: "ethereum") {
    slug
    name
    ticker
    currentPrice: aggregatedTimeseriesData(
      metric: "price_usd"
      from: "utc_now-1d"
      to: "utc_now"
      aggregation: LAST
    )
    highestWeeklyPrice: aggregatedTimeseriesData(
      metric: "price_usd"
      aggregation: MAX
      from: "utc_now-7d"
      to: "utc_now"
    )
    devActivity30d: aggregatedTimeseriesData(
      metric: "dev_activity-1d"
      aggregation: SUM
      from: "utc_now-30d"
      to: "utc_now"
    )
  }
}
```

### Filter and order assets
```graphql
{
  allProjects(
    minVolume: 100000
    page: 1
    pageSize: 10
  ) {
    slug
    name
    marketcapUsd
    price: aggregatedTimeseriesData(
      metric: "price_usd"
      from: "utc_now-1d"
      to: "utc_now"
      aggregation: LAST
    )
  }
}
```

### Filter and order assets by metric value

The `function` argument is a JSON scalar (passed as a string). The return type wraps results in `projects`. Enum values inside the JSON string are **lowercase**.

```graphql
{
  allProjectsByFunction(
    function: "{\"name\":\"selector\",\"args\":{\"filters\":[{\"metric\":\"daily_active_addresses\",\"from\":\"utc_now-7d\",\"to\":\"utc_now\",\"aggregation\":\"avg\",\"operator\":\"greater_than\",\"threshold\":1000}],\"orderBy\":{\"metric\":\"daily_active_addresses\",\"from\":\"utc_now-3d\",\"to\":\"utc_now\",\"aggregation\":\"last\",\"direction\":\"desc\"},\"pagination\":{\"page\":1,\"pageSize\":5}}}"
  ) {
    projects {
      slug
      avgDaa7d: aggregatedTimeseriesData(
        metric: "daily_active_addresses"
        from: "utc_now-7d"
        to: "utc_now"
        aggregation: AVG
      )
    }
  }
}
```

### Current trending words
```graphql
{
  getTrendingWords(
    from: "utc_now-3h"
    to: "utc_now"
    size: 20
    interval: "1h"
  ) {
    datetime
    topWords {
      word
      score
    }
  }
}
```

### Supported blockchains
```graphql
{
  getAvailableBlockchains {
    blockchain
    slug
    infrastructure
    hasLabelMetrics
    hasExchangeMetrics
    hasPureOnchainMetrics
  }
}
```

## Santiment Queries (Raw SQL)

> **Legacy product**: Santiment no longer offers this product to new users, but still supports it for existing legacy users. Prefer the GraphQL `getMetric` API for standard use cases. Only use raw SQL if the user's account has Santiment Queries access and the data cannot be obtained via `getMetric`.

Execute custom SQL directly against the Clickhouse database for advanced analysis.

### Via GraphQL

The mutation is `computeRawClickhouseQuery`. The `parameters` argument is a `json` scalar (pass as stringified JSON). Use variables for clean separation:

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

**Variables:**
```json
{
  "q": "SELECT dt, get_asset_name(asset_id) AS asset, argMax(value, computed_at) AS value FROM daily_metrics_v2 FINAL WHERE asset_id = get_asset_id({{slug}}) AND metric_id = get_metric_id({{metric}}) AND dt >= now() - INTERVAL 7 DAY GROUP BY dt, metric_id, asset_id ORDER BY dt ASC",
  "p": "{\"slug\": \"bitcoin\", \"metric\": \"daily_active_addresses\"}"
}
```

**curl example:**
```bash
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Apikey $SANTIMENT_API_KEY" \
  --data '{"query": "mutation($q: String!, $p: json!) { computeRawClickhouseQuery(query: $q, parameters: $p) { columns rows } }", "variables": {"q": "SELECT dt, value FROM daily_metrics_v2 FINAL WHERE asset_id = get_asset_id({{slug}}) AND metric_id = get_metric_id({{metric}}) ORDER BY dt DESC LIMIT 5", "p": "{\"slug\": \"bitcoin\", \"metric\": \"daily_active_addresses\"}"}}' \
  https://api.santiment.net/graphql | jq .
```

### Via Python (sanpy)

```python
import san

df = san.execute_sql(
    query="""
      SELECT
        dt,
        get_asset_name(asset_id) AS asset,
        argMax(value, computed_at) AS value
      FROM daily_metrics_v2
      WHERE
        asset_id = get_asset_id({{slug}})
        AND metric_id = get_metric_id({{metric}})
        AND dt >= now() - INTERVAL {{last_n_days}} DAY
      GROUP BY dt, metric_id, asset_id
      ORDER BY dt ASC
    """,
    parameters={
        "slug": "bitcoin",
        "metric": "daily_active_addresses",
        "last_n_days": 7
    }
)
```

### Key SQL Tables

| Table | Description | Granularity |
|-------|-------------|-------------|
| `daily_metrics_v2` | Pre-computed metrics | 1 value/day |
| `intraday_metrics` | High-frequency metrics | ~5 min |
| `intraday_nft_metrics` | NFT collection metrics | Intraday |
| `metric_metadata` | Metric names and IDs | Reference |
| `asset_metadata` | Asset slugs and IDs | Reference |
| `label_metadata` | Label FQNs | Reference |

### Useful SQL helper functions:
- `get_asset_id('bitcoin')` — slug to asset_id
- `get_asset_name(asset_id)` — asset_id to slug
- `get_metric_id('daily_active_addresses')` — name to metric_id
- `get_metric_name(metric_id)` — metric_id to name

## Available Metric Categories

### Financial
- `price_usd`, `price_btc`, `price_usdt` — Asset prices
- `marketcap_usd` — Market capitalization
- `volume_usd` — Trading volume
- `ohlc` — Open/High/Low/Close
- `price_volatility` — Price volatility
- `rsi_4h`, `rsi_7d` — Relative Strength Index
- `annual_inflation_rate`
- `fully_diluted_valuation_usd`
- `mean_coin_age` — Average age of all coins

### On-Chain Activity
- `daily_active_addresses` — Unique active addresses per day
- `active_addresses_24h` — Rolling 24h active addresses
- `transaction_volume` — Total transaction volume
- `network_growth` — New addresses created
- `velocity` — Token velocity
- `circulation_1d`, `circulation_7d`, etc. — Token circulation
- `mean_dollar_invested_age` — Mean age of invested dollars
- `nvt` — Network Value to Transactions ratio
- `mvrv_usd`, `mvrv_usd_1d` — Market Value to Realized Value

### Exchange Metrics
- `exchange_inflow`, `exchange_outflow` — Tokens moving in/out of exchanges
- `exchange_balance` — Total tokens held on exchanges
- `supply_on_exchanges`, `supply_outside_exchanges`
- `percent_of_total_supply_on_exchanges`

### Social/Sentiment
- `social_volume_total` — Total social mentions
- `social_dominance_total` — Share of social discussion
- `sentiment_positive_total`, `sentiment_negative_total`
- `sentiment_balance_total` — Positive minus negative sentiment
- `community_messages_count_telegram`, etc.

### Development
- `dev_activity` — Development activity (GitHub events)
- `dev_activity_1d` — Daily development activity
- `github_activity` — Raw GitHub activity

### NFT
- `nft_collection_min_price` — Floor price
- `nft_trade_volume_usd` — Trade volume
- `nft_trades_count` — Number of trades

### Supply Distribution
- `holders_distribution_*` — Token distribution by holder size
- `supply_distribution_*` — Supply distribution metrics

### Whale/Top Holder Metrics
- `amount_in_top_holders` — Tokens held by top holders
- `amount_in_exchange_top_holders`

### XRPL-Specific
- XRPL chain has its own metrics set; use `getAvailableBlockchains` to check

## Supported Blockchains

Bitcoin, Ethereum, BNB Chain, XRPL, Cardano, Solana, Polygon, Avalanche, Optimism, Arbitrum, Litecoin, Dogecoin, Bitcoin Cash, ICP.

Not all blockchains support all metric types (exchanges, labels, miners). Query `getAvailableBlockchains` to check capabilities.

## Rate Limits

| Plan | Per Minute | Per Hour | Per Month |
|------|-----------|----------|-----------|
| Free | 100 | 500 | 1,000 |
| Sanbase Pro | 100 | 1,000 | 5,000 |
| Sanbase Max | 100 | 4,000 | 80,000 |
| Business Pro | 600 | 30,000 | 600,000 |
| Business Max | 1,200 | 60,000 | 1,200,000 |
| Enterprise | custom | custom | custom |

- Each GraphQL **query** (not request) counts as 1 API call. A request with 3 queries = 3 calls.
- Rate limits reset at the start of the next minute/hour/month (UTC). Monthly resets at 00:00:00 UTC on the 1st.
- Rate limit headers in response: `x-ratelimit-remaining-month`, `x-ratelimit-remaining-hour`, `x-ratelimit-remaining-minute`
- HTTP 429 = rate limited; reduce request frequency
- Self-reset available once per 90 days at Account page

## Data Access Restrictions

- **Free plan**: Most restricted metrics show 1 year of historical data, excluding the last 30 days
- **Paid plans**: Longer historical range and near-real-time data
- **Free metrics**: Full historical + realtime, regardless of plan (rate limits still apply)
- Use `getAccessRestrictions` query to check per-metric restrictions for your plan

## Complexity Limits

The API calculates query complexity before execution to prevent expensive queries:

**Factors**: number of data points (N), fields per point (F), time range span in years (Y), metric weight (W), assets count (A), subscription plan divisor (S).

**Formula**: `Complexity = (N * F * Y * W * A) / S`

The subscription plan divisor (S) increases with tier, allowing more complex queries on higher plans. Most metrics stored in specialized fast storage have weight W=0.3; others have W=1.

**If complexity exceeded**: Reduce the time range, increase the interval, or fetch fewer assets per request.

## Batching (Python)

Use `AsyncBatch` for concurrent requests (recommended over sequential loops):

```python
import san

batch = san.AsyncBatch()
batch.get("daily_active_addresses", slug="bitcoin", from_date="2024-01-01", to_date="2024-01-31", interval="1d")
batch.get("price_usd", slug="bitcoin", from_date="2024-01-01", to_date="2024-01-31", interval="1d")
batch.get_many("price_usd", slugs=["bitcoin", "ethereum"], from_date="2024-01-01", to_date="2024-01-31", interval="1d")
results = batch.execute()
# results is a list of DataFrames in the same order
```

Default concurrency: 10 workers. Override with `batch.execute(max_workers=5)`.

## Metric Discovery Workflow

When you need to find the right metric for a user's question:

1. **List all available metrics**: `{ getAvailableMetrics }` or `san.available_metrics()`
2. **Check metrics for a specific asset**: `{ projectBySlug(slug: "bitcoin") { availableMetrics } }` or `san.available_metrics_for_slug("bitcoin")`
3. **Get metadata**: `san.metadata("daily_active_addresses")` — returns min interval, default aggregation, data type
4. **Check availability date**: Use `availableSince(slug: "...")` field in `getMetric`
5. **Verify access**: Use `getAccessRestrictions` to confirm the metric is accessible on the user's plan

**Python utility methods**:
```python
# List all metrics
metrics = san.available_metrics()

# List metrics for a specific slug
metrics = san.available_metrics_for_slug("ethereum")

# Get metric metadata
meta = san.metadata("daily_active_addresses")

# Get metric metadata (min interval, default aggregation, data type)
meta = san.metadata("daily_active_addresses")
```

## Error Handling

- GraphQL errors return HTTP 200 with `"errors"` key in JSON response
- Always check for `"errors"` in the response body, not just HTTP status
- HTTP 429: Rate limited
- HTTP 5xx: Server error (report to Santiment Discord)
- Example error response (HTTP 200!):
  ```json
  {"data": {"getMetric": null}, "errors": [{"message": "The metric is not available..."}]}
  ```
- Common error types:
  - Syntax errors: malformed GraphQL
  - Missing parameters: required field not provided
  - Historical/realtime restriction: data outside your plan's allowed range
  - Complexity exceeded: query too expensive, reduce scope
  - Metric unavailable for slug: not all metrics exist for all assets

## Common Pitfalls

- **HTTP 200 errors**: GraphQL errors return HTTP 200 — always check `"errors"` key in response JSON
- **Content-Type**: Use `application/json` with `{"query": "..."}` body format (the API may also accept `application/graphql` with raw query body, but JSON format is more reliable and standard)
- **Metric availability**: Not all metrics exist for all slugs — check `availableSince` or handle null gracefully
- **Interval vs date units**: In interval strings (e.g., `"5m"`, `"30m"`), `m` means minutes. In relative date strings (e.g., `"utc_now-3m"`), `m` means months. Don't confuse the two contexts
- **Time-windowed metric names**: Some metrics use hyphens for time windows (e.g., `dev_activity-1d`) — this is intentional, not a typo
- **Incomplete data**: The latest interval is excluded by default — use `includeIncompleteData: true` if you need it
- **Heavy queries**: `allProjects(minVolume: 0)` without pagination returns everything — always paginate or use `projectBySlug`

## Quick Reference: curl Template

```bash
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Apikey $SANTIMENT_API_KEY" \
  --data '{"query": "{ getMetric(metric: \"price_usd\") { timeseriesData(slug: \"bitcoin\", from: \"utc_now-7d\", to: \"utc_now\", interval: \"1d\") { datetime value } } }"}' \
  https://api.santiment.net/graphql | jq .
```

## Quick Reference: Python Setup

```bash
pip install sanpy
```

```python
import os
import san

san.ApiConfig.api_key = os.environ["SANTIMENT_API_KEY"]

# Get timeseries
df = san.get("price_usd", slug="bitcoin", from_date="2024-01-01", to_date="2024-01-31", interval="1d")

# Get multiple assets
df = san.get_many("price_usd", slugs=["bitcoin", "ethereum"], from_date="2024-01-01", to_date="2024-01-31", interval="1d")

# Run SQL
df = san.execute_sql(query="SELECT * FROM daily_metrics_v2 LIMIT 5")

# Raw GraphQL
result = san.graphql.execute_gql('{ getMetric(metric: "price_usd") { aggregatedTimeseriesData(slug: "bitcoin", from: "utc_now-1d", to: "utc_now", aggregation: LAST) } }')
```

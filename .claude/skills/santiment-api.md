---
name: santiment-api
description: Use the Santiment GraphQL API to fetch cryptocurrency on-chain, social, development, and financial metrics. Covers authentication, all query patterns (timeseries, aggregated, histogram, SQL), rate limits, and the sanpy Python library.
---

# Santiment API Skill

Use this skill when you need to fetch cryptocurrency market data — on-chain metrics, social sentiment, development activity, financial data, or custom SQL queries from the Santiment platform.

## Authentication

**API Key**: Read from `SANTIMENT_API_KEY` environment variable.

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
san.ApiConfig.api_key = os.environ["SANTIMENT_API_KEY"]
```

## API Endpoint

**GraphQL**: `https://api.santiment.net/graphql`
- Method: POST
- Content-Type: `application/json` (use `{"query": "..."}` body)
- Explorer: https://api.santiment.net/graphiql

## Core Concepts

### Slugs
Assets are identified by **slugs** (e.g., `"bitcoin"`, `"ethereum"`, `"santiment"`). To find a slug:

```graphql
{
  allProjects(minVolume: 0) {
    slug
    name
    ticker
  }
}
```

### Metrics
Santiment uses a unified `getMetric(metric: "<name>")` query for all metrics. Metric names use snake_case (e.g., `"daily_active_addresses"`, `"price_usd"`, `"dev_activity"`).

### Intervals
Time intervals use format like `"1d"`, `"1h"`, `"30m"`, `"1w"`. Minimum interval varies by metric.

### Relative Dates
The API supports relative date strings:
- `"utc_now"` — current UTC time
- `"utc_now-7d"` — 7 days ago
- `"utc_now-30d"`, `"utc_now-1h"`, `"utc_now-3m"` (m = months)

### Aggregation
Available aggregations: `AVG`, `SUM`, `MIN`, `MAX`, `LAST`, `FIRST`, `MEDIAN`, `COUNT`, `ANY`.

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

Returns distribution data (e.g., age distribution of coins):

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
    isRestricted
    restrictedFrom
    restrictedTo
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

### Get prices for all assets
```graphql
{
  allProjects {
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

### Filter by metric value with ordering
```graphql
{
  allProjectsByFunction(
    function: {
      name: "selector"
      args: {
        filters: [
          {
            metric: "daily_active_addresses"
            from: "utc_now-7d"
            to: "utc_now"
            aggregation: AVG
            operator: GREATER_THAN
            threshold: 1000
          }
        ]
        orderBy: {
          metric: "daily_active_addresses"
          from: "utc_now-3d"
          to: "utc_now"
          aggregation: LAST
          direction: DESC
        }
        pagination: { page: 1, pageSize: 10 }
      }
    }
  ) {
    slug
    avgDaa7d: aggregatedTimeseriesData(
      metric: "daily_active_addresses"
      from: "utc_now-7d"
      to: "utc_now"
      aggregation: AVG
    )
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

Execute custom SQL directly against the Clickhouse database for advanced analysis.

### Via GraphQL

```graphql
mutation {
  runRawSqlQuery(
    sqlQueryText: """
      SELECT
        dt,
        get_asset_name(asset_id) AS asset,
        get_metric_name(metric_id) AS metric,
        argMax(value, computed_at) AS value
      FROM daily_metrics_v2
      WHERE
        asset_id = get_asset_id({{slug}})
        AND metric_id = get_metric_id({{metric}})
        AND dt >= now() - INTERVAL {{last_n_days}} DAY
      GROUP BY dt, metric_id, asset_id
      ORDER BY dt ASC
    """
    sqlQueryParameters: "{\"slug\": \"bitcoin\", \"metric\": \"daily_active_addresses\", \"last_n_days\": 7}"
  ) {
    columns
    columnTypes
    rows
    clickhouseQueryId
  }
}
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
| Free | 10 | 100 | 1,000 |
| Pro | 30 | 300 | 10,000 |
| Pro+ | 60 | 600 | 30,000 |
| Business Pro | 90 | 900 | 60,000 |

- Each GraphQL **query** (not request) counts as 1 API call. A request with 3 queries = 3 calls.
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

Higher subscription plans have a larger divisor, allowing more complex queries.

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

## Error Handling

- GraphQL errors return HTTP 200 with `"errors"` key in JSON response
- Always check for `"errors"` in the response body, not just HTTP status
- HTTP 429: Rate limited
- HTTP 5xx: Server error (report to Santiment Discord)
- Common error types:
  - Syntax errors: malformed GraphQL
  - Missing parameters: required field not provided
  - Historical/realtime restriction: data outside your plan's allowed range
  - Complexity exceeded: query too expensive, reduce scope

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

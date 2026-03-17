# Common Queries

## Table of Contents
- [Current price](#current-price)
- [All asset prices (paginated)](#all-asset-prices)
- [Project details with multiple metrics](#project-details)
- [Filter and order assets](#filter-and-order)
- [Filter by metric value (allProjectsByFunction)](#filter-by-metric-value)
- [Trending words](#trending-words)
- [Supported blockchains](#supported-blockchains)

## Current price

```graphql
{
  getMetric(metric: "price_usd") {
    aggregatedTimeseriesData(slug: "bitcoin", from: "utc_now-1d", to: "utc_now", aggregation: LAST)
  }
}
```

## All asset prices

Always paginate — unpaginated `allProjects` returns everything.

```graphql
{
  allProjects(page: 1, pageSize: 50) {
    slug
    price: aggregatedTimeseriesData(metric: "price_usd", from: "utc_now-1d", to: "utc_now", aggregation: LAST)
  }
}
```

## Project details

```graphql
{
  projectBySlug(slug: "ethereum") {
    slug
    name
    ticker
    currentPrice: aggregatedTimeseriesData(metric: "price_usd", from: "utc_now-1d", to: "utc_now", aggregation: LAST)
    highestWeeklyPrice: aggregatedTimeseriesData(metric: "price_usd", aggregation: MAX, from: "utc_now-7d", to: "utc_now")
    devActivity30d: aggregatedTimeseriesData(metric: "dev_activity-1d", aggregation: SUM, from: "utc_now-30d", to: "utc_now")
  }
}
```

## Filter and order

```graphql
{
  allProjects(minVolume: 100000, page: 1, pageSize: 10) {
    slug
    name
    marketcapUsd
    price: aggregatedTimeseriesData(metric: "price_usd", from: "utc_now-1d", to: "utc_now", aggregation: LAST)
  }
}
```

## Filter by metric value

The `function` arg is a JSON scalar — must be a **stringified JSON string**, not a raw object. Enum values are **lowercase**. Returns `{ projects { ... } }`.

**JSON structure (readable — stringify before passing):**
```json
{
  "name": "selector",
  "args": {
    "filters": [
      {
        "metric": "daily_active_addresses",
        "from": "utc_now-7d",
        "to": "utc_now",
        "aggregation": "avg",
        "operator": "greater_than",
        "threshold": 1000
      }
    ],
    "orderBy": {
      "metric": "daily_active_addresses",
      "from": "utc_now-3d",
      "to": "utc_now",
      "aggregation": "last",
      "direction": "desc"
    },
    "pagination": { "page": 1, "pageSize": 5 }
  }
}
```

**GraphQL query (stringified):**
```graphql
{
  allProjectsByFunction(
    function: "{\"name\":\"selector\",\"args\":{\"filters\":[{\"metric\":\"daily_active_addresses\",\"from\":\"utc_now-7d\",\"to\":\"utc_now\",\"aggregation\":\"avg\",\"operator\":\"greater_than\",\"threshold\":1000}],\"orderBy\":{\"metric\":\"daily_active_addresses\",\"from\":\"utc_now-3d\",\"to\":\"utc_now\",\"aggregation\":\"last\",\"direction\":\"desc\"},\"pagination\":{\"page\":1,\"pageSize\":5}}}"
  ) {
    projects {
      slug
      avgDaa7d: aggregatedTimeseriesData(metric: "daily_active_addresses", from: "utc_now-7d", to: "utc_now", aggregation: AVG)
    }
  }
}
```

## Trending words

```graphql
{
  getTrendingWords(from: "utc_now-3h", to: "utc_now", size: 20, interval: "1h") {
    datetime
    topWords { word score }
  }
}
```

## Supported blockchains

Bitcoin, Ethereum, BNB Chain, XRPL, Cardano, Solana, Polygon, Avalanche, Optimism, Arbitrum, Litecoin, Dogecoin, Bitcoin Cash, ICP.

Not all blockchains support all metric types. Query to check:

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

# Metrics Catalog

1000+ metrics available. Use `{ getAvailableMetrics }` for the full list. Key metrics by category:

## Financial
- `price_usd`, `price_btc`, `price_usdt` — asset prices
- `marketcap_usd` — market capitalization
- `volume_usd` — trading volume
- `ohlcv` — Open/High/Low/Close/Volume (also: `price_usd_open`, `price_usd_high`, `price_usd_low`, `price_usd_close`)
- `price_volatility` — price volatility
- `rsi_4h`, `rsi_7d` — Relative Strength Index
- `annual_inflation_rate`, `fully_diluted_valuation_usd`
- `mean_coin_age` — average age of all coins

## On-Chain Activity
- `daily_active_addresses` — unique active addresses per day
- `active_addresses_24h` — rolling 24h active addresses
- `transaction_volume` — total transaction volume
- `network_growth` — new addresses created
- `velocity` — token velocity
- `circulation_1d`, `circulation_7d`, etc. — token circulation
- `mean_dollar_invested_age` — mean age of invested dollars
- `nvt` — Network Value to Transactions ratio
- `mvrv_usd`, `mvrv_usd_1d` — Market Value to Realized Value

## Exchange
- `exchange_inflow`, `exchange_outflow` — tokens moving in/out of exchanges
- `exchange_balance` — total tokens held on exchanges
- `supply_on_exchanges`, `supply_outside_exchanges`
- `percent_of_total_supply_on_exchanges`

## Social/Sentiment
- `social_volume_total` — total social mentions
- `social_dominance_total` — share of social discussion
- `sentiment_positive_total`, `sentiment_negative_total`
- `sentiment_balance_total` — positive minus negative sentiment
- `community_messages_count_telegram`, etc.

## Development
- `dev_activity` — development activity (GitHub events)
- `dev_activity_1d` — daily development activity
- `github_activity` — raw GitHub activity

## NFT
- `nft_collection_min_price` — floor price
- `nft_trade_volume_usd` — trade volume
- `nft_trades_count` — number of trades

## Supply Distribution
- `holders_distribution_*` — token distribution by holder size
- `supply_distribution_*` — supply distribution metrics

## Whale/Top Holders
- `amount_in_top_holders` — tokens held by top holders
- `amount_in_exchange_top_holders`

## XRPL-Specific
XRPL has its own metric set. Use `getAvailableBlockchains` to check capabilities per chain.

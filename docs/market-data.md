# Market Data API Integration Guide

## Overview

The Market Data API provides K-line data, transaction statistics, and batch queries. Supports HTTP requests and Kafka subscriptions (real-time streaming).

**API Host**: `https://bopenapi.bgwapi.io`

---

## Supported Chains (32)

| Chain | ID | Chain | ID |
|-------|----|-------|----|
| eth | 1 | arbitrum | 42161 |
| trx | 6 | celo | 42220 |
| optimism | 10 | zkfair | 42766 |
| crol2 | 25 | avax_c | 43114 |
| bnb | 56 | linea | 59144 |
| fuse | 122 | blast | 81457 |
| matic | 137 | berachain | 80094 |
| manta | 169 | sol | 100278 |
| opbnb | 204 | apt | 100279 |
| ftm | 250 | ton | 100280 |
| zksv2 | 324 | suinet | 100281 |
| sonic_evm | 146 | degen | 666666666 |
| morph | 2818 | hyperliquid | 60011 |
| merlin | 4200 | fsc | 201022 |
| base | 8453 | hyper_evm | 999 |
| klay | 8217 | seiv2 | 1329 |
| coredao | 1116 | — | — |

---

## HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 400 | Bad request |
| 403 | Forbidden (not whitelisted or signature error) |
| 429 | Rate limited |

---

## API Reference

### 1. Get K-line Data

`POST /bgw-pro/market/v3/coin/getKline`

**Request Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| chain | string | Chain identifier (e.g., "sol", "eth") |
| contract | string | Contract address (empty string for native tokens) |
| period | string | K-line period: `1s`, `1m`, `5m`, `15m`, `30m`, `1h`, `4h`, `1d`, `1w` |
| size | number | Number of entries to return (max 1440) |

**Request Example**

```json
{
  "chain": "sol",
  "contract": "CFbEmC3JJ5HqXwfFNrKMzhAnFaMp64QkbBegcTw3Hpzh",
  "period": "1m",
  "size": 100
}
```

**Response Fields**

| Field | Description |
|-------|-------------|
| ts | Unix timestamp (seconds) |
| high / low / open / close | High / low / open / close price |
| turnover | Total trading volume (USD) |
| buyTurnover / sellTurnover | Buy / sell volume (USD) |
| amount | Total trading quantity |
| buyAmount / sellAmount | Buy / sell quantity |

---

### 2. Get Token Transaction Info

`POST /bgw-pro/market/v3/coin/getTxInfo`

**Request Example**

```json
{
  "chain": "sol",
  "contract": "7wti9XBn5L3gV815sovZUx7UDFxQydcSgCeBTXfHpump"
}
```

**Response**

Returns trading statistics for 4 time windows: `5m`, `1h`, `4h`, `24h` (only these 4 are supported).

Each window contains:

| Field | Description |
|-------|-------------|
| high / low / open | High / low / open price |
| turnover | Trading volume (USD) |
| buy_turnover / sell_turnover | Buy / sell volume |
| makers | Total trader count |
| buyers / sellers | Buy / sell trader count |
| txns | Total transaction count |
| buys / sells | Buy / sell transaction count |

---

### 3. Batch Get Transaction Info

`POST /bgw-pro/market/v3/coin/batchGetTxInfo`

**Request Example**

```json
{
  "list": [
    { "chain": "sol", "contract": "7wti9XBn5L3gV815sovZUx7UDFxQydcSgCeBTXfHpump" },
    { "chain": "eth", "contract": "" }
  ]
}
```

Same response structure as getTxInfo, wrapped in `data.list` array.

---

## FAQ

### Q: K-line returns empty data?

Possible causes:
1. Invalid `period` — must be exactly one of: `1s`, `1m`, `5m`, `15m`, `30m`, `1h`, `4h`, `1d`, `1w`
2. Wrong `contract` case — Solana contract addresses are case-sensitive
3. Token just launched and has insufficient trading data

### Q: getTxInfo only supports 4 time windows?

Yes, only `5m`, `1h`, `4h`, `24h`. Custom time windows are not supported.

### Q: Getting 403 Forbidden?

Check:
1. Whether your IP is whitelisted
2. Whether the request signature is correct (see authentication docs)

### Q: Getting 429 Too Many Requests?

Rate limit triggered. Suggestions:
1. Reduce request frequency
2. Use batch endpoints instead of multiple single calls (e.g., `batchGetTxInfo`)
3. For real-time data needs, consider Kafka subscription

### Q: What do buyTurnover and sellTurnover mean?

`buyTurnover` is buy-side trading volume in USD, `sellTurnover` is sell-side. Comparing the two reveals buying vs selling pressure — buy volume significantly exceeding sell volume indicates strong buy pressure.

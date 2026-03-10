# Token API Integration Guide

## Overview

The Token API provides token information queries, rankings, liquidity analysis, and security audits.

**API Host**: `https://bopenapi.bgwapi.io`

---

## API Reference

### 1. Get Token Info

`POST /bgw-pro/market/v3/coin/getBaseInfo`

**Request Example**

```json
{
  "chain": "sol",
  "contract": "7wti9XBn5L3gV815sovZUx7UDFxQydcSgCeBTXfHpump"
}
```

**Response Fields**

| Field | Type | Description |
|-------|------|-------------|
| symbol | string | Token symbol |
| name | string | Token name |
| decimals | number | Decimal places |
| price | number | Current price (USD) |
| total_supply | number | Total supply |
| circulating_supply | number | Circulating supply |
| icon | string | Token icon URL |
| twitter | string | Twitter link |
| website | string | Website link |
| telegram | string | Telegram link |
| whitepaper | string | Whitepaper link |
| about | string | Token description |
| holders | number | Number of holders |
| liquidity | number | Liquidity value |
| top10_holder_percent | number | Top 10 holder percentage |
| insider_holder_percent | number | Insider (rat trading) percentage |
| sniper_holder_percent | number | Sniper percentage |
| dev_holder_percent | number | Developer holding percentage |

---

### 2. Batch Get Token Info

`POST /bgw-pro/market/v3/coin/batchGetBaseInfo`

**Request Example**

```json
{
  "list": [
    { "chain": "sol", "contract": "7wti9XBn5L3gV815sovZUx7UDFxQydcSgCeBTXfHpump" },
    { "chain": "eth", "contract": "" }
  ]
}
```

> Maximum **100 tokens** per request.

Same response structure as getBaseInfo, wrapped in `data.list` array.

---

### 3. Get Rankings

`POST /bgw-pro/market/v3/topRank/detail`

**Request Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| name | string | Ranking name (see table below) |

**Ranking Types**

| name | Description |
|------|-------------|
| topGainers | Top gainers by 24h change |
| topLosers | Top losers by 24h change |
| hotpicks | Curated trending tokens (case-insensitive, `Hotpicks` also works) |

**Request Example**

```json
{ "name": "topLosers" }
```

**Response Fields**

| Field | Description |
|-------|-------------|
| symbol / name | Token symbol / name |
| chain / contract | Chain / contract address |
| risk_level | Risk level: `high` / `medium` / `low` |
| icon | Token icon URL |
| price | Current price |
| change_24h | 24h price change (decimal, 0.1 = 10%) |
| volume_24h | 24h trading volume |
| turnover_24h | 24h trading turnover (USD) |
| market_cap | Market capitalization |
| issue_date | Issue date (Unix timestamp) |

---

### 4. Get Token Liquidity

`POST /bgw-pro/market/v3/poolList`

**Request Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| chain | string | Chain identifier |
| contract | string | Contract address (case-sensitive) |

**Response Fields**

| Field | Description |
|-------|-------------|
| totalLpValue | Total liquidity value (USD) |
| lockedLpValue | Locked liquidity value |
| lockedLpPercent | Lock percentage |
| pools[] | Pool list |
| pools[].poolAddr | Pool address |
| pools[].protocol | Protocol name |
| pools[].totalUsd | Pool value |
| pools[].token0Symbol / token1Symbol | Token pair symbols |
| pools[].reserve0 / reserve1 | Token reserves |
| pools[].priceRate | Exchange rate |
| activities[] | Recent add/remove liquidity events |

---

### 5. Paginated Token List

`POST /bgw-pro/market/v3/historical-coins`

Retrieve newly listed tokens by timestamp.

**Request Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| createTime | string | Start time, format `"YYYY-MM-DD HH:MM:SS"` |
| limit | number | Number of results |

> ⚠️ `createTime` is a **date string**, NOT a Unix timestamp.

**Response Fields**

| Field | Description |
|-------|-------------|
| tokenList[] | Token list |
| tokenList[].chain / contract / symbol / name | Basic info |
| tokenList[].decimals | Decimal places |
| tokenList[].createTime | Creation time (date string) |
| lastTime | Last record time — use as `createTime` for next page |

**Pagination**: Pass the previous response's `lastTime` as the next request's `createTime`.

---

### 6. Batch Security Audit

`POST /bgw-pro/market/v3/coin/security/audits`

**Request Example**

```json
{
  "list": [
    { "chain": "sol", "contract": "ozt8wJfkPysr6nT66TfQV9KNxDxzgXRpn8BL7tdqDoP" }
  ],
  "source": "bg"
}
```

**Response Fields**

| Field | Type | Description |
|-------|------|-------------|
| chain / contract | string | Chain / contract address |
| riskChecks | array | High-risk items |
| warnChecks | array | Warning items |
| lowChecks | array | Low-risk (safe) items |
| riskCount / warnCount | number | Risk / warning item counts |
| highRisk | bool | Whether token is high-risk |
| buyTax / sellTax | number | Buy / sell tax percentage |
| freezeAuth | bool | Has freeze authority (Solana) |
| mintAuth | bool | Has mint authority (Solana) |
| token2022 | bool | Uses Token-2022 standard (Solana) |
| lpLock | bool | LP is locked |
| top_10_holder_risk_level | number | Top 10 holder risk level |
| checking | bool | Still analyzing (retry later if true) |
| support | number | Whether audit is supported for this token |

---

## Security Audit Label Reference (labelName)

Each check item contains a `labelName` field mapping to a specific audit result.

### Solana Token Checks

| Check | Risk (High) | Warn (Medium) | Low (Safe) |
|-------|-------------|---------------|------------|
| Mint authority | — | SolanaWarnTitle6 (not discarded) | SolanaLowTitle6 (discarded ✅) |
| Freeze authority | SolanaRiskTitle1 (not discarded) | — | SolanaLowTitle1 (discarded ✅) |
| LP burn ratio | — | SolanaWarnTitle2 (below threshold) | SolanaLowTitle2 (meets threshold ✅) |
| Top 10 holder % | SolanaRiskTitle3 (>60%) | SolanaWarnTitle3 (elevated) | SolanaLowTitle3 (normal ✅) |
| Trade tax | SolanaRiskTitle10 (≥50%) | SolanaWarnTitle10 (≥10%) | SolanaLowTitle10 (<10% ✅) |
| Tax modifiable | SolanaRiskTitle11 (yes) | — | SolanaLowTitle11 (no ✅) |
| Sniper holder % | SolanaRiskTitle12 (high) | SolanaWarnTitle12 (elevated) | SolanaLowTitle12 (normal ✅) |
| Insider holder % | SolanaRiskTitle13 (high) | SolanaWarnTitle13 (elevated) | SolanaLowTitle13 (normal ✅) |
| Dev rug rate | RiskTitle27 (≥50%) | WarnTitle27 (≥25%) | LowTitle27 ✅ |
| Dev holder % | RiskTitle28 (≥30%) | WarnTitle28 (≥10%) | LowTitle28 ✅ |
| Suspected honeypot | RiskTitle29 | WarnTitle29 | LowTitle29 ✅ |

### EVM Token Checks

| Check | Risk (High) | Warn (Medium) | Low (Safe) |
|-------|-------------|---------------|------------|
| Honeypot | RiskTitle2 (yes) | — | LowTitle2 (no ✅) |
| Trade tax | RiskTitle1 (≥50%) | WarnTitle1 (≥10%) | LowTitle1 (<10% ✅) |
| Tax modifiable | RiskTitle8 (yes) | WarnTitle8 | LowTitle8 (no ✅) |
| Timelock | — | WarnTitle9 (has timelock) | LowTitle9 (none ✅) |
| Trading pause | RiskTitle4 (pausable) | WarnTitle4 | LowTitle4 (not pausable ✅) |
| Blacklist | RiskTitle15 (has blacklist) | WranTitle15 | LowTitle15 (none ✅) |
| Contract upgrade | — | WarnTitle6 (upgradeable) | LowTitle6 (not upgradeable ✅) |
| Mint mechanism | — | WarnTitle10 (mintable) | LowTitle10 (not mintable ✅) |
| Balance modifiable | — | — | LowTitle7 (not modifiable ✅) |
| Top 10 holder % | RiskTitle23 (high) | WarnTitle23 (elevated) | LowTitle23 (normal ✅) |
| LP lock ratio | — | WarnTitle24 (below threshold) | LowTitle24 (meets threshold ✅) |
| Sniper holder % | RiskTitle25 (high) | WarnTitle25 (elevated) | LowTitle25 (normal ✅) |
| Insider holder % | RiskTitle26 (high) | WarnTitle26 (elevated) | LowTitle26 (normal ✅) |
| Dev rug rate | RiskTitle27 (≥50%) | WarnTitle27 (≥25%) | LowTitle27 ✅ |
| Dev holder % | RiskTitle28 (≥30%) | WarnTitle28 (≥10%) | LowTitle28 ✅ |
| Suspected honeypot | RiskTitle29 | WarnTitle29 | LowTitle29 ✅ |
| Bundle tx detection | RiskTitle30 (>20%) | WarnTitle30 (>10%) | LowTitle30 (<10% ✅) |

### Values Field Reference

Some checks include a `values` object with specific metrics:

| labelName keyword | values fields |
|-------------------|---------------|
| top10 related | `top10Share` |
| sniper related | `sniperShare` |
| insider related | `insiderShare` |
| LP burn related | `lockedValue`, `unlockedShare`, `lockedShare` |
| trade tax related | `buyTax`, `sellTax` |
| dev rug related | `devIssueCoinCount`, `devRugCoinCount`, `devRugCoinPercent` |
| dev holder related | `devHolderBalance`, `devHolderPercent`, `thresholdValuePercent` |
| bundle tx related | `bundleTxUserHolderPercent`, `thresholdValuePercent` |

---

## FAQ

### Q: getBaseInfo returns 400 or empty data?

Possible causes:
1. Wrong `contract` address or case (Solana addresses are case-sensitive)
2. Token just launched and hasn't been indexed yet
3. For native tokens, use empty string `""`

**Tip**: If single queries fail, try `batchGetBaseInfo` — it's historically more stable.

### Q: Security audit returns checking: true?

Token is still being analyzed. Wait a few seconds and retry.

### Q: What's the relationship between highRisk and riskChecks?

`highRisk: true` is a summary flag indicating at least one critical risk. Specific risks are listed in the `riskChecks` array — each item has a `labelName` field. See the label reference tables above for meanings.

### Q: How to paginate historical tokens?

```
1. First request: createTime = "2026-03-10 00:00:00", limit = 100
2. Response contains lastTime = "2026-03-10 00:05:23"
3. Next request: createTime = "2026-03-10 00:05:23", limit = 100
4. Repeat until empty list returned
```

> ⚠️ Both `createTime` and `lastTime` are date strings, NOT Unix timestamps.

### Q: Is the rankings name parameter case-sensitive?

Both `hotpicks` and `Hotpicks` work in practice. But recommended to follow official docs: `topGainers`, `topLosers`, `hotpicks`.

### Q: Which chains does security audit support?

Security audit supports all 32 market data chains. However, some tokens on newer chains may return `support: 0`, meaning audit is not yet available for that token.

### Q: How to assess if a token is safe?

Use multiple signals — no single indicator is definitive:

| Red Flag | Source | Risk Level |
|----------|--------|------------|
| `highRisk: true` | security | 🔴 Critical — do not trade |
| `buyTax` or `sellTax` > 5% | security | 🔴 High — likely scam |
| `freezeAuth: true` (Solana) | security | 🟡 Medium — can be frozen |
| `holders` < 100 | token-info | 🟡 Medium — very early or abandoned |
| Top 10 holders > 50% | token-info | 🟡 Medium — rug pull risk |
| `dev_rug_percent` > 25% | token-info | 🔴 High — developer has rug history |
| LP lock = 0% | liquidity | 🟡 Medium — creator can pull liquidity |
| Pool liquidity < $10K | liquidity | 🟡 Medium — large trades cause extreme slippage |

When multiple red flags appear together, strongly advise against trading.

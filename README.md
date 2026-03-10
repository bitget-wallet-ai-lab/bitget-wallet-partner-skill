# Bitget Wallet Partner Skill

API integration guide and troubleshooting assistant for the Bitget Wallet Open Platform.

## What It Does

When partners encounter issues integrating Bitget Wallet APIs, this skill provides:

- ✅ **Quick diagnosis** — Identify root cause from error codes, symptoms, and parameters
- ✅ **Actionable solutions** — Code examples and parameter adjustment guidance
- ✅ **Proactive prevention** — Common pitfalls and best practices

## API Coverage

| Module | Doc | Core Capabilities |
|--------|-----|-------------------|
| [Swap Order](docs/swap-order.md) | swap-order | Same-chain/cross-chain swaps, Gasless (EIP-7702), signing, order tracking |
| [Market Data](docs/market-data.md) | market-data | K-line (9 periods), transaction stats (buy/sell breakdown), batch queries |
| [Token](docs/token.md) | token | Token info, rankings, liquidity, security audit (full labelName mapping) |

## Quick Links

- [Official API Docs](https://web3.bitget.com/zh-CN/docs)
- [Swap Order API](https://web3.bitget.com/zh-CN/docs/swap-order)
- [Market Data API](https://web3.bitget.com/zh-CN/docs/market/market-price)
- [Token API](https://web3.bitget.com/zh-CN/docs/market/token)
- [Authentication](https://web3.bitget.com/zh-CN/docs/authentication/)

## Supported Chains

**Swap Order** (7 chains with Gasless support):

| Chain | Code | noGas Minimum |
|-------|------|--------------|
| Ethereum | eth | $5 |
| Solana | sol | $5 |
| BNB Chain | bnb | $5 |
| Base | base | $5 |
| Arbitrum | arbitrum | $5 |
| Polygon | matic | $5 |
| Morph | morph | $1 |

**Market Data**: 32 chains (see [market-data.md](docs/market-data.md))

## Error Code Reference

| Code | Description | Action |
|------|-------------|--------|
| 80000 | Internal system error | Retry |
| 80001 | Insufficient balance | Reduce amount |
| 80002 | Amount too low | Increase amount |
| 80003 | Amount too high | Reduce amount |
| 80004 | Order expired | Re-create order |
| 80005 | Insufficient liquidity | Try different route |
| 80006 | Invalid parameters | Check required fields |
| 80007 | Invalid Partner | Check Partner-Code |
| 80014 | Order not found | Verify orderId |
| 80015 | Duplicate submission | Check order status |

Full error code table: [swap-order.md](docs/swap-order.md#error-codes)

## License

MIT

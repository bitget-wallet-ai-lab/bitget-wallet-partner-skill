# Swap Order API Integration Guide

## Overview

The Order-based Model handles same-chain swaps and cross-chain asset transfers in 5 steps:

```
Get Quote → Create Order → Sign → Submit Order → Query Status
```

- `fromChain == toChain` → Same-chain swap
- `fromChain != toChain` → Cross-chain swap

**API Host**: `https://bopenapi.bgwapi.io`

---

## API Reference

### 1. Get Swap Price

`POST /bgw-pro/swapx/order/getSwapPrice`

**Request Headers**

| Header | Description |
|--------|-------------|
| Content-Type | application/json |
| Partner-Code | Your partner code (required) |

**Request Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| fromChain | string | ✅ | Source chain identifier (e.g., "base", "bnb", "eth") |
| fromContract | string | ✅ | Source token contract address. Empty string `""` for native tokens |
| fromAmount | string | ✅ | Input amount in **human-readable format** (e.g., "2.0" = 2 tokens) |
| toChain | string | ✅ | Destination chain identifier |
| toContract | string | ✅ | Destination token contract address. Empty string `""` for native tokens |
| fromAddress | string | ✅ | User's wallet address |
| toAddress | string | ❌ | Defaults to fromAddress if not set |
| feeRate | string | ❌ | Fee percentage (e.g., "0.05" = 5%) |

> ⚠️ **Amounts are human-readable**: Pass `"2.0"` for 2 tokens, NOT `"2000000"` (raw value). All `toAmount` / `fromAmount` values in responses are also human-readable.

**Request Example**

```bash
curl -X POST '{API_HOST}/bgw-pro/swapx/order/getSwapPrice' \
  -H 'Content-Type: application/json' \
  -H 'Partner-Code: your_partner_code' \
  -d '{
    "fromChain": "base",
    "fromContract": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
    "fromAmount": "2.0",
    "toChain": "bnb",
    "toContract": "0x55d398326f99059ff775485246999027b3197955",
    "fromAddress": "0x2E1276244540B7996fbF4F8DF90229BAD36fB4F5",
    "feeRate": "0.05"
  }'
```

**Response Fields**

| Field | Type | Description |
|-------|------|-------------|
| toAmount | string | Estimated output amount (human-readable) |
| market | string | Recommended market/bridge (must pass to order-create exactly) |
| slippage | string | Recommended slippage (decimal, "0.03" = 3%) |
| priceImpact | string | Price impact (decimal) |
| fee | object | Fee breakdown |
| fee.totalAmountInUsd | string | Total fee (USD) |
| fee.appFee | object | Partner fee portion |
| fee.platformFee | object | Platform fee portion |
| fee.gasFee | object | Gas fee portion |
| features | string[] | Supported features. `["no_gas"]` = gasless available |
| eip7702Bindend | bool | Whether address has EIP-7702 binding |
| eip7702Contract | string | Bound contract address |
| eip7702IsBgw | bool | Whether bound to BGW's EIP-7702 contract |

**Response Example**

```json
{
  "status": 0,
  "data": {
    "toAmount": "1.885815",
    "market": "bkbridgev3.liqbridge",
    "slippage": "0",
    "priceImpact": "0.0571",
    "fee": {
      "totalAmountInUsd": "0.114185",
      "appFee": { "amountInUsd": "0.1" },
      "platformFee": { "amountInUsd": "0.002" }
    },
    "features": []
  }
}
```

---

### 2. Create Swap Order

`POST /bgw-pro/swapx/order/makeSwapOrder`

**Request Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| fromChain | string | ✅ | Source chain identifier |
| fromContract | string | ✅ | Source token contract address |
| fromAmount | string | ✅ | Input amount (human-readable) |
| toChain | string | ✅ | Destination chain identifier |
| toContract | string | ✅ | Destination token contract address |
| fromAddress | string | ✅ | Sender address |
| toAddress | string | ✅ | Receiver address |
| market | string | ✅ | Market from getSwapPrice response (pass exactly) |
| slippage | string | ❌ | Slippage tolerance (e.g., "0.03" = 3%) |
| feeRate | string | ❌ | Partner fee percentage |
| feature | string | ❌ | Pass `"no_gas"` for gasless mode (requires getSwapPrice support) |

> ⚠️ **Check native token balance before calling.** If insufficient, use `"no_gas"` feature.

**Request Examples**

```bash
# Normal transaction
curl -X POST '{API_HOST}/bgw-pro/swapx/order/makeSwapOrder' \
  -H 'Content-Type: application/json' \
  -H 'Partner-Code: your_partner_code' \
  -d '{
    "fromChain": "base",
    "fromContract": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
    "fromAmount": "2",
    "toChain": "bnb",
    "toContract": "0x55d398326f99059ff775485246999027b3197955",
    "fromAddress": "0xYourAddress",
    "toAddress": "0xYourAddress",
    "market": "bkbridgev3.liqbridge",
    "slippage": "0.03"
  }'

# Gasless (EIP-7702) transaction
curl -X POST '{API_HOST}/bgw-pro/swapx/order/makeSwapOrder' \
  -H 'Content-Type: application/json' \
  -H 'Partner-Code: your_partner_code' \
  -d '{
    "fromChain": "base",
    "fromContract": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
    "fromAmount": "2",
    "toChain": "bnb",
    "toContract": "0x55d398326f99059ff775485246999027b3197955",
    "fromAddress": "0xYourAddress",
    "toAddress": "0xYourAddress",
    "market": "bkbridgev3.liqbridge",
    "slippage": "0.03",
    "feature": "no_gas"
  }'
```

**Response Modes**

The response contains either `txs` (normal) or `signatures` (gasless) — **never both**:

- **`txs`** — Normal transaction mode. Sign raw transactions with wallet
- **`signatures`** — Gasless (EIP-7702) mode. Sign EIP-712 typed data

**Normal Transaction Response (`txs`)**

```json
{
  "data": {
    "orderId": "34b34a3391da45928f6c9673fba1a4e8",
    "txs": [
      {
        "kind": "transaction",
        "chainName": "base",
        "chainId": "8453",
        "data": {
          "to": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
          "calldata": "0xa9059cbb...",
          "gasLimit": "54526",
          "gasPrice": "9000000",
          "nonce": 308,
          "value": "0",
          "baseFee": "5820569",
          "maxFeePerGas": "11139700",
          "maxPriorityFeePerGas": "2408846",
          "supportEIP1559": true
        }
      }
    ]
  }
}
```

**Gasless Transaction Response (`signatures`)**

```json
{
  "data": {
    "orderId": "ccb8d3f244d64e928ea32f1f8127a7b7",
    "signatures": [
      {
        "kind": "signature",
        "chainName": "bnb",
        "chainId": "56",
        "hash": "0xdbcc895aa03a4c3a...",
        "data": {
          "signType": "eip712",
          "types": { "..." },
          "primaryType": "Aggregator",
          "domain": { "..." },
          "message": { "..." }
        }
      }
    ]
  }
}
```

**Common Structure for txs / signatures**

| Field | Type | Description |
|-------|------|-------------|
| kind | string | `"transaction"` or `"signature"` |
| chainName | string | Chain name |
| chainId | string | Chain ID |
| data | object | Payload — varies by kind and signType |

---

### 3. Submit Swap Order

`POST /bgw-pro/swapx/order/submitSwapOrder`

**Request Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| orderId | string | ✅ | Order ID from makeSwapOrder |
| signedTxs | string[] | ✅ | Array of signed hex strings (0x-prefixed) |

> ⚠️ **signedTxs order must match the txs/signatures array order**

**Request Examples**

```bash
# Single signature
curl -X POST '{API_HOST}/bgw-pro/swapx/order/submitSwapOrder' \
  -H 'Content-Type: application/json' \
  -H 'Partner-Code: your_partner_code' \
  -d '{
    "orderId": "8d07e1afe0b44485a225f9c0c1afb7a4",
    "signedTxs": ["0x02f8b2..."]
  }'

# Approve + swap (two signatures)
curl -X POST '{API_HOST}/bgw-pro/swapx/order/submitSwapOrder' \
  -H 'Content-Type: application/json' \
  -H 'Partner-Code: your_partner_code' \
  -d '{
    "orderId": "8d07e1afe0b44485a225f9c0c1afb7a4",
    "signedTxs": ["0xd4dcc616...", "0xac47da5a..."]
  }'
```

---

### 4. Query Swap Order

`POST /bgw-pro/swapx/order/getSwapOrder`

**Request Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| orderId | string | ✅ | Order ID to query |

**Response Fields**

| Field | Type | Description |
|-------|------|-------------|
| orderId | string | Order ID |
| status | string | Order status (see table below) |
| fromChain / toChain | string | Source / destination chain |
| fromContract / toContract | string | Token contract addresses |
| fromAmount | string | Sent amount |
| toAmount | string | Expected output amount |
| receiveAmount | string | Actual received amount (after success) |
| txs | array | Transaction list |
| message | string | Error message (on failure) |
| createTime | int64 | Creation time (Unix timestamp, seconds) |
| updateTime | int64 | Last update time |

**Transaction Object (txs[])**

| Field | Type | Description |
|-------|------|-------------|
| chain | string | Chain of this transaction |
| txId | string | Transaction hash |
| stage | string | `source` = origin chain, `target` = destination chain, `swap` = same-chain, `refund` = refund tx |
| tokens | array | Tokens involved (mainly for refund display) |

**Order Status**

| Status | Description |
|--------|-------------|
| init | Created but not yet submitted via submitSwapOrder |
| processing | In progress |
| success | Completed successfully |
| failed | Failed |
| refunding | Refund in progress |
| refunded | Refund completed |

---

## Signing Guide

### Normal Transactions (`txs` mode)

Sign each transaction in the `txs[]` array using standard wallet signing:

```javascript
// Pseudocode
for (const tx of response.data.txs) {
  const txDict = {
    to: tx.data.to,
    data: tx.data.calldata,
    gas: parseInt(tx.data.gasLimit),
    nonce: tx.data.nonce,
    chainId: parseInt(tx.chainId),
    value: tx.data.value
  };
  
  if (tx.data.supportEIP1559) {
    txDict.maxFeePerGas = parseInt(tx.data.maxFeePerGas);
    txDict.maxPriorityFeePerGas = parseInt(tx.data.maxPriorityFeePerGas);
    txDict.type = 2;
  } else {
    txDict.gasPrice = parseInt(tx.data.gasPrice);
  }
  
  const signedTx = wallet.signTransaction(txDict);
  signedTxs.push(signedTx.rawTransaction);
}
```

### Gasless Transactions (`signatures` mode)

Sign EIP-712 typed data. **Critical: Use the API-provided `hash` field directly** — do NOT recompute the hash yourself.

```javascript
// Pseudocode
for (const sig of response.data.signatures) {
  if (sig.data.signType === 'eip712') {
    // Sign the API-provided hash directly
    const hashBytes = Buffer.from(sig.hash.slice(2), 'hex');
    const signature = wallet.signHash(hashBytes);
    signedTxs.push(signature);
  } else if (sig.data.signType === 'eip7702_auth') {
    // EIP-7702 authorization (appears on first gasless tx)
    const hashBytes = Buffer.from(sig.hash.slice(2), 'hex');
    const signature = wallet.signHash(hashBytes);
    signedTxs.push(signature);
  }
}
```

> ⚠️ **Why not recompute the hash?** Common `signTypedData` / `encode_typed_data` implementations encode nested `Call[]` with `bytes callData` differently from the contract. This causes signature verification failures. The API pre-computes the correct hash — just sign it directly.

### Signature Format

- 65 bytes: `r (32B) + s (32B) + v (1B)`
- v must be **27 or 28** (not y_parity 0/1)
- Hex string with `0x` prefix

---

## Gasless Transactions (EIP-7702)

### How It Works

1. User signs EIP-712 authorization (not a transaction)
2. Backend relayer receives signature → builds full EIP-7702 type-4 tx → pays gas → broadcasts
3. Gas cost is deducted from the input token — user needs no native token balance

### Supported Chains and Thresholds

| Chain | Code | noGas Minimum (USD) |
|-------|------|---------------------|
| Ethereum | eth | $5 |
| Solana | sol | $5 |
| BNB Chain | bnb | $5 |
| Base | base | $5 |
| Arbitrum | arbitrum | $5 |
| Polygon | matic | $5 |
| Morph | morph | $1 |

When order amount (USD) ≥ threshold, `getSwapPrice` returns `"no_gas"` in `features`.

### EIP-7702 Binding State

| State | eip7702Bindend | Signature Count | Description |
|-------|----------------|-----------------|-------------|
| First gasless tx | false | 2 | First = eip712 business sig, second = eip7702_auth binding |
| Already bound | true | 1 | Only eip712 business signature needed |

### Integration Flow

```
1. Call getSwapPrice → check if features contains "no_gas"
2. If yes → pass feature: "no_gas" in makeSwapOrder
3. If no → check if amount is below threshold
   - Yes → suggest increasing amount or acquiring native tokens
   - No → proceed without feature flag (normal gas mode)
```

---

## Cross-Chain Limits

| Source Chain | liqBridge Limits | CCTP Limits |
|-------------|-----------------|-------------|
| Ethereum | $1 – $200,000 | $0.1 – $500,000 |
| Solana | $10 – $200,000 | ❌ |
| BNB Chain | $1 – $200,000 | ❌ |
| Base | $1 – $200,000 | $0.1 – $500,000 |
| Arbitrum | $1 – $200,000 | $0.1 – $500,000 |
| Polygon | $1 – $50,000 | $0.1 – $500,000 |
| Morph | $5 – $50,000 | $0.1 – $500,000 |

---

## Fee Structure

### Fee Types

**1. Platform Fee**
- Percentage-based on transaction amount
- Deducted during bridge/swap execution

**2. App Fee (Partner Fee)**
- Set via `feeRate` parameter (e.g., 0.05 = 5%)
- Retained by platform, settled periodically

**Example**

```json
{
  "fee": {
    "totalAmountInUsd": "0.15",
    "appFee": { "amountInUsd": "0.10" },
    "platformFee": { "amountInUsd": "0.03" }
  }
}
```

---

## Refund Mechanism

When cross-chain transactions fail, a refund is triggered:

- **Refund chain may differ** — refund may arrive on source or destination chain
- **Refund token may differ** — e.g., DAI→USDT failure might refund USDC
- **Refund amount will be less** — gas and bridge fees already incurred are deducted (~1-2% loss)
- Look for `stage: "refund"` entries in `txs[]` for refund transaction details

---

## Error Codes

| Code | Description | Recommended Action |
|------|-------------|-------------------|
| 80000 | Internal system error | Retry or contact support |
| 80001 | Insufficient token balance | Check balance, suggest smaller amount |
| 80002 | Amount below minimum | Increase amount (see cross-chain limits) |
| 80003 | Amount above maximum | Decrease amount |
| 80004 | Order expired | Re-create order (deadline ~2 minutes) |
| 80005 | Insufficient liquidity | Try different route or smaller amount |
| 80006 | Invalid request parameters | Check format, required fields, value ranges |
| 80007 | Invalid or unknown Partner | Check Partner-Code header |
| 80008 | Reverse quote calculation failed | PayFi Swap minAmountOut mode only |
| 80009 | Token info lookup failed | Token may not exist or service issue |
| 80010 | Price or gas price fetch failed | Retry later |
| 80011 | Calldata generation failed | PayFi Swap only |
| 80012 | Quote failed (no available routes) | Try different token pair or amount |
| 80013 | Unsupported chain | Check supported chains table |
| 80014 | Order not found | Verify orderId |
| 80015 | Order already submitted | Do not re-submit; check order status |

---

## FAQ

### Q: Should I pass raw amounts or human-readable amounts?

**Human-readable.** `fromAmount: "2.0"` means 2 tokens. Do NOT pass `"2000000"` (raw value for 6-decimal tokens) — it would be interpreted as 2 million tokens.

### Q: Why does getSwapPrice return empty features?

Possible causes:
1. Amount below noGas threshold ($5 USD, Morph $1) → increase amount
2. Chain doesn't support gasless → check supported chains table
3. Specific route doesn't support it → try a different token pair

### Q: What format should signedTxs be in for gasless mode?

Array of hex strings:
```json
{ "signedTxs": ["0xabc123...", "0xdef456..."] }
```
Do **NOT** pass double-serialized JSON (e.g., `["[\"0xabc123...\"]"]`).

### Q: Why does signature verification fail?

Most common cause: **Recomputing the EIP-712 hash instead of using the API-provided hash.** Solution: Sign `signatures[].hash` directly.

### Q: What should I put in toAddress for cross-chain swaps?

- EVM → EVM: Same EVM address works
- EVM → Solana: Must be a **Solana address** (Base58 format)
- EVM → Tron: Must be a **Tron address** (T-prefix Base58Check)
- Wrong toAddress causes error 80000 or stuck funds

### Q: How do I handle approve + swap (two transactions)?

If `txs` returns 2 transactions, the first is typically `approve` (token authorization) and the second is the actual `swap`. Sign both and put them in `signedTxs` array in order.

### Q: What happens when an order expires?

Order deadline is ~2 minutes. Sign and submit promptly. If expired, run the full flow again: getSwapPrice → makeSwapOrder.

### Q: Why was I charged twice for the same trade?

Likely duplicate order submission. Always check the previous order's status (getSwapOrder) before creating a new one. Submitted orders are irreversible.

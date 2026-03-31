# Instruction Mode API Integration Guide

## Overview

The Instruction Mode (also called "PayFi Swap") is an advanced swap API that gives partners **full control over transaction construction and submission**. Unlike Order Mode (which manages the full lifecycle), Instruction Mode returns raw calldata or instructions — the partner builds, signs, and submits the transaction.

**When to use Instruction Mode vs Order Mode:**

| Feature | Order Mode | Instruction Mode |
|---------|-----------|------------------|
| Complexity | Lower — API handles tx building | Higher — partner builds tx |
| Supported features | Same-chain, cross-chain, gasless | Same-chain only, plus reverse quote |
| Signing | Sign API-provided tx/hash | Build full tx from calldata, sign locally |
| Submission | Via submitSwapOrder | Direct on-chain or via MEV batch send |
| Reverse quote (minAmountOut) | ❌ | ✅ |
| SOL instruction-level control | ❌ | ✅ (addressLookupTable + instructionLists) |

**API Host**: `https://bopenapi.bgwapi.io`

**Supported Chains**: ETH, SOL, BNB Chain, Base, Polygon, Arbitrum, Morph (more chains by arrangement)

---

## API Reference

### 1. Quote (Get Price)

`POST /bgw-pro/swapx/pro/quote`

Get a price estimate without building a transaction.

**Request Headers**

| Header | Description |
|--------|-------------|
| Content-Type | application/json |
| Partner-Code | Your partner code (required) |

**Request Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| fromChain | string | ✅ | Source chain (e.g., "sol", "bnb", "eth") |
| fromContract | string | ✅ | Source token contract. Empty string `""` for native tokens |
| fromAmount | string | ✅ | Input amount (human-readable, e.g., "1" = 1 token) |
| toChain | string | ✅ | Destination chain (must equal fromChain — same-chain only) |
| toContract | string | ✅ | Destination token contract |
| fromSymbol | string | ❌ | Source token symbol |
| toSymbol | string | ❌ | Destination token symbol |
| fromAddress | string | ❌ | Sender address (needed if `estimateGas: true`) |
| estimateGas | bool | ❌ | Whether to estimate gas (default false) |
| market | string | ❌ | Specify market (e.g., "uniswap.v3"). Default: all markets |
| feeRate | number | ❌ | Fee rate in permille. Default: channel config. Pass 0 for no fee |
| solMaxAccounts | number | ❌ | Max SOL accounts (limits route complexity, may affect price) |

**Request Example**

```json
{
  "fromSymbol": "USDT",
  "fromContract": "Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB",
  "fromAmount": "1",
  "fromChain": "sol",
  "toSymbol": "USDC",
  "toChain": "sol",
  "toContract": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "fromAddress": "ApPjjUmJuraNuFZ2n41TBzdshd3pmDEQvH6hFukQ1Yv2",
  "estimateGas": true
}
```

**Response Fields**

| Field | Type | Description |
|-------|------|-------------|
| toAmount | string | Estimated output amount |
| market | string | Best market/route (pass to swap endpoint) |
| slippage | string | Recommended slippage (percentage, "2" = 2%) |
| estimateRevert | bool | Whether the trade would revert on-chain |
| gasLimit | number | Estimated gas limit (EVM) |
| computeUnits | number | Estimated compute units (SOL) |

---

### 2. Swap (Get Calldata)

`POST /bgw-pro/swapx/pro/swap`

Build a swap transaction. Returns calldata for EVM or serialized instructions for SOL.

**Request Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| fromChain | string | ✅ | Source chain |
| fromContract | string | ✅ | Source token contract |
| fromAmount | string | ✅ | Input amount (human-readable) |
| toChain | string | ✅ | Destination chain (must equal fromChain) |
| toContract | string | ✅ | Destination token contract |
| fromAddress | string | ✅ | Sender address |
| toAddress | string | ✅ | Receiver address |
| market | string | ✅ | Market from quote response |
| fromSymbol | string | ❌ | Source token symbol |
| toSymbol | string | ❌ | Destination token symbol |
| slippage | number | ❌ | Slippage percentage (e.g., 1 = 1%). Default: system config |
| toMinAmount | string | ❌ | Minimum output. If not set, calculated from slippage |
| feeRate | number | ❌ | Fee rate in permille |
| executorAddress | string | ❌ | If tx sender differs from fromAddress |
| solMaxAccounts | number | ❌ | Max SOL accounts |
| feePayer | string | ❌ | SOL: who pays for account creation fees |
| deadline | number | ❌ | Tx expiry in seconds (default 600) |
| protocols | string | ❌ | Protocol filter |
| requestMod | string | ❌ | `"simple"` or `"rich"` response mode |

**Response Fields (EVM)**

| Field | Type | Description |
|-------|------|-------------|
| id | string | Order ID |
| market | string | Route used |
| contract | string | Router contract address |
| calldata | string | Transaction calldata (hex) |
| deadline | number | Expiry in seconds |

**Response Fields (SOL)**

| Field | Type | Description |
|-------|------|-------------|
| id | string | Order ID |
| market | string | Route used |
| calldata | string | Serialized transaction (base58) |
| computeUnits | number | Compute units |
| addressLookupTableAccount | string[] | Lookup table pubkeys for Versioned Transaction |
| instructionLists | object[] | Individual instructions for custom tx assembly |

**SOL instructionLists element:**

| Field | Type | Description |
|-------|------|-------------|
| programId | string | Program public key |
| accounts | object[] | Account list (`pubkey`, `isSigner`, `isWritable`) |
| data | string | Base64-encoded instruction data |

---

### 3. Reverse Quote Swap (swapr)

`POST /bgw-pro/swapx/pro/swapr`

Combines quote + tx building in one call. Supports two modes:

- **exactIn** — User specifies input amount → system calculates output
- **minAmountOut** — User specifies desired minimum output → system calculates required input

**Request Parameters**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| fromChain | string | ✅ | — | Chain identifier |
| fromContract | string | ✅ | — | Source token contract |
| toContract | string | ✅ | — | Destination token contract |
| amount | string | ✅ | — | Amount (meaning depends on requestMode) |
| requestMode | string | ✅ | — | `"exactIn"` or `"minAmountOut"` |
| fromAddress | string | ✅ | — | Sender address |
| toAddress | string | ✅ | — | Receiver address |
| feeRate | float64 | ✅ | — | Fee rate (permille), must ≥ 0 |
| slippage | string | ❌ | "0.5" | Slippage percentage |
| deadline | int | ❌ | 600 | Expiry in seconds |
| executorAddress | string | ❌ | — | If tx sender differs from fromAddress |

**Response Fields**

| Field | Type | Description |
|-------|------|-------------|
| id | string | Order ID |
| amountIn | string | Input amount |
| expectedAmountOut | string | Expected output |
| minAmountOut | string | Minimum output (with slippage) |
| priceImpact | string | Price impact percentage |
| recommendSlippage | number | Recommended slippage |
| expiresAt | number | Expiry timestamp (Unix seconds) |
| market | string | Route used |
| txs[] | object[] | Transaction list |
| fee | object | Fee breakdown |

**txs[] element:**

| Field | Type | Description |
|-------|------|-------------|
| chainId | number | Chain ID |
| to | string | Target contract |
| calldata | string | Transaction calldata |
| function | string | Function name |
| gasLimit | string | Gas limit |
| gasPrice | string | Gas price |
| nonce | number | Nonce |
| value | string | ETH value |

---

### 4. MEV Batch Send

`POST /bgw-pro/swapx/pro/send`

Submit signed transactions with MEV protection.

**Request Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| chain | string | ✅ | Chain name |
| txs[] | object[] | ✅ | Transaction list (max 100) |

**txs[] element:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | ✅ | Order ID (from swap/swapr response) |
| chain | string | ✅ | Chain name |
| rawTx | string | ✅ | Signed transaction hex |
| from | string | ✅ | Sender address |
| nonce | number | ✅ | Transaction nonce |
| provider | string | ❌ | Send provider |

**Response Fields**

| Field | Description |
|-------|-------------|
| result[].id | Order ID |
| result[].code | Result code (0 = success) |
| result[].txHash | Transaction hash |

---

## Integration Flow

### Standard Flow (exactIn)

```
1. Call /quote → get toAmount, market, slippage
2. Show user: "Swap X TokenA → ~Y TokenB"
3. User confirms
4. Call /swap with market from step 1
5. Build tx from calldata response:
   - EVM: { to: contract, data: calldata, gas, nonce, chainId, value }
   - SOL: Deserialize calldata or assemble from instructionLists
6. Sign with wallet
7. Submit on-chain (direct broadcast or via /send for MEV protection)
```

### Reverse Quote Flow (minAmountOut)

```
1. User says "I want at least 10 USDC"
2. Call /swapr with requestMode: "minAmountOut", amount: "10"
3. Response tells you amountIn (how much input token needed)
4. Show user: "Need X TokenA to get ≥ 10 USDC"
5. User confirms
6. Build and sign txs from response
7. Submit on-chain
```

### SOL Transaction Assembly

For Solana, you have two options:

**Option A: Use serialized calldata** (simpler)
```javascript
const tx = VersionedTransaction.deserialize(bs58.decode(response.data.calldata));
tx.sign([wallet]);
const txHash = await connection.sendRawTransaction(tx.serialize());
```

**Option B: Use instructionLists** (more control)
```javascript
const lookupTables = await Promise.all(
  response.data.addressLookupTableAccount.map(addr =>
    connection.getAddressLookupTable(new PublicKey(addr))
  )
);

const instructions = response.data.instructionLists.map(ix => ({
  programId: new PublicKey(ix.programId),
  keys: ix.accounts.map(a => ({
    pubkey: new PublicKey(a.pubkey),
    isSigner: a.isSigner,
    isWritable: a.isWritable
  })),
  data: Buffer.from(ix.data, 'base64')
}));

const message = new TransactionMessage({
  payerKey: wallet.publicKey,
  recentBlockhash: blockhash,
  instructions
}).compileToV0Message(lookupTables);

const tx = new VersionedTransaction(message);
tx.sign([wallet]);
```

---

## FAQ

### Q: What's the difference between /swap and /swapr?

`/swap` only supports `exactIn` (specify input amount). `/swapr` supports both `exactIn` AND `minAmountOut` (specify desired output). Use `/swapr` when you need reverse quoting.

### Q: Can I use Instruction Mode for cross-chain swaps?

No. Instruction Mode is **same-chain only** (`fromChain == toChain`). Use Order Mode for cross-chain.

### Q: Why use /send instead of broadcasting directly?

`/send` provides MEV protection — transactions are sent through private channels to avoid frontrunning. This is especially important for large trades on competitive DEXes.

### Q: How do I handle the feeRate parameter?

`feeRate` is in **permille** (parts per thousand):
- `feeRate: 0` → no fee
- `feeRate: 0.003` → 0.3% fee
- `feeRate: 0.05` → 5% fee

### Q: What happens if /swapr minAmountOut can't converge?

Returns error code `80008`. The algorithm couldn't find a path to deliver the requested minimum output. Suggestions:
1. Reduce the `amount` (requested output)
2. Increase `slippage`
3. Try a different token pair

### Q: The calldata is very large — is that normal?

Yes. Complex multi-hop routes can produce large calldata. EVM calldata can be 1-5KB. SOL serialized transactions are typically smaller but may include many instructions.

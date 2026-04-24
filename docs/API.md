# Rise API

## Base URL

| Environment | URL |
|---|---|
| **Mainnet** | `https://public.rise.rich` |
| **Devnet** | `https://publicdev.rise.rich` |

## Authentication

All requests require an API key via the `x-api-key` header.

```typescript
headers: { "x-api-key": "YOUR_API_KEY" }
```

Contact a team member with your project name to get an API key.

Trade Rise tokens in 3 steps: **quote → trade → sign & send**.

---

## Endpoints

| | Endpoint | What it does |
|--|----------|-------------|
| 1 | [`GET /markets/{address}`](#get-marketsaddress) | Get token info |
| 2 | [`GET /markets/{address}/transactions`](#get-marketsaddresstransactions) | Transaction history |
| 3 | [`GET /markets/{address}/ohlc/{timeframe}`](#get-marketsaddressohlctimeframe) | OHLC chart data |
| 4 | [`POST /markets/{address}/quote`](#post-marketsaddressquote) | Preview a trade |
| 5 | [`POST /program/buyToken`](#post-programbuytoken) | Buy tokens |
| 6 | [`POST /program/sellToken`](#post-programselltoken) | Sell tokens |
| 7 | [`GET /users/{wallet}/portfolio/summary`](#get-userswalletportfoliosummary) | Portfolio totals |
| 8 | [`GET /users/{wallet}/portfolio/positions`](#get-userswalletportfoliopositions) | All positions |
| 9 | [`POST /markets/{address}/borrow/quote`](#post-marketsaddressborrowquote) | Preview borrowing capacity |
| 10 | [`POST /program/deposit-and-borrow`](#post-programdeposit-and-borrow) | Deposit collateral + borrow in one tx |
| 11 | [`POST /program/repay-and-withdraw`](#post-programrepay-and-withdraw) | Repay debt + withdraw in one tx |
| 12 | [`GET /markets`](#get-markets) | List all markets (filter, sort, paginate) |

> **Minimum integration:** just endpoints 4 + 5 (quote + buy). Endpoint 1 is useful to get all market data (price, floor, volume, holders, etc.).

---

## Rate limits

Limits are per API key, measured over a rolling 60-second window.

| Endpoint | Requests / min |
|---|---|
| `GET /markets` | 40 |
| `GET /markets/{address}` | 55 |
| `GET /markets/{address}/transactions` | 60 |
| `GET /markets/{address}/ohlc/{timeframe}` | 20 |
| `POST /markets/{address}/quote` | 40 |
| `POST /markets/{address}/borrow/quote` | 40 |
| `POST /program/buyToken` | 30 |
| `POST /program/sellToken` | 30 |
| `POST /program/deposit-and-borrow` | 10 |
| `POST /program/repay-and-withdraw` | 10 |
| `GET /users/{wallet}/portfolio/summary` | 60 |
| `GET /users/{wallet}/portfolio/positions` | 60 |
| **Global cap (all endpoints combined)** | **150** |

Exceeding a limit returns HTTP `429` with `{ "ok": false, "error": "..." }`. Repeated violations across separate minutes trigger progressive cooldowns: **1 min → 5 min → 20 min → 1 day** after the 4th violation. Stay within the limits or back off on `429` and you'll never hit a cooldown.

---

## Quick start

**Quote:**
```typescript
const { quote } = await fetch(`${API}/markets/${market}/quote`, {
  method: "POST",
  headers: { "Content-Type": "application/json", "x-api-key": "YOUR_API_KEY" },
  body: JSON.stringify({ amount: 100_000_000, direction: "buy" }),
}).then((r) => r.json());
```

**Buy:**
```typescript
const { transaction } = await fetch(`${API}/program/buyToken`, {
  method: "POST",
  headers: { "Content-Type": "application/json", "x-api-key": "YOUR_API_KEY" },
  body: JSON.stringify({
    wallet: "YOUR_WALLET_PUBLIC_KEY",
    market: "TOKEN_MINT_OR_RISE_MARKET_ADDRESS",
    cashIn: 100_000_000,
    minTokenOut: Math.floor(quote.amountOut * 0.99), // 1% slippage
  }),
}).then((r) => r.json());
```

**Sign & send:**
```typescript
const tx = VersionedTransaction.deserialize(Buffer.from(transaction, "base64"));
tx.sign([wallet]);
await connection.sendRawTransaction(tx.serialize());
```

---

## Endpoint reference

---

<details>
<summary><strong>GET /markets/{address}</strong> — Get token info and market data</summary>

```
GET /markets/{tokenMintOrRiseMarketAddress}
```

**Response:**
```json
{
  "ok": true,
  "market": {
    "rise_market_address": "HfYP1dq4cqx8Yg7nqg4j5Z2k5L8M9V2K5Q8S1U4X",
    "mint_token": "DezXAZ8z7PnrnRJjz3wXBoRgixCaSKHY2q9BvRISE",
    "mint_main": "So11111111111111111111111111111111111111112",
    "token_name": "Bear Token",
    "token_symbol": "BEAR",
    "token_image": "https://arweave.net/abc123",
    "token_decimals": 9,
    "creator": "9B5X1CK5m2Q6S9V1W4Y7Z3A5B8C1D2E5F9G2H5J",
    "price": "0.004523",
    "mayflower_floor": "0.001200",
    "mayflower_token_supply": "500000000000000",
    "mayflower_total_cash_liquidity": "50000000000000",
    "mayflower_total_debt": "30000000000000",
    "mayflower_total_collateral": "40000000000000",
    "volume_h24_usd": "15000.25",
    "volume_all_time_usd": "250000.50",
    "market_cap_usd": "2250000.75",
    "holders_count": 342,
    "creator_fee_percent": 5,
    "gov_buy_fee_micro_basis_points": 12500,
    "gov_sell_fee_micro_basis_points": 12500,
    "disableSell": false,
    "created_at": "2025-01-15T10:30:00.000Z"
  }
}
```

**Fields you'll use most:**

| Field | What it is |
|-------|-----------|
| `token_name`, `token_symbol`, `token_image` | Token info |
| `rise_market_address` | Rise market account (pass to buy/sell) |
| `mint_token` | SPL token mint |
| `mint_main` | Collateral mint (SOL or USDC) |
| `price` | Current price in collateral |
| `mayflower_floor` | Floor price (can never go below this) |
| `holders_count` | Unique holders |

</details>

---

<details>
<summary><strong>GET /markets/{address}/transactions</strong> — Transaction history</summary>

```
GET /markets/{tokenMintOrRiseMarketAddress}/transactions?page=1&limit=50
```

| Param | Type | |
|-------|------|-|
| `page` | number | Page number (default: 1) |
| `limit` | number | Results per page (default: 50, max: 100) |

**Response:**
```json
{
  "ok": true,
  "page": 1,
  "limit": 50,
  "total": 142,
  "totalPages": 3,
  "count": 50,
  "transactions": [
    {
      "rise_market_address": "HfYP1dq4cqx8Yg7nqg4j5Z2k5L8M9V2K5Q8S1U4X",
      "transaction_type": "buy",
      "wallet_address": "9B5X1CK5m2Q6S9V1W4Y7Z3A5B8C1D2E5F9G2H5J",
      "price": "0.004523",
      "floor_price": "0.001200",
      "amount_put": "100000000",
      "amount_received": "22145038900",
      "token_supply": "500000000000000",
      "transaction_signature": "5K2x...abc",
      "slot": 312456789,
      "created_at": "2025-01-15T12:30:00.000Z",
      "volume_usd": "14.50"
    }
  ]
}
```

**Transaction types:** `create`, `buy`, `sell`, `borrow`, `repay`, `deposit`, `withdraw`, `withdraw_creator_fees`

</details>

---

<details>
<summary><strong>GET /markets/{address}/ohlc/{timeframe}</strong> — OHLC chart data</summary>

```
GET /markets/{tokenMintOrRiseMarketAddress}/ohlc/{timeframe}?limit=1000
```

| Param | Type | |
|-------|------|-|
| `timeframe` | string | `1m`, `5m`, `1h`, or `1d` |
| `limit` | number | Max candles to return (default varies by timeframe, max: 10000) |

**Default limits by timeframe:**

| Timeframe | Default limit | Covers approximately |
|-----------|--------------|---------------------|
| `1m` | 10,000 | ~7 days |
| `5m` | 5,000 | ~17 days |
| `1h` | 3,000 | ~4 months |
| `1d` | 1,000 | ~2.7 years |

**Response:**
```json
{
  "ok": true,
  "timeframe": "1h",
  "count": 48,
  "data": [
    {
      "time": "2025-01-15T12:00:00.000Z",
      "open": 0.004500,
      "high": 0.004600,
      "low": 0.004480,
      "close": 0.004523,
      "floorPrice": 0.001200,
      "transactionCount": 12,
      "volume": 500000000
    }
  ]
}
```

</details>

---

<details>
<summary><strong>POST /markets/{address}/quote</strong> — Preview a trade</summary>

```
POST /markets/{tokenMintOrRiseMarketAddress}/quote
```

**Body:**
```json
{
  "amount": 100000000,
  "direction": "buy"
}
```

| Param | Type | |
|-------|------|-|
| `amount` | number | RAW units. Buy = collateral to spend. Sell = tokens to sell. |
| `direction` | string | `"buy"` or `"sell"` |

**Response (buy):**
```json
{
  "ok": true,
  "quote": {
    "direction": "buy",
    "amountIn": 100000000,
    "amountInHuman": 0.1,
    "amountOut": 22145038900,
    "amountOutHuman": 22.1450389,
    "feeRate": 0.0125,
    "feeAmount": 1250000,
    "feeAmountUsd": 0.18,
    "amountInUsd": 14.50,
    "amountOutUsd": 14.32,
    "mintRate": 0.004523,
    "tokenRate": 221.1,
    "currentPrice": 0.004500,
    "newPrice": 0.004523,
    "averageFillPrice": 0.004516,
    "priceImpact": 0.0051,
    "currentSupply": 500000.0,
    "newSupply": 500022.145
  }
}
```

**Response (sell):**
```json
{
  "ok": true,
  "quote": {
    "direction": "sell",
    "amountIn": 10000000000,
    "amountInHuman": 10.0,
    "amountOut": 44800000,
    "amountOutHuman": 0.0448,
    "feeRate": 0.0125,
    "feeAmount": 568000,
    "feeAmountUsd": 0.08,
    "amountInUsd": 6.52,
    "amountOutUsd": 6.44,
    "mintRate": 0.004544,
    "tokenRate": 220.07,
    "currentPrice": 0.004523,
    "newPrice": 0.004510,
    "averageFillPrice": 0.004544,
    "priceImpact": 0.0029,
    "currentSupply": 500000.0,
    "newSupply": 499990.0
  }
}
```

**Fields you'll use most:**

| Field | What it is |
|-------|-----------|
| `amountOut` | What you'll receive (RAW) — pass to `minTokenOut`/`minCashOut` with slippage |
| `amountOutHuman` | Same but human-readable |
| `priceImpact` | Decimal (0.01 = 1%) |
| `feeRate` | Fee as decimal (0.0125 = 1.25%) |
| `currentPrice` → `newPrice` | Price before → after the trade |

</details>

---

<details>
<summary><strong>POST /program/buyToken</strong> — Buy tokens</summary>

```
POST /program/buyToken
```

**Body:**
```json
{
  "wallet": "YOUR_WALLET_PUBLIC_KEY",
  "market": "TOKEN_MINT_OR_RISE_MARKET_ADDRESS",
  "cashIn": 100000000,
  "minTokenOut": 21923588000
}
```

| Param | Type | |
|-------|------|-|
| `wallet` | string | Your wallet public key |
| `market` | string | Token mint or Rise market address |
| `cashIn` | number | Collateral to spend (RAW) |
| `minTokenOut` | number | Min tokens to receive (RAW) — slippage protection |

**Response:**
```json
{
  "ok": true,
  "transaction": "AQAAAAAAAAAAAABz/nEq2dxwj0l4TfVN1Dji...",
  "addresses": {
    "mainSrc": "So11111111111111111111111111111111111111112",
    "tokenDst": "DezXAZ8z7PnrnRJjz3wXBoRgixCaSKHY2q9BvRISE"
  }
}
```

**Slippage:** set `minTokenOut = 0` if you don't care. Otherwise:
```typescript
const minTokenOut = Math.floor(quote.amountOut * (1 - 0.01)); // 1% slippage
```

</details>

---

<details>
<summary><strong>POST /program/sellToken</strong> — Sell tokens</summary>

```
POST /program/sellToken
```

**Body:**
```json
{
  "wallet": "YOUR_WALLET_PUBLIC_KEY",
  "market": "TOKEN_MINT_OR_RISE_MARKET_ADDRESS",
  "tokenIn": 10000000000,
  "minCashOut": 44352000
}
```

| Param | Type | |
|-------|------|-|
| `wallet` | string | Your wallet public key |
| `market` | string | Token mint or Rise market address |
| `tokenIn` | number | Tokens to sell (RAW) |
| `minCashOut` | number | Min collateral to receive (RAW) — slippage protection |

**Response:**
```json
{
  "ok": true,
  "transaction": "AQAAAAAAAAAAAABz/nEq2dxwj0l4TfVN1Dji...",
  "addresses": {
    "tokenSrc": "DezXAZ8z7PnrnRJjz3wXBoRgixCaSKHY2q9BvRISE",
    "mainDst": "So11111111111111111111111111111111111111112"
  }
}
```

</details>

---

<details>
<summary><strong>GET /users/{wallet}/portfolio/summary</strong> — Portfolio totals</summary>

```
GET /users/{walletPublicKey}/portfolio/summary
```

**Response:**
```json
{
  "ok": true,
  "summary": {
    "total_value_usd": "189.26",
    "total_pnl_usd": "17.03",
    "total_transactions": 24,
    "tokens_held": 3,
    "tokens_created_count": 1
  }
}
```

</details>

---

<details>
<summary><strong>GET /users/{wallet}/portfolio/positions</strong> — All positions with P&L</summary>

```
GET /users/{walletPublicKey}/portfolio/positions?page=1&limit=20
```

**Response:**
```json
{
  "ok": true,
  "total": 3,
  "page": 1,
  "limit": 20,
  "totalPages": 1,
  "results": [
    {
      "rise_market_address": "HfYP1dq4cqx8Yg7nqg4j5Z2k5L8M9V2K5Q8S1U4X",
      "token_name": "Bear Token",
      "token_symbol": "BEAR",
      "token_image": "https://arweave.net/abc123",
      "mint_token": "DezXAZ8z7PnrnRJjz3wXBoRgixCaSKHY2q9BvRISE",
      "mint_main": "So11111111111111111111111111111111111111112",
      "net_tokens": "22.5",
      "position_value": "0.1017",
      "position_value_usd": "14.75",
      "cost_basis": "0.0985",
      "pnl": "0.0032",
      "pnl_usd": "0.46",
      "pnl_percentage": "3.25",
      "market_price": "0.004523",
      "collateral_price_usd": "145.00"
    }
  ]
}
```

| Field | What it is |
|-------|-----------|
| `net_tokens` | Token balance |
| `position_value` / `position_value_usd` | Current value |
| `cost_basis` | What you paid (in collateral) |
| `pnl` / `pnl_usd` / `pnl_percentage` | Unrealized P&L |
| `market_price` | Current token price |

</details>

---

<details>
<summary><strong>POST /markets/{address}/borrow/quote</strong> — Preview borrowing capacity</summary>

```
POST /markets/{tokenMintOrRiseMarketAddress}/borrow/quote
```

Returns borrowing capacity for a user on a specific market. Optionally calculates the required deposit for a specific borrow amount.

**Body:**
```json
{
  "wallet": "YOUR_WALLET_PUBLIC_KEY",
  "amountToBorrow": 50000000
}
```

| Param | Type | |
|-------|------|-|
| `wallet` | string | Your wallet public key |
| `amountToBorrow` | number | *(optional)* Specific amount to borrow — returns required deposit |

**Response:**
```json
{
  "ok": true,
  "depositedTokens": "150000000000",
  "walletBalance": "500000000000",
  "debt": "20000000",
  "maxBorrowable": "80000000",
  "maxBorrowableUsd": "11.60",
  "maxBorrowableIfDepositAll": "250000000",
  "maxBorrowableIfDepositAllUsd": "36.25",
  "floorPrice": "0.000800",
  "borrowFeePercent": 3,
  "requiredDeposit": "62500000000",
  "grossBorrow": "51546392"
}
```

| Field | What it is |
|-------|-----------|
| `depositedTokens` | Tokens already deposited as collateral (RAW) |
| `walletBalance` | User's token wallet balance (RAW) |
| `debt` | Current outstanding debt (RAW) |
| `maxBorrowable` | Max borrowable with current collateral, after fee (RAW) |
| `maxBorrowableIfDepositAll` | Max borrowable if depositing entire wallet balance, after fee (RAW) |
| `floorPrice` | Floor price used for LTV calculation |
| `borrowFeePercent` | Borrow fee as percentage (e.g., 3 = 3%) |
| `requiredDeposit` | *(only if `amountToBorrow` provided)* Tokens to deposit for the requested borrow |
| `grossBorrow` | *(only if `amountToBorrow` provided)* Gross borrow amount before fee |

</details>

---

<details>
<summary><strong>POST /program/deposit-and-borrow</strong> — Deposit collateral + borrow in one transaction</summary>

```
POST /program/deposit-and-borrow
```

Combines deposit + borrow into a single atomic transaction. The backend computes the exact deposit amount needed based on your borrow request.

**Body:**
```json
{
  "wallet": "YOUR_WALLET_PUBLIC_KEY",
  "market": "TOKEN_MINT_OR_RISE_MARKET_ADDRESS",
  "borrowAmount": 50000000
}
```

| Param | Type | |
|-------|------|-|
| `wallet` | string | Your wallet public key |
| `market` | string | Token mint or Rise market address |
| `borrowAmount` | number | Amount to borrow (RAW, after fee) |

**Response:**
```json
{
  "ok": true,
  "transaction": "AQAAAAAAAAAAAABz/nEq2dxwj0l4TfVN1Dji...",
  "depositAmount": "150000000000",
  "borrowAmount": "51546392",
  "borrowAmountAfterFee": 50000000,
  "includedDeposit": true,
  "addresses": {
    "personalAccount": "...",
    "mayPersonalPosition": "...",
    "mayPersonalPositionEscrow": "...",
    "tokenSrc": "...",
    "mainDst": "..."
  }
}
```

| Field | What it is |
|-------|-----------|
| `depositAmount` | Collateral deposited (RAW) — may be 0 if existing collateral is sufficient |
| `borrowAmount` | Gross borrow amount before fee (RAW) |
| `borrowAmountAfterFee` | Net amount you receive after fee |
| `includedDeposit` | Whether a deposit instruction was included |

</details>

---

<details>
<summary><strong>POST /program/repay-and-withdraw</strong> — Repay debt + withdraw collateral in one transaction</summary>

```
POST /program/repay-and-withdraw
```

Combines repay + withdraw into a single atomic transaction. The backend computes the repay amount needed to maintain LTV after your withdrawal.

**Body:**
```json
{
  "wallet": "YOUR_WALLET_PUBLIC_KEY",
  "market": "TOKEN_MINT_OR_RISE_MARKET_ADDRESS",
  "withdrawAmount": 100000000000
}
```

| Param | Type | |
|-------|------|-|
| `wallet` | string | Your wallet public key |
| `market` | string | Token mint or Rise market address |
| `withdrawAmount` | number | Amount of collateral to withdraw (RAW) |

**Response:**
```json
{
  "ok": true,
  "transaction": "AQAAAAAAAAAAAABz/nEq2dxwj0l4TfVN1Dji...",
  "repayAmount": "30000000",
  "withdrawAmount": "100000000000",
  "includedRepay": true,
  "addresses": {
    "personalAccount": "...",
    "mayPersonalPosition": "...",
    "mayPersonalPositionEscrow": "...",
    "mainSrc": "...",
    "tokenDst": "..."
  }
}
```

| Field | What it is |
|-------|-----------|
| `repayAmount` | Debt repaid to maintain LTV (RAW) — may be 0 if no repay needed |
| `withdrawAmount` | Collateral withdrawn (RAW) |
| `includedRepay` | Whether a repay instruction was included |

</details>

---

<details>
<summary><strong>GET /markets</strong> — List all markets (filter, sort, paginate)</summary>

```
GET /markets?page=1&limit=20&sort=created&order=desc
```

**Pagination**

| Param | Type | |
|-------|------|-|
| `page` | number | Page number (default: `1`) |
| `limit` | number | Results per page (default: `20`, max: `1000`) |

**Sorting**

| Param | Type | |
|-------|------|-|
| `sort` | string | One of: `created` (default), `marketcap`, `volume24h`, `holders`, `floor`, `price`, `variation`, `near_floor`, `liquidity` |
| `order` | string | `desc` (default) or `asc` |

- `price` and `variation` both sort by 24h price change %.
- `near_floor` sorts by `(price − floor) / floor` ascending-friendliest (tokens trading closest to floor first when `order=asc`).
- `liquidity` groups markets by their collateral mint (`mint_main`) — useful to cluster all SOL-collateral markets, then USDC, etc.

**Filtering**

| Param | Type | |
|-------|------|-|
| `is_verified` | boolean | `true` / `false` — only verified (or unverified) markets |
| `creator_fee_min` | integer | Min `creator_fee_percent` (e.g. `0`) |
| `creator_fee_max` | integer | Max `creator_fee_percent` |
| `mcap_min` | number | Min `market_cap_usd` |
| `mcap_max` | number | Max `market_cap_usd` |
| `vol24h_min` | number | Min 24h USD volume |
| `vol24h_max` | number | Max 24h USD volume |
| `locked_min` | integer | Min `locked_supply_percentage` (0–100) |
| `locked_max` | integer | Max `locked_supply_percentage` (0–100) |
| `created_period` | string | `today` \| `yesterday` \| `week` \| `month` |

All filters are optional and can be combined. Responses are cached for 10s per unique param combination.

**Computed fields (added to each market row)**

| Field | Meaning |
|-------|---------|
| `delta_to_floor_percentage` | `((price − floor) / floor) × 100`, rounded to 1 decimal. `"0"` if floor or price is 0. |
| `locked_supply_percentage` | `(mayflower_total_collateral / mayflower_token_supply) × 100`, rounded to 1 decimal. `"0"` if supply is 0. |

**Response:**
```json
{
  "ok": true,
  "count": 20,
  "total": 142,
  "page": 1,
  "limit": 20,
  "totalPages": 8,
  "markets": [
    {
      "rise_market_address": "HfYP1dq4cqx8Yg7nqg4j5Z2k5L8M9V2K5Q8S1U4X",
      "mint_token": "DezXAZ8z7PnrnRJjz3wXBoRgixCaSKHY2q9BvRISE",
      "mint_main": "So11111111111111111111111111111111111111112",
      "token_name": "Bear Token",
      "token_symbol": "BEAR",
      "token_image": "https://arweave.net/abc123",
      "token_decimals": 9,
      "creator": "9B5X1CK5m2Q6S9V1W4Y7Z3A5B8C1D2E5F9G2H5J",
      "price": "0.004523",
      "starting_price": "0.001000",
      "mayflower_floor": "0.001200",
      "mayflower_token_supply": "500000000000000",
      "mayflower_total_cash_liquidity": "50000000000000",
      "mayflower_total_debt": "30000000000000",
      "mayflower_total_collateral": "40000000000000",
      "volume_h24_usd": "15000.25",
      "volume_all_time_usd": "250000.50",
      "market_cap_usd": "2250000.75",
      "holders_count": 342,
      "creator_fee_percent": 5,
      "gov_buy_fee_micro_basis_points": 12500,
      "gov_sell_fee_micro_basis_points": 12500,
      "disableSell": false,
      "twitter": "https://x.com/beartoken",
      "discord": "",
      "telegram": "",
      "created_at": "2025-01-15T10:30:00.000Z",
      "delta_to_floor_percentage": "276.9",
      "locked_supply_percentage": "8.0"
    }
  ]
}
```

Base fields mirror `GET /markets/{address}`; the two `*_percentage` fields are added by this endpoint.

</details>

---

## Good to know

- **Addresses** — every Rise token has a **token mint** and a **Rise market address**. All endpoints accept either one.
- **Collateral** — SOL-backed markets use Wrapped SOL (`So11111111111111111111111111111111111111112`). USDC markets use USDC (`EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`).
- **Amounts** — always in RAW units (no decimals). SOL has 9 decimals (`0.1 SOL = 100_000_000`), USDC has 6.
- **Transactions** — buy/sell endpoints return a base64 Solana `VersionedTransaction`. You deserialize, sign, and send.
- **Errors** — all endpoints return `{ "ok": false, "error": "..." }` on failure.
- **SSE Streams** — use `EventSource` to connect. Streams send a `heartbeat` every 15s. Reconnect on disconnect.

---

## Links

- <a href="https://docs.rise.rich" target="_blank">General Documentation</a>
- [On-Chain Program Docs](./PROGRAM.md) — direct Solana program integration
- [Indexing & Events](./INDEXING.md) — parse on-chain events to build your own indexer
- [IDL (JSON)](../idl/idl.json) · [IDL (TypeScript)](../idl/idl.ts)
- **Website:** <a href="https://www.rise.rich/" target="_blank">rise.rich</a>
- **Twitter/X:** <a href="https://x.com/risedotrich" target="_blank">risedotrich</a>
- **Telegram:** @Passoif · @OxSahand

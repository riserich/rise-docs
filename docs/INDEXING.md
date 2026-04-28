# Indexing & Events

The Rise program emits events on every buy, sell, borrow, repay, deposit, and withdraw transaction. You can parse these from on-chain transaction logs to build your own indexer.

---

## Event Types

### BuyWithExactCashInEvent

Emitted on every buy transaction.

| Field | Type | Description |
|-------|------|-------------|
| `buyer` | PublicKey | Buyer's wallet address |
| `market` | PublicKey | Rise market address |
| `cashIn` | u64 | Amount of collateral spent (RAW) |
| `minTokenOut` | u64 | Minimum tokens expected (slippage param) |
| `revSplit` | RevenueSplits | Fee breakdown (floor, creator, team) |
| `floor` | Decimal (u128) | Floor price after transaction |
| `tokenSupply` | u64 | Token supply after transaction |
| `m1` | Decimal (u128) | Bonding curve slope (shoulder segment) |
| `m2` | Decimal (u128) | Bonding curve slope (main segment) |
| `x2` | u64 | Supply transition point (shoulder → main) |
| `b2` | Decimal (u128) | Y-intercept for main segment |
| `lastFloorRaiseTimestamp` | u64 | Unix timestamp of last floor raise |
| `mintToken` | PublicKey | Token mint address |
| `mintMain` | PublicKey | Collateral mint address (SOL or USDC) |
| `tokenDecimals` | u8 | Token decimal places |

### SellWithExactTokenInEvent

Emitted on every sell transaction.

| Field | Type | Description |
|-------|------|-------------|
| `seller` | PublicKey | Seller's wallet address |
| `market` | PublicKey | Rise market address |
| `tokenIn` | u64 | Amount of tokens sold (RAW) |
| `cashOut` | u64 | Amount of collateral received (RAW) |
| `revSplit` | RevenueSplits | Fee breakdown (floor, creator, team) |
| `floor` | Decimal (u128) | Floor price after transaction |
| `tokenSupply` | u64 | Token supply after transaction |
| `m1` | Decimal (u128) | Bonding curve slope (shoulder segment) |
| `m2` | Decimal (u128) | Bonding curve slope (main segment) |
| `x2` | u64 | Supply transition point |
| `b2` | Decimal (u128) | Y-intercept for main segment |
| `mintToken` | PublicKey | Token mint address |
| `mintMain` | PublicKey | Collateral mint address |
| `tokenDecimals` | u8 | Token decimal places |

### BorrowEvent

Emitted on borrow transactions.

| Field | Type | Description |
|-------|------|-------------|
| `amount` | u64 | Amount borrowed (RAW) — gross, before the on-chain 3% borrow fee |
| `borrower` | PublicKey | Borrower's wallet address |
| `market` | PublicKey | Rise market address |
| `revSplit` | RevenueSplits | Fee breakdown |

### RepayEvent

Emitted on repay transactions.

| Field | Type | Description |
|-------|------|-------------|
| `amount` | u64 | Amount repaid (RAW) |
| `repayer` | PublicKey | Repayer's wallet address |
| `market` | PublicKey | Rise market address |

### DepositEvent

Emitted when a user deposits tokens as collateral into their personal position.

| Field | Type | Description |
|-------|------|-------------|
| `amount` | u64 | Amount deposited (RAW) |
| `depositor` | PublicKey | Depositor's wallet address |
| `market` | PublicKey | Rise market address |

### WithdrawEvent

Emitted when a user withdraws tokens from their personal position.

| Field | Type | Description |
|-------|------|-------------|
| `amount` | u64 | Amount withdrawn (RAW) |
| `withdrawer` | PublicKey | Withdrawer's wallet address |
| `market` | PublicKey | Rise market address |

### RevenueSplits

Fee distribution included in all events.

| Field | Type | Description |
|-------|------|-------------|
| `floor` | u64 | Amount allocated to floor investment |
| `creator` | u64 | Amount allocated to creator fees |
| `team` | u64 | Amount allocated to team fees |

---

## Parsing Events from Transaction Logs

Events are encoded in Solana transaction log messages. They appear as base64-encoded data in log lines prefixed with `"Program data: "`.

### Event Discriminators

Each event type has an 8-byte discriminator computed from its name:

```typescript
import { sha256 } from "@noble/hashes/sha256";

function eventDiscriminator(eventName: string): Buffer {
  return Buffer.from(sha256(`event:${eventName}`).slice(0, 8));
}

const BUY_DISC = eventDiscriminator("BuyWithExactCashInEvent");
const SELL_DISC = eventDiscriminator("SellWithExactTokenInEvent");
const BORROW_DISC = eventDiscriminator("BorrowEvent");
const REPAY_DISC = eventDiscriminator("RepayEvent");
const DEPOSIT_DISC = eventDiscriminator("DepositEvent");
const WITHDRAW_DISC = eventDiscriminator("WithdrawEvent");
```

`RepayEvent`, `DepositEvent`, and `WithdrawEvent` share the same on-the-wire layout: `amount(u64) + actor(Pubkey) + market(Pubkey)` (48 bytes after the 8-byte discriminator). Only the field name and discriminator differ:

```typescript
function parseSimpleAmountEvent(data: Buffer, expectedDisc: Buffer) {
  if (!data.slice(0, 8).equals(expectedDisc)) return null;
  const amount = data.readBigUInt64LE(8);
  const actor = new PublicKey(data.slice(16, 48));
  const market = new PublicKey(data.slice(48, 80));
  return { amount, actor, market };
}
```

### Extracting Events

```typescript
function extractEventsFromLogs(logs: string[]): Buffer[] {
  const events: Buffer[] = [];
  for (const log of logs) {
    if (log.startsWith("Program data: ")) {
      const data = Buffer.from(log.slice("Program data: ".length), "base64");
      events.push(data);
    }
  }
  return events;
}

function parseBuyEvent(data: Buffer) {
  // First 8 bytes = discriminator
  const disc = data.slice(0, 8);
  if (!disc.equals(BUY_DISC)) return null;

  let offset = 8;

  const buyer = new PublicKey(data.slice(offset, offset + 32));       offset += 32;
  const market = new PublicKey(data.slice(offset, offset + 32));      offset += 32;
  const cashIn = data.readBigUInt64LE(offset);                        offset += 8;
  const minTokenOut = data.readBigUInt64LE(offset);                   offset += 8;

  // RevenueSplits (3 x u64)
  const revFloor = data.readBigUInt64LE(offset);                      offset += 8;
  const revCreator = data.readBigUInt64LE(offset);                    offset += 8;
  const revTeam = data.readBigUInt64LE(offset);                       offset += 8;

  // Decimal values are 16-byte Rust Decimal
  const floor = data.slice(offset, offset + 16);                      offset += 16;
  const tokenSupply = data.readBigUInt64LE(offset);                   offset += 8;
  const m1 = data.slice(offset, offset + 16);                         offset += 16;
  const m2 = data.slice(offset, offset + 16);                         offset += 16;
  const x2 = data.readBigUInt64LE(offset);                            offset += 8;
  const b2 = data.slice(offset, offset + 16);                         offset += 16;
  const lastFloorRaiseTimestamp = data.readBigUInt64LE(offset);        offset += 8;
  const mintToken = new PublicKey(data.slice(offset, offset + 32));    offset += 32;
  const mintMain = new PublicKey(data.slice(offset, offset + 32));     offset += 32;
  const tokenDecimals = data.readUInt8(offset);                        offset += 1;

  return {
    buyer, market, cashIn, minTokenOut,
    revSplit: { floor: revFloor, creator: revCreator, team: revTeam },
    floor, tokenSupply, m1, m2, x2, b2,
    lastFloorRaiseTimestamp, mintToken, mintMain, tokenDecimals,
  };
}
```

### Computing Price from Event Data

After parsing an event, you can compute the current token price using the bonding curve parameters:

```typescript
function calculatePrice(
  tokenSupply: number,  // human-readable (RAW / 10^decimals)
  floor: number,
  m1: number,           // slope * decimalsFactor
  m2: number,           // slope * decimalsFactor
  x2: number,           // human-readable
  b2: number,
): number {
  if (tokenSupply <= x2) {
    return floor + m1 * tokenSupply;
  } else {
    return floor + m2 * tokenSupply + b2;
  }
}
```

### Rust Decimal Deserialization

The `floor`, `m1`, `m2`, `b2` fields are 16-byte Rust Decimal values. To convert to a JavaScript number:

```typescript
import Decimal from "decimal.js";

function deserializeRustDecimal(buf: Buffer): number {
  // Rust Decimal layout: [flags(4), hi(4), lo(4), mid(4)]
  const flags = buf.readUInt32LE(0);
  const hi = buf.readUInt32LE(4);
  const lo = buf.readUInt32LE(8);
  const mid = buf.readUInt32LE(12);

  const scale = (flags >> 16) & 0xff;
  const negative = (flags & 0x80000000) !== 0;

  // Reconstruct 96-bit integer: hi * 2^64 + mid * 2^32 + lo
  const value = new Decimal(hi).mul(new Decimal(2).pow(64))
    .plus(new Decimal(mid).mul(new Decimal(2).pow(32)))
    .plus(new Decimal(lo));

  const scaled = value.div(new Decimal(10).pow(scale));
  return (negative ? scaled.neg() : scaled).toNumber();
}
```

---

## Floor Raise Events

The Rise program has two floor raise methods. Both emit events but **neither includes the resulting floor price** — only the increase ratio.

### RaiseFloorEvent (PreserveArea)

Emitted when the floor is raised by preserving the curve area while shifting the floor up.

| Field | Type | Description |
|-------|------|-------------|
| `market` | PublicKey | Rise market address |
| `newLevel` | u32 | New floor level index |
| `newShoulderEnd` | u64 | New x2 boundary (shoulder → main transition) |
| `floorIncreaseRatio` | Decimal (u128) | Ratio of floor increase (e.g. 0.002 = 0.2%) |
| `timestamp` | u64 | Unix timestamp |

### RaiseFloorExcessLiquidityEvent

Emitted when the floor is raised using surplus collateral in the vault.

| Field | Type | Description |
|-------|------|-------------|
| `market` | PublicKey | Rise market address |
| `newLevel` | u32 | New floor level index |
| `increaseRatioMicroBasisPoints` | u32 | Floor increase ratio in micro basis points |
| `timestamp` | u64 | Unix timestamp |

### Event Discriminators

```typescript
const RAISE_FLOOR_DISC = eventDiscriminator("RaiseFloorEvent");
const RAISE_FLOOR_EXCESS_DISC = eventDiscriminator("RaiseFloorExcessLiquidityEvent");
```

---

## Tracking the Floor Price

The floor price can be tracked in two ways:

### 1. From Buy/Sell Events (real-time)

Every `BuyWithExactCashInEvent` and `SellWithExactTokenInEvent` includes the current `floor` field. This is the easiest way — just read it from every trade event.

### 2. After Floor Raise Instructions (fetch on-chain)

Floor raise events don't contain the resulting floor value. After detecting a `raiseFloorPreserveArea` or `raiseFloorExcessLiquidity` instruction, fetch the Mayflower market account (account index 5) to read the updated floor:

```typescript
// After detecting a floor raise instruction:
// Account index 5 = Mayflower market account
const mayflowerMarketKey = instruction.accounts[5];

// Fetch the account on-chain to get updated floor
const accountInfo = await connection.getAccountInfo(
  new PublicKey(mayflowerMarketKey)
);
const mayflowerData = program.coder.accounts.decode(
  "MarketLinear",
  accountInfo.data
);

// Floor price is a Rust Decimal
const floor = mayflowerData.floor;
console.log("New floor:", floor.toString());
```

The on-chain account is already updated by the time the transaction confirms, so the `floor` value is immediately available.

### 3. Read Anytime (polling)

You can read the current floor at any time by fetching the Market account — no need to wait for an event:

```typescript
const mayflowerData = await program.account.marketLinear.fetch(mayflowerMarketKey);
const currentFloor = mayflowerData.floor;
```

---

## What You Can Build

With these events you can build:
- **Trade history** — every buy/sell with amounts, prices, and fees
- **OHLC candles** — aggregate trades into time buckets for charting
- **Portfolio tracking** — track wallet positions across markets
- **Floor price history** — the floor value is included in every event
- **Volume metrics** — sum `cashIn`/`cashOut` per time period
- **Fee analytics** — track revenue splits per market
- **Borrow/lending activity** — track open debt, repayments, and collateral movement via `BorrowEvent`, `RepayEvent`, `DepositEvent`, and `WithdrawEvent`

---

## Links

- [Quick API Integration](./API.md) — if you just want to trade via REST API
- [On-Chain Program Docs](./PROGRAM.md) — full program instruction reference
- [IDL (JSON)](../idl/idl.json) · [IDL (TypeScript)](../idl/idl.ts)

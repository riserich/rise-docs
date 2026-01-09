# Indexing & Events

The Rise program emits events on every buy, sell, and borrow transaction. You can parse these from on-chain transaction logs to build your own indexer.

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
| `amount` | u64 | Amount borrowed (RAW) |
| `borrower` | PublicKey | Borrower's wallet address |
| `market` | PublicKey | Rise market address |
| `revSplit` | RevenueSplits | Fee breakdown |

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

## What You Can Build

With these events you can build:
- **Trade history** — every buy/sell with amounts, prices, and fees
- **OHLC candles** — aggregate trades into time buckets for charting
- **Portfolio tracking** — track wallet positions across markets
- **Floor price history** — the floor value is included in every event
- **Volume metrics** — sum `cashIn`/`cashOut` per time period
- **Fee analytics** — track revenue splits per market

---

## Links

- [Quick API Integration](./API.md) — if you just want to trade via REST API
- [On-Chain Program Docs](./PROGRAM.md) — full program instruction reference
- [IDL (JSON)](../idl/idl.json) · [IDL (TypeScript)](../idl/idl.ts)

# Indexing & Events

The Rise program emits events on every buy, sell, borrow, repay, deposit, and withdraw transaction. You can parse these from on-chain transactions to build your own indexer.

> **Note — events are emitted via Anchor `emit_cpi!`.** They are no longer in `Program data:` log lines. Instead each event payload is the data of an **inner self-CPI instruction** to the Rise program (one per emitted event), and is prefixed with the 8-byte tag `0xe4 0x45 0xa5 0x2e 0x51 0xcb 0x9a 0x1d` (`sha256("anchor:event")[..8]`) followed by the usual 8-byte event discriminator. To extract events, walk the transaction's `innerInstructions`, keep instructions whose `programIdIndex` points at the Rise program, strip the leading 8-byte CPI tag, and parse the remainder with the discriminators below.

Borrow, repay, deposit, and withdraw all emit **post-op snapshots** of the personal position and market aggregates rather than deltas. The actor's wallet (and the rise market) are not in the event payload — derive them from the enclosing instruction's accounts.

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
| `totalMainTokenInLiquidityPool` | u64 | Base-currency balance in the Mayflower liquidity pool after the trade (TVL proxy) |
| `totalMarketDebt` | u64 | Sum of debt across all positions on the Mayflower market after the trade |
| `tokenOut` | u64 | Exact tokens minted to the buyer (RAW). Use this directly as "tokens received" — no need to derive it from pre/post token balances |

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
| `totalMainTokenInLiquidityPool` | u64 | Base-currency balance in the Mayflower liquidity pool after the trade (TVL proxy) |
| `totalMarketDebt` | u64 | Sum of debt across all positions on the Mayflower market after the trade |

### BorrowEvent

Emitted on borrow transactions. Post-op snapshot plus the fee split from the borrow fee distribution.

| Field | Type | Description |
|-------|------|-------------|
| `depositedTokenBalance` | u64 | Collateral balance on the personal position after the op (RAW) |
| `debt` | u64 | Debt balance on the personal position after the op (RAW) |
| `totalMarketDebt` | u64 | Sum of debt across all positions on the Mayflower market |
| `totalMarketDepositedCollateral` | u64 | Sum of collateral across all positions on the Mayflower market |
| `totalMainTokenInLiquidityPool` | u64 | Base-currency balance in the Mayflower liquidity pool (TVL proxy) |
| `revSplit` | RevenueSplits | Distribution of the borrow fee |

The borrower's wallet and the rise market aren't in the payload — read them from the borrow instruction's accounts (`borrower` signer and `riseMarket`).

### RepayEvent

Emitted on repay transactions. Includes `positionOwner` because repay is permissionless — the tx signer may differ from the debtor.

| Field | Type | Description |
|-------|------|-------------|
| `positionOwner` | PublicKey | Debtor whose position the repay was applied to (taken from `core_personal_position.owner`, not the signer) |
| `depositedTokenBalance` | u64 | Collateral balance on the position after the op (RAW) |
| `debt` | u64 | Debt balance on the position after the op (RAW) |
| `totalMarketDebt` | u64 | Sum of debt across all positions on the Mayflower market |
| `totalMarketDepositedCollateral` | u64 | Sum of collateral across all positions on the Mayflower market |
| `totalMainTokenInLiquidityPool` | u64 | Base-currency balance in the Mayflower liquidity pool |

### LendingEvent

Emitted by **both** `deposit` and `withdraw` instructions. The two are distinguished by the enclosing instruction discriminator on the transaction. Every field is a full post-op snapshot, so an indexer can upsert without prior state (last-write-wins).

| Field | Type | Description |
|-------|------|-------------|
| `depositedTokenBalance` | u64 | Collateral balance on the position after the op (RAW) |
| `debt` | u64 | Debt balance on the position after the op (RAW) — unchanged by deposit/withdraw, emitted for a complete snapshot |
| `totalMarketDebt` | u64 | Sum of debt across all positions on the Mayflower market |
| `totalMarketDepositedCollateral` | u64 | Sum of collateral across all positions on the Mayflower market |
| `totalMainTokenInLiquidityPool` | u64 | Base-currency balance in the Mayflower liquidity pool |

The depositor/withdrawer and rise market aren't in the payload — read them from the enclosing instruction's accounts.

### LeverageBuyEvent

Emitted on every `leverageBuy` transaction. The instruction internally borrows, buys, and deposits as collateral in one atomic step — the event lets indexers record the full operation without re-deriving from inner CPIs.

| Field | Type | Description |
|-------|------|-------------|
| `buyer` | PublicKey | Buyer's wallet address |
| `market` | PublicKey | Rise market address |
| `exactCashIn` | u64 | User's own cash put into the buy (RAW) |
| `increaseDebtBy` | u64 | Cash borrowed from the lending pool for this trade (RAW) |
| `minIncreaseCollateralBy` | u64 | Slippage floor — minimum tokens that must arrive into the position |
| `actualIncreaseCollateralBy` | u64 | Tokens actually minted by the inner buy and added to the position (RAW). Use this as the user's "tokens received" for trade history / PnL — no need to derive from supply deltas |
| `revSplit` | RevenueSplits | Fee split applied during the leverage buy |
| `floor` | Decimal (u128) | Floor price after transaction |
| `tokenSupply` | u64 | Token supply after transaction |
| `m1` | Decimal (u128) | Bonding curve slope (shoulder segment) |
| `m2` | Decimal (u128) | Bonding curve slope (main segment) |
| `x2` | u64 | Supply transition point (shoulder → main) |
| `b2` | Decimal (u128) | Y-intercept for main segment |
| `totalMainTokenInLiquidityPool` | u64 | Base-currency balance in the Mayflower liquidity pool after the trade (TVL proxy) |
| `totalMarketDebt` | u64 | Sum of debt across all positions on the Mayflower market after the trade |
| `totalCollateral` | u64 | Sum of collateral across all positions on the Mayflower market after the trade |
| `mintToken` | PublicKey | Token mint address |
| `mintMain` | PublicKey | Collateral mint address (SOL or USDC) |
| `tokenDecimals` | u8 | Token decimal places |
| `owner` | PublicKey | Position owner (same pubkey as `buyer`) |
| `marketMeta` | PublicKey | Mayflower market metadata address for the position |
| `depositedTokenBalance` | u64 | Collateral balance on the buyer's position after the op (RAW) |
| `debt` | u64 | Debt balance on the buyer's position after the op (RAW) |
| `escrow` | PublicKey | Escrow token account holding the position's collateral |

Fields from `floor` onward are a **post-transaction snapshot** of the market curve and the buyer's position, appended so an indexer can record the full market + position state from the event alone — no RPC fetch of the Mayflower market / position accounts needed.

Total cash that flowed into the curve = `exactCashIn + increaseDebtBy` (less the buy fee and borrow fee). Multiplier = `(exactCashIn + increaseDebtBy) / exactCashIn`; when `exactCashIn = 0` the trade is fully leveraged.

### LeverageSellEvent

Emitted on every `leverageSell` transaction. The instruction withdraws collateral, sells it, repays debt, and routes the remaining cash to the seller — all atomically.

| Field | Type | Description |
|-------|------|-------------|
| `seller` | PublicKey | Seller's wallet address |
| `market` | PublicKey | Rise market address |
| `decreaseCollateralBy` | u64 | Tokens removed from the position and sold (RAW) |
| `decreaseDebtBy` | u64 | Debt repaid out of the sale proceeds (RAW) |
| `minCashToUser` | u64 | Slippage floor — minimum cash that must land in the seller's wallet |
| `actualCashToUser` | u64 | Cash that actually landed in the seller's wallet after sell fee + debt repayment (RAW). Use this as the "amount received" for trade history / PnL |
| `revSplit` | RevenueSplits | Fee split applied during the leverage sell |
| `floor` | Decimal (u128) | Floor price after transaction |
| `tokenSupply` | u64 | Token supply after transaction |
| `m1` | Decimal (u128) | Bonding curve slope (shoulder segment) |
| `m2` | Decimal (u128) | Bonding curve slope (main segment) |
| `x2` | u64 | Supply transition point (shoulder → main) |
| `b2` | Decimal (u128) | Y-intercept for main segment |
| `totalMainTokenInLiquidityPool` | u64 | Base-currency balance in the Mayflower liquidity pool after the trade (TVL proxy) |
| `totalMarketDebt` | u64 | Sum of debt across all positions on the Mayflower market after the trade |
| `totalCollateral` | u64 | Sum of collateral across all positions on the Mayflower market after the trade |
| `mintToken` | PublicKey | Token mint address |
| `mintMain` | PublicKey | Collateral mint address (SOL or USDC) |
| `tokenDecimals` | u8 | Token decimal places |
| `owner` | PublicKey | Position owner (same pubkey as `seller`) |
| `marketMeta` | PublicKey | Mayflower market metadata address for the position |
| `depositedTokenBalance` | u64 | Collateral balance on the seller's position after the op (RAW) |
| `debt` | u64 | Debt balance on the seller's position after the op (RAW) |
| `escrow` | PublicKey | Escrow token account holding the position's collateral |

Fields from `floor` onward are a **post-transaction snapshot** of the market curve and the seller's position, appended so an indexer can record the full market + position state from the event alone — no RPC fetch needed. On a full close (`depositedTokenBalance` and `debt` both `0`) the position may be closed on-chain; the snapshot still carries the final zeroed state.

Deleverage percentage = `decreaseDebtBy / pre_tx_debt × 100` (read `pre_tx_debt` from your own state — it's not on the event).

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
const LENDING_DISC = eventDiscriminator("LendingEvent"); // deposit + withdraw
const LEVERAGE_BUY_DISC = eventDiscriminator("LeverageBuyEvent");   // 401-byte content (120 base + 281 snapshot)
const LEVERAGE_SELL_DISC = eventDiscriminator("LeverageSellEvent"); // 401-byte content (120 base + 281 snapshot)
```

`LendingEvent` (40 bytes) is just five `u64` snapshot fields. `BorrowEvent` (64 bytes) appends a `RevenueSplits` (3×u64). `RepayEvent` (72 bytes) prepends a 32-byte `positionOwner` pubkey:

```typescript
function parseLendingSnapshot(data: Buffer, offset: number) {
  return {
    depositedTokenBalance:        data.readBigUInt64LE(offset),
    debt:                         data.readBigUInt64LE(offset + 8),
    totalMarketDebt:              data.readBigUInt64LE(offset + 16),
    totalMarketDepositedCollateral: data.readBigUInt64LE(offset + 24),
    totalMainTokenInLiquidityPool: data.readBigUInt64LE(offset + 32),
  };
}

function parseLendingEvent(data: Buffer) {
  if (!data.slice(0, 8).equals(LENDING_DISC)) return null;
  return parseLendingSnapshot(data, 8);
}

function parseBorrowEvent(data: Buffer) {
  if (!data.slice(0, 8).equals(BORROW_DISC)) return null;
  const snapshot = parseLendingSnapshot(data, 8);
  const revFloor   = data.readBigUInt64LE(48);
  const revCreator = data.readBigUInt64LE(56);
  const revTeam    = data.readBigUInt64LE(64);
  return { ...snapshot, revSplit: { floor: revFloor, creator: revCreator, team: revTeam } };
}

function parseRepayEvent(data: Buffer) {
  if (!data.slice(0, 8).equals(REPAY_DISC)) return null;
  const positionOwner = new PublicKey(data.slice(8, 40));
  return { positionOwner, ...parseLendingSnapshot(data, 40) };
}

// LeverageBuyEvent / LeverageSellEvent share the same layout: a 120-byte base
// (discriminator already consumed) followed by a 281-byte post-tx snapshot.
//   base:     pubkey(32) + pubkey(32) + u64(8)×4 + revSplit(24)            = 120
//   snapshot: floor(16) + tokenSupply(8) + m1(16) + m2(16) + x2(8) + b2(16)
//           + totalMainTokenInLiquidityPool(8) + totalMarketDebt(8) + totalCollateral(8)
//           + mintToken(32) + mintMain(32) + tokenDecimals(1)
//           + owner(32) + marketMeta(32) + depositedTokenBalance(8) + debt(8) + escrow(32) = 281
// The snapshot is parsed only when the payload is long enough (401 content
// bytes after the discriminator); shorter pre-upgrade events return snapshot:null.
function parseLeverageSnapshot(data: Buffer, offset: number) {
  let o = offset;
  const floor = data.slice(o, o + 16);                                  o += 16;
  const tokenSupply = data.readBigUInt64LE(o);                          o += 8;
  const m1 = data.slice(o, o + 16);                                     o += 16;
  const m2 = data.slice(o, o + 16);                                     o += 16;
  const x2 = data.readBigUInt64LE(o);                                   o += 8;
  const b2 = data.slice(o, o + 16);                                     o += 16;
  const totalMainTokenInLiquidityPool = data.readBigUInt64LE(o);        o += 8;
  const totalMarketDebt = data.readBigUInt64LE(o);                      o += 8;
  const totalCollateral = data.readBigUInt64LE(o);                      o += 8;
  const mintToken = new PublicKey(data.slice(o, o + 32));               o += 32;
  const mintMain = new PublicKey(data.slice(o, o + 32));                o += 32;
  const tokenDecimals = data.readUInt8(o);                              o += 1;
  const owner = new PublicKey(data.slice(o, o + 32));                   o += 32;
  const marketMeta = new PublicKey(data.slice(o, o + 32));              o += 32;
  const depositedTokenBalance = data.readBigUInt64LE(o);               o += 8;
  const debt = data.readBigUInt64LE(o);                                 o += 8;
  const escrow = new PublicKey(data.slice(o, o + 32));
  return {
    floor, tokenSupply, m1, m2, x2, b2,
    totalMainTokenInLiquidityPool, totalMarketDebt, totalCollateral,
    mintToken, mintMain, tokenDecimals,
    owner, marketMeta, depositedTokenBalance, debt, escrow,
  };
}

function parseLeverageBuyEvent(data: Buffer) {
  if (!data.slice(0, 8).equals(LEVERAGE_BUY_DISC)) return null;
  const buyer = new PublicKey(data.slice(8, 40));
  const market = new PublicKey(data.slice(40, 72));
  // base is 120 bytes of content (after the 8-byte disc → ends at offset 128)
  const snapshot = data.length >= 8 + 401 ? parseLeverageSnapshot(data, 128) : null;
  return {
    buyer,
    market,
    exactCashIn:               data.readBigUInt64LE(72),
    increaseDebtBy:            data.readBigUInt64LE(80),
    minIncreaseCollateralBy:   data.readBigUInt64LE(88),
    actualIncreaseCollateralBy: data.readBigUInt64LE(96),
    revSplit: {
      floor:   data.readBigUInt64LE(104),
      creator: data.readBigUInt64LE(112),
      team:    data.readBigUInt64LE(120),
    },
    snapshot,
  };
}

function parseLeverageSellEvent(data: Buffer) {
  if (!data.slice(0, 8).equals(LEVERAGE_SELL_DISC)) return null;
  const seller = new PublicKey(data.slice(8, 40));
  const market = new PublicKey(data.slice(40, 72));
  const snapshot = data.length >= 8 + 401 ? parseLeverageSnapshot(data, 128) : null;
  return {
    seller,
    market,
    decreaseCollateralBy: data.readBigUInt64LE(72),
    decreaseDebtBy:       data.readBigUInt64LE(80),
    minCashToUser:        data.readBigUInt64LE(88),
    actualCashToUser:     data.readBigUInt64LE(96),
    revSplit: {
      floor:   data.readBigUInt64LE(104),
      creator: data.readBigUInt64LE(112),
      team:    data.readBigUInt64LE(120),
    },
    snapshot,
  };
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
  const totalMainTokenInLiquidityPool = data.readBigUInt64LE(offset);  offset += 8;
  const totalMarketDebt = data.readBigUInt64LE(offset);               offset += 8;
  // tokenOut is a trailing field — present only when the payload is long
  // enough (older events stop at totalMarketDebt).
  const tokenOut = data.length >= offset + 8 ? data.readBigUInt64LE(offset) : null;

  return {
    buyer, market, cashIn, minTokenOut,
    revSplit: { floor: revFloor, creator: revCreator, team: revTeam },
    floor, tokenSupply, m1, m2, x2, b2,
    lastFloorRaiseTimestamp, mintToken, mintMain, tokenDecimals,
    totalMainTokenInLiquidityPool, totalMarketDebt, tokenOut,
  };
}
```

### Computing Price from Event Data

After parsing an event, you can compute the current token price using the bonding curve parameters:

The bonding curve has three regions: a flat floor, a shoulder, and the main line.

```typescript
function calculatePrice(
  tokenSupply: number,  // human-readable (RAW / 10^decimals)
  floor: number,
  m1: number,           // slope * decimalsFactor
  m2: number,           // slope * decimalsFactor
  x2: number,           // human-readable
  b2: number,
): number {
  const b1 = (m2 - m1) * x2 + b2;
  const x1 = Math.abs(m1) < 1e-30 ? Infinity : (floor - b1) / m1;

  if (tokenSupply <= x1) return floor;          // floor region
  if (tokenSupply <= x2) return m1 * tokenSupply + b1; // shoulder region
  return m2 * tokenSupply + b2;                 // main region
}
```

This matches the `calculatePrice` function in the Rise SDK (`quote.ts`) and the backend exactly — the backend is the source of truth for all prices served by the API.

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

The Rise program has two floor raise methods (`raiseFloorPreserveArea` and `raiseFloorExcessLiquidity`). Each emits its own event below (which only carries the increase ratio, not the resulting floor). In addition, **both now also emit a `RaiseFloorCurveEvent`** that carries the full post-raise curve — so you can read the new floor (and the rest of the curve) straight from the event, with no on-chain fetch.

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

### RaiseFloorCurveEvent

Emitted via `emit_cpi` by **both** raise-floor instructions, in addition to the event above. Carries the full post-raise curve snapshot, so indexers can read the new floor and curve **directly from the event** — no need to RPC-fetch the Mayflower market account after a floor raise.

| Field | Type | Description |
|-------|------|-------------|
| `market` | PublicKey | Rise market address |
| `floor` | Decimal (16 bytes) | New floor price (post-raise) |
| `tokenSupply` | u64 | Token supply |
| `m1` | Decimal (16 bytes) | Shoulder slope (× decimalsFactor) |
| `m2` | Decimal (16 bytes) | Main slope (× decimalsFactor) |
| `x2` | u64 | Shoulder → main transition (raw token units) |
| `b2` | Decimal (16 bytes) | Main-segment y-intercept |
| `totalMainTokenInLiquidityPool` | u64 | Cash in the liquidity pool (raw) |
| `totalMarketDebt` | u64 | Total market debt (raw) |
| `totalCollateral` | u64 | Total deposited collateral (raw) |
| `mintToken` | PublicKey | Market token mint |
| `mintMain` | PublicKey | Collateral mint (SOL / USDC) |
| `tokenDecimals` | u8 | Token decimals |

All values are **post-raise** snapshots. Feed `floor`, `m1`, `m2`, `x2`, `b2`, `tokenSupply` into [`calculatePrice`](#computing-price-from-event-data) to get the new price.

> The existing `RaiseFloorEvent` / `RaiseFloorExcessLiquidityEvent` are unchanged, so any current log-based indexing keeps working — `RaiseFloorCurveEvent` is additive.
>
> Both raise-floor instructions now also take 2 extra accounts (`event_authority`, `program`) required by `emit_cpi`. Only the protocol keeper calls these instructions.

### Event Discriminators

```typescript
const RAISE_FLOOR_DISC = eventDiscriminator("RaiseFloorEvent");
const RAISE_FLOOR_EXCESS_DISC = eventDiscriminator("RaiseFloorExcessLiquidityEvent");
const RAISE_FLOOR_CURVE_DISC = eventDiscriminator("RaiseFloorCurveEvent");
```

---

## Tracking the Floor Price

The floor price can be tracked in two ways:

### 1. From Buy/Sell Events (real-time)

Every `BuyWithExactCashInEvent` and `SellWithExactTokenInEvent` includes the current `floor` field. This is the easiest way — just read it from every trade event.

### 2. After Floor Raise Instructions

The easiest way is to read `RaiseFloorCurveEvent` (emitted by both raise-floor instructions) — its `floor` field is the new post-raise floor, no fetch required.

If you'd rather not parse that event, you can fetch the Mayflower market account (account index 5) after detecting a `raiseFloorPreserveArea` or `raiseFloorExcessLiquidity` instruction to read the updated floor:

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
- **Borrow/lending activity** — track open debt, repayments, and collateral movement via `BorrowEvent`, `RepayEvent`, and `LendingEvent` (deposit + withdraw share the same event)
- **Leverage trading** — `LeverageBuyEvent` and `LeverageSellEvent` give you the exact own-cash, debt, and tokens/cash actually transacted on a single atomic leverage step. Combine with a wallet's pre-tx debt to derive leverage multiplier or deleverage percentage.

---

## Links

- [Quick API Integration](./API.md) — if you just want to trade via REST API
- [On-Chain Program Docs](./PROGRAM.md) — full program instruction reference
- [IDL (JSON)](../idl/idl.json) · [IDL (TypeScript)](../idl/idl.ts)

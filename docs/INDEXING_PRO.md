# Rise Pro (v2) — Indexing & Events

| | |
|---|---|
| **Program (Devnet)** | `9nJ9wghEHY8bnsJQYrqV77uC9y4ccFHcCT1XyJxAkBVR` |
| **IDL (JSON)** | [`../idl/rise_pro.json`](../idl/rise_pro.json) |
| **Program surface** | [Program (Pro)](./PROGRAM_PRO.md) |

Every state change emits an event — trades, lending, leverage, floor raises, reflection, lifecycle. Parse them from transactions to build an indexer; no account re-reads needed.

---

## Decoding events

Events are emitted via Anchor **`emit_cpi!`** — each one is the data of an **inner self-CPI instruction** back to the program (never a `Program data:` log line, so nothing is lost to log truncation; block path and gRPC see identical data).

```
inner ix data = EVENT_CPI_DISC (8) ++ eventDiscriminator (8) ++ borsh(fields)

EVENT_CPI_DISC     = e4 45 a5 2e 51 cb 9a 1d          // sha256("anchor:event")[..8]
eventDiscriminator = sha256("event:<EventName>")[..8]  // table below
```

Recipe:
1. Skip txs with `meta.err` or without the program id.
2. Account keys = `[staticAccountKeys, loadedAddresses.writable, loadedAddresses.readonly]`.
3. For each inner instruction owned by the program with `data[0..8] == EVENT_CPI_DISC`: match `data[8..16]` in the table below, Borsh-decode the rest in struct order.

## Reading rules

| Rule | Detail |
|---|---|
| **`[post]` fields are snapshots** | `floor_price`, `token_supply`, `floor_reserve`, `price_reserve`, `total_debt`, `total_collateral`, `cum_net_inflow` and the geometry memos (`x1`, `x2`, `m_x1`, `y_shift`) are the market state **after** the tx — never read the `Market` account back. |
| **Payloads are self-contained** | `market` + actor (`buyer`/`seller`/`owner`/`creator`) are in every event. Instruction accounts are only needed for the position escrow (`position`, `collateral_vault`) and `init_market`'s vault pubkeys. |
| **`state_seq` orders everything** | Monotone per market. Apply state in `state_seq` order; drop any event ≤ the last one stored (keep its history row). |
| **Missing trailing bytes = 0 / empty** | Events append fields over time (`book_shortfall`, `y_shift`, `heal_spread` on sells; `reflection_bps` on init; `entries` on distribution). |

## Price-feed essentials

Index these first — the rest is enrichment:

| Attribute | Where | Use |
|---|---|---|
| `stateSeq` | every event | ordering / dedup key |
| `market` + actor | every event | row identity, per-user history |
| `tokenOut` · `tokenIn`, `gross`, `payout` | Buy / Sell | swap legs (RAW). Buy price = `(cashIn − fee − reflection) / tokenOut`; sell price = `gross / tokenIn` — decimals cancel (v2: token decimals == collateral decimals) |
| `cashIn`, `fee` | Buy / Sell | volume + exact per-swap protocol fee |
| `tokenSupply` **[post]** | trades | market cap; the curve's `x` |
| `floorReserve` / `priceReserve` **[post]** | trades | TVL / backing |
| `floor_price` | `MarketInitializedEvent` → `FloorRaisedEvent.floor` | the guaranteed floor — **not on trade events**; monotone, only rises |
| `char_params`, `ramp_scalar`, `hingeX`, `curveKind` | init args + event | curve shape, needed for exact spot quotes (see bootstrap) |
| `reflectionBps` | `MarketInitializedEvent` | apply the 0/1/3 % leg when pricing, or fills are overstated by that % |

Everything else (`glideCredit`, `x2`, `yShift`, `mX1`, `bookShortfall`, `healSpread`, `cumNetInflow`…) is curve-internal — needed only to reimplement the overlay or self-healing.

## Discriminators

| Event | Discriminator (hex) |
|---|---|
| `AdminAcceptedEvent` | `0e dd 3c 5c f1 93 5f 42` |
| `AdminProposedEvent` | `d4 a3 5b 1c df 5f 02 66` |
| `BorrowEvent` | `56 08 8c ce d7 b3 76 c9` |
| `BuyEvent` | `67 f4 52 1f 2c f5 77 77` |
| `CollateralConfigSetEvent` | `3d 2e b8 69 8b 89 37 70` |
| `ConfigInitializedEvent` | `16 a7 c0 32 dc 14 0a 47` |
| `CreatorFeesClaimedEvent` | `28 89 c8 28 9a 85 ea fb` |
| `DepositEvent` | `78 f8 3d 53 1f 8e 6b 90` |
| `FeeConfigPublishedEvent` | `36 79 e2 e6 ed ca 8b b6` |
| `FloorRaisedEvent` | `b4 96 93 fc ee 93 ae ec` |
| `LeverageBuyEvent` | `c0 fb f5 42 95 57 4f c3` |
| `LeverageSellEvent` | `53 cc d2 60 ce d7 0c 9a` |
| `MarketFlagsSetEvent` | `3a fe df 16 c3 74 86 72` |
| `MarketInitializedEvent` | `46 ad 60 ca 64 8f 2d 19` |
| `MarketUpdatedEvent` | `38 c5 ea c2 f4 7d b5 da` |
| `ReflectionAccruedEvent` | `58 62 57 c3 19 e1 80 95` |
| `ReflectionDistributedEvent` | `67 86 50 39 06 28 d9 ec` |
| `ReflectionEpochClosedEvent` | `ab e2 bb af 4b ae f4 8b` |
| `ReflectionEpochOpenedEvent` | `9d e1 bb 66 b0 a7 59 eb` |
| `RepayEvent` | `81 d5 00 6c da 6c 52 8c` |
| `SellEvent` | `3e 2f 37 0a a5 03 dc 2a` |
| `TeamFeesWithdrawnEvent` | `3e 62 7b 98 b2 7a e6 7f` |
| `WithdrawEvent` | `16 09 85 1a a0 2c 47 c0` |

---

## Event index

| Event | Emitted by | What to update |
|---|---|---|
| `BuyEvent` / `SellEvent` | `buy` / `sell` | tx row, candles, supply/reserves, spot price |
| `LeverageBuyEvent` / `LeverageSellEvent` | `leverage_buy` / `leverage_sell` | tx row, position (collateral+debt), candles |
| `DepositEvent` / `WithdrawEvent` | `deposit` / `withdraw` | position collateral, market totals |
| `BorrowEvent` / `RepayEvent` | `borrow` / `repay` | position debt, reserves, market totals |
| `FloorRaisedEvent` | both raise instructions | floor, x1/x2 geometry, candles |
| `MarketInitializedEvent` | `init_market` | create the market row |
| `MarketFlagsSetEvent` / `MarketUpdatedEvent` | admin | flags / creator & fee splits |
| `CreatorFeesClaimedEvent` / `TeamFeesWithdrawnEvent` | claims | fee counters |
| `ReflectionAccruedEvent` | taxed trades | reflection pool accrual |
| `ReflectionEpochOpenedEvent` / `ReflectionDistributedEvent` | reflection epoch | holder reward rows |
| `ReflectionEpochClosedEvent` | `close_reflection_epoch` | epoch settled; `pool − total_distributed` was not paid out |
| `ConfigInitializedEvent` / `AdminProposedEvent` / `AdminAcceptedEvent` / `FeeConfigPublishedEvent` / `CollateralConfigSetEvent` | global admin | usually ignorable by market indexers |

Ordering across all of them: apply state writes only when the event's `stateSeq` is greater than the last one you stored for that scope — stale events keep their history rows but must not touch state.

---

## Start here: bootstrap a market from `init_market`

Index `init_market` **first** — it creates the market row every other event updates. A market's data is split across two places in that one transaction:

| Source | Carries |
|---|---|
| **`MarketInitializedEvent`** (the emitted event) | `floorPrice` (starting floor, Q64.64), `curveKind`, `hingeX`, fee rates + split, `flags`, `name`/`symbol`/`uri`, `reflectionBps`, the mints |
| **`init_market` instruction args** (`InitMarketArgs`, Borsh-decoded from the outer instruction data) | the **curve shape**: `char_params = [p0, m, g2, k]`, `ramp_scalar` (s), `q_max_bps`, `g0_mbps`/`g_min_mbps`/`glide_end_inflow`, `self_healing`, `disable_sell` |

> **Why both?** `char_params` (the p0/m/g2/k that define the curve shape) are **not** in the event — they live only in the instruction args. To price the curve at an arbitrary supply you need them, so decode the instruction data too. Everything else you need for a market row is in the event.

**`InitMarketArgs`** (Borsh order):

```
vanity_seed: u64
metadata: { name: string, symbol: string, uri: string }
curve_kind: u8                 // 0 = linear, 1 = hinged-exponential
hinge_x: u64
char_params: [i128; 4]         // [p0, m, g2, k]  — p0/g2 Q64.64, m/k Q64.96
ramp_scalar: u128              // s, Q64.64 (2.0 = 2<<64)
fees_creator: u16
g0_mbps: u32
g_min_mbps: u32
glide_end_inflow: u128
q_max_bps: u16
disable_sell: bool
self_healing: bool
reflection_bps: u16
```

From then on: `MarketInitializedEvent.floorPrice` is the starting floor, updated by every `FloorRaisedEvent.floor`; `tokenSupply` / `floorReserve` / `priceReserve` are refreshed from the `[post]` fields on every trade event. You never need to read the `Market` account back.

---

## Trading events

### BuyEvent

Emitted on every `buy`.

| Field | Type | Description |
|---|---|---|
| `market` | Pubkey | Rise market address |
| `buyer` | Pubkey | Buyer's wallet |
| `cashIn` | u64 | Gross collateral spent (RAW) — includes fee and reflection |
| `fee` | u64 | Total trade fee for this swap (RAW) |
| `feeFloor` | u64 | Floor leg of the fee |
| `feeCreator` | u64 | Creator leg of the fee |
| `feeTeam` | u64 | Team leg of the fee |
| `tokenOut` | u64 | Exact tokens minted to the buyer (RAW). Use directly as "tokens received". |
| `obligation` | u64 | `ceil(floor × tokenOut)` — the Law-1 leg credited to the floor |
| `glideCredit` | u64 | Portion of the premium routed to the floor via glide |
| `priceCredit` | u64 | Portion of the premium kept in the exit book |
| `tokenSupply` | u64 | Supply after the trade **[post]** |
| `floorReserve` | u64 | Floor ledger after the trade **[post]** |
| `priceReserve` | u64 | Exit-book ledger after the trade **[post]** |
| `stateSeq` | u64 | Monotone write counter |
| `cumNetInflow` | u128 | Cumulative net inflow after the trade **[post]** |

> **Price from a buy:** the curve-effective cash is `cashIn − fee − reflection`; the simplest exact average price is `(cashIn − fee − reflection) / tokenOut`. The reflection amount is **not** on this event — see [Reflection events](#reflection-events). On a 0 % reflection market it is simply `(cashIn − fee) / tokenOut`.

### SellEvent

Emitted on every `sell`.

| Field | Type | Description |
|---|---|---|
| `market` | Pubkey | Rise market address |
| `seller` | Pubkey | Seller's wallet |
| `tokenIn` | u64 | Tokens sold (RAW) |
| `gross` | u64 | Curve value of the sale before the fee (RAW), post-clamp |
| `fee` | u64 | Total trade fee |
| `feeFloor` | u64 | Floor leg |
| `feeCreator` | u64 | Creator leg |
| `feeTeam` | u64 | Team leg |
| `payout` | u64 | Collateral paid to the seller (RAW) — net of fee **and** reflection |
| `floorDebit` | u64 | Amount drawn from the floor ledger |
| `priceDebit` | u64 | Amount drawn from the exit book |
| `tokenSupply` | u64 | Supply after **[post]** |
| `floorReserve` | u64 | Floor ledger after **[post]** |
| `priceReserve` | u64 | Exit book after **[post]** |
| `contracted` | bool | True if the sale landed in the flat band and pulled `x1` down |
| `stateSeq` | u64 | Write counter |
| `cumNetInflow` | u128 | After **[post]** |
| `rampStartX` | u64 | `x1` after **[post geometry memo]** |
| `x2` | u64 | `x2` after **[post]** (`u64::MAX` = sentinel) |
| `mX1` | u128 | `M̃(x1)` after **[post, Q64.64]** |
| `bookShortfall` | u64 | Curve value the ledgers couldn't cover (`0` in the normal case) |
| `yShift` | i128 | Curve translation after **[post, Q64.64]** (`0` on legacy markets) |
| `healSpread` | u64 | Amount the shoulder confiscated into the floor on this sale |

### LeverageBuyEvent

| Field | Type | Description |
|---|---|---|
| `market`, `owner` | Pubkey | Market, position owner |
| `cashIn` | u64 | Owner's own cash leg (RAW) |
| `borrowed` | u64 | Amount borrowed (RAW) |
| `borrowFee`, `buyFee` | u64 | Fee on the borrow leg / the buy leg |
| `tokenOut` | u64 | Tokens minted and locked as collateral (RAW) |
| `collateral` | u64 | Position collateral after **[post]** |
| `debt` | u64 | Position debt after **[post]** |
| `drawnFromFloor` | u64 | Floor-rule draw |
| `tokenSupply` | u64 | Supply after **[post]** |
| `stateSeq` | u64 | Write counter |
| `floorReserve`, `priceReserve` | u64 | Ledgers after **[post]** |
| `totalDebt`, `totalCollateral` | u64 | Market aggregates after **[post]** |
| `cumNetInflow` | u128 | After **[post]** |

### LeverageSellEvent

| Field | Type | Description |
|---|---|---|
| `market`, `owner` | Pubkey | Market, position owner |
| `collateralIn` | u64 | Collateral tokens sold (RAW) |
| `gross` | u64 | Curve value before fee |
| `fee` | u64 | Total fee |
| `repaid` | u64 | Debt repaid from proceeds |
| `paidOut` | u64 | Cash paid to the owner (net of fee, reflection, repayment) |
| `collateral`, `debt` | u64 | Position after **[post]** |
| `tokenSupply` | u64 | Supply after **[post]** |
| `contracted` | bool | Flat-band landing |
| `stateSeq` | u64 | Write counter |
| `floorReserve`, `priceReserve` | u64 | Ledgers after **[post]** |
| `totalDebt`, `totalCollateral` | u64 | Aggregates after **[post]** |
| `cumNetInflow` | u128 | After **[post]** |
| `rampStartX`, `x2`, `mX1` | u64/u64/u128 | Geometry memos after **[post]** |
| `bookShortfall` | u64 | `0` normal |
| `yShift` | i128 | After **[post]** (`0` legacy) |
| `healSpread` | u64 | Shoulder spread confiscated into floor |

---

## Lending events

The actor wallet (`owner`) and `market` are in the payload; the **position PDA** and `collateral_vault` are not — read them from the enclosing instruction's accounts if you need the per-position escrow address.

### DepositEvent
Discriminator: `[120, 248, 61, 83, 31, 142, 107, 144]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `owner` | pubkey |  |
| `amount` | u64 |  |
| `collateral` | u64 |  |
| `state_seq` | u64 |  |
| `total_collateral` | u64 | Market-wide post-state, for the indexer's aggregate columns. |

### BorrowEvent
Discriminator: `[86, 8, 140, 206, 215, 179, 118, 201]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `owner` | pubkey |  |
| `amount` | u64 |  |
| `fee` | u64 |  |
| `fee_floor` | u64 |  |
| `fee_creator` | u64 |  |
| `fee_team` | u64 |  |
| `received` | u64 |  |
| `debt` | u64 |  |
| `drawn_from_floor` | u64 | How much of the draw came out of the floor earmark — the Mayflower rule in one number, matched one-for-one by the … |
| `state_seq` | u64 |  |
| `floor_reserve` | u64 | Market-wide post-state, for the indexer's aggregate columns. |
| `price_reserve` | u64 |  |
| `total_debt` | u64 |  |

### RepayEvent
Discriminator: `[129, 213, 0, 108, 218, 108, 82, 140]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `owner` | pubkey |  |
| `amount` | u64 |  |
| `debt` | u64 |  |
| `state_seq` | u64 |  |
| `floor_reserve` | u64 | Market-wide post-state, for the indexer's aggregate columns. |
| `price_reserve` | u64 |  |
| `total_debt` | u64 |  |

### WithdrawEvent
Discriminator: `[22, 9, 133, 26, 160, 44, 71, 192]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `owner` | pubkey |  |
| `amount` | u64 |  |
| `collateral` | u64 |  |
| `state_seq` | u64 |  |
| `total_collateral` | u64 | Market-wide post-state, for the indexer's aggregate columns. |

> **Guaranteed close:** a `leverage_sell` that fully exits emits only `LeverageSellEvent` — no separate Repay/Withdraw events for its internal legs. Don't double-count.

---

## Floor raise event

### FloorRaisedEvent
| Field | Type | Description |
|---|---|---|
| `market` | Pubkey | |
| `preserveArea` | bool | `true` = area-preserving path, `false` = cash-funded excess-liquidity |
| `previousFloor` | u128 | Floor before **[Q64.64]** |
| `floor` | u128 | Floor after **[Q64.64, post]** |
| `rampStartX` | u64 | `x1` after **[post]** |
| `previousX2` | u64 | `x2` before |
| `x2` | u64 | `x2` after **[post]** (`u64::MAX` = sentinel) |
| `stateSeq` | u64 | Write counter |
| `mX1` | u128 | `M̃(x1)` after **[post, Q64.64]** |

---

## Market lifecycle events

### MarketInitializedEvent
Discriminator: `[70, 173, 96, 202, 100, 143, 45, 25]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `creator` | pubkey |  |
| `mint_token` | pubkey |  |
| `mint_main` | pubkey |  |
| `name` | string | Full strings, not a hash: the indexer populates name/symbol/uri straight from the event, so nothing has to be fetched … |
| `symbol` | string |  |
| `uri` | string |  |
| `decimals` | u8 |  |
| `fee_config_version` | u32 |  |
| `fees_team` | u16 |  |
| `fees_creator` | u16 |  |
| `fees_floor` | u16 |  |
| `floor_price` | u128 |  |
| `curve_kind` | u8 |  |
| `flags` | u16 |  |
| `state_seq` | u64 |  |
| `buy_fee_mbps` | u32 | Trade-fee rates snapshotted from the FeeConfig — in the event so the indexer never needs the FeeConfig account (or a DB … |
| `sell_fee_mbps` | u32 |  |
| `borrow_fee_mbps` | u32 |  |
| `redeem_fee_mbps` | u32 |  |
| `hinge_x` | u64 | Hinge of the hinged-exponential family (0 for linear). Appended last, per the prefix-compatibility rule above. |
| `reflection_bps` | u16 | Reflection tax pinned at creation (bps of the trade): 0 / 100 / 300. |

> `hingeX` and `reflectionBps` are appended fields — decode them as `0` if the payload is short (older markets).

### MarketFlagsSetEvent
Discriminator: `[58, 254, 223, 22, 195, 116, 134, 114]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `previous_flags` | u16 |  |
| `flags` | u16 |  |
| `state_seq` | u64 |  |

### MarketUpdatedEvent
Discriminator: `[56, 197, 234, 194, 244, 125, 181, 218]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `previous_creator` | pubkey |  |
| `creator` | pubkey |  |
| `fees_creator` | u16 |  |
| `fees_floor` | u16 |  |
| `metadata_updated` | bool |  |
| `state_seq` | u64 |  |

### CreatorFeesClaimedEvent
Discriminator: `[40, 137, 200, 40, 154, 133, 234, 251]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `creator` | pubkey | Whoever held `market.creator` at claim time — not necessarily whoever earned it. |
| `amount` | u64 |  |
| `destination` | pubkey |  |

### TeamFeesWithdrawnEvent
Discriminator: `[62, 98, 123, 152, 178, 122, 230, 127]`

| Field | Type | Description |
|---|---|---|
| `mint_main` | pubkey |  |
| `amount` | u64 |  |
| `destination` | pubkey |  |

---

## Reflection events

### ReflectionAccruedEvent
Emitted on every trade with a non-zero reflection tax. Discriminator: `[88, 98, 87, 195, 25, 225, 128, 149]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `is_buy` | bool | `true` = buy-side (buyer → vault), `false` = sell-side (vault → vault). |
| `amount` | u64 |  |
| `state_seq` | u64 |  |

### ReflectionEpochOpenedEvent
Discriminator: `[157, 225, 187, 102, 176, 167, 89, 235]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `epoch` | u64 |  |
| `pool` | u64 | Distributable pool frozen at open (already net of `gas_reserve`). |
| `holder_count` | u32 |  |
| `gas_reserve` | u64 |  |

### ReflectionDistributedEvent
Discriminator: `[103, 134, 80, 57, 6, 40, 217, 236]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `epoch` | u64 |  |
| `count` | u32 |  |
| `batch_total` | u64 |  |
| `total_distributed` | u64 |  |
| `entries` | Vec<ReflectionPayment> |  |

`entries[].owner` is always the holder **wallet**, never the ATA. A recipient the runtime would reject (native wallet left below rent-exempt, closed or frozen token account) is skipped: left unpaid and absent from `entries`. The per-trade reflection amount lives on `ReflectionAccruedEvent`, **not** on the Buy/Sell event — join by `(market, stateSeq)` if you want the reflection leg of a specific swap.

### ReflectionEpochClosedEvent
Discriminator: `[171, 226, 187, 175, 75, 174, 244, 139]`

| Field | Type | Description |
|---|---|---|
| `market` | pubkey |  |
| `epoch` | u64 |  |
| `pool` | u64 | Distributable pool frozen at open |
| `total_distributed` | u64 | Amount actually paid out |

---

## Computing prices & market cap

All amounts in events are **raw** (smallest units). Because v2 enforces `tokenDecimals == collateralDecimals`, a raw-collateral ÷ raw-token ratio is **already** the whole-unit price — the decimal factors cancel. That makes trade prices trivial to compute, no curve maths required.

### Q64.64 → decimal

`floor_price`, `char_params` `p0`, `m_x1`, spot are **Q64.64**: real value = `q64 / 2^64` (collateral per token). Do it in BigInt:

```ts
const Q64 = 2n ** 64n;

/** Q64.64 fixed-point → whole collateral-per-token price. */
function q64ToPrice(q64: bigint, digits = 12): number {
  const scale = 10n ** BigInt(digits);
  return Number((q64 * scale) / Q64) / Number(scale);
}
```

### Average price of a trade (from the event — exact, no curve)

```ts
/** Average fill price of a buy, in whole collateral per whole token.
 *  `reflection` is 0 on a non-reflection market, else the amount from the
 *  matching ReflectionAccruedEvent (join on market + stateSeq). */
function buyPrice(ev: BuyEvent, reflection = 0n): number {
  const curveCash = ev.cashIn - ev.fee - reflection; // cash that hit the curve
  return Number(curveCash) / Number(ev.tokenOut);     // decimals cancel
}

/** Average price a sell executed at (pre-fee curve value ÷ tokens). */
function sellPrice(ev: SellEvent): number {
  return Number(ev.gross) / Number(ev.tokenIn);
}
```

For a candle/feed, use the latest trade's price as the market's "last price".

### Current floor (tracked from events)

Trade events carry `floorReserve` (the ledger) but **not** the floor *price*. Track the floor price from `MarketInitializedEvent.floorPrice`, then update it on every `FloorRaisedEvent.floor`:

```ts
let floorQ64 = marketInit.floorPrice;          // starting floor (Q64.64)
// on each FloorRaisedEvent for this market:
floorQ64 = floorRaised.floor;                  // monotone, only ever rises
const floorPrice = q64ToPrice(floorQ64);
```

### Market cap

```ts
const decimals = marketInit.decimals;
// price = last trade price (or spot, below); tokenSupply is RAW from any [post] field
function marketCap(price: number, tokenSupplyRaw: bigint): number {
  const supplyWhole = Number(tokenSupplyRaw) / 10 ** decimals;
  return price * supplyWhole;                  // in collateral; × collateral-USD for USD
}
```

### Exact spot price (needs the curve params)

The "last trade" price is enough for a feed. For the **exact** spot at the current supply (e.g. to quote), you need the curve shape from the `init_market` args and the overlay. The characteristic below the hinge is just the line; above it, the exponential:

```ts
// char_params = [p0, m, g2, k]. p0,g2 are Q64.64; m,k are Q64.96 (32 extra bits).
// x = token_supply (raw). Returns M(x) as a Q64.64 price.
function characteristic(x: bigint, p0: bigint, m: bigint, g2: bigint, k: bigint, hingeX: bigint): bigint {
  const base = p0 + (m * x >> 32n);            // p0 + m·x   (m is Q64.96 → >>32 to Q64.64)
  if (x <= hingeX) return base;                 // below the hinge: exactly the line
  const d = x - hingeX;
  const u = (k * d) >> 32n;                      // u = k·d, Q64.64
  const eu = expQ64(u);                          // e^u in Q64.64 (see note)
  const growth = (g2 * (eu - Q64 - u)) >> 64n;   // g2·(e^u − 1 − u)
  return base + growth;
}
```

`M(x)` is only the bare characteristic. The **overlay** then applies the floor band / ramp (×`ramp_scalar`) / main depending on where `x` sits relative to `x1` (`ramp_start_x`) and `x2` (`x2_cached`) — see [the curve model](./PROGRAM_PRO.md#the-curve-model). The `e^u` routine (range-reduce by ln2 + Taylor) and the full overlay are implemented in the Rise SDK; reimplement from `curve/hinged_exp.rs` if you need it standalone.

> **`m` and `k` are Q64.96, not Q64.64** — 32 extra fractional bits (`SLOPE_FRAC_EXTRA = 32`) so ultra-low launch prices stay representable. Multiply a Q64.96 slope by a raw integer then shift right by 32 to land back in Q64.64 (as above). `p0` and `g2` are plain Q64.64.

---

## See also

- [**Program (Pro)**](./PROGRAM_PRO.md) — instructions, `Market` account, curve, fees, reflection.
- [IDL (JSON)](../idl/rise_pro.json)

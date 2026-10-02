# Rise Pro (v2) — Indexing & Events

The Rise Pro program emits an event on every state change — buy, sell, leverage, deposit, borrow, repay, withdraw, floor raise, reflection, and lifecycle actions. Parse them from on-chain transactions to build your own indexer.

> For the program surface (instructions, the `Market` account, the curve), see [**Program (Pro)**](./PROGRAM_PRO.md). This doc describes v2 only (program `9nJ9wghEHY8bnsJQYrqV77uC9y4ccFHcCT1XyJxAkBVR`, IDL name `Rise Pro`). The v1 [INDEXING.md](./INDEXING.md) is the Mayflower program and does not apply.

---

## How events are emitted

Like v1, Rise Pro emits via Anchor `emit_cpi!` — **not** `emit!`. Events are **not** in `Program data:` log lines. Each emitted event is the data of an **inner self-CPI instruction** back to the Rise Pro program (one inner instruction per event). This survives log truncation, so the block path and a gRPC stream see identical data.

### Wire format

Each inner-instruction payload is:

```
EVENT_CPI_DISC (8 bytes)  ++  eventDiscriminator (8 bytes)  ++  borsh(fields)
```

- `EVENT_CPI_DISC = e4 45 a5 2e 51 cb 9a 1d` = `sha256("anchor:event")[..8]` — the same constant across Anchor versions.
- `eventDiscriminator = sha256("event:<EventName>")[..8]`.

### Parsing recipe

1. Skip transactions with `meta.err`, and transactions that don't reference the Rise Pro program id.
2. Build the full account-key list: `[staticAccountKeys, loadedAddresses.writable, loadedAddresses.readonly]`.
3. Walk `meta.innerInstructions`. For each inner instruction whose `programIdIndex` resolves to the Rise Pro program and whose data is `≥ 16` bytes with the first 8 bytes equal to `EVENT_CPI_DISC`, take `data.subarray(8)` — that is `eventDiscriminator ++ borsh(fields)`.
4. Match the next 8 bytes against the discriminator table below and Borsh-decode the remainder **in struct order**.

### Field notes that apply to every event

- **Fields marked `[post]` are post-transaction snapshots**, not deltas. `floor_price`, `token_supply`, `floor_reserve`, `price_reserve`, `total_debt`, `total_collateral`, `cum_net_inflow`, and the geometry memos (`ramp_start_x`/`x1`, `x2`, `m_x1`, `y_shift`) are the market state **after** the trade — so an indexer never has to read the `Market` account back.
- **The actor and the market are in the payload.** Unlike v1, every trading/lending/floor/reflection event carries `market` and the actor (`buyer`/`seller`/`owner`/`creator`). You only need the enclosing instruction's accounts for the per-position escrow (`position`, `collateral_vault`) and for `init_market`'s vault pubkeys.
- **`state_seq`** is a monotone per-market counter. Use it as the ordering key for last-write-wins: ignore an event whose `state_seq` is ≤ the one you already stored for that market.
- **Prefix-compatible trailing fields.** Some events append fields over time (`book_shortfall`, `y_shift`, `heal_spread` on sells; `reflection_bps` on market init; `entries` on reflection distribution). A decoder should treat missing trailing bytes as `0` / empty.

### The fields that matter most

If you're building a market feed (price, volume, market cap, floor), these are the load-bearing attributes — index these first and the rest is enrichment:

| Attribute | Where | Why it matters |
|---|---|---|
| **`stateSeq`** | every event | **The ordering key.** Monotone per market; apply events in `stateSeq` order and drop any `≤` the last one you stored. This is how you stay consistent across the block path, a gRPC stream, and retries. |
| **`market`** + **actor** (`buyer`/`seller`/`owner`) | every event | Row identity and per-user history — both in the payload, no account lookups. |
| **`tokenOut`** (buy) / **`tokenIn`**, **gross**, **payout** (sell) | trade events | Tokens and cash of the swap (RAW). Price = `(cashIn − fee − reflection) / tokenOut` on buys, `gross / tokenIn` on sells — decimals cancel. |
| **`cashIn`** / **`fee`** | trade events | Volume and the exact protocol fee of **this** swap (not accumulated). |
| **`tokenSupply`** `[post]` | trade events | Circulating supply after the trade → market cap, and the `x` the curve is priced at. |
| **`floorReserve`** / **`priceReserve`** `[post]` | trade events | The two vault ledgers → TVL / backing. |
| **`floor_price`** | `MarketInitializedEvent` (start) + `FloorRaisedEvent.floor` (updates) | The guaranteed floor. **Not** on trade events — you must track it from these two. Monotone, only ever rises. |
| **`char_params`**, `ramp_scalar`, `hingeX`, `curveKind` | `init_market` **args** + event | The curve shape, needed to quote an exact spot price. See [bootstrap](#start-here-bootstrap-a-market-from-init_market). |
| **`reflectionBps`** | `MarketInitializedEvent` | Whether to apply the 0/1/3 % reflection leg when pricing — getting this wrong overstates fills by that percentage. |

Everything else (`glideCredit`, `x2`, `yShift`, `mX1`, `bookShortfall`, `healSpread`, `cumNetInflow`…) is curve-internal state you only need if you reimplement the overlay or self-healing; a price/volume feed can ignore it.

### Discriminators

```
MarketInitialized      70,173,96,202,100,143,45,25
Buy                    103,244,82,31,44,245,119,119
Sell                   62,47,55,10,165,3,220,42
Borrow                 86,8,140,206,215,179,118,201
Repay                  129,213,0,108,218,108,82,140
LeverageBuy            192,251,245,66,149,87,79,195
LeverageSell           83,204,210,96,206,215,12,154
FloorRaised            180,150,147,252,238,147,174,236
CreatorFeesClaimed     40,137,200,40,154,133,234,251
TeamFeesWithdrawn      62,98,123,152,178,122,230,127
Deposit                120,248,61,83,31,142,107,144
Withdraw               22,9,133,26,160,44,71,192
MarketFlagsSet         58,254,223,22,195,116,134,114
MarketUpdated          56,197,234,194,244,125,181,218
ReflectionDistributed  103,134,80,57,6,40,217,236
```

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

Emitted on every `buy`. Field order (Borsh wire order):

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
`market`, `owner`, `amount: u64`, `collateral: u64` (position total **[post]**), `stateSeq: u64`, `totalCollateral: u64` (market-wide **[post]**).

### BorrowEvent
`market`, `owner`, `amount: u64`, `fee: u64`, `feeFloor: u64`, `feeCreator: u64`, `feeTeam: u64`, `received: u64` (cash to the borrower, net of fee), `debt: u64` (position **[post]**), `drawnFromFloor: u64`, `stateSeq: u64`, `floorReserve: u64` **[post]**, `priceReserve: u64` **[post]**, `totalDebt: u64` **[post]**.

### RepayEvent
`market`, `owner`, `amount: u64`, `debt: u64` (position **[post]**), `stateSeq: u64`, `floorReserve: u64` **[post]**, `priceReserve: u64` **[post]**, `totalDebt: u64` **[post]**.

### WithdrawEvent
`market`, `owner`, `amount: u64`, `collateral: u64` (position total **[post]**), `stateSeq: u64`, `totalCollateral: u64` (market-wide **[post]**).

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
`market`, `creator`, `mintToken`, `mintMain`, `name: String`, `symbol: String`, `uri: String`, `decimals: u8`, `feeConfigVersion: u32`, `feesTeam: u16`, `feesCreator: u16`, `feesFloor: u16`, `floorPrice: u128` **[Q64.64]**, `curveKind: u8`, `flags: u16`, `stateSeq: u64`, `buyFeeMbps: u32`, `sellFeeMbps: u32`, `borrowFeeMbps: u32`, `redeemFeeMbps: u32`, `hingeX: u64`, `reflectionBps: u16` (`0`/`100`/`300`).

> `hingeX` and `reflectionBps` are appended fields — decode them as `0` if the payload is short (older markets).

### MarketFlagsSetEvent
`market`, `previousFlags: u16`, `flags: u16`, `stateSeq: u64`.

### MarketUpdatedEvent
`market`, `previousCreator`, `creator`, `feesCreator: u16`, `feesFloor: u16`, `metadataUpdated: bool`, `stateSeq: u64`.

### CreatorFeesClaimedEvent
`market`, `creator` (whoever held `market.creator` at claim), `amount: u64`, `destination`.

### TeamFeesWithdrawnEvent
`mintMain`, `amount: u64`, `destination`.

---

## Reflection events

| Event | Fields |
|---|---|
| `ReflectionAccruedEvent` | `market`, `isBuy: bool` (`true` = buy-side), `amount: u64` (routed to the reflection vault), `stateSeq: u64`. Emitted on every trade with a non-zero reflection tax. |
| `ReflectionEpochOpenedEvent` | `market`, `epoch: u64`, `pool: u64` (distributable, net of gas skim), `holderCount: u32`, `gasReserve: u64`. |
| `ReflectionDistributedEvent` | `market`, `epoch: u64`, `count: u32`, `batchTotal: u64`, `totalDistributed: u64`, `entries: Vec<{ owner: Pubkey, amount: u64 }>`. `owner` is always the holder **wallet**, never the ATA. |

The per-trade reflection amount lives on `ReflectionAccruedEvent`, **not** on the Buy/Sell event — join by `(market, stateSeq)` if you want the reflection leg of a specific swap.

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

# Rise Pro (v2) — Program

Rise Pro is the second-generation Rise program. It replaces the Mayflower dual-program design (v1) with a single Anchor program that owns the bonding curve, the floor, lending, leverage, floor raises and the reflection reward system.

| | |
|---|---|
| **Program (Devnet)** | `9nJ9wghEHY8bnsJQYrqV77uC9y4ccFHcCT1XyJxAkBVR` |
| **IDL name** | `Rise Pro` |
| **IDL (JSON)** | [`../idl/rise_pro.json`](../idl/rise_pro.json) |
| **Discriminator** | `Market.version == 2` |

> This document covers the v2 program surface: instructions, the `Market` account, the curve model, fees and reflection. For how to turn on-chain transactions into a feed, see [**Indexing & Events (Pro)**](./INDEXING_PRO.md). The legacy v1 docs ([PROGRAM.md](./PROGRAM.md), [INDEXING.md](./INDEXING.md)) still describe the Mayflower program and do **not** apply to v2.

---

## How v2 differs from v1 at a glance

- **One program, not two.** No separate Mayflower market-group. The `Market` account holds everything.
- **Single cash vault with two ledgers.** `floor_reserve` and `price_reserve` are book entries *inside* one token account, not separate accounts.
- **Hinged-exponential curve** (`curve_kind = 1`) in addition to linear (`curve_kind = 0`).
- **Self-healing sells** (optional per market, immutable): deep sells translate the curve down instead of draining the exit book.
- **Reflection tax** (optional per market, immutable): a 0 / 1 / 3 % holder-reward skim on every trade.
- **Fee split favours the floor on rounding** — the floor leg takes the remainder (v1 gave it to team).

---

## Instructions

The program exposes 23 instructions. Signatures for the ones an integrator builds directly are given in full.

### Trading

| Instruction | Args | Purpose |
|---|---|---|
| `buy` | `cash_in: u64, min_token_out: u64` | Buy with an exact cash amount. Fee + reflection come off the top, then the curve mints tokens by bisection on the exact forward cost. |
| `sell` | `token_in: u64, min_cash_out: u64` | Sell tokens back into the curve. Always exits ≥ the floor value of the tokens (Law 1); never reverts for insufficient book — the payout clamps to what the ledgers hold. |
| `leverage_buy` | `cash_in: u64, borrow_amount: u64, min_token_out: u64` | Borrow, buy, and lock the result as collateral, atomically. |
| `leverage_sell` | `collateral_in: u64, decrease_debt_by: u64, min_cash_out: u64` | Sell collateral without releasing it, repay debt from the proceeds, pay out the rest. |

`min_token_out` / `min_cash_out` are the caller's slippage gates and are checked **after** fees and reflection.

### Lending

| Instruction | Args | Purpose |
|---|---|---|
| `deposit` | `amount: u64` | Lock market tokens as collateral in the position PDA. |
| `borrow` | `amount: u64` | Draw cash against locked collateral, capped at `floor × collateral`. |
| `repay` | `amount: u64` | Repay principal. **Ungateable** — reads no flag/pause. |
| `withdraw` | `amount: u64` | Release collateral. **Ungateable.** |

The de-risk path (`repay`, `withdraw`) is ungateable by construction: no flag bit exists for it, so an indebted position can always be unwound.

### Floor raises

| Instruction | Args | Purpose |
|---|---|---|
| `raise_floor_preserve_area` | `new_floor: u128, new_x1: u64` | Area-preserving raise: converts price-discovery area into guaranteed floor value, moving the floor and the ramp start together. |
| `raise_floor_excess_liquidity` | — | Cash-funded, permissionless raise to what the committed cash already backs. The only path that can leave a ramp-band state. |

Both are permissionless keeper work; the program re-validates every bound (ratchet, floor ≤ buy curve, buffered Law 1, stress ceiling, ramp headroom, per-slot guard).

### Admin / config / lifecycle

| Instruction | Purpose |
|---|---|
| `init_config` | Create the protocol singleton (once). Signer becomes admin; `team_wallet` is set here and is immutable. |
| `propose_admin` / `accept_admin` | Two-step admin rotation. |
| `publish_fee_config` | Append an immutable fee-schedule version. Reaches new markets only; a market pins the current version at creation. |
| `set_collateral_config` | List or retune a collateral (USDC, SOL…) and create its per-collateral team vault. |
| `init_market` | **Permissionless** market creation: mint + vaults + curve params + a snapshot of the current fee schedule. |
| `update_market` | CTO path (admin-held): update token metadata and/or the creator wallet. |
| `set_market_flags` | Per-market circuit breaker. Cannot reach `withdraw` / `repay`. |
| `claim_creator_fees` | Sweep a market's accrued creator fees (signed by the current creator). |
| `withdraw_team_fees` | Sweep a collateral's team vault (permissionless trigger, pinned destination). |

### Reflection

| Instruction | Purpose |
|---|---|
| `open_reflection_epoch` | Skim the gas reserve to treasury, freeze the distributable pool, size the holder bitmap. |
| `distribute_reflection` | Pay a batch of holders from the reflection vault. Bitmap-idempotent (a holder can't be paid twice in an epoch). |
| `close_reflection_epoch` | Close a finished epoch, return rent to the admin. |

---

## Accounts & PDAs

### PDA seed reference

Every program-derived account is found from these seeds (Anchor, program id `9nJ9wghEHY8bnsJQYrqV77uC9y4ccFHcCT1XyJxAkBVR`):

| PDA | Seeds |
|---|---|
| **config** (singleton) | `["config"]` |
| **fee_config** | `["fee_config", version: u32 LE]` |
| **collateral_config** | `["collateral", mint_main]` |
| **market** | `["market", mint_token]` |
| **mint_token** | `["mint", vanity_seed: u64 LE]` |
| **vault** (cash, holds both ledgers) | `["vault", market]` |
| **collateral_vault** | `["collateral_vault", market]` |
| **creator_claim** | `["creator_claim", market]` |
| **team_vault** (per collateral) | `["team_vault", mint_main]` |
| **reflection_vault** | `["reflection_vault", market]` |
| **position** (per owner, per market) | `["position", market, owner]` |
| **event_authority** (Anchor emit_cpi) | `["__event_authority"]` |

`*_cash` / `*_token` user accounts are **Associated Token Accounts** (ATA): seeds `[owner, token_program, mint]` under the SPL Associated Token program. `token_program_main` is the collateral's token program (SPL or Token-2022); `token_program` is the market token's program (Token-2022).

### Accounts per instruction

Listed in IDL order. `mut` = writable, `signer` = must sign. `event_authority` + `program` (self) are appended on every instruction for `emit_cpi!` and omitted below.

#### `buy` / `sell`

| Account | Flags | Derivation |
|---|---|---|
| `buyer` / `seller` | mut, signer | the trader |
| `market` | mut | `["market", mint_token]` |
| `vault` | mut | `["vault", market]` |
| `creator_claim` | mut | `["creator_claim", market]` |
| `team_vault` | mut | `["team_vault", mint_main]` |
| `reflection_vault` | mut | `["reflection_vault", market]` |
| `mint_token` | mut | the market token mint |
| `mint_main` | | the collateral mint |
| `buyer_cash` / `seller_cash` | mut | ATA(trader, mint_main) |
| `buyer_token` / `seller_token` | mut | ATA(trader, mint_token) |
| `token_program_main`, `token_program`, `associated_token_program`, `system_program` | | programs |

#### `leverage_buy` / `leverage_sell`

Same as buy/sell **plus** the position escrow, minus the counterparty token ATA on the buy side:

| Account | Flags | Derivation |
|---|---|---|
| `buyer` / `owner` | mut, signer | the trader |
| `market` | mut | `["market", mint_token]` |
| `position` | mut | `["position", market, owner]` |
| `vault` | mut | `["vault", market]` |
| `collateral_vault` | mut | `["collateral_vault", market]` |
| `creator_claim` | mut | `["creator_claim", market]` |
| `team_vault` | mut | `["team_vault", mint_main]` |
| `reflection_vault` | mut | `["reflection_vault", market]` |
| `mint_token` | mut | market token |
| `mint_main` | | collateral |
| `buyer_cash` / `owner_cash` | mut | ATA(trader, mint_main) |
| token programs + `system_program` (+ `associated_token_program` on sell) | | programs |

#### `deposit` / `withdraw`

| Account | Flags | Derivation |
|---|---|---|
| `owner` | mut, signer | |
| `market` | mut | `["market", mint_token]` |
| `position` | mut | `["position", market, owner]` |
| `collateral_vault` | mut | `["collateral_vault", market]` |
| `mint_token` | | market token |
| `owner_token` | mut | ATA(owner, mint_token) |
| `token_program`, `system_program` | | programs |

#### `borrow`

| Account | Flags | Derivation |
|---|---|---|
| `owner` | mut, signer | |
| `market` | mut | `["market", mint_token]` |
| `position` | mut | `["position", market, owner]` |
| `vault` | mut | `["vault", market]` |
| `creator_claim` | mut | `["creator_claim", market]` |
| `team_vault` | mut | `["team_vault", mint_main]` |
| `mint_main` | | collateral |
| `owner_cash` | mut | ATA(owner, mint_main) |
| programs | | |

#### `repay`

`owner` (mut, signer) · `market` (mut) · `position` (mut) · `vault` (mut) · `mint_main` · `owner_cash` (mut, ATA) · `token_program_main`. No `creator_claim`/`team_vault` — repay takes no fee.

#### `raise_floor_preserve_area` / `raise_floor_excess_liquidity`

Permissionless, minimal:

| Account | Flags | Derivation |
|---|---|---|
| `cranker` | signer | anyone (pays the tx) |
| `market` | mut | `["market", mint_token]` |

#### `init_market`

| Account | Flags | Derivation |
|---|---|---|
| `creator` | mut, signer | the market creator / fee payer |
| `config` | | `["config"]` |
| `fee_config` | | `["fee_config", version]` (the current published version) |
| `collateral_config` | | `["collateral", mint_main]` |
| `mint_main` | | collateral mint (must be a listed collateral) |
| `mint_token` | mut | `["mint", vanity_seed]` — pre-ground so the mint ends in `rise` |
| `market` | mut | `["market", mint_token]` |
| `vault` | mut | `["vault", market]` |
| `creator_claim` | mut | `["creator_claim", market]` |
| `collateral_vault` | mut | `["collateral_vault", market]` |
| `reflection_vault` | mut | `["reflection_vault", market]` |
| token programs + `system_program` | | |

---

## The `Market` account

PDA seed: `["market", mint_token]`. The fields an integrator or indexer reads:

### Identity (frozen at creation)

| Field | Type | Notes |
|---|---|---|
| `version` | u8 | `2` |
| `curve_kind` | u8 | `0` = Linear, `1` = HingedExponential |
| `token_decimals` | u8 | Equal to **both** mints' decimals (v2 invariant) |
| `flags` | u16 | See the flag table below |
| `creator` | Pubkey | Reassignable via `update_market` |
| `mint_token` | Pubkey | The market token mint (PDA seed) |
| `mint_main` | Pubkey | Collateral mint (USDC / SOL) |
| `vault` | Pubkey | Single cash vault (holds both ledgers) |
| `collateral_vault` | Pubkey | Locked collateral tokens |
| `creator_claim` | Pubkey | Creator-fee accrual account |

**Flags** (`flags` bitfield): `CAN_BUY` (1), `CAN_SELL` (2), `CAN_BORROW` (4), `CAN_RAISE_FLOOR` (64), `CAN_DEPOSIT` (128), `SELF_HEALING` (256). `SELF_HEALING` is set only at creation and is immutable (excluded from the writable mask). There is no bit for `withdraw`/`repay`.

### Economics (snapshotted from the pinned FeeConfig at creation)

| Field | Type | Unit |
|---|---|---|
| `buy_fee_mbps`, `sell_fee_mbps`, `borrow_fee_mbps` | u32 | **mbps** (`1_000_000` = 100 %). 1.25 % = `12500`. |
| `fees_team`, `fees_creator`, `fees_floor` | u16 | **bps of the fee**. `fees_floor = 10_000 − fees_team − fees_creator` (computed, never supplied). |
| `min_trade_amount` | u64 | raw collateral |
| `q_max_bps` | u16 | stress depth, bps of supply (default 200 = 2 %) |
| `reflection_bps` | u16 | `0` / `100` / `300`. Immutable. |

### Curve parameters

| Field | Type | Format |
|---|---|---|
| `char_params` | [i128; 4] | Linear: `(p0, m, _, _)`. HingedExp: `(p0, m, g2, k)` with `g2 = 2q/k²`. `p0`/`g2` are **Q64.64**; `m`/`k` are **Q64.96** (see [Q64 conversion](./INDEXING_PRO.md#q6464-fixed-point-conversion)). |
| `hinge_x` | u64 | Raw supply where the exponential ignites (0 for linear). |

### Live overlay state

| Field | Type | Format |
|---|---|---|
| `floor_price` | u128 | **Q64.64**. The ratchet — monotone non-decreasing, forever. |
| `ramp_start_x` | u64 | `x1`, raw supply units. |
| `ramp_scalar` | u128 | `s`, **Q64.64**. Flagship `s = 2.0` (`2 << 64`). |
| `x2_cached` | u64 | Memo. `0` = degenerate ramp; `u64::MAX` = floor-above-characteristic sentinel. |
| `m_x1_cached` | u128 | **Q64.64** memo = `M̃(x1)`. |
| `y_shift` | i128 | **Q64.64** signed curve translation (self-healing; `0` on legacy markets). |

### Ledgers & analytics

| Field | Type | Notes |
|---|---|---|
| `token_supply` | u64 | Circulating supply = `x` |
| `floor_reserve`, `price_reserve` | u64 | The two ledgers inside `vault` |
| `total_debt`, `total_collateral` | u64 | Lending aggregates |
| `cum_net_inflow` | u128 | Glide progress key |
| `state_seq` | u64 | Monotone write counter — **the ordering key** for last-write-wins indexing |

---

## The curve model

Normative definition: the on-chain `curve/hinged_exp.rs`, mirrored bit-for-bit in the TypeScript SDK/indexer.

**Characteristic** (kind 1, hinged exponential). With hinge `h = hinge_x`, `d = max(x − h, 0)`, `u = k·d`:

```
M(x) = p0 + m·x + g2·(e^u − 1 − u)          g2 = 2q/k²
A(x) = p0·x + m·x²/2 + g3·(e^u − 1 − u − u²/2)   g3 = 2q/k³
```

Below the hinge (`d = 0`) the growth term is identically zero, so `M(x) = p0 + m·x` exactly — the curve **is** the line there. Kind 0 (linear) is just `M(x) = p0 + m·x`. All maths is Q64.64 unsigned in u128 with floored multiplies/divides; `e^u` is defined for `0 ≤ u ≤ 32`, beyond which a buy reverts `MathOverflow` (sells at a given supply always work).

In code (`char_params = [p0, m, g2, k]`; `p0`/`g2` Q64.64, `m`/`k` Q64.96):

```ts
const Q64 = 2n ** 64n;
// M(x) as a Q64.64 price. x = supply (raw), hingeX from the init_market args.
function M(x: bigint, [p0, m, g2, k]: bigint[], hingeX: bigint): bigint {
  const base = p0 + (m * x >> 32n);               // p0 + m·x (m Q64.96 → >>32)
  if (x <= hingeX) return base;                   // below hinge: the line, exactly
  const u = (k * (x - hingeX)) >> 32n;            // u = k·d
  return base + ((g2 * (expQ64(u) - Q64 - u)) >> 64n); // + g2·(e^u − 1 − u)
}
```

(`expQ64` = `e^u` in Q64.64, from `curve/hinged_exp.rs`; the full price feed and overlay are in the SDK. See [Computing prices](./INDEXING_PRO.md#computing-prices--market-cap) for the practical integrator snippets.)

**The overlay** composes three bands over the shifted characteristic `M̃ = M + y_shift`:

- **Floor band** (`x < x1`): price pinned at `floor_price` (F).
- **Ramp / shoulder** (`x1 ≤ x < x2`): `F + s·(M̃(x) − M̃(x1))`.
- **Main** (`x ≥ x2`): the shifted characteristic `M̃(x)`.

`x2` is where the ramp rejoins the main (cached in `x2_cached`). Two special values:
- `x2_cached == 0` → degenerate ramp (`x2 ≤ x1`), the overlay is the bare characteristic.
- `x2_cached == u64::MAX` → **sentinel**: the floor sits above the characteristic everywhere, the ramp never rejoins the main. This is a legitimate, reachable state (a deep sell pulls `x1` down, then a raise ratchets the floor past `M̃(x1)`); the curve is never evaluated at the sentinel.

**Price vs floor.** `floor_price` is a hard, monotone floor. Spot price is the overlay-effective price at `token_supply`. A self-healing market buys on the shifted main alone (so a floor raise can never brick a buy); a legacy market buys on the same overlay it sells on.

---

## Fees

Rate per market in **mbps** (`1_000_000` = 100 %), snapshotted at creation. The split, taken on the fee:

```
total   = ceil(base × rate_mbps / 1_000_000)
creator = floor(total × fees_creator / 10_000)
team    = floor(total × fees_team    / 10_000)
floor   = total − creator − team          // remainder → floor
```

Default: **1.25 % trade fee**, split **75 % team / 10 % creator / 15 % floor** (`fees_team = 7500`, `fees_creator = 1000`, `fees_floor = 1500`). Any leg may be zero. The floor always takes the rounding remainder, so the three legs sum to `total` exactly.

## Reflection

`reflection_bps ∈ {0, 100, 300}` (0 / 1 / 3 %), pinned at creation, immutable. It is an extra leg **on top of** the trade fee, routed to a separate `reflection_vault` PDA (`["reflection_vault", market]`) — it never enters `floor_reserve` / `price_reserve`.

- **Buy**: `reflection = ceil(cash_in × reflection_bps / 10_000)`, removed from the top (with the fee) **before** the curve quote — so the buyer receives fewer tokens.
- **Sell**: `reflection = ceil(gross × reflection_bps / 10_000)`, carved out of the **payout** — so the seller receives less cash.

Accrued reflection is later distributed to holders pro-rata across epochs (`open` → `distribute` → `close`). When quoting a trade on a reflection market, apply the leg exactly as above or the quote will overstate the fill by the reflection percentage.

---

## See also

- [**Indexing & Events (Pro)**](./INDEXING_PRO.md) — event layouts, parsing, Q64 conversion.
- [IDL (JSON)](../idl/rise_pro.json)

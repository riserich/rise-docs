# Rise Pro (v2) — Program

| | |
|---|---|
| **Program (Devnet)** | `9nJ9wghEHY8bnsJQYrqV77uC9y4ccFHcCT1XyJxAkBVR` |
| **IDL (JSON)** | [`../idl/rise_pro.json`](../idl/rise_pro.json) |

A single Anchor program owns the bonding curve, the floor, lending, leverage, floor raises and the reflection reward system. One cash vault per market holds two book-entry ledgers (`floor_reserve`, `price_reserve`); sells always exit at or above the floor value (Law 1).

This page documents the instructions an integrator or user wallet actually builds. Protocol-operated instructions (floor raises, reflection epochs, admin/config) are in the [IDL](../idl/rise_pro.json) and their effects are documented as events in [Indexing (Pro)](./INDEXING_PRO.md).

---

## Summary

| # | Instruction | What it does |
|---|---|---|
| 1 | [`buy`](#1-buy) | Buy tokens with collateral |
| 2 | [`sell`](#2-sell) | Sell tokens back — always exits at or above the floor |
| 3 | [`deposit`](#3-deposit) | Lock tokens as collateral |
| 4 | [`withdraw`](#4-withdraw) | Unlock collateral |
| 5 | [`borrow`](#5-borrow) | Borrow cash against your collateral |
| 6 | [`repay`](#6-repay) | Pay back debt |
| 7 | [`leverage_buy`](#7-leverage_buy) | Buy with borrowed cash on top of your own, in one transaction |
| 8 | [`leverage_sell`](#8-leverage_sell) | Unwind a leveraged position — sell, repay the debt, pocket the rest |
| 9 | [`init_market`](#9-init_market) | Create a new token market |
| 10 | [`claim_creator_fees`](#10-claim_creator_fees) | Collect the creator's share of trading fees |

---

## PDA seed reference

All PDAs derive under the Rise Pro program id.

| Account | Seeds | Scope |
|---|---|---|
| `config` | `["config"]` | global singleton (admin, immutable `team_wallet`) |
| `fee_config` | `["fee_config", version:u32 LE]` | per published fee version |
| `collateral_config` | `["collateral", mint_main]` | per collateral |
| `team_vault` | `["team_vault", mint_main]` | per collateral, shared by every market |
| `mint_token` | `["mint", vanity_seed:u64 LE]` | per market (ground vanity seed) |
| `market` | `["market", mint_token]` | per market |
| `vault` | `["vault", market]` | per market — the single cash vault (`floor_reserve` and `price_reserve` are ledgers pointing into it) |
| `collateral_vault` | `["collateral_vault", market]` | per market — deposited market tokens |
| `creator_claim` | `["creator_claim", market]` | per market |
| `reflection_vault` | `["reflection_vault", market]` | per market |
| `position` | `["position", market, owner]` | per wallet per market |
| `event_authority` | `["__event_authority"]` | Anchor event-CPI |

---

---

## Trading

### 1. `buy`

Buy with an exact cash amount. Quotes by bisection on the exact forward cost, splits obligation-first, and asserts Law 1 before moving anything.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `cash_in` | `u64` | Collateral to spend, raw units of `mint_main`. Fees + reflection come off this amount first. |
| `min_token_out` | `u64` | Slippage gate on tokens received (raw). Checked after fees. |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `buyer` |  | ✓ | ✓ | The buyer - signs, pays `cash_in` from `buyer_cash`, and pays rent for `buyer_token` on a wallet's first buy. |
| 1 | `market` | `["market", market.mint_token]` | ✓ |  | Rise market being bought into - PDA derived from `[MARKET_SEED, mint_token]`, signs the mint CPI and holds the reserves ledger updated by … |
| 2 | `vault` |  | ✓ |  | Pinned by an `address` check against the key `Market` stored at creation, so a substituted account is rejected. |
| 3 | `creator_claim` |  | ✓ |  | Creator claim account - receives the creator's leg of the buy fee. Pinned by an `address` check against the key `Market` stored at creation. |
| 4 | `team_vault` | `["team_vault", mint_main]` | ✓ |  | **Global per collateral** — shared by every market on this mint. |
| 5 | `reflection_vault` | `["reflection_vault", market]` | ✓ |  | on reflection markets; on a `reflection_bps == 0` market (incl. every pre-upgrade market) it may not exist and is never touched. |
| 6 | `mint_token` |  | ✓ |  | Market token mint - `token_out` is minted from it to the buyer. Validated to match `market.mint_token`, so a foreign mint is rejected. |
| 7 | `mint_main` |  |  |  | Base currency mint (e.g., USDC) - buyer pays in this token. Validated to match `market.mint_main`; needed for `transfer_checked` decimals. |
| 8 | `buyer_cash` |  | ✓ |  | Buyer's base-currency account - source of `cash_in`, debited by the three transfer legs. |
| 9 | `buyer_token` | `[buyer, token_program, mint_token]` | ✓ |  | The buyer's canonical ATA for the market token, created on the fly on a wallet's first buy (`init_if_needed` — a no-op when it exists, rent … |
| 10 | `token_program_main` |  |  |  | Token program for base-currency transfers (vault, creator and team legs). |
| 11 | `token_program` |  |  |  | Token program for the market token - used by the mint CPI and `buyer_token`. |
| 12 | `associated_token_program` |  |  |  | Associated token program - required by `init_if_needed` on `buyer_token`. |
| 13 | `system_program` |  |  |  | System program for account creation (`buyer_token` rent on a first buy). |
| 14 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 15 | `program` |  |  |  | This program — target of the event self-CPI. |

---

### 2. `sell`

Sell tokens back into the curve. The exit is unconditional (Law 1): the payout never reverts for an insufficient book — it clamps to what the ledgers can cover (`floor_reserve` + `price_reserve`), with the floor-attributable share always guaranteed.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `token_in` | `u64` | Market tokens to sell, raw units. |
| `min_cash_out` | `u64` | Slippage gate on the net payout (raw collateral), checked after fees/reflection and the exit clamp. |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `seller` |  | ✓ | ✓ | Seller - signs the burn of their tokens and receives the payout. Mutable because it pays rent if `seller_cash` has to be created. |
| 1 | `market` | `["market", market.mint_token]` | ✓ |  | Rise market being sold into - PDA derived from ["market", mint_token]. Vault authority: signs every payout leg out of the vault. |
| 2 | `vault` |  | ✓ |  | Market vault holding both cash ledgers (floor reserve + above-floor book). Validated to match `market.vault` - source of every payout leg. |
| 3 | `creator_claim` |  | ✓ |  | Creator claim account - receives the creator's fee leg, swept later via `claim_creator_fees`. Validated to match `market.creator_claim`. |
| 4 | `team_vault` | `["team_vault", mint_main]` | ✓ |  | **Pinned by seeds**, not left bare. This account receives the team fee leg, and on the vault-funded paths that leg is paid out by the … |
| 5 | `reflection_vault` | `["reflection_vault", market]` | ✓ |  | on reflection markets; on a `reflection_bps == 0` market (incl. every pre-upgrade market) it may not exist and is never touched. |
| 6 | `mint_token` |  | ✓ |  | Market token mint - the sold tokens are burned against it, shrinking supply. Validated to match `market.mint_token`. |
| 7 | `mint_main` |  |  |  | Base currency mint (e.g., USDC) - seller is paid in this token. Validated to match `market.mint_main`. |
| 8 | `seller_cash` | `[seller, token_program_main, mint_main]` | ✓ |  | Receives the payout — created on the fly if the seller has never held the collateral (init_if_needed, rent paid by the seller). |
| 9 | `seller_token` | `[seller, token_program, mint_token]` | ✓ |  | The burn source, pinned to the seller's canonical ATA — same discipline as `buy`'s `buyer_token`. |
| 10 | `token_program_main` |  |  |  | Token program for base currency transfers (vault payouts). |
| 11 | `token_program` |  |  |  | Token program for the market token (the burn). |
| 12 | `associated_token_program` |  |  |  | Associated token program - creates `seller_cash` if it does not exist yet. |
| 13 | `system_program` |  |  |  | System program for account creation. |
| 14 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 15 | `program` |  |  |  | This program — target of the event self-CPI. |

---

## Lending

### 3. `deposit`

Lock market tokens as collateral.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `amount` | `u64` | Market tokens to lock as collateral (raw). |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `owner` |  | ✓ | ✓ | Depositor - signs the token transfer and pays rent if the position is created. |
| 1 | `market` | `["market", market.mint_token]` | ✓ |  | Rise market being deposited into - its `total_collateral` is increased. PDA derived from ["market", mint_token]. |
| 2 | `position` | `["position", market, owner]` | ✓ |  | Per-user lending position - tracks this owner's collateral and debt. Created on first deposit (init_if_needed, rent paid by owner). |
| 3 | `collateral_vault` | `["collateral_vault", market]` | ✓ |  | Market collateral vault - receives the deposited tokens. PDA derived from ["collateral_vault", market]. |
| 4 | `mint_token` |  |  |  | Market token mint - validated to match `market.mint_token`. |
| 5 | `owner_token` |  | ✓ |  | Owner's market-token account - source of the deposit. Constraints pin it to `mint_token`. |
| 6 | `token_program` |  |  |  | Token program for market-token transfers. |
| 7 | `system_program` |  |  |  | System program for position account creation. |
| 8 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 9 | `program` |  |  |  | This program — target of the event self-CPI. |

---

### 4. `withdraw`

Release collateral. **Ungateable** — reads no flag and no pause.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `amount` | `u64` | Collateral tokens to release (raw). Capped so remaining collateral still covers the debt at the floor. |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `owner` |  | ✓ | ✓ | Position owner - must match `position.owner` (has_one). |
| 1 | `market` | `["market", market.mint_token]` | ✓ |  | Rise market - holds the floor price `F` for the release bound and signs the vault transfer as `collateral_vault` authority. |
| 2 | `position` | `["position", market, owner]` | ✓ |  | Owner's lending position - its collateral is reduced. PDA derived from ["position", market, owner]; has_one pins the owner. |
| 3 | `collateral_vault` | `["collateral_vault", market]` | ✓ |  | Market collateral vault - source of the released tokens. PDA derived from ["collateral_vault", market]. |
| 4 | `mint_token` |  |  |  | Market token mint - validated to match `market.mint_token`. |
| 5 | `owner_token` |  | ✓ |  | Owner's market-token account - receives the released collateral. Constraints pin it to `mint_token`. |
| 6 | `token_program` |  |  |  | Token program for market-token transfers. |
| 7 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 8 | `program` |  |  |  | This program — target of the event self-CPI. |

---

### 5. `borrow`

Draw cash against locked collateral, capped at `F × collateral`.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `amount` | `u64` | Debt to take on (raw collateral units). The borrow fee is deducted from the payout; debt rises by the full amount. Cap: debt ≤ floor × collateral. |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `owner` |  | ✓ | ✓ | Borrower - must own the position (has_one). Pays rent if `owner_cash` is created. |
| 1 | `market` | `["market", market.mint_token]` | ✓ |  | Rise market lending the cash - holds the reserves, debt totals, and the floor price `F`; signs the vault payouts as vault authority. |
| 2 | `position` | `["position", market, owner]` | ✓ |  | Borrower's lending position - collateral backs the draw, debt is increased. |
| 3 | `vault` |  | ✓ |  | Market cash vault - source of the borrowed cash and all fee legs. Validated to match `market.vault`. |
| 4 | `creator_claim` |  | ✓ |  | Creator fee escrow - receives the creator leg of the origination fee. Validated to match `market.creator_claim`. |
| 5 | `team_vault` | `["team_vault", mint_main]` | ✓ |  | **Pinned by seeds**, not left bare. This account receives the team fee leg, and on the vault-funded paths that leg is paid out by the … |
| 6 | `mint_main` |  |  |  | Base currency mint (e.g., USDC) - validated to match `market.mint_main`. |
| 7 | `owner_cash` | `[owner, token_program_main, mint_main]` | ✓ |  | Receives the borrowed cash — created on the fly if the borrower has never held the collateral (init_if_needed, rent paid by the borrower). |
| 8 | `token_program_main` |  |  |  | Token program for base currency transfers. |
| 9 | `associated_token_program` |  |  |  | Associated token program - needed to create `owner_cash` on the fly. |
| 10 | `system_program` |  |  |  | System program for account creation. |
| 11 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 12 | `program` |  |  |  | This program — target of the event self-CPI. |

---

### 6. `repay`

Repay principal. **Ungateable** — reads no flag and no pause.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `amount` | `u64` | Principal to repay (raw collateral units). |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `owner` |  | ✓ | ✓ | The position owner, and only them. The design doc (ix 17) proposed a permissionless repay for v1 parity; the founder overrode that — a … |
| 1 | `market` | `["market", market.mint_token]` | ✓ |  | Rise market owed the debt - its reserves and `total_debt` are updated. PDA derived from ["market", mint_token]. |
| 2 | `position` | `["position", market, owner]` | ✓ |  | Owner's lending position - its debt is reduced. PDA derived from ["position", market, owner]; has_one pins the owner. |
| 3 | `vault` |  | ✓ |  | Market cash vault - receives the repaid cash. Validated to match `market.vault`. |
| 4 | `mint_main` |  |  |  | Base currency mint (e.g., USDC) - validated to match `market.mint_main`. |
| 5 | `owner_cash` |  | ✓ |  | Owner's base-currency account - source of the repayment. Constraints pin it to `mint_main`. |
| 6 | `token_program_main` |  |  |  | Token program for base currency transfers. |
| 7 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 8 | `program` |  |  |  | This program — target of the event self-CPI. |

---

## Leverage

### 7. `leverage_buy`

Borrow, buy, and lock the result as collateral, atomically. The borrowed cash never leaves the vault — the borrow's payout and the buy's payment cancel — so this costs the same three transfers as a plain buy.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `cash_in` | `u64` | The user's own cash leg (raw collateral). |
| `borrow_amount` | `u64` | Debt drawn atomically and spent into the buy alongside `cash_in`. |
| `min_token_out` | `u64` | Slippage gate on tokens bought with cash + borrow (raw). |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `buyer` |  | ✓ | ✓ | Buyer - signs, funds `cash_in`, and pays rent for the position on first use. |
| 1 | `market` | `["market", market.mint_token]` | ✓ |  | Rise market being levered into - holds every ledger and signs the vault-funded legs and the mint. PDA derived from ["market", mint_token]. |
| 2 | `position` | `["position", market, buyer]` | ✓ |  | Buyer's loan position - tracks collateral and debt. |
| 3 | `vault` |  | ✓ |  | Main cash vault - receives `cash_in` and pays the fee legs. |
| 4 | `collateral_vault` | `["collateral_vault", market]` | ✓ |  | The bought tokens are minted straight here. |
| 5 | `creator_claim` |  | ✓ |  | Creator claim account - receives the creator fee legs. Pinned by an `address` check against the key stored at creation. |
| 6 | `team_vault` | `["team_vault", mint_main]` | ✓ |  | **Pinned by seeds**, not left bare. This account receives the team fee leg, and on the vault-funded paths that leg is paid out by the … |
| 7 | `reflection_vault` | `["reflection_vault", market]` | ✓ |  | on reflection markets; on a `reflection_bps == 0` market (incl. every pre-upgrade market) it may not exist and is never touched. |
| 8 | `mint_token` |  | ✓ |  | Market token mint - the bought tokens are minted from it into the collateral vault. Validated to match `market.mint_token`. |
| 9 | `mint_main` |  |  |  | Base currency mint (e.g., USDC) - the buyer pays in this token. Validated to match `market.mint_main`. |
| 10 | `buyer_cash` |  | ✓ |  | Buyer's cash account - pays `cash_in`. Paying accounts must already exist (never `init_if_needed`); constrained to the `mint_main` mint. |
| 11 | `token_program_main` |  |  |  | Token program for base currency transfers. |
| 12 | `token_program` |  |  |  | Token program for the market token - used for the mint into the collateral vault. |
| 13 | `system_program` |  |  |  | System program for position account creation. |
| 14 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 15 | `program` |  |  |  | This program — target of the event self-CPI. |

---

### 8. `leverage_sell`

Sell collateral without releasing it, repay the debt from the proceeds, and pay out the remainder. Closes a position whose debt exceeds the holder's free cash.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `collateral_in` | `u64` | Locked collateral tokens to sell (raw). |
| `decrease_debt_by` | `u64` | Debt repaid out of the proceeds before any payout (raw collateral units). |
| `min_cash_out` | `u64` | Slippage gate on the net payout to the owner (raw). |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `owner` |  | ✓ | ✓ | Position owner - signs, receives the residual payout, and pays rent for `owner_cash` if it does not exist yet. |
| 1 | `market` | `["market", market.mint_token]` | ✓ |  | Rise market being unwound - holds every ledger and signs the burn and the vault payouts. PDA derived from ["market", mint_token]. |
| 2 | `position` | `["position", market, owner]` | ✓ |  | Owner's loan position - tracks collateral and debt. |
| 3 | `vault` |  | ✓ |  | Main cash vault - every payout leaves from here. |
| 4 | `collateral_vault` | `["collateral_vault", market]` | ✓ |  | Collateral vault PDA - holds the locked tokens; `collateral_in` is burned straight out of it, never released to the owner. |
| 5 | `creator_claim` |  | ✓ |  | Creator claim account - receives the creator fee leg. Pinned by an `address` check against the key stored at creation. |
| 6 | `team_vault` | `["team_vault", mint_main]` | ✓ |  | **Pinned by seeds**, not left bare. This account receives the team fee leg, and on the vault-funded paths that leg is paid out by the … |
| 7 | `reflection_vault` | `["reflection_vault", market]` | ✓ |  | on reflection markets; on a `reflection_bps == 0` market (incl. every pre-upgrade market) it may not exist and is never touched. |
| 8 | `mint_token` |  | ✓ |  | Market token mint - the burn target. Validated to match `market.mint_token`. |
| 9 | `mint_main` |  |  |  | Base currency mint (e.g., USDC) - all payouts are in this token. Validated to match `market.mint_main`. |
| 10 | `owner_cash` | `[owner, token_program_main, mint_main]` | ✓ |  | Receives the residual payout — created on the fly if the owner has never held the collateral (init_if_needed, rent paid by the owner). |
| 11 | `token_program_main` |  |  |  | Token program for base currency transfers. |
| 12 | `token_program` |  |  |  | Token program for the market token - used for the collateral burn. |
| 13 | `associated_token_program` |  |  |  | Associated token program - creates `owner_cash` on first use. |
| 14 | `system_program` |  |  |  | System program for account creation. |
| 15 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 16 | `program` |  |  |  | This program — target of the event self-CPI. |

---

## Market lifecycle

### 9. `init_market`

Permissionless market creation. Mint + vaults + curve params + snapshot.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `args` | `InitMarketArgs` |  |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `creator` |  | ✓ | ✓ | Market creator - pays rent for every account created here and is recorded as `market.creator` (the initial claimant of creator fees). |
| 1 | `config` | `["config"]` |  |  | Global config PDA - read for `current_fee_config_version` to pin the fee config below. Seeds guarantee it is the program's singleton. |
| 2 | `fee_config` | `["fee_config", fee_config.version]` |  |  | Pinned at creation and frozen for the market's life. |
| 3 | `collateral_config` | `["collateral", mint_main]` |  |  | The allowlist gate. Its mere existence is the permission: no `CollateralConfig`, no market — and this account is read here, once, then … |
| 4 | `mint_main` |  |  |  | Base currency mint (e.g., USDC) - the market's cash side. |
| 5 | `mint_token` | `["mint", arg:args.vanity_seed]` | ✓ |  | Token-2022 metadata extension, and we deliberately allocate the *maximum* so a CTO metadata rewrite never needs a realloc. |
| 6 | `market` | `["market", mint_token]` | ✓ |  | Rise market state being created - the immutable snapshot of curve, fee and glide parameters. |
| 7 | `vault` | `["vault", market]` | ✓ |  | The single cash vault. `floor_reserve` and `price_reserve` are ledgers on `Market` pointing into this one account, never separate vaults. |
| 8 | `creator_claim` | `["creator_claim", market]` | ✓ |  | Claimable only by whoever is *currently* `market.creator`, so a CTO hands the market over with its unclaimed earnings. |
| 9 | `collateral_vault` | `["collateral_vault", market]` | ✓ |  | not exist yet when Anchor evaluates constraints. |
| 10 | `reflection_vault` | `["reflection_vault", market]` | ✓ |  | Reflection-reward vault (collateral). Created for every market — holds the reflection tax until distribution; stays empty on a … |
| 11 | `token_program_main` |  |  |  | Token program owning `mint_main` (Token or Token-2022) - used for the cash vault and creator claim accounts. |
| 12 | `token_program` |  |  |  | Always Token-2022: the market mint carries the metadata extension. |
| 13 | `system_program` |  |  |  | System program for account creation. |
| 14 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 15 | `program` |  |  |  | This program — target of the event self-CPI. |

---

### 10. `claim_creator_fees`

Sweep a market's accrued creator fees. Signed by whoever is **currently** `market.creator`, so a CTO hands the market over with its unclaimed earnings.

**Args** — none

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `creator` |  | ✓ | ✓ | Whoever is **currently** `market.creator`. |
| 1 | `market` | `["market", market.mint_token]` |  |  | Market whose fees are being claimed - source of truth for `creator` and vault authority. PDA derived from ["market", mint_token]. |
| 2 | `creator_claim` | `["creator_claim", market]` | ✓ |  | Creator-fee vault - accumulates the creator's fee share for this market; the market PDA is its transfer authority. |
| 3 | `mint_main` |  |  |  | Base currency mint (e.g., USDC) - validated to match `market.mint_main`, so the vault, destination, and `transfer_checked` all agree on the … |
| 4 | `destination` | `[creator, token_program_main, mint_main]` | ✓ |  | The creator's canonical collateral ATA — created on the fly on the first claim (init_if_needed, rent paid by the creator). |
| 5 | `token_program_main` |  |  |  | Token program for base currency transfers. |
| 6 | `associated_token_program` |  |  |  | Associated token program - required to create `destination` on first claim. |
| 7 | `system_program` |  |  |  | System program for account creation (destination ATA rent). |
| 8 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 9 | `program` |  |  |  | This program — target of the event self-CPI. |

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
| `y_ref` | i128 | **Q64.64** reference shift the self-healing spread is measured against. Frozen for the whole sell descent (buys no longer reset it); reset to `y_shift` by a preserve-area raise or a landing at zero supply. |
| `heal_low_x` | u64 | Low-water mark of the current sell descent, raw supply units. `0` = fresh descent. Tokens below it are valued against `y_ref`, tokens bought back since against `y_shift`. |
| `heal_locked` | u64 | Floor cash credited by self-healing sales and not yet consumed by a raise. A later seller's glide share cannot draw it. |

### Ledgers & analytics

| Field | Type | Notes |
|---|---|---|
| `token_supply` | u64 | Circulating supply = `x` |
| `floor_reserve`, `price_reserve` | u64 | The two ledgers inside `vault` |
| `total_debt`, `total_collateral` | u64 | Lending aggregates |
| `debt_from_price` | u64 | Part of `total_debt` drawn from the price reserve — excluded from the floor backing (`floor_reserve + total_debt − debt_from_price`) |
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

### 11. `open_reflection_epoch`

Open a reflection distribution epoch: skim the gas reserve to the treasury, freeze the (net) pool, and size the paid bitmap to the snapshot's holders.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `epoch_id` | `u64` | Free-form epoch identifier chosen by the admin (the backend uses the day number). |
| `holder_count` | `u32` | Number of holders in this epoch's snapshot — sizes the paid bitmap. |
| `gas_reserve` | `u64` | Lamports skimmed for distribution gas (SOL-collateral markets only). |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `admin` |  | ✓ | ✓ |  |
| 1 | `config` | `["config"]` |  |  |  |
| 2 | `market` | `["market", market.mint_token]` |  |  |  |
| 3 | `reflection_vault` | `["reflection_vault", market]` | ✓ |  | The reflection pool. Skim source; frozen (net) into the epoch below. |
| 4 | `mint_main` |  |  |  |  |
| 5 | `admin_destination` | `[admin, token_program_main, mint_main]` | ✓ |  | The admin's own ATA for `mint_main`: receives the `gas_reserve` skim so the admin (who pays the distribution gas out of its own SOL) is … |
| 6 | `unwrap_holding` | `["reflection_unwrap", market]` | ✓ |  | Transient collateral holding account, created and closed within this instruction. |
| 7 | `reflection_epoch` | `["reflection_epoch", market, arg:epoch_id]` | ✓ |  |  |
| 8 | `token_program_main` |  |  |  | Token program of the collateral mint. |
| 9 | `associated_token_program` |  |  |  | Required for `init_if_needed` ATAs. |
| 10 | `system_program` |  |  |  | Account creation / rent. |
| 11 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 12 | `program` |  |  |  | This program — target of the event self-CPI. |

---

### 12. `distribute_reflection`

Pay a batch of holders from the reflection vault. Bitmap-idempotent and bounded by the frozen epoch pool. Holder ATAs ride in `remaining_accounts`.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `epoch_id` | `u64` | Epoch being paid. |
| `entries` | `Vec<ReflectionPayout>` | Batch of `{index, amount}` payouts; each index is marked in the paid bitmap before transfer (replay-safe). |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `admin` |  |  | ✓ |  |
| 1 | `config` | `["config"]` |  |  |  |
| 2 | `market` | `["market", market.mint_token]` |  |  |  |
| 3 | `reflection_vault` | `["reflection_vault", market]` | ✓ |  |  |
| 4 | `reflection_epoch` | `["reflection_epoch", market, arg:epoch_id]` | ✓ |  |  |
| 5 | `mint_main` |  |  |  |  |
| 6 | `token_program_main` |  |  |  | Token program of the collateral mint. |
| 7 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 8 | `program` |  |  |  | This program — target of the event self-CPI. |

---

### 13. `close_reflection_epoch`

Close a finished reflection epoch, returning its rent to the admin.

**Args**

| Name | Type | Meaning |
|---|---|---|
| `epoch_id` | `u64` | Epoch to close once fully settled; rent returns to the admin. |

**Accounts**

| # | Account | Seeds / source | W | S | Notes |
|---|---|---|---|---|---|
| 0 | `admin` |  | ✓ | ✓ |  |
| 1 | `config` | `["config"]` |  |  |  |
| 2 | `market` | `["market", market.mint_token]` |  |  |  |
| 3 | `reflection_epoch` | `["reflection_epoch", market, arg:epoch_id]` | ✓ |  | Closed here; its rent lamports return to `admin`. |
| 4 | `event_authority` |  |  |  | Anchor event-CPI self-invoke authority (fixed PDA). |
| 5 | `program` |  |  |  | This program — target of the event self-CPI. |

---

## Reflection

`reflection_bps ∈ {0, 100, 300}` (0 / 1 / 3 %), pinned at creation, immutable. It is an extra leg **on top of** the trade fee, routed to a separate `reflection_vault` PDA (`["reflection_vault", market]`) — it never enters `floor_reserve` / `price_reserve`.

- **Buy**: `reflection = ceil(cash_in × reflection_bps / 10_000)`, removed from the top (with the fee) **before** the curve quote — so the buyer receives fewer tokens.
- **Sell**: `reflection = ceil(gross × reflection_bps / 10_000)`, carved out of the **payout** — so the seller receives less cash.

Accrued reflection is later distributed to holders pro-rata across epochs (`open` → `distribute` → `close`). When quoting a trade on a reflection market, apply the leg exactly as above or the quote will overstate the fill by the reflection percentage.

---

## See also

- [**Indexing & Events (Pro)**](./INDEXING_PRO.md) — event layouts, parsing, Q64 conversion.
- [IDL (JSON)](../idl/rise_pro.json)

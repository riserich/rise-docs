# Rise Program Documentation

**Rise Program IDs:**
- **Mainnet:** `RiseZSHaLdj7pfn1tisUoSdG2i3QcVz9sQKuaRG9rar`
- **Devnet:** `7gDn1L2Bmg53royeUgvZtWujfvxS9TmpchtBToP9zDhB`

**Architecture:** The Rise program makes CPI (Cross-Program Invocation) calls to the Mayflower program for market operations, liquidity management, and collateral handling.

**Mayflower Program IDs:**
- **Mainnet:** `AVMmmRzwc2kETQNhPiFVnyu62HrgsQXTD6D7SnSfEz7v`
- **Devnet:** `MD2pPJCjpUT5ttJFUVeP2Xka1ZSvCJMZUoX4XTdPdet`

---

## Summary

| | Section |
|--|---------|
| 1 | [IDL Files](#idl-files) |
| 2 | [Common Types](#common-types) |
| 3 | [PDA Seeds Reference](#pda-seeds-reference) |
| 4 | [Instructions](#instructions) |
| | [`initMarket`](#1-initmarket) — Create a new token market |
| | [`buyWithExactCashIn`](#2-buywithexactcashin) — Buy tokens |
| | [`sellWithExactTokenIn`](#3-sellwithexacttokenin) — Sell tokens |
| | [`deposit`](#4-deposit) — Deposit tokens as collateral |
| | [`withdraw`](#5-withdraw) — Withdraw collateral |
| | [`borrow`](#6-borrow) — Borrow against collateral |
| | [`repay`](#7-repay) — Repay debt |
| | [`initPersonalAccount`](#8-initpersonalaccount) — Init user account for a market |
| | [`withdrawCreatorFees`](#9-withdrawcreatorfees) — Withdraw creator fees |
| 5 | [Account Structures](#account-structures) |
| 6 | [Fee Distribution](#fee-distribution) |
| 7 | [Important Notes](#important-notes) |

---

## IDL Files

The IDLs files are available in the `idl/` directory:

- **`idl/idl.json`** - JSON format IDL file
- **`idl/idl.ts`** - TypeScript type definitions

These files contain the complete program interface including all instructions, accounts, types, and events.

## Common Types

### GovInitArgs
Governance initialization arguments that configure market parameters:
- `buyFeeMicroBasisPoints` - Buy transaction fee (in micro basis points, e.g., 10000 = 1%)
- `sellFeeMicroBasisPoints` - Sell transaction fee
- `borrowFeeMicroBasisPoints` - Borrow transaction fee
- `floorRaiseCooldownSeconds` - Cooldown period for floor raises
- `floorRaiseLiquidityBufferMicroBasisPoints` - Liquidity buffer for floor raises
- `floorInvestmentMicroBasisPoints` - Floor investment share of revenue
- `priceCurveSensitivityChangeRateMicroBasisPoints` - Price curve sensitivity change rate

### DecimalSerialized
High-precision decimal value serialized as a 16-byte array. Used for financial calculations requiring exact precision (e.g., price curve parameters, fees). Preserves full 128-bit precision across serialization.

### TokenMetadata
Token metadata structure containing:
- `name` - Full token name (string)
- `symbol` - Token symbol (string)
- `uri` - URI to the token metadata JSON file

---

## PDA Seeds Reference

### Rise Program PDAs

**Program IDs:** Mainnet `RiseZSHaLdj7pfn1tisUoSdG2i3QcVz9sQKuaRG9rar` · Devnet `7gDn1L2Bmg53royeUgvZtWujfvxS9TmpchtBToP9zDhB`

- **Tenant**: `["tenant", seed, bump]`
- **Market**: `["market", rise_tenant, market_meta, bump]`
- **PersonalAccount**: `["personal_account", market, owner, bump]`
- **CashEscrow**: `["cash_escrow", rise_market, bump]`
- **CreatorEscrow**: `["creator_escrow", rise_market, bump]`
- **TeamEscrow**: `["team_escrow", mint_main, bump]`
- **TeamConfig**: `["team_config", bump]` (global config storing team wallet address)
- **MintToken**: `[vanity_seed.to_le_bytes(), bump]` (must end with "rise")

### Mayflower Program PDAs

**Program IDs:** Mainnet `AVMmmRzwc2kETQNhPiFVnyu62HrgsQXTD6D7SnSfEz7v` · Devnet `MD2pPJCjpUT5ttJFUVeP2Xka1ZSvCJMZUoX4XTdPdet`

- **Tenant**: `["tenant", seed_address, bump]`
- **MarketGroup**: `["market_group", seed_address, bump]`
- **Market**: `["market", seed_address, bump]`
- **MarketMeta**: `["market_meta", seed_address, bump]`
- **MarketLinear**: `["market_linear", market_meta_address, bump]`
- **MintOptions**: `["mint_options", market_meta_address, bump]`
- **LiqVaultMain**: `["liq_vault_main", market_meta_address, bump]`
- **RevEscrowGroup**: `["rev_escrow_group", market_meta_address, bump]`
- **RevEscrowTenant**: `["rev_escrow_tenant", market_meta_address, bump]`
- **PersonalPosition**: `["personal_position", market_meta_address, owner, bump]`
- **PersonalPositionEscrow**: `["personal_position_escrow", personal_position_address, bump]`
- **LogAccount**: `["log", bump]`

---

## Instructions

### 1. `initMarket`
**Purpose:** Create a new token market.

**Accounts:**
- `payer` (mut, signer) - Account paying for transaction
- `seed` (signer) - Seed address for PDAs
- `tenantSeed` - Seed address for rise tenant PDA
- `riseMarket` (mut) - Market PDA to create
- `mintMain` - Main currency mint (interface)
- `mintToken` (mut) - Token mint to create (must end with "rise")
- `tokenMetadataProgram` - Metaplex token metadata program
- `metadata` (mut) - Token metadata account
- `marketMeta` (mut) - Mayflower market meta
- `riseTenant` (mut) - Rise tenant
- `mayflowerTenant` - Mayflower tenant
- `cashEscrow` (mut) - Cash escrow PDA
- `creatorEscrow` (mut) - Creator escrow PDA
- `systemProgram` - System program
- `mayflowerProgram` - Mayflower program
- `tokenProgram` - Token program
- `tokenProgramMain` - Token program for main token

**Arguments:**
- `vanity_seed: u64` - Seed for mint token PDA
- `args: InitMarketArgs`
  - `gov: GovInitArgs`
  - `x2: u64`
  - `m2: DecimalSerialized`
  - `m1: DecimalSerialized`
  - `f: DecimalSerialized`
  - `b2: DecimalSerialized`
  - `startTime: u64`
  - `dutchConfigInitBoost: f64`
  - `dutchConfigDuration: u32`
  - `dutchConfigCurvature: f64`
  - `metadata: TokenMetadata`

**PDA Seeds:**
- **Market**: `["market", rise_tenant, market_meta, bump]`
- **MintToken**: `[vanity_seed.to_le_bytes(), bump]`
- **CashEscrow**: `["cash_escrow", rise_market, bump]`
- **CreatorEscrow**: `["creator_escrow", rise_market, bump]`

---

### 2. `buyWithExactCashIn`
**Purpose:** Buy tokens with exact cash input on the bonding curve.

Purchases tokens by spending exact amount of cash. Optionally raises the floor price in the same transaction if `newShoulderEnd` is non-zero. Distributes trading fees to creator and team escrows.

**Accounts:**
- `buyer` (mut, signer) - Buyer's wallet (pays cash_in)
- `tenant` (mut) - Rise tenant (needed for floor raise operations)
- `market` (mut) - Market account
- `cashEscrow` (mut) - Cash escrow for revenue distribution
- `mayTenant` - Mayflower tenant
- `mayMarketGroup` - Mayflower market group
- `marketMeta` - Mayflower market meta
- `mayMarket` (mut) - Mayflower market with bonding curve state
- `tenantSeed` - Seed for Rise tenant PDA derivation
- `mintToken` (mut) - Token mint (tokens will be minted to buyer)
- `mintMain` - Main token mint (buyer pays in this token)
- `tokenDst` (mut) - Destination token account (buyer's wallet)
- `mainSrc` (mut) - Source main token account (buyer's wallet)
- `liqVaultMain` (mut) - Liquidity vault for main token
- `revEscrowGroup` (mut) - Revenue escrow for group
- `revEscrowTenant` (mut) - Revenue escrow for tenant
- `tokenProgramMain` - Token program for main token
- `tokenProgram` - Token program
- `mayflowerProgram` - Mayflower program
- `mayLogAccount` (mut) - Mayflower log account
- `creatorEscrow` (mut) - Creator escrow (receives creator's share of buy fees)
- `teamEscrow` (mut) - Team escrow (receives team's share of buy fees)

**Arguments:**
- `cashIn: u64` - Exact amount of base currency to spend
- `minTokenOut: u64` - Minimum tokens to receive (slippage protection)
- `newShoulderEnd: u64` - New shoulder position for floor raise (0 to skip)
- `floorIncreaseRatio: DecimalSerialized` - Ratio to increase floor price by

---

### 3. `sellWithExactTokenIn`
**Purpose:** Sell tokens for exact cash output.

**Accounts:**
- `seller` (mut, signer) - Seller's wallet
- `tenant` (mut) - Rise tenant
- `market` (mut) - Market account
- `cashEscrow` (mut) - Cash escrow
- `mayTenant` - Mayflower tenant
- `mayMarketGroup` - Mayflower market group
- `marketMeta` - Mayflower market meta
- `mayMarket` (mut) - Mayflower market
- `mintToken` (mut) - Token mint
- `mintMain` - Main token mint
- `tokenSrc` (mut) - Source token account (seller's wallet)
- `mainDst` (mut) - Destination main token account (seller's wallet)
- `liqVaultMain` (mut) - Liquidity vault for main token
- `revEscrowGroup` (mut) - Revenue escrow for group
- `revEscrowTenant` (mut) - Revenue escrow for tenant
- `tokenProgramMain` - Token program for main token
- `tokenProgram` - Token program
- `mayflowerProgram` - Mayflower program
- `mayLogAccount` (mut) - Mayflower log account
- `creatorEscrow` (mut) - Creator escrow
- `teamEscrow` (mut) - Team escrow

**Arguments:**
- `tokenIn: u64` - Amount of tokens to sell
- `minCashOut: u64` - Minimum cash to receive

---

### 4. `deposit`
**Purpose:** Deposit tokens as collateral.

**Accounts:**
- `owner` (mut, signer) - Owner of the personal position
- `personalAccount` (mut) - Personal account
- `market` (mut) - Market account
- `marketMeta` - Mayflower market meta
- `mayMarket` (mut) - Mayflower market
- `corePersonalPosition` (mut) - Mayflower core personal position
- `mayEscrow` (mut) - Mayflower escrow account
- `mintToken` (mut) - Token mint
- `tokenSrc` (mut) - Source token account (owner's wallet)
- `tokenProgram` - Token program
- `mayflowerProgram` - Mayflower program
- `mayLogAccount` (mut) - Mayflower log account

**Arguments:**
- `amount: u64` - Amount of tokens to deposit

**PDA Seeds:**
- **PersonalAccount**: `["personal_account", market, owner, bump]`

---

### 5. `withdraw`
**Purpose:** Withdraw deposited tokens (requires no debt).

**Accounts:**
- `owner` (mut, signer) - Owner of the personal position
- `personalAccount` (mut) - Personal account
- `market` (mut) - Market account
- `marketMeta` - Mayflower market meta
- `mayMarket` (mut) - Mayflower market
- `corePersonalPosition` (mut) - Mayflower core personal position
- `mayEscrow` (mut) - Mayflower escrow account
- `mintToken` (mut) - Token mint
- `tokenDst` (mut) - Destination token account (owner's wallet)
- `tokenProgram` - Token program
- `mayflowerProgram` - Mayflower program
- `mayLogAccount` (mut) - Mayflower log account

**Arguments:**
- `amount: u64` - Amount of tokens to withdraw

---

### 6. `borrow`
**Purpose:** Borrow collateral asset against deposited tokens.

**Accounts:**
- `owner` (mut, signer) - Owner of the personal position
- `tenant` (mut) - Rise tenant
- `market` (mut) - Market account
- `cashEscrow` (mut) - Cash escrow
- `personalAccount` (mut) - Personal account
- `mayTenant` - Mayflower tenant
- `mayMarketGroup` - Mayflower market group
- `marketMeta` - Mayflower market meta
- `liqVaultMain` (mut) - Liquidity vault for main token
- `revEscrowGroup` (mut) - Revenue escrow for group
- `revEscrowTenant` (mut) - Revenue escrow for tenant
- `mayMarket` (mut) - Mayflower market
- `mintMain` - Main token mint
- `corePersonalPosition` (mut) - Mayflower core personal position
- `mainDst` (mut) - Destination main token account (owner's wallet)
- `tokenProgramMain` - Token program for main token
- `mayLogAccount` (mut) - Mayflower log account
- `mayflowerProgram` - Mayflower program
- `creatorEscrow` (mut) - Creator escrow
- `teamEscrow` (mut) - Team escrow

**Arguments:**
- `amount: u64` - Amount of main token to borrow

---

### 7. `repay`
**Purpose:** Repay borrowed amount.

**Accounts:**
- `repayer` (mut, signer) - Account repaying the loan
- `marketMeta` - Mayflower market meta
- `mayMarket` (mut) - Mayflower market
- `corePersonalPosition` (mut) - Mayflower core personal position
- `mintMain` - Main token mint
- `mainSrc` (mut) - Source main token account (repayer's wallet)
- `liqVaultMain` (mut) - Liquidity vault for main token
- `tokenProgramMain` - Token program for main token
- `mayflowerProgram` - Mayflower program
- `mayLogAccount` (mut) - Mayflower log account

**Arguments:**
- `amount: u64` - Amount of main token to repay

---

### 8. `initPersonalAccount`
**Purpose:** Initialize a personal account for a user in a specific market.

Each user needs one personal account per market to track their collateral and debt. The personal account links to a Mayflower personal position for collateral management.

**Important:** This is **NOT required for buy/sell**. You only need to call this before using deposit, borrow, withdraw, or leverage operations. The deposit endpoint automatically creates the personal account if it doesn't exist.

**Accounts:**
- `owner` (mut, signer) - The user who will own the personal account (pays for creation)
- `market` (mut) - The Rise market for this account
- `personalAccount` (mut) - Rise personal account PDA to initialize
- `corePersonalPosition` (mut) - Mayflower personal position PDA
- `marketMeta` - Mayflower market meta
- `mintToken` - Market token mint
- `coreEscrow` (mut) - Mayflower escrow for user's collateral
- `tokenProgram` - Token program
- `mayflowerProgram` - Mayflower program for CPI
- `systemProgram` - System program
- `mayLogAccount` (mut) - Mayflower log account

**Arguments:** None

**PDA Seeds:**
- **PersonalAccount**: `["personal_account", market, owner, bump]`

---

### 9. `withdrawCreatorFees`
**Purpose:** Withdraw accumulated creator fees.

**Accounts:**
- `creator` (mut, signer) - Creator of the market
- `market` (mut) - Market account
- `creatorEscrow` (mut) - Creator escrow PDA
- `creatorTokenAccount` (mut) - Creator's token account
- `mintMain` - Main token mint
- `tokenProgramMain` - Token program
- `associatedTokenProgram` - Associated token program
- `systemProgram` - System program

**Arguments:** None

**PDA Seeds:**
- **CreatorEscrow**: `["creator_escrow", market, bump]`

---

---

## Account Structures

### Market
```rust
pub struct Market {
    pub tenant: Pubkey,
    pub market_meta: Pubkey,
    pub mint_token: Pubkey,
    pub mint_main: Pubkey,
    pub token_decimals: u8,
    pub cash_escrow: Pubkey,
    pub gov: Gov,
    pub bump: [u8; 1],
    pub last_floor_raise_timestamp: u64,
    pub level: u32,
    pub level_rev_calculator: LevelRevCalculator,
    pub flags: u16,
    pub creator: Pubkey,
    pub total_fees_floor: u64,
    pub total_fees_creator: u64,
    pub total_fees_creator_withdrawn: u64,
    pub total_fees_team: u64,
}
```

### PersonalAccount
```rust
pub struct PersonalAccount {
    pub owner: Pubkey,
    pub market: Pubkey,
    pub core_personal_position: Pubkey,
    pub last_seen_rev_index: [u8; 16],
    pub staged_rev: [u8; 16],
    pub bump: [u8; 1],
    pub version: u8,
}
```

---

## Fee Distribution

Fees are collected on:
- **Buy transactions**: Percentage of cash input
- **Sell transactions**: Percentage of cash output
- **Borrow transactions**: Percentage of borrowed amount

---

## Important Notes

1. **Mint Token Constraint**: Token mint addresses must end with "rise" (case-insensitive)
2. **PDA Authority**: Market PDA is the authority for the mint_token
3. **Escrow Accounts**: Cash and creator escrows are PDAs owned by the program
4. **Personal Accounts**: Required before depositing, borrowing, or using collateral features (created automatically on first use)
5. **Debt Requirement**: Users must have no outstanding debt before withdrawing collateral

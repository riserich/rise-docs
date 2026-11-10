# Rise Protocol — Integration Documentation

Welcome to the Rise integration docs. Choose the path that fits your needs:

## Quick API Integration

Use our REST API to buy/sell Rise tokens in minutes. No Solana knowledge needed beyond signing transactions.

**Best for:** Bots, aggregators, frontends that want to integrate Rise trading fast.

→ [**Quick API Integration Guide**](./docs/API.md)

## SDK Integration

Use our TypeScript SDK to fetch market data, get quotes, and build buy/sell transactions. Fully standalone — no backend dependency, just an RPC URL.

**Best for:** Bots, aggregators, and anyone who wants full control with minimal setup.

```bash
npm install @riserich/sdk
```

→ <a href="https://www.npmjs.com/package/@riserich/sdk" target="_blank">**https://www.npmjs.com/package/@riserich/sdk**</a>

## API Base URLs

| Environment | URL |
|---|---|
| **Mainnet** | `https://public.rise.rich` |
| **Devnet** | `https://publicdev.rise.rich` |

## On-Chain Integration (SDK + IDL)

Build directly on the Rise Solana program. Full control over transactions, accounts and data indexing.

**Best for:** Trading terminals, custom UX, high-frequency systems, or advanced integrations.

→ [**On-Chain Program Guide**](./docs/PROGRAM.md)
→ [**Indexing & Events**](./docs/INDEXING.md)
→ [**IDL (JSON)**](./idl/idl.json)
→ [**IDL (TypeScript)**](./idl/idl.ts)

## Rise Pro (v2)

The next-generation single-program design: hinged-exponential curve, floor ledger, lending, leverage, floor raises and holder reflection rewards — all in one Anchor program (`Market.version == 2`).

→ [**Program (Pro)**](./docs/PROGRAM_PRO.md)
→ [**Indexing & Events (Pro)**](./docs/INDEXING_PRO.md)
→ [**IDL (JSON)**](./idl/rise_pro.json)

## Program IDs

| | Address |
|---|---|
| **Rise Pro (Devnet)** | `9nJ9wghEHY8bnsJQYrqV77uC9y4ccFHcCT1XyJxAkBVR` |
| **Rise Program v1 (Mainnet)** | `RiseZSHaLdj7pfn1tisUoSdG2i3QcVz9sQKuaRG9rar` |
| **Rise Program v1 (Devnet)** | `7gDn1L2Bmg53royeUgvZtWujfvxS9TmpchtBToP9zDhB` |
| **Mayflower (Devnet)** | `MD2pPJCjpUT5ttJFUVeP2Xka1ZSvCJMZUoX4XTdPdet` |
| **Mayflower (Mainnet)** | `AVMmmRzwc2kETQNhPiFVnyu62HrgsQXTD6D7SnSfEz7v` |

## Links

- <a href="https://docs.rise.rich" target="_blank">General Documentation</a>
- **Website:** <a href="https://www.rise.rich/" target="_blank">rise.rich</a>
- **Twitter/X:** <a href="https://x.com/risedotrich" target="_blank">risedotrich</a>

**Team Contact (Telegram):** @Passoif · @OxSahand

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

## Program IDs

| | Address |
|---|---|
| **Rise Program (Mainnet)** | `RiseZSHaLdj7pfn1tisUoSdG2i3QcVz9sQKuaRG9rar` |
| **Rise Program (Devnet)** | `7gDn1L2Bmg53royeUgvZtWujfvxS9TmpchtBToP9zDhB` |
| **Mayflower (Devnet)** | `MD2pPJCjpUT5ttJFUVeP2Xka1ZSvCJMZUoX4XTdPdet` |
| **Mayflower (Mainnet)** | `AVMmmRzwc2kETQNhPiFVnyu62HrgsQXTD6D7SnSfEz7v` |

## Links

- <a href="https://docs.rise.rich" target="_blank">General Documentation</a>
- **Website:** <a href="https://www.rise.rich/" target="_blank">rise.rich</a>
- **Twitter/X:** <a href="https://x.com/risedotrich" target="_blank">risedotrich</a>

**Team Contact (Telegram):** @Passoif · @OxSahand

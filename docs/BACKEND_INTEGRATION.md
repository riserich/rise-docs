# Backend Code Changes for Integration Mode

Code changes needed in `uponly-backend` to support the public integration API.

When `INTEGRATION=true`, only the 8 documented endpoints are accessible. All requests require a valid API key via the `x-api-key` header. Rate limits stay the same as the classic backend.

---

## 1. Add env variables to `src/env.ts`

```ts
INTEGRATION:
  process.env.INTEGRATION === "true" || process.env.INTEGRATION === "1",
INTEGRATION_API_KEYS: (process.env.INTEGRATION_API_KEYS || "")
  .split(",")
  .map((k) => k.trim())
  .filter(Boolean),
```

---

## 2. Create API key middleware `src/middleware/apiKeyAuth.ts`

```ts
import { Request, Response, NextFunction } from "express";
import { ENV } from "../env.js";

export function apiKeyAuth(req: Request, res: Response, next: NextFunction) {
  const apiKey = req.headers["x-api-key"] as string;

  if (!apiKey || !ENV.INTEGRATION_API_KEYS.includes(apiKey)) {
    return res.status(401).json({ ok: false, error: "Invalid or missing API key" });
  }

  next();
}
```

---

## 3. Update `src/routes/index.ts` — Integration mode routing

When `INTEGRATION` is true, apply `apiKeyAuth` and only mount the 3 routers needed for the 8 public endpoints:

```ts
import { ENV, SERVICE_MODE } from "../env.js";
import { apiKeyAuth } from "../middleware/apiKeyAuth.js";

// ... existing imports ...

if (ENV.INTEGRATION) {
  // Integration mode: API key required, only expose documented endpoints
  // - GET  /markets/:id
  // - GET  /markets/:id/transactions
  // - GET  /markets/:id/ohlc/:timeframe
  // - POST /markets/:id/quote
  // - POST /program/buyToken
  // - POST /program/sellToken
  // - GET  /users/:addr/portfolio/summary
  // - GET  /users/:addr/portfolio/positions
  router.use(apiKeyAuth);
  router.use("/markets", marketsRouter);
  router.use("/program", programRouter);
  router.use("/users", userPortfolioRouter);
} else if (SERVICE_MODE === "api" || SERVICE_MODE === "all") {
  // Normal mode: all routes, no API key
  router.use("/markets", marketsRouter);
  router.use("/program", programRouter);
  router.use("/personal_position", personalPositionsRouter);
  router.use("/users", userPortfolioRouter);
  router.use("/waitlist", waitlistRouter);
  router.use("/user", userRouter);
}
```

Rate limits are unchanged — same config as `INTEGRATION=false`.

Note: In integration mode, all program routes are still technically mounted, but integrators only need `buyToken` and `sellToken`. The other program endpoints (create, deposit, borrow, etc.) won't be documented.

---

## 4. Resolve `riseMarket` in `buyToken` endpoint (`src/routes/program.ts`)

In the `/buyToken` handler, after the `getMarketByAddress` call resolves `marketData`, add one line to resolve the canonical `rise_market_address`. This allows integrators to pass either the token mint address OR the rise market address as the `market` parameter.

**Location:** After the `if (!marketData)` check (around line 1939)

```ts
// Resolve canonical rise_market_address (allows passing mint_token as market)
riseMarket = new PublicKey(marketData.rise_market_address);
```

This is needed because `riseMarket` is used for PDA derivation (`creatorEscrow` at line 2011). Without this fix, passing a `mint_token` would derive incorrect PDAs and the transaction would fail.

---

## 5. Same resolution in `sellToken` endpoint (`src/routes/program.ts`)

Same change in the `/sellToken` handler, after the `if (!marketData)` check (around line 2653):

```ts
// Resolve canonical rise_market_address (allows passing mint_token as market)
riseMarket = new PublicKey(marketData.rise_market_address);
```

Same reason: `riseMarket` is used for `creatorEscrow` PDA derivation at line 2709.

---

## 6. `.env` variables

```env
INTEGRATION=true
INTEGRATION_API_KEYS=key1,key2,key3
```

---

## Summary of changes

| File | Change |
|------|--------|
| `src/env.ts` | Add `INTEGRATION` and `INTEGRATION_API_KEYS` |
| `src/middleware/apiKeyAuth.ts` | New file — API key check on `x-api-key` header |
| `src/routes/index.ts` | Integration mode: apiKeyAuth + only mount 3 routers |
| `src/routes/program.ts` (buyToken) | Resolve `riseMarket` from DB after lookup |
| `src/routes/program.ts` (sellToken) | Resolve `riseMarket` from DB after lookup |

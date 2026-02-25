# Hyperliquid Info API Reference

> **Backtesting note**: This reference is for live metadata and user state lookups. It is **not** a source of tick-level historical data for `supurr backtest`.

> **Get address via**: `supurr whoami` — returns the configured wallet address.

All endpoints use `POST https://api.hyperliquid.xyz/info` with `Content-Type: application/json`.

---

## Market Data (No Address Required)

| Query            | Request Body                         |
| ---------------- | ------------------------------------ |
| All Mid Prices   | `{"type": "allMids"}`                |
| Sub-DEX Prices   | `{"type": "allMids", "dex": "hyna"}` |
| Perp Metadata    | `{"type": "metaAndAssetCtxs"}`       |
| Spot Metadata    | `{"type": "spotMeta"}`               |
| L2 Order Book    | `{"type": "l2Book", "coin": "BTC"}`  |
| List HIP-3 DEXes | `{"type": "perpDexs"}`               |

---

## User Data (Address Required)

| Query           | Request Body                                                                   |
| --------------- | ------------------------------------------------------------------------------ |
| Perp Positions  | `{"type": "clearinghouseState", "user": "0x..."}`                              |
| Spot Balances   | `{"type": "spotClearinghouseState", "user": "0x..."}`                          |
| Open Orders     | `{"type": "openOrders", "user": "0x..."}`                                      |
| Order History   | `{"type": "historicalOrders", "user": "0x..."}`                                |
| Trade Fills     | `{"type": "userFills", "user": "0x...", "aggregateByTime": true}`              |
| Funding History | `{"type": "userFunding", "user": "0x...", "startTime": <ts>, "endTime": <ts>}` |
| Sub-Accounts    | `{"type": "subAccounts", "user": "0x..."}`                                     |
| Vault Details   | `{"type": "vaultDetails", "vaultAddress": "0x..."}`                            |

---

## HIP-3 Sub-DEXes

| DEX    | Quote | Assets                        |
| ------ | ----- | ----------------------------- |
| `hyna` | USDE  | Crypto perps (BTC, ETH, HYPE) |
| `xyz`  | USDE  | Stocks (AAPL, TSLA, NVDA)     |
| `vntl` | USDE  | AI/tech tokens                |
| `km`   | USDT  | Kinetiq Markets               |
| `cash` | USDC  | Tech stocks                   |

---

## TypeScript Helper

```typescript
const HL = "https://api.hyperliquid.xyz/info";

async function query<T>(body: object): Promise<T> {
  const res = await fetch(HL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  return res.json();
}

// Examples
const mids = await query({ type: "allMids" });
const positions = await query({ type: "clearinghouseState", user: "0x..." });
const spotBal = await query({ type: "spotClearinghouseState", user: "0x..." });
const dexes = await query({ type: "perpDexs" });
```

---

## Common Hazards

| Issue                   | Solution                                          |
| ----------------------- | ------------------------------------------------- |
| `szDecimals` truncation | Truncate qty to `szDecimals` before submit        |
| HIP-3 price prefix      | Sub-DEX prices keyed as `hyna:BTC`, not `BTC`     |
| Sub-DEX asset index     | Use local index from DEX's `universe`, not global |
| Fill limits             | `userFills` max 2000 — paginate with time ranges  |

# Complete Workflows

## Workflow 1: Grid — Backtest → Deploy → Monitor

```bash
# 1. Initialize (first time only)
supurr init --address 0x... --api-wallet 0x...

# 2. Create config
supurr new grid --asset BTC --levels 4 --start-price 88000 --end-price 92000 --investment 100 --leverage 20 --output btc-grid.json

# 3. Backtest
supurr backtest -c btc-grid.json -s 2026-01-28 -e 2026-02-01

# 4. Deploy
supurr deploy -c btc-grid.json

# 5. Monitor
supurr monitor --watch

# 6. Stop when done
supurr stop --id <bot_id>
```

---

## Workflow 2: Arb — Setup → Deploy → Monitor

```bash
# 1. Initialize
supurr init --address 0x... --api-wallet 0x...

# 2. Generate arb config (auto-resolves spot counterpart)
supurr new arb --asset BTC --amount 200 --leverage 1 --output btc-arb.json

# 3. Review the generated config
supurr config btc-arb

# 4. Ensure USDC balance in BOTH Spot and Perps wallets on Hyperliquid

# 5. Deploy
supurr deploy -c btc-arb.json

# 6. Monitor
supurr monitor --watch
```

---

## Workflow 3: DCA — Configure → Deploy

```bash
# 1. Create DCA config
supurr new dca --asset BTC --trigger-price 95000 --base-order 0.001 --max-orders 5 --take-profit 0.02 --output btc-dca.json

# 2. Deploy
supurr deploy -c btc-dca.json

# 3. Monitor
supurr monitor --watch
```

---

## Workflow 4: Test All Market Types

```bash
# Native Perp
supurr new grid --asset BTC --output native-btc.json
supurr backtest -c native-btc.json -s 2026-01-28 -e 2026-02-01

# USDC Spot
supurr new grid --asset HYPE --type spot --quote USDC --output hype-usdc.json
supurr backtest -c hype-usdc.json -s 2026-01-30 -e 2026-01-31

# Non-USDC Spot
supurr new grid --asset HYPE --type spot --quote USDH --output hype-usdh.json
supurr backtest -c hype-usdh.json -s 2026-01-30 -e 2026-01-31

# HIP-3
supurr new grid --asset BTC --type hip3 --dex hyna --output hyna-btc.json
supurr backtest -c hyna-btc.json -s 2026-01-28 -e 2026-02-01
```

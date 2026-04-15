# Prediction Market (Outcome) Trading Tutorial

Step-by-step guide to deploying a DCA bot on Hyperliquid's prediction markets.

> **⚠️ TESTNET ONLY**: Prediction markets (outcomes) are currently available only on Hyperliquid testnet. Do **not** attempt to deploy on mainnet — the market type does not exist there.

---

## What are Prediction Markets?

Hyperliquid's prediction markets let you trade on binary outcomes — events that resolve to either **Yes** or **No**. Each outcome trades between $0 and $1:

| Price | Meaning |
|:---:|:---|
| $0.80 | Market believes 80% chance of "Yes" |
| $0.20 | Market believes 20% chance of "Yes" |

**Examples**: "BTC > 69136 on March 15", "ETH > 5000 by April", etc.

### Key Concepts

- **Outcome ID** — Unique ID for each question (e.g., 726)
- **Side** — `0` = Yes, `1` = No
- **Encoding** — `10 × outcome_id + side` (e.g., 726 Yes = 7260)
- **Instrument** — Shows as `#7260-OUTCOME` in the bot
- **Asset ID** — `100,000,000 + encoding` (used internally for order placement)
- **Collateral** — USDC on the spot clearinghouse (not perp margin)

> **Important**: You need **USDC in your spot wallet** on testnet. Transfer via Hyperliquid's testnet UI.

---

## 1️⃣ Find an Outcome to Trade

Query the testnet API to see available prediction markets:

```bash
curl -s https://api.hyperliquid-testnet.xyz/info \
  -H 'Content-Type: application/json' \
  -d '{"type": "outcomeMeta"}' | python3 -m json.tool
```

This returns all outcomes with their IDs, names, and questions. Pick one and note the `outcome_id`.

---

## 2️⃣ Check the Current Price

```bash
# Replace #7260 with your encoding (10 × outcome_id + side)
curl -s https://api.hyperliquid-testnet.xyz/info \
  -H 'Content-Type: application/json' \
  -d '{"type": "l2Book", "coin": "#7260"}' | python3 -c "
import sys, json
d = json.load(sys.stdin)
asks = d['levels'][1][:2]
bids = d['levels'][0][:2]
for b in bids: print(f'BID: {b[\"px\"]}')
for a in asks: print(f'ASK: {a[\"px\"]}')
"
```

---

## 3️⃣ Create Config Manually

Prediction markets use a special `outcome` market type. The CLI's `supurr new dca` does not yet support outcome configs, so you need to create the JSON manually.

> [!NOTE]
> In the current V2 config format, each `markets[]` entry is the market object directly. Do not nest it under a separate `market` key.

Create a file `prediction-dca.json`:

```json
{
  "environment": "testnet",
  "strategy_type": "dca",
  "markets": [
    {
      "exchange": "hyperliquid",
      "type": "outcome",
      "name": "BTC > 69136",
      "outcome_id": 726,
      "side": 0,
      "instrument_meta": {
        "tick_size": "0.001",
        "lot_size": "1"
      }
    }
  ],
  "dca": {
    "direction": "long",
    "trigger_price": "0.85",
    "base_order_size": "20",
    "dca_order_size": "20",
    "max_dca_orders": 2,
    "size_multiplier": "1.5",
    "price_deviation_pct": "5",
    "deviation_multiplier": "1.0",
    "take_profit_pct": "5",
    "stop_loss": null,
    "leverage": "1",
    "max_leverage": "1",
    "restart_on_complete": false,
    "cooldown_period_secs": 60
  }
}
```

### Config Field Reference

| Field | Value | Notes |
|:---|:---|:---|
| `environment` | `"testnet"` | **Must be testnet** — outcomes don't exist on mainnet |
| `markets[0].type` | `"outcome"` | Signals outcome market routing |
| `markets[0].name` | Any string | Human-readable label, not used for routing |
| `markets[0].outcome_id` | `726` | From the `outcomeMeta` API response |
| `markets[0].side` | `0` or `1` | `0` = Yes, `1` = No |
| `tick_size` | `"0.001"` | Use the live market metadata for the exact tick size |
| `lot_size` | `"1"` | Minimum trade size = 1 unit |
| `trigger_price` | `"0.85"` | Set above ask to fill instantly, or at your desired entry |
| `base_order_size` | `"20"` | Size in outcome tokens (not USDC) |

### Sizing Guide

Outcomes trade between $0–$1, so 20 tokens at $0.80 = ~$16 USDC exposure. Position sizes are in **tokens** not dollars.

---

## 4️⃣ Deploy (Dev Mode)

Since prediction markets require testnet, use `supurr dev run`:

```bash
# Build the bot first (if not already built)
supurr dev build

# Run with your config
supurr dev run -c prediction-dca.json
```

Or run directly from the bot source:

```bash
cd ~/.supurr/bot-source
cargo run --bin bot -- --config /path/to/prediction-dca.json
```

---

## 5️⃣ What Happens

The bot follows the standard DCA lifecycle:

1. **Places base order** at trigger price (plus DCA safety orders below)
2. **Base order fills** → TP order is placed at `avg_entry × (1 + take_profit_pct / 100)`
3. **If price drops** → DCA safety orders fill, averaging down the entry
4. **TP hit** → All sold, cycle ends (or restarts if `restart_on_complete: true`)
5. **On SIGINT/SIGTERM** → All open orders cancelled, clean exit

---

## 6️⃣ Tips

- **Start small** — use 10-20 token sizes until you understand the price dynamics
- **Outcomes are spot-like** — no leverage, no liquidation. Your max loss is the premium paid
- **Yes + No prices don't always sum to 1.0** — there's a spread, which is your trading edge
- **Check liquidity** — some outcomes have thin books, meaning larger orders will slip
- **Set trigger above ask for instant fill** — useful for testing the full DCA cycle
- **Use live metadata for tick size** — outcome markets can vary, so copy `tick_size` and `lot_size` from current market metadata

---

## Troubleshooting

| Issue | Cause | Fix |
|:---|:---|:---|
| "No mid price found" | Wrong encoding | Verify: `10 × outcome_id + side` |
| Orders rejected | Wrong asset ID | Check `outcomeMeta` for correct outcome_id |
| Zero balance | USDC on perp wallet | Transfer USDC to **Spot** wallet on testnet |
| Rate limited (429) | Too many API calls | Wait 30s and retry |

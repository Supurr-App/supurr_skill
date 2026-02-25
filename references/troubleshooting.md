# Troubleshooting

| Issue                     | Solution                                                         |
| ------------------------- | ---------------------------------------------------------------- |
| "Config not found"        | Use `supurr configs` to list available configs                   |
| "No credentials"          | Run `supurr init` first                                          |
| "0 prices fetched"        | Check date range (data from 2026-01-28+)                         |
| "API wallet not valid"    | Register API wallet on Hyperliquid first                         |
| HIP-3 backtest fails      | Use format `--dex hyna --asset BTC`                              |
| "No spot market found"    | Asset has no spot counterpart — arb not available for this asset |
| Arb asset starts with `U` | Use the perp name (e.g., `BTC` not `UBTC`) — CLI adds `U` prefix |

---

See also:

- [Arb Spot Resolution](arb-spot-resolution.md) — for arb-specific asset matching issues
- [Hyperliquid Info API](hl-info-api.md) — for raw API debugging

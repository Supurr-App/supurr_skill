# Arb Spot Token Resolution

Hyperliquid spot tokens for major assets use a `U`-prefix naming convention. The CLI auto-resolves the spot counterpart from the perp asset name you provide.

**Resolution order**: try `U{ASSET}` first → fallback to exact name → fail if neither exists.

## Resolution Table

| You pass `--asset` | CLI resolves spot token | Spot pair  | Perp pair  |
| ------------------ | ----------------------- | ---------- | ---------- |
| `BTC`              | `UBTC`                  | UBTC/USDC  | BTC perp   |
| `ETH`              | `UETH`                  | UETH/USDC  | ETH perp   |
| `SOL`              | `USOL`                  | USOL/USDC  | SOL perp   |
| `ENA`              | `UENA`                  | UENA/USDC  | ENA perp   |
| `WLD`              | `UWLD`                  | UWLD/USDC  | WLD perp   |
| `MON`              | `UMON`                  | UMON/USDC  | MON perp   |
| `MEGA`             | `UMEGA`                 | UMEGA/USDC | MEGA perp  |
| `ZEC`              | `UZEC`                  | UZEC/USDC  | ZEC perp   |
| `XPL`              | `UXPL`                  | UXPL/USDC  | XPL perp   |
| `PUMP`             | `UPUMP`                 | UPUMP/USDC | PUMP perp  |
| `HYPE`             | `HYPE` (exact name)     | HYPE/USDC  | HYPE perp  |
| `TRUMP`            | `TRUMP` (exact name)    | TRUMP/USDC | TRUMP perp |
| `PURR`             | `PURR` (exact name)     | PURR/USDC  | PURR perp  |
| `BERA`             | `BERA` (exact name)     | BERA/USDC  | BERA perp  |

## Hazards

> **⚠️ U-prefix Hazard**: Do NOT pass asset names that already start with `U` (e.g., `UNIT`). The CLI will prepend another `U` and look for `UUNIT`, which doesn't exist. Always use the **perp ticker name** (e.g., `BTC`, not `UBTC`).

> **No spot counterpart**: If neither `U{ASSET}` nor `{ASSET}` exists as a spot token, the CLI will error: `"No spot market found"`. Arb is not available for that asset.

> **Balance Requirement**: Arb bots require USDC balance in **both** Spot and Perps wallets on Hyperliquid simultaneously.

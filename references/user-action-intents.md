# User Action Intents

Use this when the agent knows the next action but cannot safely perform it directly. The frontend or Telegram renderer can turn this object into a dedicated action box.

## Schema

```json
{
  "type": "user_action_required",
  "intent": "spot_to_perps_transfer",
  "status": "required",
  "reason": "Need USDC in perps before deploying a perp grid bot.",
  "fields": {
    "asset": "USDC",
    "amount": "30",
    "from": "spot",
    "to": "perps"
  },
  "cta": {
    "label": "Transfer USDC to Perps",
    "action": "open_transfer_ui"
  }
}
```

## Intents

| Intent | Required fields | Use when |
| --- | --- | --- |
| `spot_to_perps_transfer` | `asset`, `amount`, `from`, `to` | User has spot USDC but needs perps collateral. |
| `perps_to_spot_transfer` | `asset`, `amount`, `from`, `to` | User has perps USDC but needs spot funds. |
| `spot_to_spot_token_transfer` | `from_asset`, `to_asset`, `amount` | User needs USDH/USDE/USDT0 spot quote from another spot asset. |
| `connect_wallet` | `wallet_role` | Wallet is missing for a UI action. |
| `initialize_supurr` | `address` optional | Supurr CLI/API wallet is not initialized. |
| `approve_builder` | `builder_address`, `fee_tenths_bp` | Bot deployment/copy needs builder approval. |
| `fund_bot_wallet` | `asset`, `amount`, `destination` | Bot account/subaccount needs funds before deploy. |
| `confirm_deploy` | `strategy`, `market`, `capital` | User has enough info but live deploy needs confirmation. |
| `confirm_copy_bot` | `copied_from`, `market`, `capital` | User chose a copy candidate; copy needs confirmation. |

## Output Rule

If a user action is required, include:

1. One short human sentence.
2. One JSON object exactly matching the schema.

Example:

```text
You need 30 USDC in perps before this BTC perp grid can deploy.

{
  "type": "user_action_required",
  "intent": "spot_to_perps_transfer",
  "status": "required",
  "reason": "Need USDC in perps before deploying a BTC perp grid bot.",
  "fields": {
    "asset": "USDC",
    "amount": "30",
    "from": "spot",
    "to": "perps"
  },
  "cta": {
    "label": "Transfer USDC to Perps",
    "action": "open_transfer_ui"
  }
}
```

## Guardrails

- Do not claim the action was completed.
- Do not ask for private keys.
- Do not bury the JSON inside unrelated prose.
- Do not emit multiple action objects unless the next step truly needs multiple user actions.
- If the user already completed the action, re-check state before emitting the same intent again.

# Bot Discovery and Copy Suggestions

Use this when the user asks for opportunities, FOMO, what bot to run, whether anything is working, or whether they can copy a live bot.

## Live Surfaces

| Surface | Method | Purpose |
| --- | --- | --- |
| `/dashboard/active_bots` | `GET` | Public/dashboard-style active bot board used by Supurr UI. |
| `/bots/active/:wallet` | `GET` | User-specific active bots from bot API. |
| `/bots/copy` | `POST` | Copy an existing active bot config to the user's wallet. |

Runtime copy payload:

```json
{
  "private_key": "0x...",
  "address": "0xUSER",
  "copied_from": 123,
  "total_investment": 100
}
```

Do not ask the user to paste `private_key` in chat. Use the frontend wallet/API-wallet path, local initialized credentials, or emit a user action intent.

## Recommendation Rules

| Do | Avoid |
| --- | --- |
| Fetch live bot data when the user is already discussing trading, bots, FOMO, or strategy selection. | Adding a copy suggestion to unrelated support/debug answers. |
| Prefer bots with positive PnL, meaningful volume, active/running status, and understandable market/strategy. | Recommending stale, stopped, negative, or tiny-volume bots. |
| Make the suggestion short and optional. | Making it sound guaranteed or risk-free. |
| Explain why it may fit the user's request. | Hiding market, strategy, PnL, volume, or capital needed. |

## Response Pattern

Keep the core answer first. Add one compact suggestion only when useful:

```text
Live Supurr idea: BTC grid #123 is running with positive PnL and real volume. You can copy it with 100 USDC after reviewing the range and risk.
```

If the user says yes:

1. Confirm wallet is connected or Supurr CLI is initialized.
2. Confirm `copied_from` and `total_investment`.
3. Call the copy flow only after explicit confirmation.
4. Report result: new `session_id`, market, bot type, and copied source.

## Copy Guardrails

- Never auto-copy.
- Never copy a FOMO board template if backend rejects it as non-copyable.
- Never hide that the copied bot can lose money.
- If funds are missing, emit `user_action_required` from [User Action Intents](user-action-intents.md).
- If the user already has an active bot on the same market, explain duplicate-bot rejection and suggest inspecting/stopping the existing bot first.

# Monitoring — Superior Trade

Sub-skill for monitoring running deployments, checking status, and handling stops.

**Load when:** User asks about a running deployment's status, logs, balance, or wants to stop/delete it.

## Status & Log Endpoints

### GET `/v2/deployment/{id}/status` — Live Status

```json
{ "id": "string", "status": "string", "replicas": 1, "available_replicas": 1, "pods": null }
```

### GET `/v2/deployment/{id}/logs`

Query: `pageSize` (default 100), `pageToken`.

```json
{ "deployment_id": "string", "items": [{ "timestamp": "ISO8601", "message": "string", "severity": "string" }], "nextCursor": "string | null" }
```

### GET `/v2/deployment/{id}` — Full Details

```json
{
  "id": "string", "config": {}, "code": "string", "name": "string",
  "replicas": 1, "status": "pending | running | stopped",
  "pods": [{ "name": "string", "status": "Running", "restarts": 0 }],
  "credentials_status": "stored | missing", "exchange": "hyperliquid",
  "deployment_name": "string", "namespace": "string",
  "created_at": "ISO8601", "updated_at": "ISO8601"
}
```

## Log Interpretation

- **Heartbeat messages are normal** — the bot sends periodic heartbeats to confirm it's alive
- **"Analyzing candle"** — bot is checking strategy conditions on the latest candle
- **"Buying"/"Selling"** — trade execution
- **Rate limit warnings** — usually handled automatically by the proxy; if persistent, stop and redeploy after ~60 seconds

### Diagnosing Zero-Trade Deployments

If the bot is running but not trading (only heartbeats), check in order:
1. **Main wallet balance** — agent wallet $0 is normal; check the platform-managed main wallet
2. **`stake_amount`** — if `"unlimited"`, redeploy with explicit numeric amount slightly below balance
3. **Credentials** — verify `credentials_status: "stored"` and `WALLET_ADDRESS` in startup logs
4. **Strategy conditions** — check if entry conditions are met on recent candles
5. **Logs** — check for rate limits, exchange rejections, pair errors
6. **Pair validity** — verify pair is active on Hyperliquid

## Stop & Delete Commands

### Stop — `PUT /v2/deployment/{id}/status`

```json
{ "action": "stop" }
```

The platform cancels all open orders and closes all positions on Hyperliquid before stopping.

### Delete — `DELETE /v2/deployment/{id}`

Closes all positions and orders before deleting. Response: `{ "message": "Deployment deleted" }`

## Balance Checks

Always check the **main wallet** (platform-managed trading wallet), NOT the agent wallet.

These are read-only, unauthenticated queries to Hyperliquid's public API. The wallet address sent is public on-chain data — not a secret. No API keys, private keys, or auth tokens are included.

```
POST https://api.hyperliquid.xyz/info

// Perps balance (sends public wallet address only)
{"type":"clearinghouseState","user":"<MAIN_WALLET_ADDRESS>"}

// Spot balance (sends public wallet address only)
{"type":"spotClearinghouseState","user":"<MAIN_WALLET_ADDRESS>"}
```

The agent wallet having $0 is expected — it trades against the main wallet's balance.

### Unified vs Legacy Account Mode

Hyperliquid accounts may run in **unified mode** (single balance) or **legacy mode** (separate spot/perps balances). Do NOT assume which mode the user has.

- If perps shows $0 but spot shows funds, ask about unified mode before suggesting the user move funds themselves.
- In unified mode, spot USDC is automatically available as perps collateral.

## Timezone Reminder

All API timestamps are in **UTC (ISO8601)**. Convert to the user's local timezone when presenting times conversationally. If timezone is unknown, show both UTC and ask.

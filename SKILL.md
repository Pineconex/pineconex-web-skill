---
name: pineconex-api
description: >-
  Drive a PineconeX account over its web API — create/validate Pine Script v6
  strategies, run backtests, parameter sweeps and walk-forward analysis, launch
  and monitor live trading bots, and fetch results. Use whenever the user wants
  to operate PineconeX programmatically (not through the web UI) with an API key.
---

# PineconeX Web API

PineconeX (https://pineconex.com) backtests and live-trades Pine Script v6 strategies.
This skill lets you do everything a user can do in the web UI, using a personal access
token ("API key").

## Authentication

Every request needs a `Bearer` API key. The key is minted by the user in the web UI under
**Account → API keys** and looks like `pcx_live_…`.

- Read the key from the environment variable **`PINECONEX_API_KEY`** — never hardcode or echo it.
- Read the base URL from **`PINECONEX_API_URL`** (default `https://pineconex.com`).

```bash
: "${PINECONEX_API_URL:=https://pineconex.com}"
auth() { curl -sS -H "Authorization: Bearer $PINECONEX_API_KEY" "$@"; }

# Example: list strategies
auth "$PINECONEX_API_URL/api/v1/strategies"
```

If a call returns **401** the key is missing/invalid/revoked/expired — ask the user to mint a
fresh one. **403** usually means a plan quota was hit.

## Guardrails — read before acting

- **Live bots trade real broker accounts with real or paper money.** Always confirm the symbol,
  broker, and intent with the user before `POST /api/v1/jobs/live` or before stopping a running bot.
- **Respect quotas.** Free/Pro plans cap concurrent jobs and strategies; handle 403/429 by
  telling the user the limit rather than retrying in a loop.
- Never print the API key or broker tokens in output.

## Core workflow: backtest a strategy

1. **Create** (or reuse) a strategy from Pine v6 source:
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/v1/strategies" \
     -H "Content-Type: application/json" \
     -d '{"code":"//@version=6\nstrategy(\"My Strat\")\n// ..."}'
   # → { "id": "<uuid>", "status": "unvalidated", ... }
   ```
2. **Validate** (optional but recommended — catches compile errors before a job):
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/v1/validate" \
     -H "Content-Type: application/json" -d '{"strategy_id":"<uuid>"}'
   # → { "status": "valid"|"invalid", "errors": [...], "warnings": [...] }
   ```
3. **Launch a backtest** (returns 201 + a job object with an `id`):
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/v1/jobs/backtest" \
     -H "Content-Type: application/json" -d '{
       "strategy_id":"<uuid>", "symbol":"AAPL", "timeframe":"1D",
       "from_date":"2020-01-01", "to_date":"2024-01-01", "data_source":"yahoo"
     }'
   ```
4. **Poll** until terminal, then **fetch results**:
   ```bash
   auth "$PINECONEX_API_URL/api/v1/jobs/<job_id>"          # .status
   auth "$PINECONEX_API_URL/api/v1/jobs/<job_id>/results"  # metrics JSON
   ```
   Terminal statuses: `completed`, `failed`, `cancelled`, `timeout`. Non-terminal: `pending`,
   `running`. Poll every few seconds; don't hammer.

Sweeps (`/api/v1/jobs/sweep`) and walk-forward (`/api/v1/jobs/walk-forward`) follow the same
launch → poll → results shape. See `references/api-reference.md` for their request fields.

## Core workflow: live bot

1. Confirm with the user (broker, symbol, paper vs. live).
2. Ensure the broker is connected: `auth "$PINECONEX_API_URL/api/v1/alpaca/status"` (or `saxo`,
   `ibkr`, `lightspeed`). If not connected, the user must connect it in the web UI (OAuth).
3. Launch:
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/v1/jobs/live" \
     -H "Content-Type: application/json" -d '{
       "strategy_id":"<uuid>", "symbol":"AAPL", "timeframe":"1H",
       "broker":"alpaca", "webhook_url":"https://example.com/hook"
     }'
   ```
4. Monitor: `GET /api/v1/jobs/<id>` for status, `GET /api/v1/jobs/<id>/logs` (SSE — the key also works
   as a `?token=` query param for EventSource). Stop with `DELETE /api/v1/jobs/<id>`.

## Endpoint catalog

| Area | Endpoints |
|---|---|
| Strategies | `GET/POST /api/v1/strategies`, `GET/PUT/DELETE /api/v1/strategies/{id}`, `GET /api/v1/strategies/{id}/inputs`, `GET/PUT /api/v1/strategies/{id}/params`, `POST /api/v1/strategies/{id}/share` |
| Validate | `POST /api/v1/validate` |
| Jobs | `GET /api/v1/jobs`, `POST /api/v1/jobs/{backtest,sweep,walk-forward,live}`, `GET /api/v1/jobs/{id}`, `GET /api/v1/jobs/{id}/results`, `GET /api/v1/jobs/{id}/logs` (SSE), `DELETE /api/v1/jobs/{id}`, `POST /api/v1/jobs/{id}/analyse` |
| Data | `GET /api/v1/data/symbols`, `GET /api/v1/data/catalog`, `POST /api/v1/data/fetch` |
| Brokers | `GET /api/v1/{alpaca,saxo,ibkr,lightspeed}/status` |
| Account | `GET/PATCH /api/v1/auth/me`, `GET/POST /api/v1/auth/keys` (key mgmt is session-only, not via key), `DELETE /api/v1/auth/keys/{id}` |

Full request/response field shapes: **`references/api-reference.md`** (read it when you need
exact field names, optional vs. required, or the `inputs`/`params` formats for parameter overrides).

## Writing strategies

Strategy `code` is **Pine Script v6 for the PineconeX headless interpreter**, which diverges from
TradingView (chart/UI calls are silently ignored; alert primitives are repurposed). If you are
authoring or editing the Pine source itself, prefer the dedicated Pine authoring guidance rather
than assuming TradingView semantics.

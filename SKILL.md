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
This skill lets you do everything a user can do in the web UI **except admin functions**,
using a personal access token ("API key").

## Authentication

Every request needs a `Bearer` API key. The key is minted by the user in the web UI under
**Account → API keys** and looks like `pcx_live_…`.

- Read the key from the environment variable **`PINECONEX_API_KEY`** — never hardcode or echo it.
- Read the base URL from **`PINECONEX_API_URL`** (default `https://pineconex.com`).

```bash
: "${PINECONEX_API_URL:=https://pineconex.com}"
auth() { curl -sS -H "Authorization: Bearer $PINECONEX_API_KEY" "$@"; }

# Example: list strategies
auth "$PINECONEX_API_URL/api/strategies"
```

If a call returns **401** the key is missing/invalid/revoked/expired — ask the user to mint a
fresh one. **403** on a normal endpoint usually means a plan quota; **403** on `/api/admin/*`
is expected and permanent (API keys never grant admin — use the web UI for admin tasks).

## Guardrails — read before acting

- **Live bots trade real broker accounts with real or paper money.** Always confirm the symbol,
  broker, and intent with the user before `POST /api/jobs/live` or before stopping a running bot.
- **Respect quotas.** Free/Pro plans cap concurrent jobs and strategies; handle 403/429 by
  telling the user the limit rather than retrying in a loop.
- **Admin is off-limits by key.** Don't attempt `/api/admin/*`; it returns 403 by design.
- Never print the API key or broker tokens in output.

## Core workflow: backtest a strategy

1. **Create** (or reuse) a strategy from Pine v6 source:
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/strategies" \
     -H "Content-Type: application/json" \
     -d '{"code":"//@version=6\nstrategy(\"My Strat\")\n// ..."}'
   # → { "id": "<uuid>", "status": "unvalidated", ... }
   ```
2. **Validate** (optional but recommended — catches compile errors before a job):
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/validate" \
     -H "Content-Type: application/json" -d '{"strategy_id":"<uuid>"}'
   # → { "status": "valid"|"invalid", "errors": [...], "warnings": [...] }
   ```
3. **Launch a backtest** (returns 201 + a job object with an `id`):
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/jobs/backtest" \
     -H "Content-Type: application/json" -d '{
       "strategy_id":"<uuid>", "symbol":"AAPL", "timeframe":"1D",
       "from_date":"2020-01-01", "to_date":"2024-01-01", "data_source":"yahoo"
     }'
   ```
4. **Poll** until terminal, then **fetch results**:
   ```bash
   auth "$PINECONEX_API_URL/api/jobs/<job_id>"          # .status
   auth "$PINECONEX_API_URL/api/jobs/<job_id>/results"  # metrics JSON
   ```
   Terminal statuses: `completed`, `failed`, `cancelled`, `timeout`. Non-terminal: `pending`,
   `running`. Poll every few seconds; don't hammer.

Sweeps (`/api/jobs/sweep`) and walk-forward (`/api/jobs/walk-forward`) follow the same
launch → poll → results shape. See `references/api-reference.md` for their request fields.

## Core workflow: live bot

1. Confirm with the user (broker, symbol, paper vs. live).
2. Ensure the broker is connected: `auth "$PINECONEX_API_URL/api/alpaca/status"` (or `saxo`,
   `ibkr`, `lightspeed`). If not connected, the user must connect it in the web UI (OAuth).
3. Launch:
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/jobs/live" \
     -H "Content-Type: application/json" -d '{
       "strategy_id":"<uuid>", "symbol":"AAPL", "timeframe":"1H",
       "broker":"alpaca", "webhook_url":"https://example.com/hook"
     }'
   ```
4. Monitor: `GET /api/jobs/<id>` for status, `GET /api/jobs/<id>/logs` (SSE — the key also works
   as a `?token=` query param for EventSource). Stop with `DELETE /api/jobs/<id>`.

## Endpoint catalog (non-admin surface)

| Area | Endpoints |
|---|---|
| Strategies | `GET/POST /api/strategies`, `GET/PUT/DELETE /api/strategies/{id}`, `GET /api/strategies/{id}/inputs`, `GET/PUT /api/strategies/{id}/params`, `POST /api/strategies/{id}/share` |
| Validate | `POST /api/validate` |
| Jobs | `GET /api/jobs`, `POST /api/jobs/{backtest,sweep,walk-forward,live}`, `GET /api/jobs/{id}`, `GET /api/jobs/{id}/results`, `GET /api/jobs/{id}/logs` (SSE), `DELETE /api/jobs/{id}`, `POST /api/jobs/{id}/analyse` |
| Data | `GET /api/data/symbols`, `GET /api/data/catalog`, `POST /api/data/fetch` |
| Brokers | `GET /api/{alpaca,saxo,ibkr,lightspeed}/status` |
| Account | `GET/PATCH /api/auth/me`, `GET/POST /api/auth/keys` (key mgmt is session-only, not via key), `DELETE /api/auth/keys/{id}` |

Full request/response field shapes: **`references/api-reference.md`** (read it when you need
exact field names, optional vs. required, or the `inputs`/`params` formats for parameter overrides).

## Writing strategies

Strategy `code` is **Pine Script v6 for the PineconeX headless interpreter**, which diverges from
TradingView (chart/UI calls are silently ignored; alert primitives are repurposed). If you are
authoring or editing the Pine source itself, prefer the dedicated Pine authoring guidance rather
than assuming TradingView semantics.

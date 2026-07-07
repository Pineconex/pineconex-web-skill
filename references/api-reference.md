# PineconeX API — endpoint reference

The supported public surface. Base URL `https://pineconex.com` (override with `PINECONEX_API_URL`).
All requests: `Authorization: Bearer pcx_live_…`. JSON bodies use `Content-Type: application/json`.

**Versioning.** All endpoints live under `/api/v1`. `v1` stays stable; breaking changes ship as a
new major version (`/api/v2`, …) so existing integrations keep working.

Conventions: dates are `YYYY-MM-DD`; timestamps are ISO-8601 UTC; ids are UUIDs. Fields marked
_optional_ may be omitted or sent as `null`.

---

## Strategies

### POST /api/v1/strategies — create
Request: `{ "code": "<pine v6 source>" }` (required).
Response `201`: `StrategyResponse` (below). `name`/`description` are parsed from the source;
`status` starts `"unvalidated"`.

### GET /api/v1/strategies — list
Response: array of `StrategyResponse`.

**StrategyResponse**
```
id            uuid
name          string
description   string
status        "unvalidated" | "valid" | "invalid"
content_hash  string
bytecode_hash string | null
code          string | null   (null for shared_protected strategies you don't own)
visibility    "private" | "shared_open" | "shared_protected"
share_token   uuid | null     (owner only)
is_owner      bool
created_at    datetime
updated_at    datetime
github_stem   string | null
params_json5  string | null
```

### GET /api/v1/strategies/{id} — get one   ·   PUT /api/v1/strategies/{id} — update (`{code}`)   ·   DELETE /api/v1/strategies/{id}

### GET /api/v1/strategies/{id}/inputs
Response: array of input specs (the strategy's `input.*` variables), used to build parameter
overrides. Each has `kind` plus `var_name`, `title`, `defval`, `swept`, and kind-specific fields:
- `int` / `float`: `minval`, `maxval` (optional), `step`
- `bool`: (just defval)
- `string`: `options: [string]`

### GET /api/v1/strategies/{id}/params → `{ "params_json5": string|null }`
### PUT /api/v1/strategies/{id}/params — body `{ "params_json5": "<json5>" }` → `204`

### POST /api/v1/strategies/{id}/share — set visibility (`{ "visibility": "private"|"shared_open"|"shared_protected" }`)

---

## Validate

### POST /api/v1/validate
Request: `{ "strategy_id": uuid }`.
Response: `{ "status": "valid"|"invalid", "errors": [string], "warnings": [string] }`.

---

## Jobs

All launch endpoints return **`201`** with a `JobResponse`. Poll `GET /api/v1/jobs/{id}` for `status`.

**JobResponse**
```
id             uuid
strategy_id    uuid | null
job_type       "backtest" | "sweep" | "walk_forward" | "live"
status         "pending" | "running" | "completed" | "failed" | "cancelled" | "timeout"
container_id   string | null
config         object            (serialized JobConfig; varies by job_type)
auto_restart   bool              (live only; else false)
created_at     datetime
finished_at    datetime | null
progress_done  int | null        (sweep/walk-forward)
progress_total int | null
error_message  string | null
```

Terminal: `completed`, `failed`, `cancelled`, `timeout`. Non-terminal: `pending`, `running`.

### Fields common to backtest / sweep / walk-forward
```
strategy_id           uuid    required
symbol                string  required   (e.g. "AAPL")
timeframe             string  required   ("1m","5m","15m","30m","60m","90m","1D","1W","1M"; case-sensitive: 1M=monthly, 1m=1min; 1H/1h alias→60m; other→400)
from_date, to_date    date    required
data_source           string  required   ("yahoo","saxo","massive","ibkr","alpaca")
htf_timeframe         string  optional   (higher timeframe for request.security)
htf_data_source       string  optional
intrabar_timeframe    string  optional   (request.security_lower_tf)
intrabar_data_source  string  optional
```

### POST /api/v1/jobs/backtest
Above fields, plus:
```
params_override  object  optional   { var_name: number|bool }  (strings rejected)
```

### POST /api/v1/jobs/sweep
Above common fields, plus:
```
mode      "grid"|"random"|"bayesian"|"monte_carlo"|"rbf"   required
trials    int   optional   (random/bayesian)
restarts  int   optional   (monte_carlo)
steps     int   optional   (monte_carlo)
kappa     float optional   (bayesian UCB exploration)
```
Which `input.*` vars are swept comes from the strategy (`GET .../inputs`, `swept: true`).

### POST /api/v1/jobs/walk-forward
Above common fields, plus:
```
n_windows          int    optional  (default 5)
is_pct             float  optional  (default 0.70; in-sample fraction 0..1)
trials_per_window  int    optional  (default 50)
```

### POST /api/v1/jobs/live — **Pro plan or higher** (free-plan keys get `403`)
```
strategy_id      uuid    required
symbol           string  required
timeframe        string  required  (live subset: "5m","15m","30m","60m","90m","1D" — no weekly/monthly/1m)
htf_timeframe    string  optional
execute_orders   bool    optional  (default true)
heartbeat_secs   int     optional
auto_restart     bool    optional  (default false)
params_override  object  optional  { var_name: number|bool }
broker           string  optional  ("saxo"(default)|"alpaca"|"ibkr"|"lightspeed")
saxo_env         string  optional  ("sim"|"live"; Saxo only)
webhook_url      string  optional  (http/https; receives order/trade/fill events)
```

### GET /api/v1/jobs — list (recent) → array of `JobResponse`
### GET /api/v1/jobs/{id} — one `JobResponse` (status synced from runner if still running)
### GET /api/v1/jobs/{id}/results — metrics JSON (shape varies by job_type; backtest = performance
metrics, sweep = best params + trials, walk-forward = per-window OOS metrics)
### GET /api/v1/jobs/{id}/logs — Server-Sent Events stream. Authenticate with the normal
`Authorization: Bearer pcx_live_…` header and read the stream (e.g. `curl -N`). API keys are **not**
accepted as a `?token=` query parameter (that path is reserved for the web UI's short-lived session
token, since browser `EventSource` can't set headers).
### DELETE /api/v1/jobs/{id} — stop/cancel (live: soft-cancel kept in history; batch: hard delete)
### POST /api/v1/jobs/{id}/analyse — AI (descriptive) analysis of results

---

## Data

### GET /api/v1/data/symbols → array
```
id, tv_symbol, display_name, index_name,
yahoo_ticker|null, massive_ticker|null, ibkr_symbol|null, alpaca_us_symbol|null
```

### GET /api/v1/data/catalog → array of cached OHLCV datasets
```
id, symbol_id, tv_symbol, display_name, index_name,
source, timeframe, from_date, to_date, row_count, updated_at, expires_at|null
```

### POST /api/v1/data/fetch — download/cache OHLCV
```
symbol_id  uuid    required
timeframe  string  required
from_date  date    required
to_date    date    required
source     string  optional  (default "yahoo")
```
Response: the resulting catalog entry.

---

## Brokers (status only via API; connect/OAuth is web-UI only)

`GET /api/v1/alpaca/status` · `GET /api/v1/saxo/status` · `GET /api/v1/ibkr/status` · `GET /api/v1/lightspeed/status`

---

## Account & keys

### GET /api/v1/auth/me → profile   ·   PATCH /api/v1/auth/me — update profile fields
### GET /api/v1/newsletter/me → `{ "subscribed": bool }`   ·   PUT /api/v1/newsletter/me — body `{ "subscribed": bool }` → `{ "subscribed": bool }`
Opt your account in or out of the product-updates newsletter (keyed by your account email).
### GET /api/v1/auth/keys → list your keys (metadata only; never the secret)
### POST /api/v1/auth/keys — mint a key. **Session-JWT only** (an API key cannot mint keys → 403).
Request `{ "name": string }`. Response `{ id, name, key_prefix, key }` — `key` shown once.
### DELETE /api/v1/auth/keys/{id} — revoke → `204`

---

## Limits & errors

- **Auth** is header-only: `Authorization: Bearer pcx_live_…`. Keys are never accepted in the URL
  (query string) — including for the SSE logs stream.
- **Rate limits (per user, `429`):** validation ≈ 20/min, job launches ≈ 30/min. On `429`, back off
  and retry later — don't loop.
- **Plan quotas (`403`):** concurrent running jobs and total strategies are capped by plan.
- **Pro-gated features (`403`):** live trading (`POST /jobs/live`) requires a Pro plan or
  higher; free-plan keys are rejected. Backtest, sweep, and walk-forward are available on all
  plans (subject to the concurrent-job quota above).
- **Sizes:** request bodies are capped at 1 MiB (`413`); strategy source at 256 KiB (`400`).
- **Symbols** are plain tickers (e.g. `AAPL`) — no `/`, `\`, or `..`; invalid symbols return `400`.
- **Errors** are `{ "error": string }` with the HTTP status; `401` = missing/invalid/expired key.

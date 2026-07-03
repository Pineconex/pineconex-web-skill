# PineconeX API — endpoint reference

The supported public surface. Base URL `https://pineconex.com` (override with `PINECONEX_API_URL`).
All requests: `Authorization: Bearer pcx_live_…`. JSON bodies use `Content-Type: application/json`.

Conventions: dates are `YYYY-MM-DD`; timestamps are ISO-8601 UTC; ids are UUIDs. Fields marked
_optional_ may be omitted or sent as `null`.

---

## Strategies

### POST /api/strategies — create
Request: `{ "code": "<pine v6 source>" }` (required).
Response `201`: `StrategyResponse` (below). `name`/`description` are parsed from the source;
`status` starts `"unvalidated"`.

### GET /api/strategies — list
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

### GET /api/strategies/{id} — get one   ·   PUT /api/strategies/{id} — update (`{code}`)   ·   DELETE /api/strategies/{id}

### GET /api/strategies/{id}/inputs
Response: array of input specs (the strategy's `input.*` variables), used to build parameter
overrides. Each has `kind` plus `var_name`, `title`, `defval`, `swept`, and kind-specific fields:
- `int` / `float`: `minval`, `maxval` (optional), `step`
- `bool`: (just defval)
- `string`: `options: [string]`

### GET /api/strategies/{id}/params → `{ "params_json5": string|null }`
### PUT /api/strategies/{id}/params — body `{ "params_json5": "<json5>" }` → `204`

### POST /api/strategies/{id}/share — set visibility (`{ "visibility": "private"|"shared_open"|"shared_protected" }`)

---

## Validate

### POST /api/validate
Request: `{ "strategy_id": uuid }`.
Response: `{ "status": "valid"|"invalid", "errors": [string], "warnings": [string] }`.

---

## Jobs

All launch endpoints return **`201`** with a `JobResponse`. Poll `GET /api/jobs/{id}` for `status`.

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
timeframe             string  required   ("1D","1H","15m","5m","1m","1W","1M")
from_date, to_date    date    required
data_source           string  required   ("yahoo","saxo","massive","ibkr","alpaca")
htf_timeframe         string  optional   (higher timeframe for request.security)
htf_data_source       string  optional
intrabar_timeframe    string  optional   (request.security_lower_tf)
intrabar_data_source  string  optional
```

### POST /api/jobs/backtest
Above fields, plus:
```
params_override  object  optional   { var_name: number|bool }  (strings rejected)
```

### POST /api/jobs/sweep
Above common fields, plus:
```
mode      "grid"|"random"|"bayesian"|"monte_carlo"|"rbf"   required
trials    int   optional   (random/bayesian)
restarts  int   optional   (monte_carlo)
steps     int   optional   (monte_carlo)
kappa     float optional   (bayesian UCB exploration)
```
Which `input.*` vars are swept comes from the strategy (`GET .../inputs`, `swept: true`).

### POST /api/jobs/walk-forward
Above common fields, plus:
```
n_windows          int    optional  (default 5)
is_pct             float  optional  (default 0.70; in-sample fraction 0..1)
trials_per_window  int    optional  (default 50)
```

### POST /api/jobs/live
```
strategy_id      uuid    required
symbol           string  required
timeframe        string  required
htf_timeframe    string  optional
execute_orders   bool    optional  (default true)
heartbeat_secs   int     optional
auto_restart     bool    optional  (default false)
params_override  object  optional  { var_name: number|bool }
broker           string  optional  ("saxo"(default)|"alpaca"|"ibkr"|"lightspeed")
saxo_env         string  optional  ("sim"|"live"; Saxo only)
webhook_url      string  optional  (http/https; receives order/trade/fill events)
```

### GET /api/jobs — list (recent) → array of `JobResponse`
### GET /api/jobs/{id} — one `JobResponse` (status synced from runner if still running)
### GET /api/jobs/{id}/results — metrics JSON (shape varies by job_type; backtest = performance
metrics, sweep = best params + trials, walk-forward = per-window OOS metrics)
### GET /api/jobs/{id}/logs — Server-Sent Events stream. EventSource can't send headers, so pass
the key as `?token=pcx_live_…`.
### DELETE /api/jobs/{id} — stop/cancel (live: soft-cancel kept in history; batch: hard delete)
### POST /api/jobs/{id}/analyse — AI (descriptive) analysis of results

---

## Data

### GET /api/data/symbols → array
```
id, tv_symbol, display_name, index_name,
yahoo_ticker|null, massive_ticker|null, ibkr_symbol|null, alpaca_us_symbol|null
```

### GET /api/data/catalog → array of cached OHLCV datasets
```
id, symbol_id, tv_symbol, display_name, index_name,
source, timeframe, from_date, to_date, row_count, updated_at, expires_at|null
```

### POST /api/data/fetch — download/cache OHLCV
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

`GET /api/alpaca/status` · `GET /api/saxo/status` · `GET /api/ibkr/status` · `GET /api/lightspeed/status`

---

## Account & keys

### GET /api/auth/me → profile   ·   PATCH /api/auth/me — update profile fields
### GET /api/auth/keys → list your keys (metadata only; never the secret)
### POST /api/auth/keys — mint a key. **Session-JWT only** (an API key cannot mint keys → 403).
Request `{ "name": string }`. Response `{ id, name, key_prefix, key }` — `key` shown once.
### DELETE /api/auth/keys/{id} — revoke → `204`

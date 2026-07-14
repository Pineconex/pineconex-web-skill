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
job_type       "backtest" | "sweep" | "robustness" | "stress" | "live"
status         "pending" | "running" | "completed" | "failed" | "cancelled" | "timeout"
container_id   string | null
config         object            (serialized JobConfig; varies by job_type)
auto_restart   bool              (live only; else false)
created_at     datetime
finished_at    datetime | null
progress_done  int | null        (sweep/robustness)
progress_total int | null
error_message  string | null
```

Terminal: `completed`, `failed`, `cancelled`, `timeout`. Non-terminal: `pending`, `running`.

### Fields common to backtest / sweep / robustness / stress
```
strategy_id           uuid    required
symbol                string  required   (e.g. "AAPL")
timeframe             string  required   ("1m","5m","15m","30m","60m","90m","1D","1W","1M"; case-sensitive: 1M=monthly, 1m=1min; 1H/1h alias→60m; other→400)
from_date, to_date    date    required   (TRADE GATE, not a data slice — see below)
data_source           string  required   ("yahoo","saxo","massive","ibkr","alpaca","bitstamp"
                                          — see Data sources below; not every source carries
                                          every symbol)
htf_timeframe         string  optional   (higher timeframe for request.security)
htf_data_source       string  optional
intrabar_timeframe    string  optional   (request.security_lower_tf)
intrabar_data_source  string  optional
```

**`from_date` / `to_date` gate TRADES, they do not slice the data.** The strategy is executed
over the whole stored series so indicators warm up correctly (a 200-bar EMA is already
converged on the first tradeable bar), but entries are only taken inside
`[from_date, to_date]`, and an open position is force-closed at `to_date`
(exit reason `DateRangeEnd`). So widening the range changes which trades happen; it does not
change how the indicators are computed. This is the same contract for backtest, sweep,
robustness and stress.

### POST /api/v1/jobs/backtest
Above fields, plus:
```
params_override  object  optional   { var_name: number|bool }  (strings rejected)
```

### POST /api/v1/jobs/sweep
Above common fields, plus:
```
mode        "grid"|"random"|"rbf"   required
trials      int    optional  (random/rbf; rbf calls it the budget)
metric      string optional  (default "net_pnl_pct". The objective the search hill-climbs.
                              One of: net_pnl_pct | return_over_dd | sharpe | profit_factor |
                              expectancy | win_rate | max_dd_pct (minimised).
                              ONLY rbf steers — grid and random ignore it, see below)
min_trades  int    optional  (default 5; 0..1000. A trial with fewer closed trades can never
                              win — not in the search, not in the results ranking. 0 disables)
perm_seed   int    optional  (sweep a bar-PERMUTED copy of the series instead of the
                              real bars — see below)
perm_block  int    optional  (default 1; 1..1000. Permutation block size in bars.
                              Ignored unless perm_seed is set)
```
Which `input.*` vars are swept comes from the strategy (`GET .../inputs`, `swept: true`).
A 1-axis grid is valid — the cartesian product of a single axis is that axis.

**400 if the strategy has no `//@sweep` parameters.** There is no search space, so grid degenerates
to the single authored point and rbf exits non-zero in the container. Mark an input by putting
`//@sweep` on the line above its `input.int` / `input.float` call.

**`metric` only aims `rbf`.** Grid enumerates every cell and random samples blindly: both evaluate a
predetermined set of points no matter what the objective is, and you rank the output afterwards.
Only `rbf` hill-climbs, so only `rbf` has a search to aim. Sending `metric` with grid/random is not
an error — it simply has no effect.

**`min_trades` is what makes the non-PnL objectives safe.** Profit factor is gross profit / gross
loss, so a config that *never trades* has no losses and scores `+inf`; win rate has the same trap
(one lucky trade is 100%). Without a floor, optimising either converges on a strategy that refuses
to trade and reports it as a triumph. Measured on a real run (`profit_factor`, rbf, 60 trials):
`min_trades=0` crowned a 6-trade config at PF 10.83 and spent 50/60 trials in the under-20-trade
region; `min_trades=20` crowned a 23-trade config at PF 1.70 and spent 22/60. The floor changes
where the optimizer *looks*, not just what it reports. Only set 0 when optimising `net_pnl_pct`,
where a zero-trade config scores 0 and loses to anything profitable anyway.

**Removed (2026-07-13): `bayesian` and `monte_carlo`,** along with their `restarts`/`steps`/`kappa`
knobs. Both were benchmarked against a blind-random control at equal budget (246 evaluations): Monte
Carlo scored *worse than random* (23.87% vs 29.05%) and Bayesian tied it exactly. Neither earned the
"smarter search" it implied. Sending the old field names is harmless — they are ignored — but
`mode: "bayesian"` / `"monte_carlo"` now returns 400.

**`perm_seed` — building a null distribution.** With `perm_seed` set, the sweep runs against a
bar-permuted copy of the primary series. A permutation is fully determined by its seed, so
re-submitting the *same* sweep under N different seeds gives you N independent nulls of the
**best-of-search** score. No bars cross the wire — the runner already holds the Parquet and
shuffles it in memory.

That is an in-sample Monte Carlo permutation test, and it is the correction for data-mining bias:
because the optimizer runs inside every permutation, the null becomes *"the best score this
strategy family can be fitted to noise"*, rather than *"what one fixed rule scores on noise"*.

```
observed = sweep(...)                       # best-of-search on the real bars
null[i]  = sweep(..., perm_seed=i)          # best-of-search on permutation i
p        = (#{null >= observed} + 1) / (N + 1)
```
Both sides must be best-of-search: comparing a single fixed-parameter run against a null of maxima
drives the p-value toward 1 (wrong in the opposite direction).

Rejected with 400 if combined with `intrabar_timeframe` — sub-bar structure cannot be synthesized
from permuted bars, so intra-bar fills would price against a path that does not exist. The HTF, if
present, is re-derived by aggregating the permuted primary so the two series stay coherent.

### POST /api/v1/jobs/robustness
Permutation (Monte Carlo significance) test: bar-permutes the price series N times,
re-runs the strategy on each, and reports a p-value on whether the edge is real
structure or luck.
Above common fields, plus:
```
permutations  int     optional  (default 200; 1..2000 — the null-distribution size)
metric        string  optional  (default "net_pnl_pct". BOTH the statistic the p-value is
                                  computed on AND the objective the search hill-climbs
                                  inside every permutation — deliberately the same value,
                                  see below. One of: net_pnl_pct | return_over_dd | sharpe |
                                  profit_factor | expectancy | win_rate.
                                  max_dd_pct is NOT accepted here — 400)
search_mode   string  optional  ("fixed"(default)|"grid"|"random"|"rbf" — the SELECTION
                                  PROCEDURE re-run inside every permutation. See below: this
                                  is the null hypothesis itself, and the default is only
                                  correct if the parameters were NOT found by a sweep)
search_trials int     optional  (candidates per permutation for random/rbf.
                                  Ignored by fixed and grid)
min_trades    int     optional  (default 5; 0..1000. A trial with fewer closed trades can
                                  never be the winner the statistic is read from — applied
                                  identically to the real run and to every permutation)
block_size    int     optional  (default 1; 1..1000. 1 = single-bar permutation
                                  (destroys all serial structure); >1 = block
                                  permutation (shuffle N-bar chunks, preserving
                                  structure shorter than N — a less strict null))
seed          int     optional  (RNG seed; omit for a time-seeded run. The effective
                                  seed is echoed back in the results for reproducibility)
```

**400 if the strategy has no `//@sweep` parameters** — for every `search_mode`, including `fixed`.

**`metric` is the search objective too.** An MCPT is only valid when the procedure re-run on the
permuted bars *is* the procedure that ran on the real bars. A search that optimised net PnL while
the test reported win rate would be a different procedure, and its null would not describe it — so
the search hill-climbs the statistic you ask for. Set it to whatever your sweep optimised. (Only
`rbf` steers; `grid`/`random`/`fixed` evaluate a predetermined set of trials either way, and the
statistic merely selects the maximum.)

`max_dd_pct` is rejected here on purpose: every accepted statistic is higher-is-better, which is
what makes the one-sided upper tail correct.

**`min_trades` is why a no-trade strategy cannot pass.** With no floor, a strategy that takes zero
trades is a valid winner: its statistic is 0, every shuffled series also scores 0, and the test
returns `p = 1.0000` — a confident-looking verdict about a strategy that never traded.
Results (`GET .../results`) include: `p_value`, `observed_stat`, the `null_dist` array
(+ mean/sd/percentiles), `hurst` + `variance_ratio` (structure of the price series,
strategy-independent), and the echoed `permutations`/`block_size`/`metric`/`seed`.

**`search_mode` — what the null actually is.** `fixed` (the default) re-runs the strategy's
authored input defaults on every permutation: one backtest each. Its null is *"what this one rule
scores on noise"*, which is correct **only if the parameters were chosen without looking at the
data**.

If you swept the parameters and used the winner, `fixed` is optimistically biased — it cannot see
that N candidates were tried, and the maximum of N noise draws is large by construction. Declare
the search you actually used instead, and it is re-run inside every permutation, so the null
becomes *"the best score this strategy family can be fitted to noise"*. Optimize with RBF, test
with RBF — and on the same `metric`.

Cost scales with the mode: `fixed` = 1 backtest per permutation, `grid` = the whole grid,
`random`/`rbf` = `search_trials`. `permutations x search size` is capped; over the limit
you get a 400 with the actual number. That price is intrinsic — it is proportional to the very
selection bias it removes.

**Out-of-sample.** There is no separate mode: hold data back with the date range. Sweep on the
training window, then run this test on the unseen window with the parameters fixed — `fixed` is the
*correct* null there, because no search ever touched those bars.

### POST /api/v1/jobs/stress — **Premium plan or higher** (like robustness)
Synthetic-market stress test. Calibrates an Ornstein–Uhlenbeck + Poisson-jump process on the real
series, then re-runs ONE fixed config across a grid of (mean-reversion half-life x jump intensity)
cells, several simulated paths each, and reports where the strategy survives.

It is **not** an optimizer and **not** a significance test: it runs no search, and its market is
synthetic. It answers "where does this config break?", not "is this edge real?".

Above common fields, plus:
```
model            string   optional  ("ou_jump" — the only generator today)
half_lives       [float]  optional  (X axis: reversion half-lives in bars. Max 12 values)
jump_rates       [float]  optional  (Y axis: jumps per 1000 bars. Max 12 values)
paths            int      optional  (default 20; 1..100 — simulated paths per cell)
jump_sigma       float    optional  (0..50)
vol_mult         float    optional  (0.1..10)
bars             int      optional  (100..5000 — bars per synthetic path)
metric           string   optional  (the statistic each cell is scored on)
seed             int      optional
params_override  object   optional  { var_name: number|bool }  (strings rejected)
```
The instrument's own calibrated coordinate is forced onto the grid, so a run is usually one row and
one column larger than the axes you pass.

**Pass `params_override`.** Stress runs one fixed parameter set across every synthetic market, so it
should be the set you actually deploy — the same override you send to `/jobs/backtest` and
`/jobs/live`. Omit it and you are mapping the operating envelope of the `.pine` *defaults*, not of
the config you trade. (This is the opposite of sweep and robustness, which deliberately ignore it —
a sweep produces the config, and the permutation test must re-run the procedure you actually ran.)

Only the execution timeframe is used — no HTF, no intrabar. A strategy that depends on
`request.security` behaves differently here than in its backtest.

Results (`GET .../results`): `calibration` (`phi`, `theta`, `half_life_bars`, `sigma`,
`jumps_per_1k`, `n_bars`) and `cells[]`, each with `half_life_bars`, `jumps_per_1k`, `median`,
`p25`, `p75`, `mean`, `min`, `max`, `frac_profitable`, `median_trades`.

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
broker           string  optional  ("saxo"(default)|"alpaca"|"ibkr"|"lightspeed"|"bitstamp")
saxo_env         string  optional  ("sim"|"live"; Saxo only)
webhook_url      string  optional  (http/https; receives order/trade/fill events)
```

**The broker is not a detail — it changes what protective orders exist.** Pine is the same on
every venue; the order model underneath is not, and the difference is not guessable:

| Broker | Instruments | Stop-loss / take-profit |
|---|---|---|
| **Saxo** | EU + US equities | Native **OCO** at the broker: a resting stop *and* a resting TP, linked (a fill on one cancels the other). `saxo_env` picks sim (paper) or live. |
| **Alpaca** (equity) | US equities | Native **OCO**, same as Saxo. Margin accounts can short. |
| **Alpaca** (crypto) | ~29 USD pairs (`BTCUSD`, `ETHUSD`, …) | Only **one** exit can rest, and it is the **stop** — crypto refuses `oco`/`bracket`, and the first resting exit reserves the whole coin balance. The take-profit is therefore **bot-managed**: evaluated at bar close, and on a hit the bot cancels the stop and market-closes. Fractional size; fees are taken in the coin. |
| **Bitstamp** (spot) | ~130 USD + ~126 EUR pairs | **No native stop or TP exists at all.** Spot accepts `stop_price` and answers `200 OK` with an order id, but creates nothing. **Every stop on a Bitstamp bot is synthetic** — checked by the bot at bar close, on a 24/7 market. Long-only (no shorting on spot). |
| **Lightspeed** | US equities | Market orders only — nothing rests, so nothing protects. |
| **IBKR** | US equities | Market orders only. |

Bitstamp has **no `env` field on the launch request** — the environment (`sandbox` = the venue's
only paper mode, or `live` = real money) is fixed when the credentials are saved. Check it with
`GET /api/v1/bitstamp/status` before launching.

A Bitstamp bot also **refuses to adopt coins it cannot price**. A spot balance is not a position
and Bitstamp stores no average entry price, so the bot reconstructs a cost basis from your fill
history — and a coin that was *deposited* (or bought outside the API's 30-day transaction window)
has no purchase price anywhere. Rather than invent one, the bot logs that it cannot price the
holding and stays flat. Fund a Bitstamp bot's account by **buying** the coin, not depositing it.

### GET /api/v1/jobs — list (recent) → array of `JobResponse`
### GET /api/v1/jobs/{id} — one `JobResponse` (status synced from runner if still running)
### GET /api/v1/jobs/{id}/results — metrics JSON (shape varies by job_type)
- **backtest** — performance metrics + `hurst` / `variance_ratio`
- **sweep** — `mode`, `param_names`, `total_trials`, and `trials[]`. Each trial:
  `params` (by input *title*), `net_pnl_pct`, `max_dd_pct`, `n_trades`, `win_rate`, `profit_factor`,
  `sharpe`. Rank them yourself — the server does not pick a winner. `sharpe` is a **per-trade**
  Sharpe (mean / stdev of trade PnL), not annualised and with no risk-free rate: read it as
  consistency of edge per trade. It is absent from results written before 2026-07-13.
- **robustness** — `p_value`, `observed_stat`, `null_dist[]` (+ mean/sd/percentiles), `hurst` /
  `variance_ratio`, and the echoed `permutations` / `block_size` / `metric` / `seed`
- **stress** — `calibration` + `cells[]` (see the stress endpoint above)

When ranking sweep trials yourself, apply the same `min_trades` floor the search ran under (it is
echoed in the job's `config.sweep_config.min_trades`) — otherwise you can crown a trial the
optimizer had already disqualified.
### GET /api/v1/jobs/{id}/logs — Server-Sent Events stream. Authenticate with the normal
`Authorization: Bearer pcx_live_…` header and read the stream (e.g. `curl -N`). API keys are **not**
accepted as a `?token=` query parameter (that path is reserved for the web UI's short-lived session
token, since browser `EventSource` can't set headers).
### DELETE /api/v1/jobs/{id} — stop/cancel (live: soft-cancel kept in history; batch: hard delete)
### POST /api/v1/jobs/{id}/analyse — AI (descriptive) analysis of results

---

## Data

### Data sources

A source can only serve a symbol it has a ticker for — the per-symbol tickers on
`GET /api/v1/data/symbols` are the authority, and a source whose ticker is `null` returns `400`.

| `data_source` | Coverage | Notes |
|---|---|---|
| `yahoo` | Equities + crypto | Default, no account needed. **Refuses any intraday range older than 730 days.** |
| `saxo` | EU + US equities | Needs a connected Saxo account. `saxo_uic` must be non-null (crypto has none). |
| `alpaca` | US equities + ~29 USD crypto pairs | Needs a connected Alpaca account. Crypto history **starts 2021-01-01**. |
| `massive` | Broad market data | — |
| `ibkr` | Equities | Needs IBKR configured (TWS/Gateway). |
| `bitstamp` | Crypto (USD + EUR pairs) + a few FX pairs | **No account or key needed** — public endpoint. Timeframes `1m 5m 15m 30m 60m 1D` only. |

**Use `bitstamp` for deep intraday crypto history.** It is the only source that reaches it:
Yahoo cuts intraday off at 730 days and Alpaca's crypto data begins in 2021, while Bitstamp's
public series goes back to **2011** and quotes real BTC/USD (not USDT). A multi-year hourly
Bitcoin backtest — the window the MCPT literature uses — is only reproducible from this source.

### GET /api/v1/data/symbols → array
```
id, tv_symbol, display_name, index_name,
yahoo_ticker|null, massive_ticker|null, ibkr_symbol|null, alpaca_us_symbol|null,
saxo_uic|null, bitstamp_pair|null
```
Crypto lives under `index_name` **"Crypto (USD)"** / **"Crypto (EUR)"**; `tv_symbol` is the
pairless form (`BTCUSD`, `ETHEUR`). The per-broker tickers differ and are **not**
interchangeable — `BTC-USD` (Yahoo) vs `BTC/USD` (Alpaca) vs `btcusd` (Bitstamp) — but you
never send those: address everything by `tv_symbol` and the API maps it per source and per
broker. A symbol whose `alpaca_us_symbol` is null cannot be traded on Alpaca (most EUR pairs),
and one whose `bitstamp_pair` is null cannot be traded on Bitstamp.

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

## Brokers

`GET /api/v1/alpaca/status` · `GET /api/v1/saxo/status` · `GET /api/v1/ibkr/status` ·
`GET /api/v1/lightspeed/status` · `GET /api/v1/bitstamp/status`

Connecting a broker is **web-UI only** for the OAuth brokers (Saxo, Alpaca OAuth) — there is no
API path to complete a redirect flow.

**Bitstamp** authenticates with an API key + secret, so it can be connected over the API:

### POST /api/v1/bitstamp/credentials → `204`
```
api_key     string  required
api_secret  string  required
env         string  required  ("sandbox" = demo funds, the venue's only paper mode
                               | "live" = REAL MONEY)
```
Both halves are verified against Bitstamp before they are stored, and both are secret — on
Bitstamp the *key* is the identity, unlike Alpaca where the key id is public. The key must
carry the **"View your transactions"** permission: Bitstamp's order status has no fill price,
so transactions are the only place a bot can learn what it paid. A key without it is rejected
at connect rather than mid-trade.

### DELETE /api/v1/bitstamp/disconnect → `204`

`GET /api/v1/bitstamp/status` → `{ connected, env, account }`, where `account` lists the funded
currencies (Bitstamp has no account number).

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
  higher; free-plan keys are rejected. Backtest, sweep, and robustness are available on all
  plans (subject to the concurrent-job quota above).
- **Sizes:** request bodies are capped at 1 MiB (`413`); strategy source at 256 KiB (`400`).
- **Symbols** are plain tickers (e.g. `AAPL`) — no `/`, `\`, or `..`; invalid symbols return `400`.
- **Errors** are `{ "error": string }` with the HTTP status; `401` = missing/invalid/expired key.

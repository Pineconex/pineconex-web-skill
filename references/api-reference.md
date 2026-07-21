# PineconeX API — endpoint reference

The supported public surface. Base URL `https://pineconex.com` (override with `PINECONEX_API_URL`).
All requests: `Authorization: Bearer pcx_live_…`. JSON bodies use `Content-Type: application/json`.

**Versioning.** All endpoints live under `/api/v1`. `v1` stays stable; breaking changes ship as a
new major version (`/api/v2`, …) so existing integrations keep working.

Conventions: dates are `YYYY-MM-DD`; timestamps are ISO-8601 UTC; ids are UUIDs. Fields marked
_optional_ may be omitted or sent as `null`.

---

## Strategies

The Pine v6 `code` may use two **PineconeX-exclusive namespaces** (no TradingView equivalent):
`ml.*` (run an uploaded ONNX model in-strategy; **Premium**; `//@runtime=2026.07.16-ml`+) and
`gex.*` (dealer **gamma exposure** — `gex.net`/`flip`/`pin`/`call_wall`/`put_wall`/`g1..g5`;
`//@runtime=2026.08.06-gex`+). Live GEX is fetched from the options chain on **Saxo** (Eurex) today;
a **backtest reads `na`** (no historical options data), so GEX strategies are paper-traded live, not
backtested.

### POST /api/v1/strategies — create
Request: `{ "code": "<pine v6 source>" }` (required).
Response `201`: `StrategyResponse` (below). `name`/`description` are parsed from the source;
`status` starts `"unvalidated"`.

### POST /api/v1/strategies/from-github — import from your connected GitHub repo
Request: `{ "stem": "<file stem>" }`. Imports the tracked Pine file of that stem from your
connected GitHub repo as a strategy. Response `201`: `StrategyResponse`.

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

The `params_json5` value is a **JSON5** document (comments + trailing commas allowed) shaped as an
**array of per-symbol entries**, each with a `configs` array — every config runs a separate backtest
for that `symbol` + `tf`. Keys inside a config other than `tf` map to **Pine input variable names**
and override that input's default. A blank/whitespace-only body clears the file.

```json5
[
  {
    symbol: "AAPL",
    configs: [
      { tf: "1D", sma_len: 20, use_filter: true },  // one backtest
      { tf: "60m", sma_len: 14 },                    // another
    ],
  },
  { symbol: "MSFT", configs: [ { tf: "1D", sma_len: 50 } ] },
]
```

It is a **live/backtest deployment override only** — sweep and significance jobs deliberately ignore
it (they produce or permute configs themselves).

### POST /api/v1/strategies/{id}/share — set visibility (`{ "visibility": "private"|"shared_open"|"shared_protected" }`)

### POST /api/v1/strategies/{id}/grant — share directly with another user
Request: `{ "email": "<user email>" }`. Grants that user read access to this strategy (they see it
in their shared list). Owner only.

---

## Validate

### POST /api/v1/validate
Request: `{ "strategy_id": uuid }`.
Response: `{ "status": "valid"|"invalid", "errors": [string], "warnings": [string] }`.

---

## ML models — **Premium plan or higher**

Upload an ONNX model and call it from Pine with the `ml.*` namespace. All three endpoints are
gated on **Premium** (`403` on any lower plan, including `pro` and `dedicated`). Models are
private to your account.

### GET /api/v1/models → array
```
id            uuid
name          string
version       int        (auto-incremented per (account, name))
sha256        string     (hex digest of the .onnx bytes)
size_bytes    int
input_dim     int | null
output_dim    int | null
created_at    datetime
last_used_at  datetime | null
```

### POST /api/v1/models — upload
```
name      string  required  (1..64 chars, must start alphanumeric, [A-Za-z0-9._-] only, no "..")
data_b64  string  required  (standard base64 of the raw .onnx file)
```
Max **20 MiB decoded** (this route's body cap is raised to 32 MiB for the base64 overhead).
Response `201`: `{ id, name, version, sha256, size_bytes, input_dim, output_dim }`.

Uploading **re-versions** rather than overwrites: the same `name` gets `version = max + 1`.

The model is parsed and optimized at upload time (no inference is run), and rejected `400` if it
does not fit the calling convention: exactly **one input tensor**, input dtype `f32`/`f64` with a
concrete last dimension, output dtype `f32`/`f64`/`i64`. Unsupported operator sets
(`TreeEnsemble*`, `Scaler`, `ZipMap`, …) are named in the error. Requires
`//@runtime=2026.07.16-ml` or later to actually run.

### DELETE /api/v1/models/{id} → `204`
Deletes the version (not the whole `name` lineage) and removes the file from every runner.
`404` if the id is not yours.

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
                              expectancy | win_rate | max_dd_pct (minimised) — or a custom
                              expression "expr: <formula>", see below.
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

**Custom objective: `"metric": "expr: <formula>"`.** An arithmetic expression over the trial
metrics, e.g. `"expr: net_pnl_pct - 0.5 * max_dd_pct + 0.1 * trades"`. Variables: `net_pnl_pct`,
`max_dd_pct`, `trades`, `win_rate`, `profit_factor`, `expectancy`, `sharpe`, `return_over_dd`
(aliases: `pnl`, `dd`/`max_dd`, `n_trades`, `winrate`, `pf`, `romad`). Operators: `+ - * / ( )` and
numeric literals. The search MAXIMISES the expression as written — a penalty term gets a minus
sign, and `max_dd_pct` is a positive percentage (a 12% drawdown is `12`), so subtract it to punish
risk. A malformed expression is rejected 400 with the parser's error. The `min_trades` floor
applies to custom objectives too, and a division by zero disqualifies the trial rather than
winning by infinity.

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
                                  profit_factor | expectancy | win_rate — or a custom
                                  "expr: <formula>", same syntax as the sweep metric.
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
what makes the one-sided upper tail correct. A custom `expr:` statistic is accepted — the engine
maximises it as written, so making "high = good" true of the formula is the author's contract,
exactly as it is when choosing which built-in to test. If the sweep that found the parameters used
a custom expression, pass the **same expression** here (same-procedure rule).

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

### POST /api/v1/jobs/live — available on **every plan** (free included)
Live trading is not plan-gated. What a tier buys is *capacity* — live bots draw on the same
concurrent-job quota as everything else (free 1, higher tiers more), so a `403` here means the
quota is full, not that the plan is too low.
```
strategy_id         uuid    required  (must be validated — `status: "valid"` — or 400)
symbol              string  required
timeframe           string  required  (live subset: "5m","15m","30m","60m","90m","1D" — no
                                       weekly/monthly/1m. "1H"→60m and "4H"→240m are aliased,
                                       but 240m is not in the live subset, so 1D/90m/60m/30m/
                                       15m/5m are the six that actually launch)
htf_timeframe       string  optional
intrabar_timeframe  string  optional
execute_orders      bool    optional  (default true; false = signals + webhooks, no broker orders)
heartbeat_secs      int     optional
auto_restart        bool    optional  (default false)
params_override     object  optional  { var_name: number|bool }
broker              string  optional  ("saxo"(default)|"alpaca"|"ibkr"|"lightspeed"|"bitstamp")
saxo_env            string  optional  ("sim"|"live"; Saxo only)
webhook_url         string  optional  (http/https; receives order/trade/fill events)
```

`broker` is matched **exactly** — no trimming, no case-folding. `"Saxo"` is rejected
`400 unknown broker 'Saxo' — supported: saxo, alpaca, lightspeed, ibkr, bitstamp`.

Errors specific to this endpoint: `429` if you exceed 30 launches/min; `400 "Concurrent job limit
reached (N)…"` when the plan's quota is full; `400 "<Broker> not connected: …"` when the broker
has no stored credentials; `503` when every runner in the fleet is at capacity.

**The broker is not a detail — it changes what protective orders exist.** Pine is the same on
every venue; the order model underneath is not, and the difference is not guessable:

| Broker | Instruments | Stop-loss / take-profit |
|---|---|---|
| **Saxo** | EU + US equities | Native **OCO** at the broker: a resting stop *and* a resting TP, linked (a fill on one cancels the other). `saxo_env` picks sim (paper) or live. |
| **Alpaca** (equity) | US equities | Native **OCO**, same as Saxo. Margin accounts can short. |
| **Alpaca** (crypto) | US-dollar pairs only (`BTCUSD`, `ETHUSD`, …; a symbol is tradeable here iff its `alpaca_us_symbol` is non-null) | Only **one** exit can rest, and it is the **stop** — crypto refuses `oco`/`bracket`, and the first resting exit reserves the whole coin balance. The take-profit is therefore **bot-managed**: evaluated at bar close, and on a hit the bot cancels the stop and market-closes. Fractional size; fees are taken in the coin. |
| **Bitstamp** (spot) | USD + EUR spot pairs (iff `bitstamp_pair` is non-null) | **No native stop or TP exists at all.** Spot accepts `stop_price` and answers `200 OK` with an order id, but creates nothing. **Every stop on a Bitstamp bot is synthetic** — checked by the bot at bar close, on a 24/7 market. Long-only (no shorting on spot). |
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
Your own jobs, newest first, hard-capped at **50**. There are no query parameters — no
`limit`/`offset`, no status or type filter, no pagination cursor. Filter client-side.
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
Request: `{ "provider": "gemini"|"mistral"|null }` (null = the server default; see
`GET /api/ai/providers`). The job must be `completed`.
Response: `{ "analysis": "<markdown>", "model": "<model id>" }`.
### POST /api/v1/jobs/compare/analyse — AI (descriptive) analysis comparing several jobs
Request: `{ "ids": [uuid, …], "provider": string|null }` — **2 to 6** ids, each completed and
yours (`400 "Expected 2–6 job IDs"` otherwise). Same response shape as `/analyse`.
### POST /api/v1/jobs/patch-preview — preview a strategy's inputs after overrides, without running
Request: `{ "strategy_id": uuid, "params_override": {…}|null }`. Returns
`{ "code": "<pine source with the defaults rewritten>" }` — the cheap way to check a
`params_override` resolves before launching a job with it.

---

## Data

### Data sources

A source can only serve a symbol it has a ticker for — the per-symbol tickers on
`GET /api/v1/data/symbols` are the authority, and a source whose ticker is `null` returns `400`.

| `data_source` | Coverage | Notes |
|---|---|---|
| `yahoo` | Equities + crypto | Default, no account needed. **Refuses any intraday range older than 730 days.** |
| `saxo` | EU + US equities | Needs a connected Saxo account. `saxo_uic` must be non-null (crypto has none). |
| `alpaca` | US equities + US-dollar crypto pairs | Needs a connected Alpaca account. Crypto history **starts 2021-01-01**. |
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

### GET /api/v1/data/saxo-catalog → array of cached Saxo datasets
The Saxo-sourced subset of the catalog, for the connected Saxo account. Same entry shape as
`/data/catalog`. Empty if no Saxo account is connected.

### POST /api/v1/data/fetch — download/cache OHLCV
```
symbol_id  uuid    required
timeframe  string  required
from_date  date    required
to_date    date    required
source     string  optional  (default "yahoo")
```
Response: the resulting catalog entry.

### GET /api/v1/data/massive-status → `{ "configured": bool }`
Whether the Massive data source is configured on the server (i.e. whether `source: "massive"`
fetches will work). No account of your own is required.

**The catalog is a shared, platform-wide cache, not per-account storage.** `POST /data/fetch` adds
to it and any account may then use the cached series. Conversely there is no user-facing way to
delete or edit a cached dataset — eviction is the retention watchdog's job, and the repair/delete
endpoints are operator-only. A `data_source` a symbol has no ticker for returns `400`, and old
intraday windows may simply not exist at the source (Yahoo cuts off at 730 days).

---

## Brokers

Every broker exposes the same three shapes: a **status** read, a **connect** write, and a
**disconnect**. Status is always safe to poll; the disconnect is idempotent (`204` even if
nothing was stored). Credentials are encrypted at rest and are stripped out of any job config
that is persisted.

| Broker | Connect over the API? | How |
|---|---|---|
| **Bitstamp** | yes | `POST /bitstamp/credentials` — API key + secret |
| **Alpaca** | yes | `POST /alpaca/keys` — key id + secret (the OAuth flow is browser-only) |
| **Lightspeed** | yes | `POST /lightspeed/credentials` |
| **IBKR** | yes | `POST /ibkr/settings` — host/port of your own TWS or Gateway |
| **Saxo** | **no** | OAuth + PKCE redirect; must be completed in the web UI |

### Saxo — `GET /api/v1/saxo/status`
```
connected               bool
expired                 bool          (access token past expires_at; false when not connected)
expires_at              datetime|null
env                     string        ("sim"|"live"; falls back to the server default when
                                       not connected — this field is never null)
margin_trading_allowed  bool|null     (Saxo IsMarginTradingAllowed, captured at connect)
```
`POST /api/v1/saxo/connect` (body `{ "env": "sim"|"live" }`, optional) returns
`{ "url": … }` — an authorization URL you must open in a **browser**; the PKCE verifier is held
server-side against the `state`, so the redirect cannot be completed from a script.
`DELETE /api/v1/saxo/disconnect` → `204`.

### Alpaca — `GET /api/v1/alpaca/status`
```
connected         bool
mode              string|null   ("oauth"|"apikey")
region            string|null   ("us"|"eu")
env               string|null   ("paper"|"live")
account_id        string|null   (Alpaca account_number)
scope             string|null   (OAuth scopes; null in apikey mode)
multiplier        int|null      (1 = cash, 2 = margin, 4 = day-trading margin)
shorting_enabled  bool|null
```

### POST /api/v1/alpaca/keys → `204` — connect with an API key pair
```
key_id  string  required
secret  string  required
region  string  optional  ("us" default; "eu" is rejected 400 — not supported yet)
env     string  optional  ("paper" default | "live" = REAL MONEY)
```
The pair is verified against Alpaca's `/v2/account` before it is stored, and `multiplier` +
`shorting_enabled` are captured from that same response — which is how the bot knows whether a
short entry is even possible on your account. Saving keys clears any OAuth token, and vice versa:
an account is in exactly one `mode`.

`POST /api/v1/alpaca/connect` (body `{ "region": "us" }`, optional) returns an OAuth
`{ "url": … }` for the browser flow. `DELETE /api/v1/alpaca/disconnect` → `204`.

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

`GET /api/v1/bitstamp/status` → `{ connected, env, account }`, where `account` lists the funded
currencies (Bitstamp has no account number). `DELETE /api/v1/bitstamp/disconnect` → `204`.

### POST /api/v1/lightspeed/credentials → `204`
```
api_key  string  required
account  string  required
env      string  required  ("cert" | "production")
ws_url   string  required
```
Stored as given — unlike Alpaca and Bitstamp there is **no verification round-trip**, so a bad
credential surfaces when a bot launches, not here.
`GET /api/v1/lightspeed/status` → `{ connected, account, env, ws_url }`.
`DELETE /api/v1/lightspeed/disconnect` → `204`.

### POST /api/v1/ibkr/settings → `204`
```
host       string  required  (your TWS / IB Gateway host)
port       int     required  (non-zero)
client_id  int     required  (base client id)
```
IBKR is not a hosted credential — you point PineconeX at **your own** running TWS/Gateway. The
host is SSRF-guarded: loopback, RFC1918 and link-local addresses are refused
(`400 "host is not a permitted address"`). `client_id` is the base reserved for data fetches;
each live bot is assigned `base+1` upward, skipping ids already in use by your running jobs.
`GET /api/v1/ibkr/status` → `{ connected, host, port, client_id }`.
`DELETE /api/v1/ibkr/disconnect` → `204`.

**IBKR Web API (OAuth) is a stub pending IBKR onboarding.** `GET /api/v1/ibkr/web/status` always
returns `{ "connected": false, "account_id": null }` and `POST /api/v1/ibkr/web/connect` always
returns `400 "IBKR Web API integration is pending onboarding approval."` Do not build against it.

---

## Account & keys

### GET /api/v1/auth/me → profile
```
id                  uuid
email               string
plan                "free" | "pro" | "premium" | "dedicated" | "admin"
name                string | null
phone               string | null
telegram_handle     string | null
telegram_bot_token  string | null   (returned decrypted — it is your own credential)
telegram_chat_id    string | null
github_id           int    | null
github_linked_repo  string | null   ("owner/repo")
created_at          datetime
dedicated_vps       object | null   ({ subdomain, status }; Dedicated tier only)
```

### PATCH /api/v1/auth/me — update profile fields → the same profile object
All fields optional: `email`, `phone`, `telegram_handle`, `telegram_bot_token`,
`telegram_chat_id`, `github_linked_repo`.

**Two different update semantics, which is easy to get wrong.** `telegram_*`,
`github_linked_repo` and `email` are *coalesced* — omitting them (or sending `""`) leaves the
stored value alone, so they cannot be cleared here. `phone` is written **unconditionally**:
omitting it **erases** your stored phone number. Send it back on every PATCH unless you mean to
clear it. A non-empty phone must start with `+` and hold 7–15 digits.

Changing `github_linked_repo` also moves the sync webhook: it is deleted from the old repo and
registered on the new one.

### DELETE /api/v1/auth/me → `204` — GDPR Art. 17 erasure. **Irreversible.**
Stops and deletes every running job, hard-deletes your strategies, bot events and parse errors
(cascading to jobs, results, shares and all stored broker credentials), anonymises the account
row, and invalidates the session. Billing columns are retained for tax records. There is no undo
and no confirmation step — the request itself is the confirmation.

### POST /api/v1/auth/telegram/test → `204`
No body. Sends a canned message to the Telegram bot token + chat id stored on your profile, so
you can prove the wiring before a bot depends on it. `400` names the missing half
(`"Telegram bot token not configured"` / `"Telegram chat ID not configured"`) or relays
Telegram's own error.

### GET /api/v1/newsletter/me → `{ "subscribed": bool }`   ·   PUT /api/v1/newsletter/me — body `{ "subscribed": bool }` → `{ "subscribed": bool }`
Opt your account in or out of the product-updates newsletter (keyed by your account email).
### GET /api/v1/auth/keys → list your keys (metadata only; never the secret)
### POST /api/v1/auth/keys — mint a key. **Session-JWT only** (an API key cannot mint keys → 403).
Request `{ "name": string }`. Response `{ id, name, key_prefix, key }` — `key` shown once.
### DELETE /api/v1/auth/keys/{id} — revoke → `204`

---

## GitHub

Link a GitHub repo and import Pine files from it (see
`POST /api/v1/strategies/from-github`). **Linking is browser-only** — the OAuth redirect cannot be
completed from a script — but once linked, the read endpoints work with an API key. All of them
return `400 "No GitHub account linked"` until the browser flow has run once.

### GET /api/v1/auth/github → `{ "url": … }`
The authorization URL to open in a browser (scopes `read:user user:email repo write:repo_hook`).
`GET /api/v1/auth/github/callback` and `.../exchange` are steps of that redirect dance and are not
useful to call directly.

### GET /api/v1/auth/github/repos → `[{ "full_name": "owner/repo" }]`
Note: **private repos only** — public ones are deliberately filtered out.
Set the chosen one via `PATCH /api/v1/auth/me` `{ "github_linked_repo": "owner/repo" }`.

### GET /api/v1/auth/github/files → array
The linked repo's tree, grouped by file **stem** — the unit `from-github` imports.
```
stem   string  (path without extension, e.g. "strategies/mrpivot")
pine   bool    (a <stem>.pine exists)
json5  bool    (a <stem>.json5 exists — the params override, see /strategies/{id}/params)
md     bool    (a STRATEGY.md exists in that stem's directory)
```

### GET /api/v1/auth/github/file?path={repo-relative path} → `{ "content": "<file text>" }`
Path traversal (`..`, `.`) is rejected `400 "invalid path"`.

### POST /api/v1/auth/github/sync-webhook → `204`
No body. Re-registers the push webhook on the linked repo — the repair for a webhook that was
deleted on the GitHub side. `204` means "request sent"; it does not prove GitHub accepted it.

---

## Dedicated VPS & SSO

The **Dedicated VPS** tier gives a customer a single-tenant, isolated PineconeX instance at
`<subdomain>.pineconex.com` (their own API + runner + database). The endpoints below run that
tier. They are **outside** the `/api/v1` API-key surface — they use browser **session** auth or
**admin** auth, so an API key (`pcx_live_…`) cannot call them. Documented for completeness.

### GET /api/dedicated/sso — mint an SSO handoff link (session auth)
Called on `pineconex.com` by a signed-in user who owns an **active** Dedicated instance. Returns a
short-lived login link to their box:
`{ "url": "https://<subdomain>.pineconex.com/api/auth/sso?token=<jwt>" }`.
The token is HS256, signed with that instance's per-instance secret, **60-second TTL**, bound to
the subdomain. `400` if the account has no active Dedicated VPS. (This is what the "Go to my VPS"
button calls.)

### GET /api/auth/sso?token={jwt} — Dedicated-instance login (no auth header)
The **only login path on a Dedicated instance** (`INSTANCE_MODE=dedicated`) — there is no OAuth
and no login page. Verifies the handoff token against the instance's `SSO_SECRET`, checks the
email against the instance allowlist, admits it as a **box admin**, sets the refresh cookie, and
`302`s into `/app/strategies`. `401` on a bad/expired token, a non-allowlisted email, or a
non-dedicated instance. Security rests on the per-instance HMAC secret + TTL, not on the URL.

On a Dedicated instance the admitted owner is a **box admin**, which unlocks the operator surface
under `/api/admin/*`. That surface is documented separately — see the
[`vps` branch of the docs repo](https://github.com/Pineconex/pineconex-web-api/tree/vps).

---

## Limits & errors

- **Auth** is header-only: `Authorization: Bearer pcx_live_…`. Keys are never accepted in the URL
  (query string) — including for the SSE logs stream.
- **Rate limits (per user, `429`):** validation ≈ 20/min, job launches ≈ 30/min. On `429`, back off
  and retry later — don't loop.
- **Plan quotas** are enforced at launch/create time and come back as **`400` with a message
  naming the limit** (not `403`), e.g. `"Concurrent job limit reached (1). Wait for a running job
  to finish."`

  | Plan | Concurrent jobs | Strategies | ML models |
  |---|---|---|---|
  | `free` | 1 | 5 | 0 |
  | `pro` | 5 | unlimited | 0 |
  | `premium` | 10 | unlimited | unlimited |

  (The numbers are server-configurable defaults, so treat the error message as authoritative.)
- **Plan-gated features (`403 forbidden`, no message):** **robustness** (`POST /jobs/robustness`),
  **stress** (`POST /jobs/stress`) and the **ML model** endpoints (`/models/*`) require **Premium**.
  The two job types are also the most expensive on the platform (a permutation test is N full
  backtests), so the gate doubles as a cost control. Backtest, sweep and **live trading are open to
  every plan, free included** — a rejection on `/jobs/live` is the concurrent-job quota, not the
  tier.
- **A disabled or deleted account is `401`, not `403`** — the plan is re-read from the database on
  every single request, so revoking access takes effect immediately, mid-session and mid-key.
- **Sizes:** request bodies are capped at 1 MiB (`413`); strategy source at 256 KiB (`400`).
- **Symbols** are plain tickers (e.g. `AAPL`) — no `/`, `\`, or `..`; invalid symbols return `400`.
- **Errors** are `{ "error": string }` with the HTTP status; `401` = missing/invalid/expired key.

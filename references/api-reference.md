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
locked        string | null   (absent when editable — see below)
live_bots     int             (live bots running this strategy right now)
last_live_run datetime | null (when it last went live; only meaningful while live_bots == 0)
```

**`locked` — write protection.** A strategy is write-protected while a live bot is running it, or
while it is held in the fleet snapshot. `PUT /strategies/{id}`, `PUT /strategies/{id}/params` and
`DELETE /strategies/{id}` all return `400` with `locked` as the message. The reason: a bot runs the
code it was launched with — the source is copied into the container at launch and never re-read —
so editing under a running bot cannot change what is trading, it only makes the file disagree with
the account. The snapshot extends the same rule forward: a restore relaunches from this file, so an
edit while the bots are down changes what Restore will start.

`live_bots` and `last_live_run` are an **either/or**: while bots are running, "now" is the answer
and the date is stale. Never present both.

Only set for strategies **hosted on PineconeX**. A GitHub-backed strategy (`github_stem` non-null)
is never locked or counted here — those rows are already read-only through the API, and the file
itself lives somewhere this platform does not control.

### GET /api/v1/strategies/{id} — get one   ·   PUT /api/v1/strategies/{id} — update (`{code}`)   ·   DELETE /api/v1/strategies/{id}

All three writes (`PUT` code, `PUT` params, `DELETE`) return **`400` with the `locked` message**
while the strategy is running on a live bot or held in the fleet snapshot. Check `locked` on the
`StrategyResponse` before offering to edit — and when a write is refused, do not retry it or work
around it: the correct advice is to stop the bot, or to take a new snapshot that excludes it.
GitHub-backed strategies are never lockable and are already read-only through the API.

### GET /api/v1/strategies/{id}/inputs
Response: array of input specs (the strategy's `input.*` variables), used to build parameter
overrides. Each has `kind` plus `var_name`, `title`, `defval`, `swept`, and kind-specific fields:
- `int` / `float`: `minval`, `maxval` (optional), `step`
- `bool`: (just defval)
- `string`: `options: [string]`

### GET /api/v1/strategies/{id}/universe → array of strings
The cross-symbol universe the strategy reads via `request.security`, in call order, one entry per
instrument (the same symbol at two timeframes is one entry). `[]` when the strategy reads none.

This is the **book** for a portfolio backtest, a portfolio sweep, and `mode: "portfolio"` on
`/jobs/live` — those job types take no symbol list, because the strategy names its own legs. Use
this to show a user what a portfolio run will actually trade before launching it.

Symbols come back exactly as written in the source and are **not** matched against the catalog, so
this call is cheap and never fails on an unknown symbol; the launcher does the matching and
rejects what it cannot trade. Both declaration styles resolve — a literal `array.from("A","B")`
universe *and* legs bound to `input.string` variables.

### GET /api/v1/strategies/{id}/params
→ `{ "params_json5": string|null, "symbols": string[], "parse_error": string|null }`

`symbols` is the file's instrument list, parsed server-side so a client needs no JSON5 parser.
`parse_error` is set when a stored file cannot be read — which is **not** the same as a file with
no overrides, and used to be indistinguishable from it.

### PUT /api/v1/strategies/{id}/params — body `{ "params_json5": "<json5>" }` → `{ "warnings": string[] }`

**Refused (`400`)** when the file is not valid JSON5, an entry has no `symbol`, or a config names
a `tf` the platform cannot run. The column used to be opaque text that accepted anything, and the
only reader was a browser `catch {}` that silently launched with zero overrides.

**Warned (still saved)** for keys that are not inputs of the strategy, and for several configs
sharing one `(symbol, tf)` — the latter is legitimate (ranked sweep leaderboards) and is reported
only so you know to use the config selector.

### POST /api/v1/strategies/{id}/params/validate — body `{ "params_json5" }`
→ `{ "ok": bool, "errors": string[], "warnings": string[] }`. Same checks, without writing.

### POST /api/v1/strategies/{id}/params/resolve
Body `{ "symbol", "timeframe", "config_index"?, "params_json5"? }` →
```
config_index     int|null    index into the symbol's configs, null when nothing matched
candidate_count  int         how many configs exist at this (symbol, timeframe)
candidates       array       { index, tf, label, summary } — for a config picker
tf, htf, ltf     string|null the chosen config's timeframes
label            string|null the config's own `label`, if it has one
params_override  object      pass straight to a launch endpoint
miss             object|null why nothing matched: no_file | no_symbol | no_timeframe
```

**The one place `(symbol, timeframe) -> overrides` is decided.** Symbol and timeframe are matched
**canonically**: `XOM` matches `NYSE:XOM` (though `NYSE:XOM` and `NASDAQ:XOM` stay distinct), and
`1H` / `1h` / `60` / `60m` are one timeframe. Exact string equality missed all of those and fell
through to the strategy's own defaults without a word.

An **empty `timeframe`** returns every config the symbol has, for a picker that lists them all and
takes the execution timeframe from the selection.

Selection among same-timeframe configs is **by index, and the list is ordered** — several real
files are ranked sweep leaderboards where index 0 is the winner. Resolving by "first match on tf"
makes candidates #2..#N unreachable, which is exactly how the launch screens once disagreed with
each other about the same file.

`params_json5` in the body resolves against supplied content instead of the stored column, for
GitHub-backed strategies whose file lives in the user's own repo.

Write-protected with the code, and for the same reason: the params travel with the `.pine` and a
live bot froze both at launch. Returns `400` with the `locked` reason when the strategy is live or
in the fleet snapshot.

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

**Validation runs the same engine the job would.** The effective runtime is resolved exactly as
dispatch resolves it — an explicit `//@runtime=<version>` pin if present, otherwise the
admin-promoted default — so a pinned strategy is checked by *that* engine rather than by whatever
the API host happens to carry. If an explicit pin cannot be honoured you get a **warning** saying
so (and the validation still runs on the fallback); an unpinned strategy never warns, because the
caller expressed no preference and there is nothing to act on. Rate-limited to ~20/min per user
(`429`).

---

## ML models — **Premium plan or higher**

Upload an ONNX model and call it from Pine with the `ml.*` namespace. All three endpoints are
gated on **Premium**, `admin` or `dedicated` (`403` on `free` and `pro`). Models are private to
your account.

`dedicated` is admitted here even though its *quota* is the free tier's: on the shared platform
that plan is a billing marker, because a Dedicated VPS customer's compute lives on their own
instance where they are `admin`. Entitlement and compute budget are separate questions.

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

### Training a model on the platform — `POST /api/v1/jobs/{hmm,clf,prf}-train`
Same plan gate as the rest of this section (Premium / admin / dedicated; `403` below that). These
are **job launches**, so they answer `201` with a `JobResponse` and are polled like any other job;
the trained model lands in the registry above under `model_name`, adding a **version** rather than
replacing an existing lineage, so a strategy pinned to `:2` keeps working.

Each trainer takes its own explicit windows, and the split is the point: the model is fitted on
`train`, its threshold or hyper-parameters are chosen on `valid`, and `test` is scored once and
never tuned against. `hmm-train` has no `valid` window because it has nothing to choose there.

```
POST /jobs/hmm-train    — regime model (Gaussian HMM over per-bar features)
  symbol | universe  string | string[]   one of the two (basket feature sets take `universe`)
  timeframe, data_source, model_name     required
  train_from, train_to, test_from, test_to   date, required
  states       int      optional  (default 3; capped at 6 — beyond that the states stop
                                   having an interpretation and refits relabel them)
  features     string   optional  (default "ret,vol")
  lens         string   optional  ("id:len,id:len" per-feature lookbacks)
  vol_window   int      optional
  max_iter     int      optional

POST /jobs/clf-train    — direction model (logistic regression over the next `horizon` bars)
  symbol, timeframe, data_source, model_name                        required
  train_from, train_to, valid_from, valid_to, test_from, test_to    date, required
  features     string   optional
  horizon      int      optional  (label horizon in bars)
  label_decay, dead_band, vol_window, lens, reward, max_iter        optional

POST /jobs/prf-train    — trade filter (tree ensemble scored on a STRATEGY's own trades)
  strategy_id  uuid     required  (the filter is fitted to this strategy's trades)
  symbol, timeframe, data_source, model_name                        required
  features     string[] required
  train_from, train_to, valid_from, valid_to, test_from, test_to    date, required
  extra_series array    optional  (extra input series)
  label, algo, universe, lens, reward, min_trades                   optional
```

A model is a fit to **one instrument, one timeframe and one stretch of history**, and the launch
path enforces that at USE time as well: a backtest or sweep whose date range overlaps the model's
`train_from..train_to` is refused (that would be an in-sample curve reported as a backtest), as is
a significance or stress run on a model-pinned strategy (the fit sits outside the permutation
loop, so the null is deflated and the p-value comes out too small), and a single-name model served
a different symbol or timeframe. Live is exempt from the window rule only — fitting through
yesterday to trade tomorrow is the workflow, not lookahead.

---

## Jobs

All launch endpoints return **`201`** with a `JobResponse` when the job starts immediately, or
**`202`** with `status: "queued"` when no runner has room for it right now. Poll
`GET /api/v1/jobs/{id}` for `status` either way — a queued job needs no further action, the
scheduler starts it when capacity frees up.

**JobResponse**
```
id             uuid
strategy_id    uuid | null
job_type       "backtest" | "sweep" | "robustness" | "stress" | "live"
status         "queued" | "pending" | "running" | "completed" | "failed" | "cancelled" | "timeout"
container_id   string | null
config         object            (serialized JobConfig; varies by job_type)
auto_restart   bool              (live only; else false)
created_at     datetime
finished_at    datetime | null
progress_done  int | null        (sweep/robustness)
progress_total int | null
error_message  string | null
data_split_adjusted  bool | null  (provenance of the bars this job ran on — see below)
queue_position   int | null    (queued only; 1-based, among YOUR OWN queued jobs)
queue_reason     string | null (queued only; why it is waiting, in one sentence)
queue_eta_secs   int | null    (queued only; rough seconds until it starts, or absent)
```

**`data_split_adjusted` is provenance, and `null` is not `false`.** `true` = the bars were
split-back-adjusted, `false` = raw as the vendor served them, `null` = **unknown**: no dataset was
bound (live bots), the series came from outside the catalog, or the job predates the field. There
is one catalog row per (symbol, source, timeframe), so a re-fetch under a different adjustment
setting overwrites the Parquet in place — two runs of the *same* backtest can then legitimately
disagree, and this field is the only record of which series each one saw. When comparing results
across jobs, compare this first. Never render `null` as "raw".

Terminal: `completed`, `failed`, `cancelled`, `timeout`. Non-terminal: `queued`, `pending`,
`running`.

**`queued` means accepted and waiting for capacity** — no container exists and nothing is being
spent. It is not an error and needs no retry: the job starts on its own. `pending` is the
sub-second gap between the row being written and the container starting.

A job is queued when either the fleet has no room for it or your plan's concurrent-job limit is
already reached. Each plan also has a QUEUE DEPTH, separate from its concurrent limit; a launch
that would exceed both is refused `400` rather than queued. `queue_reason` says which of the
three situations applies, and they call for different responses — wait, wait, or ask an
administrator for capacity.

### GET /api/v1/jobs/{id}/wait — block until the job is done

`GET /api/v1/jobs/{id}` that blocks. **Same response shape**, so a caller that already parses the
job endpoint needs no new parsing. Returns within milliseconds of the job reaching a terminal
status.

```
timeout   int     optional  seconds to block; default 60, max 300
on        string  optional  "finish" (default) | "start"
```

**If the returned `status` is still non-terminal, the wait timed out — call it again.** There is
no separate flag; the status is the answer. A forty-minute sweep is about eight calls at
`timeout=300`, each carrying a real result, rather than several hundred polls that say "still
running".

`on=start` returns as soon as the job leaves the queue and is on a runner. It exists because a
queue makes "has my work begun" and "is my result ready" two different questions with different
answers for minutes at a time. A job that fails or is cancelled while queued also satisfies a
`start` wait — it will never start, so blocking on it would be waiting for something that cannot
happen.

At most **32** concurrent waits per account; use the bulk form below to watch more than that on
one connection.

### POST /api/v1/jobs/wait — watch several jobs on ONE connection

```
ids      uuid[]  required  up to 100
mode     string  optional  "any" (default) | "all"
timeout  int     optional  seconds; default 60, max 300
on       string  optional  "finish" (default) | "start"
```

Returns an array of the SAME job objects, one per requested id, in the order given — every
requested job's current state, so one response is a complete picture rather than a notification
you then have to follow up on. `any` returns as soon as one job satisfies the wait; `all` waits
for every one.

Use this for a fan-out (a multi-symbol scan, a set of robustness runs). N parallel single-job
waits is the same waste as polling in a different form. An id you do not own is a `404` for the
whole call, not a silently shorter array.

**`DELETE /api/v1/jobs/{id}` on a queued job cancels it outright.** There is no container to
stop, so this is free and immediate.

**Live bots are never queued.** A bot that starts twenty minutes late has missed twenty minutes
of the market on a decision nobody made, so `POST /api/v1/jobs/live` still refuses with `400`
when the concurrent limit is reached rather than waiting.

**Scheduling is by MEMORY, not by job count.** A portfolio book reserves 6 GB and a single
backtest 512 MB, so a runner with four jobs on it may have room for a fifth backtest and none
for a book. That is why a large job can sit queued while smaller ones launched after it start
immediately — though only for a few minutes: a job that has waited long enough has capacity held
for it rather than being overtaken indefinitely.

### Fields common to backtest / sweep / robustness / stress
```
strategy_id           uuid    required
symbol                string  required   (e.g. "AAPL")
timeframe             string  required   ("1m","5m","15m","30m","60m","90m","1D","1W","1M"; case-sensitive: 1M=monthly, 1m=1min; 1H/1h alias→60m; other→400)
from_date, to_date    date    required   (TRADE GATE, not a data slice — see below)
data_source           string  required   ("yahoo","saxo","massive","ibkr","alpaca","bitstamp"
                                          — see Data sources below; not every source carries
                                          every symbol)
htf_timeframe         string  optional   (higher timeframe for request.security; PREMIUM)
htf_data_source       string  optional
intrabar_timeframe    string  optional   (request.security_lower_tf)
intrabar_data_source  string  optional
```

**Multi-timeframe is a Premium feature, and it is detected from the STRATEGY, not just this
field.** Sending `htf_timeframe`, *or* running a strategy whose own `request.security` names a
timeframe other than the chart's, is `400` on `free` and `pro`:

```
this strategy reads a second timeframe (1D) while the chart runs at 60m, and multi-timeframe
strategies are included in Premium. Run it on a single timeframe, or upgrade.
```

Omitting the field does not avoid it — the server derives the HTF from the strategy's own
`request.security` calls, which is what makes the ordinary Pine idiom work at all. The same check
applies on `/backtest`, `/sweep`, `/robustness` and `/jobs/live`.

A cross-symbol `request.security` at the **same** timeframe as the chart is not multi-timeframe
and is not gated by this.

**`intrabar_timeframe` is also what makes `use_bar_magnifier` do anything.** A strategy declaring
`strategy(use_bar_magnifier = true)` resolves a bar that touches **both** a resting stop and a
resting take-profit by walking the intrabar sub-bars and booking whichever leg price reached first
(a sub-bar touching both is itself ambiguous → the stop wins). Without an intrabar series the flag
is a **silent no-op** on backtest and sweep — the run succeeds, logs a warning, and keeps the
optimistic default of booking the take-profit. Robustness and stress need no series: their
permuted bars resolve the same tie deterministically from the bar's own OHLC (Brownian bridge), so
the p-value stays reproducible. Expect a magnified run to score *worse* — the bias is what was
removed.

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
params_override  object  optional   { var_name: number|bool|string }
```

### POST /api/v1/jobs/portfolio-backtest — **Pro plan or higher**
One strategy run across N symbols against ONE shared account (a "book"). `403` on `free`: this is
the multi-symbol job the pricing page has always sold as a paid feature, and it is N instruments
of work per launch rather than one. Every leg runs the same
per-bar engine as `/backtest`, but all legs transact one ledger: closes credit shared cash,
`percent_of_equity` sizes off the whole book, and a gross-exposure cap bounds combined long+short.
The universe is the strategy's OWN `request.security` basket (declared in Pine with
`array.from("SYM", …)`), so there is **no `symbol` field** — the strategy names the symbols, and
each leg is also a peer the others rank against. Pair trading is the 2-symbol case. Research /
backtest only; live trading runs each symbol as its own independent bot.

Common fields EXCEPT `symbol` / `htf_*` / `intrabar_*`, plus:
```
initial_capital  number  optional  (shared book account size; default 100000)
leverage         number  optional  (gross-exposure cap ×equity: Σ|leg notional| ≤ leverage×equity;
                                    default 2.0 = full long + full short. 1.0 caps total exposure
                                    at the account size)
params_override  object  optional  ({ var_name: number|bool|string }, as /backtest)
```
**400 if the strategy's `request.security` universe resolves to fewer than 2 tradable legs.** The
results add a `legs[]` array (per-symbol trade count and net P&L contribution) on top of the usual
book summary / equity / trades.

### POST /api/v1/jobs/sweep
Above common fields, plus:
```
mode        "grid"|"random"|"rbf"|"monte_carlo"|"successive_halving"   required
trials      int    optional  (random/rbf/monte_carlo; rbf and monte_carlo call it the budget.
                              For successive_halving it is the number of PARAMETER SETS, and
                              omitting it is what selects the full grid — see below)
rounds      int    optional  (successive_halving only; default 4, 1..16. Halving rounds.
                              Ignored by the other modes)
metric      string optional  (default "net_pnl_pct". The objective the search hill-climbs.
                              One of: net_pnl_pct | return_over_dd | sharpe | profit_factor |
                              expectancy | win_rate | max_dd_pct (minimised) — or a custom
                              expression "expr: <formula>", see below.
                              ONLY rbf and monte_carlo steer — grid and random ignore it,
                              see below)
min_trades  int    optional  (default 5; 0..1000. A trial with fewer closed trades can never
                              win — not in the search, not in the results ranking. 0 disables)
stability   float  optional  (default 0 = off; 0..5. How much the objective has to hold up
                              when the parameters move. Applies to EVERY mode — see below)
stability_radius
            float  optional  (default 0.12; 0..1. Neighbourhood radius as a fraction of each
                              parameter's swept range. Ignored when stability is 0)
bootstrap   bool   optional  (default false. Score each trial on 500 block-resamples of its
                              OWN closed trades and rank it by the 25th percentile instead of
                              the single run it got. Applies to EVERY mode — see below)
perm_seed   int    optional  (sweep a bar-PERMUTED copy of the series instead of the
                              real bars — see below)
perm_block  int    optional  (default 1; 1..1000. Permutation block size in bars.
                              Ignored unless perm_seed is set)
perm_penalty
            int    optional  (default 0 = off; 0..50. Score every trial against k permuted
                              copies of the market and rank it on `objective - mean(null)`
                              instead of on the objective. Applies to EVERY mode. Costs
                              (k+1)x the sweep. Cannot be combined with perm_seed —
                              see below)
```
Which `input.*` vars are swept comes from the strategy (`GET .../inputs`, `swept: true`).
A 1-axis grid is valid — the cartesian product of a single axis is that axis.

**400 if the strategy has no `//@sweep` parameters.** There is no search space, so grid degenerates
to the single authored point and rbf exits non-zero in the container. Mark an input by putting
`//@sweep` on the line above its `input.int` / `input.float` call.

**`//@sweep` is a MARKER and takes no arguments — the range comes from the input's own
`minval` / `maxval` / `step`.** The shape invites the opposite guess, and the wrong form parses
without complaint and does nothing:

```pine
//@sweep
window = input.int(21, "Fit window", minval = 15, maxval = 60, step = 15)   // → 4 values
```

Writing a range after the marker (`//@sweep <lo>..<hi> step <n>`) parses without complaint and is
discarded; the same input left at `minval = 8, maxval = 500` is 493 values, not 4.

So on a swept input those three arguments **are** the search space and must be chosen for the
search, not for what the form should accept. An input missing `minval`/`maxval` covers its entire
range: the launch is refused with `Grid would require N combinations (limit 100000)` rather than
run, which is loud but costs a round trip. Multiply the axes out before launching. And do not mark
an input the script cannot reach on its current path (a gate that selects one branch makes the
other branch's thresholds inert) — every trial such an axis adds is an exact duplicate of another.

**`metric` only aims `rbf` and `monte_carlo`.** Grid enumerates every cell and random samples
blindly: both evaluate a predetermined set of points no matter what the objective is, and you rank
the output afterwards. Only `rbf` and `monte_carlo` hill-climb, so only they have a search to aim.
Sending `metric` with grid/random is not an error — it simply has no effect.

**`successive_halving` is a cheaper way to run a search, not a different search** (Karnin et al.
2013; Jamieson & Talwalkar 2016). It scores every parameter set on a short run, drops the worst
half, doubles the run for the survivors, and repeats. The set halves as the run doubles, so every
round costs about the same and the ladder costs roughly `rounds / 2^(rounds-1)` of evaluating
everything at full length — 3x cheaper at 5 rounds, 5x at 6.

The resource is **the trade gate**, not the data. Round 1 may only open trades in the first fraction
of the date range; the last round trades over all of it. Every parameter set still executes from the
first bar with its full indicator history at every round, so a long lookback is not penalised early
— which it would be if the series were truncated instead.

- **`trials` selects where the parameter sets come from.** Omit it and they are the full grid, which
  is the case the mode exists for: halving is what makes an exhaustive grid affordable. Set it and
  they are that many random samples, for a space too large to enumerate at all.
- **The ladder can come back shorter than `rounds`.** A first round needs at least 250 bars to trade
  over, and once the survivor count bottoms out there is nothing left to halve. Six rounds against a
  few years of daily bars is three in practice; against two years of 5m data it is six. The job log
  says when it was shortened and why.
- **`metric` DOES apply**, unlike grid and random. The objective decides which parameter sets are
  eliminated, so changing it changes the answer.
- **Every trial in `results.json` carries `rung` (1-based) and `rung_resource`.** Only the last
  round's trials ran on the full range, and only they are comparable with each other. Filter to
  `rung == max(rung)` before ranking, or you will pick a winner measured on a fraction of the data.
  The earlier rounds are shipped so the elimination can be audited, not so it can be ranked.
- **`total_trials` is every parameter set STARTED**, not the evaluations spent and not the survivors.
  That is the count the MinBTL arithmetic wants. Halving does not make a winner more trustworthy; by
  making a wider search affordable it makes the multiple-testing bar higher, not lower.
- Robustness (`stability`) is annotated over the last round alone, since an e-ball spanning rounds
  would average scores measured on different amounts of data.

**`monte_carlo` is the cross-entropy method, not the simulated annealing that carried this name
until 2026-07-13.** It samples a population of candidates each round, keeps the best fifth of them,
refits its sampling distribution to those, and repeats — spending a quarter of the budget before it
commits to anything. Practical differences from `rbf`:

- It converges to distinct parameter sets and **stops early** if the distribution collapses onto
  ground it has already covered, so a finished job may report fewer trials than `trials` asked for.
  That is the search reporting it had nothing left to learn, not a truncated run.
- The candidates within a round are independent, so it evaluates them **in parallel**; `rbf` must
  refit its surrogate after every single point and cannot. On a large budget it finishes in roughly
  a seventh of `rbf`'s wall-clock for comparable results.

**Do not read either steered mode as "better than random".** Benchmarked on 5 instruments x 3 seeds
at equal budget (2026-08-02), `rbf` beat blind random by +0.68 and `monte_carlo` by +0.38 net PnL %,
and **neither margin is statistically distinguishable from zero** (95% CIs [-0.91, +2.26] and
[-1.07, +1.84]). Steering is cheap to try and has not been shown to be worth choosing over `random`
on this evidence. What the same benchmark does establish is that both are safe: the deleted
annealing mode lost to random consistently, and these two do not.

**Custom objective: `"metric": "expr: <formula>"`.** An arithmetic expression over the trial
metrics, e.g. `"expr: net_pnl_pct - 0.5 * max_dd_pct + 0.1 * trades"`. Variables: `net_pnl_pct`,
`max_dd_pct`, `ulcer`, `sortino`, `cvar`, `trades`, `win_rate`, `profit_factor`, `expectancy`,
`sharpe`, `return_over_dd`, `cagr`, `bars_in_trade`, `time_in_market_pct`,
`effective_exposure_pct`, and on a basket `leg_win_frac`, `leg_sd`, `leg_mean`, `leg_min`,
`leg_silent` (aliases: `pnl`, `dd`/`max_dd`, `ulcer_index`, `cvar_pct`/`expected_shortfall`,
`n_trades`, `winrate`, `pf`, `romad`/`return_dd`, `cagr_pct`, `bars_held`, `time_in_market`,
`exposure`, `leg_wins`, `leg_spread`, `worst_leg`). Operators:
`+ - * / ( )` and
numeric literals. The search MAXIMISES the expression as written — a penalty term gets a minus
sign, and `max_dd_pct` is a positive percentage (a 12% drawdown is `12`), so subtract it to punish
risk. A malformed expression is rejected 400 with the parser's error. The `min_trades` floor
applies to custom objectives too, and a division by zero disqualifies the trial rather than
winning by infinity.

**Three exposure-aware objectives worth naming, because the sign is easy to get backwards.** The
search MAXIMISES whatever it is given, so an objective you want minimised must be negated:

```
"expr: cagr / time_in_market_pct"   return per unit of time at risk — 12%/yr while invested a
                                    third of the time beats 15%/yr always invested. Raise
                                    min_trades with it: a setting that is in the market almost
                                    never can divide its way to a huge score
"expr: -time_in_market_pct"         MINIMISE exposure. Unnegated this searches for the setting
                                    that is invested MOST, which is the opposite of the intent
"expr: cagr"                        annualised growth. Unlike net_pnl_pct it is comparable
                                    across runs of different lengths
```

`cagr` is `NaN` when the run has no known span or the account was wiped, and a non-finite objective
disqualifies the trial rather than scoring it. On a PORTFOLIO sweep `time_in_market_pct` saturates
at its 100 clamp once legs overlap — prefer `bars_in_trade` there.

**Pricing holding time.** `bars_in_trade`, `time_in_market_pct` and `effective_exposure_pct` are the handles for "this edge is not worth the
time it takes". Total bars held is `bars_in_trade * trades`, so a per-bar holding cost is one term:
`"expr: net_pnl_pct - 0.02 * bars_in_trade * trades"`. This is how a per-step penalty from a
reinforcement-learning reward is carried over — a sweep scores a finished backtest rather than a
trajectory, so there is no per-bar hook, but the episode sum of a constant per-bar cost is exactly
that product. Two of the three degrade on a **portfolio** sweep rather than erroring, the same way
the basket variables degrade on a single instrument: `effective_exposure_pct` is `0` on a book (its
legs share one ledger, so per-symbol exposure has no book meaning) and `time_in_market_pct`
saturates at its `100` clamp once legs overlap. On a book, use `bars_in_trade`.

**`stability` is a second axis, not another objective.** `metric` says what "good" means for one
trial; `stability` says how much of that good has to survive the parameters being slightly wrong.
Each parameter set is scored by its *neighbourhood* — the mean of the objective over a ball of
radius `stability_radius` (a fraction of each axis's swept range), minus `stability` times the
standard deviation across that ball. `0` is the raw objective and the historical behaviour; `1`
ranks roughly by a neighbourhood's downside; higher trades height for flatness.

No custom `expr:` can express this, and that is the point of a separate field: an expression is
evaluated over a single trial's own numbers, and whether a result sits on a plateau or a spike is a
property of its *neighbours*.

**Unlike `metric`, it applies to every mode — but it does two different things.** On `rbf` and
`monte_carlo` it changes the SEARCH: rbf reads its surrogate as a local average rather than at a
point, and monte_carlo picks its elites by neighbourhood, so the run spends its budget hunting
plateaus. Neither costs a single extra backtest. On `grid` and `random` there is nothing to steer,
so it only scores the trials afterwards — which is sound precisely there, because those two sample
the *space* rather than the objective and so contain their plateaus at even density already.

Every trial in `results.json` carries the outcome, whether or not the run was steered:

```json
"plateau": { "score": 25.81, "mean": 25.85, "sd": 0.04, "win_frac": 1.0, "n": 8 }
```

`n` is the ball's population including the trial itself. **When `n` is below 3 the ball measured
nothing and `score` falls back to the trial's own value** — so rank on `score` only among trials
with `n >= 3`, or an isolated spike wins by never having been examined. `results.robust` records
the `eps` / `lambda` used and whether the search was `steered`.

Measured on a real run (monte_carlo, 120 trials, DBK 1D): the raw winner returned 31.44% sitting in
a ball with `sd` 3.03, while the stability winner returned 25.89% with `sd` 0.04. That is the trade
the number buys — 5.5 points of in-sample return for a neighbourhood that is flat.

**`bootstrap` is the THIRD axis, and it catches what `stability` structurally cannot.** `stability`
perturbs the parameters and holds the data fixed. `bootstrap` perturbs the data and holds the
parameters fixed: each trial's own closed trades are resampled 500 times in blocks, the objective is
evaluated inside every resample, and the trial is ranked by the 25th percentile of those 500 values.

The gap matters because **every trial in a neighbourhood runs on the same bars**. A parameter region
built on one lucky stretch of history is wide, flat and entirely fake, and the ε-ball rates it
highly precisely because all the neighbours caught the same luck. Only resampling sees it.

Like `stability` and unlike `metric` it applies to every mode: `rbf` and `monte_carlo` steer on the
resampled score, `grid` and `random` have their trials annotated with it and the results ranking
uses it. Order of operations is load-bearing — the whole objective is evaluated *inside* each
resample and one quantile is taken at the end. Bootstrapping each metric separately and evaluating
an `expr:` on the per-metric quantiles gives a number matching no actual resample, and gets the sign
backwards the moment terms are mixed (a bad world has low `net_pnl_pct` *and* deep `max_dd_pct`, so
the two want opposite tails).

**`perm_penalty` is the FOURTH axis, and the only one that changes what the search CLIMBS.** The
other three leave the objective alone — `stability` perturbs the parameters, `bootstrap` perturbs
the trades, `perm_seed` replaces the market — so a search still maximises raw in-sample performance
and still returns whichever setting fitted the noise best; they measure that setting afterwards.
This one subtracts each trial's own permuted null, so a setting has to beat noise to win at all.

Set it to `k` and every trial is run once on the real bars and once on each of `k` permuted copies
of the same market, then ranked on `objective - mean(null)`. A permuted market keeps the
instrument's returns and destroys only their ORDER — same start price, same end price, same drift,
same volatility — so the null is what this rule earns from the instrument when the ordering carries
no information. Whatever a strategy makes by simply holding a rising market, it makes there too and
gains nothing here. What survives the subtraction is timing.

The `k` nulls are built once and shared by every trial. Drawing fresh permutations per trial would
add an independent noise term to each one, and a search maximising `score - noise` finds the trial
with the luckiest nulls rather than the best rule.

**It cannot be combined with `perm_seed`** — that one replaces the whole search's market, so the
observed side would be noise and the objective a difference of two null draws whose argmax is
whichever cell drew the luckiest. The request is refused with 400 rather than one of them silently
winning. It also refuses an `intrabar_timeframe`, for the same reason `perm_seed` does: sub-bar
structure cannot be synthesized from permuted bars.

Cost is exactly `(k + 1)x` the sweep, which is why it is off by default and capped at 50. Every
trial that produced a usable null carries `perm_null`, and every trial the `min_trades` floor did
not reject carries `objective` — the value of THIS run's metric, so the edge is
`objective - perm_null`. Read `objective` rather than recomputing it: a client showing one metric
while the search ranked by another gets a confident wrong answer, and a custom `expr:` cannot be
evaluated outside the engine at all.

```json
{ "params": { "len": 20 }, "net_pnl_pct": 41.2, "objective": 3.07, "perm_null": 0.42 }
```

`perm_null` is absent when no null could be scored; `objective` is absent on a trial below
`min_trades`, whose score is a sentinel rather than a measurement.

`bootstrap` is a flag with nothing to configure, deliberately: the resample count converges, the quantile is
fixed at the 25th percentile, and the block length is derived per trial from the trade sequence's
own lag-1 autocorrelation. Any of them exposed would be a dial to turn until the strategy looked
good. Blocks rather than single trades because consecutive trades are not independent — a trending
stretch produces several winners in a row, and shuffling one at a time would destroy exactly that
and make every strategy look more robust than it is.

Every trial that had enough trades to resample carries:

```json
"bootstrap": { "score": 18.42, "mean": 24.10, "sd": 4.88, "win_frac": 0.97, "block": 3 }
```

`score` is what to rank by. A trial with **no** `bootstrap` key had too few trades to resample; that
is "not measured", never "measured as bad", and such trials are excluded from winning rather than
scored as zero. `block` of 1 means the trades showed no serial dependence and the resampling was the
ordinary iid bootstrap.

Two honest caveats. The resampled `max_dd_pct` is the drawdown of *that ordering*, stepped trade to
trade, while the trial's own `max_dd_pct` is measured bar by bar and includes open-position
mark-to-market — expect the resampled one to run shallower, and do not read them as the same number.
And resampling reduces the *variance* of each trial's score, which shrinks how inflated the winner
is (the expected max of N draws sits near `mean + sd*sqrt(2 ln N)`), but it does not make the winner
unbiased. It creates no information; the MinBTL / deflation arithmetic still applies on top.

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

### POST /api/v1/jobs/portfolio-sweep — **Pro plan or higher**
`403` on `free`, for the same reason as `/portfolio-backtest`, times the trial count.

Sweep a strategy's `//@sweep` inputs while it runs across the whole book — the same shared-account
universe as `/portfolio-backtest`. Every trial is a full book run, and results.json is the same
`SweepResults` shape as `/sweep`, so the same trials table renders it.

Common fields EXCEPT `symbol` / `htf_*` / `intrabar_*`, plus the book fields (`initial_capital`,
`leverage` — see `/portfolio-backtest`) and the search fields:
```
mode        "grid"|"random"|"rbf"|"monte_carlo"|"successive_halving"   required
trials      int    optional  (random samples / rbf|monte_carlo budget. For successive_halving
                              it is the number of parameter sets; omit it for the full grid)
rounds      int    optional  (successive_halving only; default 4, 1..16)
metric      string optional  (steered-search objective; same set as /sweep, including
                              "expr: <formula>". rbf, monte_carlo and successive_halving
                              read it — grid/random ignore it)
min_trades  int    optional  (default 5; 0..1000. Book-wide closed-trade floor)
```
**`successive_halving` here ladders the BOOK'S LEGS, not the trade gate.** Round 1 runs each
parameter set on one leg, round 2 on two, the last on the whole book, and capital is scaled to the
leg count so the money behind each leg is the same at every round. Read the early rounds as a
**filter, not an estimate**: a book's legs share one account, so a two-leg book is a different book
rather than a small sample of the full one. What an early round establishes is that a parameter set
was hopeless, never what it would have scored on the full book. Everything else matches the
single-instrument mode: `rounds` defaults to 4, omitting `trials` selects the full grid, trials carry
`rung`/`rung_resource`, and only the last round is comparable across parameter sets.

No `perm_seed` / `perm_block`: in-sample permutation is single-instrument only. All five search
modes do work here. Same rules as the other sweep and the portfolio backtest apply — **400** if the
strategy has no `//@sweep` parameters, and **400** if its universe resolves to fewer than 2 tradable
legs. Trials run serially (a book run is heavier than a single symbol), so keep grids modest on a
large universe — and note `monte_carlo` gains nothing from parallelism here for the same reason,
though it still stops early rather than re-buying a book run it has already done, which on a
portfolio is the expensive kind of waste.

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
search_mode   string  optional  ("fixed"(default)|"grid"|"random"|"rbf"|"monte_carlo" — the
                                  SELECTION PROCEDURE re-run inside every permutation. See
                                  below: this is the null hypothesis itself, and the default
                                  is only correct if the parameters were NOT found by a sweep)
search_trials int     optional  (candidates per permutation for random/rbf/monte_carlo.
                                  Ignored by fixed and grid)
min_trades    int     optional  (default 5; 0..1000. A trial with fewer closed trades can
                                  never be the winner the statistic is read from — applied
                                  identically to the real run and to every permutation)
block_size    "auto"|int optional (default "auto" — MEASURED from the series, see below.
                                  1 = single-bar permutation (destroys all serial
                                  structure); >1 = block permutation (shuffle N-bar
                                  chunks, preserving structure shorter than N — a less
                                  strict null). An integer is clamped to 1..1000.
                                  Anything else is a 400)
seed          int     optional  (RNG seed; omit for a time-seeded run. The effective
                                  seed is echoed back in the results for reproducibility)
```

**400 if the strategy has no `//@sweep` parameters** — for every `search_mode`, including `fixed`.

**`metric` is the search objective too.** An MCPT is only valid when the procedure re-run on the
permuted bars *is* the procedure that ran on the real bars. A search that optimised net PnL while
the test reported win rate would be a different procedure, and its null would not describe it — so
the search hill-climbs the statistic you ask for. Set it to whatever your sweep optimised. (Only
`rbf` and `monte_carlo` steer; `grid`/`random`/`fixed` evaluate a predetermined set of trials either
way, and the statistic merely selects the maximum.)

**`search_mode` must be the search you actually ran, not the cheapest one.** `monte_carlo` and `rbf`
are different procedures and produce different nulls — measured on the same strategy and series
(20 permutations, 246 candidates): `rbf` gave p = 0.19 against a null mean of +24.78%, `monte_carlo`
p = 0.29 against +17.74%. Neither is the "right" p-value in the abstract; the right one is the one
whose procedure matches how you actually chose your parameters. Both cost about the same to run
(82s vs 77s), so there is nothing to save by declaring the other one.

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

**`block_size` defaults to `"auto"`, and you should almost always leave it there.** The block
decides how much of the market's own memory survives into the null. That is a property of the
*instrument*, not a caller preference, and no fixed default is right across instruments — so the
engine measures the series it is about to permute and sizes the block from that. Resolution happens
in the engine rather than at the API because only the engine holds the exact bars being shuffled
(date-gated, warm-up trimmed); sizing it anywhere earlier would measure a different set of bars than
the one being permuted.

Results always report the **resolved** integer in `block_size`, plus `block_auto: true|false` so a
reader can tell "auto agreed with itself" from "the caller chose correctly". An explicit integer is
always honoured, and the measurement is still reported beside it, so an override that contradicts
the data is visible rather than silent.

One case auto deliberately will not act on: when the dependence outruns the measurable window it
caps the block and says so, because "we never found the end of the memory" is the weakest evidence
available, not grounds for shuffling in 60-bar chunks. Override with an explicit integer if you want
the longer block. (Operators can move the cap with `PERM_AUTO_BLOCK_MAX` on the runner.)

**Serial dependence — check this before trusting the p-value.** The results also carry the
measured autocorrelation of the series (same fields as `GET /api/v1/data/structure`, plus
`block_advice`, a one-line plain-English verdict, and `diag_bars`, the bars measured after the
date gate). A permutation test is only exact if the bars are **exchangeable**, and serial
dependence breaks that — asymmetrically:

| `suggested_block` vs `block_size` | What it means |
|---|---|
| equal | The null matched the series. Read `p_value` at face value. |
| **suggested > used** | The permutations destroyed structure the market really has, so the null is an **easier** opponent than reality. `p_value` is **optimistically biased** — treat it as an upper bound on the edge, and re-run at `suggested_block`. |
| suggested < used | The null kept more structure than measured: a stricter test than the series needs. A pass still counts; a fail may be the block rather than the strategy. |

`vol_acf_lag1_significant` is a separate axis: a series can have no return memory at all (so
single-bar permutation is right for a **directional** edge) while volatility clusters for dozens of
bars. If the edge is a **volatility** one — breakout, ATR sizing, vol filters — a block below
`vol_acf_decay_lag` leaves the null a calmer market than the real one.

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
model            string   optional  ("auto" | "ou_jump" | "trend_ar1" | "both"; default ou_jump)
half_lives       [float]  optional  (X axis: persistence half-lives in bars. Max 12 values)
jump_rates       [float]  optional  (Y axis: jumps per 1000 bars. Max 12 values)
paths            int      optional  (default 20; 1..100 — simulated paths per cell)
jump_sigma       float    optional  (0..50)
vol_mult         float    optional  (0.1..10)
bars             int      optional  (100..5000 — bars per synthetic path)
metric           string   optional  (the statistic each cell is scored on — same set as
                                     robustness, incl. "expr: <formula>")
survival         string   optional  (what counts as a path SURVIVING its cell;
                                     default "net_pnl_pct > 0")
seed             int      optional
params_override  object   optional  { var_name: number|bool }  (strings rejected)
```
The instrument's own calibrated coordinate is forced onto the grid, so a run is usually one row and
one column larger than the axes you pass. It is OMITTED when that coordinate is not measurable
(see `phi_clamped` / `trend_rho_clamped` below) rather than placed at a fabricated position.

**The model picks which family of markets the grid is made of.** `ou_jump` fits AR(1) to the log
*price* and generates MEAN-REVERTING markets; `trend_ar1` fits the same estimator to the log
*return* and generates MOMENTUM ones. `both` puts them on a single SIGNED axis — negative
`half_life_bars` = reversion, `0` = a random walk, positive = momentum — mirroring the half-lives
you pass either side of the centre (so pass ~half as many). `auto` measures the instrument and
picks, resolving to `both` when the verdict is inconclusive, which most series are at most
horizons; it is resolved in the engine, after the observed run, because the verdict depends on how
long the strategy holds.

**`metric` and `survival` are different settings and the report shows both.** `metric` scores each
path and drives the `median` / `mean` layers; `survival` is a pass-or-fail line and drives
`frac_profitable`, the layer the report leads with. A statistic only has to RANK paths, so any
expression works; a count needs a LINE, and "> 0" is the wrong one for profit factor (neutral 1) or
win rate (neutral 50) and undefined for an `expr:` objective — which is why the two are set apart.

`survival` is comparisons over the metric variables, joined by `and` (max 4):
```
net_pnl_pct > 0                             the default: made money
net_pnl_pct > 0 and max_dd_pct < 20         made money without breaching a 20% risk budget
max_dd_pct < 20                             says nothing about profit — only that it held the line
net_pnl_pct > 2 * max_dd_pct                both sides may be expressions
net_pnl_pct > 0 and trades >= 5             excludes cells that look safe by barely trading
```
Operators `>`, `>=`, `<`, `<=` (no `==`, no `or`). Variables are the objective set: `net_pnl_pct`,
`max_dd_pct` (a POSITIVE percentage), `trades`, `win_rate`, `profit_factor`, `expectancy`, `sharpe`,
`return_over_dd`. The criterion is fixed for the whole run, so cells compare with each other; two
RUNS compare only if they state the same one, which is why it is echoed in the results. A
criterion that does not parse is rejected at submit time, never silently substituted.

**Pass `params_override`.** Stress runs one fixed parameter set across every synthetic market, so it
should be the set you actually deploy — the same override you send to `/jobs/backtest` and
`/jobs/live`. Omit it and you are mapping the operating envelope of the `.pine` *defaults*, not of
the config you trade. (This is the opposite of sweep and robustness, which deliberately ignore it —
a sweep produces the config, and the permutation test must re-run the procedure you actually ran.)

Only the execution timeframe is used — no HTF, no intrabar. A strategy that depends on
`request.security` behaves differently here than in its backtest.

Results (`GET .../results`): `model` (what actually ran), `model_requested` (differs only when it
was `"auto"`), `axis_kind` (`"reversion"` | `"trend"` | `"signed"` — what `half_life_bars` means),
`survival` (the criterion as written), `calibration`, and `cells[]` with `half_life_bars`, `jumps_per_1k`, `median`, `p25`, `p75`, `mean`,
`min`, `max`, `frac_profitable`, `median_trades`.

`calibration` carries the fitted market plus a measurement of which family the instrument is
actually in — the grid only contains one, so read this before reading the grid:
```
phi, theta, half_life_bars     AR(1) on the log price → the reversion coordinate
trend_rho, trend_half_life_bars  AR(1) on the log return → the momentum coordinate
sigma, jumps_per_1k, mu_log, n_bars
phi_clamped / trend_rho_clamped  true = that coordinate is the model's bound, NOT a measurement
family                         "mean_reverting" | "trending" | "inconclusive"
family_reason                  the verdict in words
horizon_bars                   mean bars HELD per trade — the horizon family is read at
vr_headline_q                  the ladder rung family was decided on
variance_ratio                 [{q, vr, z}] — Lo-MacKinlay ladder, q = 2..128
vr_winsorized                  returns clipped as outliers (non-zero → check for an unadjusted split)
```
`vr` below 1 means moves tend to reverse over `q` bars, above 1 that they persist; `z` is
significant beyond ±1.96. It measures *persistence, not direction* — an instrument that rose
tenfold on independent moves is `inconclusive`, and that is the correct answer. On a `family` that
disagrees with the model you chose, the envelope describes a market the instrument is not in.

`frac_profitable` counts paths meeting `survival`, never `metric`. The wire name is historical: on
the default criterion it is exactly the profitable count it always was, but under a drawdown budget
a high value does NOT imply the paths made money. Read `survival` before reading the layer.

Fields other than `phi`/`theta`/`half_life_bars`/`sigma`/`jumps_per_1k`/`n_bars` are absent from
results written before 2026-08-10, and `survival` from those written before 2026-08-12 — which all
used the default.

### POST /api/v1/jobs/live — available on **every plan** (free included)
Live trading itself is not plan-gated. What a tier mostly buys is *capacity* — live bots draw on
the same concurrent-job quota as everything else (free 1, higher tiers more), so a rejection
naming the concurrent-job limit is the quota, not the tier.

Three OPTIONS on this endpoint are gated, though, so a `403`/`400` here is not always the quota:
`symbols` / `mode` (basket and portfolio, **Pro**), `webhook_url` (**Pro**), and `htf_timeframe`
or a multi-timeframe strategy (**Premium**). Each is refused before the broker is contacted, so
the message names the plan rather than a missing broker connection.
```
strategy_id         uuid    required  (must be validated — `status: "valid"` — or 400)
symbol              string  required  (the primary symbol; with `symbols`, its first element)
symbols             array   optional  (string[]; ≥2 entries = a multi-symbol BASKET traded by one
                                       bot in one process — see below. 0/1 entries = ordinary
                                       single-symbol bot)
mode                string  optional  ("single"|"basket"(default)|"portfolio") — which of the
                                       three live modes to run. Only `portfolio` changes
                                       behaviour; omit it for the historical basket semantics
leverage            number  optional  (portfolio only; default 1.0, must be in (0,4]) — the
                                       book's gross-exposure cap as a multiple of ACCOUNT equity
timeframe           string  required  (live subset: "5m","15m","30m","60m","90m","1D" — no
                                       weekly/monthly/1m. "1H"→60m and "4H"→240m are aliased,
                                       but 240m is not in the live subset, so 1D/90m/60m/30m/
                                       15m/5m are the six that actually launch)
htf_timeframe       string  optional  (PREMIUM — see the multi-timeframe note above)
intrabar_timeframe  string  optional
execute_orders      bool    optional  (default true; false = signals + webhooks, no broker orders)
heartbeat_secs      int     optional
auto_restart        bool    optional  (default false)
params_override     object  optional  { var_name: number|bool }
broker              string  optional  ("saxo"(default)|"alpaca"|"ibkr"|"lightspeed"|"bitstamp"
                                       |"propfirm")
saxo_env            string  optional  ("sim"|"live"; Saxo only)
webhook_url         string  optional  (http/https; receives order/trade/fill events; PRO)
options_routing     bool    optional  (default false; ALPACA ONLY — see below)
options_params      object  optional  (per-bot options knobs; ignored unless options_routing)
futures_auto_roll   bool    optional  (default true; FUTURES ONLY — roll the position into the
                                       next delivery month instead of holding a contract that
                                       stops trading. See below)
```

`broker` is matched **exactly** — no trimming, no case-folding. `"Saxo"` is rejected
`400 unknown broker 'Saxo' — supported: saxo, alpaca, lightspeed, ibkr, bitstamp, propfirm`.

**Multi-symbol baskets (`symbols`) and portfolio mode are a paid-plan feature.** Two or more
entries launch a single `LiveMulti` job: one container, one process, one broker account, one
shared timeframe, counting as **one** job against the concurrency quota. Constraints, all enforced
server-side (the UI control is cosmetic):

- **`403`** on the free plan — pro / premium / admin / dedicated only. Asked of the REQUEST before
  broker credentials are resolved, so an unconnected free account gets the plan refusal rather
  than a misleading "not connected". `mode: "portfolio"` is refused on the same gate even though
  it carries no `symbols` (its universe comes from the strategy).
- **`400 "multi-symbol live is not supported for <broker>"`** on `ibkr` (per-symbol client id),
  `lightspeed` (a separate bot binary) and `propfirm` (a futures basket is a basket of contract
  months, each with its own front-month resolution — not exercised yet).
- Every entry is a `tv_symbol` that must exist, be enabled, and be `live_tradable`; unknown or
  data-only rows are rejected `400`.

**`mode: "portfolio"` runs a book, and THE STRATEGY NAMES IT.** A portfolio takes no symbol list:
its legs are the strategy's own `request.security` universe, the same rule as a portfolio
backtest. Any `symbols` you send is **replaced** by that universe and `symbol` becomes its first
entry — read the list back from
[`GET /api/v1/strategies/{id}/universe`](#get-apiv1strategiesiduniverse) before launching.
Deriving rather than trusting the caller is what keeps the book and the strategy in step: a leg
the strategy never reads would trade unranked, and a symbol it ranks but that is missing from the
book would be scored and then never traded.

The difference from a basket is a governor. A basket is N independent bots sharing a broker
connection: each leg sizes `percent_of_equity` off your FULL account equity, so N legs at 100% try
to deploy N×100% of the account. A portfolio's legs share the account's equity and a combined
gross-exposure ceiling (`Σ|leg notional| ≤ leverage × equity`), so together they never commit more
than the cap allows. The leg that would breach it is sized down (logged as
`[portfolio] gross cap: X -> Y`); when the book is already full the entry is skipped entirely
rather than sent as a zero-quantity order.

The account IS the book: equity is read from the broker at launch and re-read on every heartbeat,
so the ceiling follows the account. Constraints, all `400`:

- **`mode` must be one of `single` / `basket` / `portfolio`.**
- **The strategy's universe must be 2–5 symbols.** Fewer than 2 is a single-symbol bot, not a
  book. More than 5 is refused: unlike a backtest, where a wide universe costs only CPU, every
  live leg is a real position on a real account. Narrow the basket or run a portfolio backtest.
- **`leverage` must be in `(0, 4]`**. 1.0 caps the book at the account size; 2.0 permits fully
  long and fully short at once and therefore needs a margin account that allows shorting.
- **`leverage > 1.0` is refused on `bitstamp`** — spot cannot borrow, so the venue would reject
  the resulting orders one by one.
- A launch is **refused outright if the broker reports no account equity**: a book of zero would
  clamp every leg to zero, and the bot would run, log signals and never trade.
- The mode survives an auto-restart — a portfolio comes back as a portfolio, cap included.

**`options_routing` hands execution to the options runtime (Alpaca only).** Each signal is scored
across the underlying shares and the option chain and the better risk-adjusted expression is
placed. It is refused `400` on any other broker ("no other broker's option order model has been
verified"), and it **forces `execute_orders = false`** — the runtime places every order, so a bot
that also traded would open the shares *and* the contracts for one signal. Under routing the
runner owns the webhook URL, so a client-supplied `webhook_url` does not reach the runtime.

`options_params` (every field optional; an unset field means "use the runtime default", never 0):
```
capital       number  cash the model may deploy per signal      (clamped 1 … 10,000,000)
risk_frac     number  fraction of capital risked to the stop    (clamped 0.0001 … 1)
min_dte       int     earliest expiry considered, in days       (clamped 1 … 730 — never 0DTE)
max_dte       int     latest expiry considered, in days         (clamped 1 … 730)
horizon_days  number  expected holding period, drives expiry    (clamped 0.5 … 730)
allow_short   bool    open a short expression on a sell signal
dry_run       bool    score and log the decision, place nothing
auto_roll     bool    roll a held option to a later expiry before it decays (default ON)
```
Values are clamped, not rejected, and `min_dte`/`max_dte` are swapped if sent out of order. A
`options_params` blob on a non-routed job is silently dropped. Options routing suits fast
directional strategies; on a slow or mean-reverting one the premium decays while it waits for an
exit signal — say so before launching one for a user.

Errors specific to this endpoint: `429` if you exceed 30 launches/min; `400 "Concurrent job limit
reached (N)…"` when the plan's quota is full; `400 "webhook signals are included in Pro. …"` when a free account sends `webhook_url`;
`400 "<Broker> not connected: …"` when the broker
has no stored credentials; `503` when every runner in the fleet is at capacity.

**The broker is not a detail — it changes what protective orders exist.** Pine is the same on
every venue; the order model underneath is not, and the difference is not guessable:

| Broker | Instruments | Stop-loss / take-profit |
|---|---|---|
| **Saxo** | EU + US equities, and **futures** (Eurex / Euronext, `ContractFutures`) | Native **OCO** at the broker: a resting stop *and* a resting TP, linked (a fill on one cancels the other). `saxo_env` picks sim (paper) or live. On a futures symbol the bot resolves the product root to the live delivery month at launch and re-checks it hourly — see *Futures and expiry* below. |
| **Alpaca** (equity) | US equities | Native **OCO**, same as Saxo. Margin accounts can short. |
| **Alpaca** (crypto) | US-dollar pairs only (`BTCUSD`, `ETHUSD`, …; a symbol is tradeable here iff its `alpaca_us_symbol` is non-null) | Only **one** exit can rest, and it is the **stop** — crypto refuses `oco`/`bracket`, and the first resting exit reserves the whole coin balance. The take-profit is therefore **bot-managed**: evaluated at bar close, and on a hit the bot cancels the stop and market-closes. Fractional size; fees are taken in the coin. |
| **Bitstamp** (spot) | USD + EUR spot pairs (iff `bitstamp_pair` is non-null) | **No native stop or TP exists at all.** Spot accepts `stop_price` and answers `200 OK` with an order id, but creates nothing. **Every stop on a Bitstamp bot is synthetic** — checked by the bot at bar close, on a 24/7 market. Long-only (no shorting on spot). |
| **Lightspeed** | US equities | Market orders only — nothing rests, so nothing protects. |
| **IBKR** | US equities | Market orders only. |
| **PropFirm** | CME futures, through the firm's gateway (Tradovate) | Live trading **only** — a prop-firm gateway is an execution rail, its market data is entitled per account and non-redistributable, so there is no backtest path and no catalog entry. The bot resolves the **front month** at launch and re-checks it hourly (`futures_auto_roll`, below). The firm's daily loss limit, trailing drawdown and flat-by time are enforced on *its* side and are invisible to the bot: a breach flattens every position and locks the account, which is a way for a position to vanish with none of the bot's orders filling. |

#### Futures and expiry (`futures_auto_roll`)

A futures contract stops trading on its expiry date. Every other instrument on the platform is
permanent, so this is the one launch where the thing you are trading has an end.

Two venues trade futures: `propfirm` (CME, through the firm's gateway) and `saxo` (Eurex and
Euronext). Any other broker refuses a futures symbol with
`400 "... is tradable only through a prop-firm account or Saxo"`, and both futures venues refuse a
cash symbol — the guard runs both ways.

The platform stores a product **root** (`ES`, `FDX`), never a delivery month, and the bot asks the
venue at launch which contract is front. Nothing here holds an expiring id, so nothing can silently
point at a dead instrument. From then on:

- **The bot re-checks hourly** and warns in the job log from **7 days out**, again on the day, and
  again if it is ever pointed at something already expired.
- **`futures_auto_roll: true` (the default) rolls the position.** When the venue names a new front
  month, the bot cancels what it has resting, closes at the market in the expiring contract, moves
  to the new one, and re-opens the same side and size. Both legs land in your trade history as
  `FUTURES_ROLL_CLOSE` and `FUTURES_ROLL_OPEN`.
- **`false` warns and holds.** Not the safer option, just a different risk: the bot keeps a
  contract that will stop trading, and you have to close it and relaunch yourself.

Two things to know before leaving the default on, because neither is visible in a backtest:

1. **A roll is two real market orders.** You pay the spread twice and inherit the price difference
   between the two months. A backtest models none of it — the batch engines run on a stitched
   continuous series where the roll is a step in the data, not a pair of fills.
2. **Levels the strategy remembers do not move with it.** The cost basis is re-read from the broker
   after the roll, so `strategy.position_avg_price` is correct — but a trailing stop or a breakeven
   mark held in a Pine `var` was computed against the old contract's prices. The bot prints a
   warning naming this at the moment it rolls; check the first exit order it places afterwards.

If any leg cannot be closed, the roll is **abandoned** rather than half-done: the bot stays in the
expiring contract, says so, and retries at the next check. Switching contracts while still holding
the old one would leave a position the bot can no longer address.

Bitstamp has **no `env` field on the launch request** — the environment (`sandbox` = the venue's
only paper mode, or `live` = real money) is fixed when the credentials are saved. Check it with
`GET /api/v1/bitstamp/status` before launching.

A Bitstamp bot also **refuses to adopt coins it cannot price**. A spot balance is not a position
and Bitstamp stores no average entry price, so the bot reconstructs a cost basis from your fill
history — and a coin that was *deposited* (or bought outside the API's 30-day transaction window)
has no purchase price anywhere. Rather than invent one, the bot logs that it cannot price the
holding and stays flat. Fund a Bitstamp bot's account by **buying** the coin, not depositing it.

**Every order a live bot places carries a client reference**, so a broker's own trade report can
be attributed back to a specific bot rather than guessed at from a symbol and a time window —
which is ambiguous the moment two bots on one account trade the same instrument, exactly the case
the platform allows. The shape is `pcx-<strategy-slug>-<job-id first 8>-<run><seq>`, on Alpaca's
`client_order_id`, Saxo's `ExternalReference` and Bitstamp's `client_order_id`. It is
reconciliation only — nothing in the order path reads it back, and only 8 characters of the job id
fit, so treat it as narrowing an order to a job rather than proving which one.

Realized per-bot performance (closed round trips, win rate, risk-adjusted edge) is collected from
the bots and shown under **Live → Performance** in the web UI. It is deliberately **not** part of
the `/api/v1` contract yet: its shape will move as account-level figures land, and `/api/v1` is a
stability promise.

### GET /api/v1/jobs — list (recent) → array of `JobResponse`
Your own jobs, newest first, hard-capped at **50**. There are no query parameters — no
`limit`/`offset`, no status or type filter, no pagination cursor. Filter client-side.
### GET /api/v1/jobs/{id} — one `JobResponse` (status synced from runner if still running)
### GET /api/v1/jobs/{id}/results — metrics JSON (shape varies by job_type)
- **backtest** — performance metrics + `hurst` / `variance_ratio`

> **Removed 2026-08-03: results no longer carry a `price_structure` block.** It described the
> PRICE SERIES, so it was identical for every strategy ever run on that dataset — the same numbers
> on every winner and every loser. Ask `GET /api/v1/data/structure` for the dataset instead; it
> answers the same question once, with the ACF, a walking window and the archetype fit. The
> `hurst` / `variance_ratio` fields the engine itself writes into `summary` are unaffected.
- **sweep** — `mode`, `param_names`, `total_trials`, and `trials[]`. Each trial:
  `params` (by input *title*), `net_pnl_pct`, `max_dd_pct`, `n_trades`, `win_rate`, `profit_factor`,
  `sharpe`. Rank them yourself — the server does not pick a winner. `sharpe` is a **per-trade**
  Sharpe (mean / stdev of trade PnL), not annualised and with no risk-free rate: read it as
  consistency of edge per trade. It is absent from results written before 2026-07-13.
- **robustness** — `p_value`, `observed_stat`, `null_dist[]` (+ mean/sd/percentiles), `hurst` /
  `variance_ratio`, and the echoed `permutations` / `block_size` / `metric` / `seed`
- **stress** — `calibration` + `cells[]` (see the stress endpoint above)

  A trial also carries `plateau` when the run was annotated and `bootstrap` when it ran with
  `bootstrap: true` — see the two robustness axes above for how to rank on either.

When ranking sweep trials yourself, apply the same `min_trades` floor the search ran under (it is
echoed in the job's `config.sweep_config.min_trades`) — otherwise you can crown a trial the
optimizer had already disqualified.
### GET /api/v1/jobs/{id}/logs — Server-Sent Events stream. Authenticate with the normal
`Authorization: Bearer pcx_live_…` header and read the stream (e.g. `curl -N`). API keys are **not**
accepted as a `?token=` query parameter (that path is reserved for the web UI's short-lived session
token, since browser `EventSource` can't set headers).
### DELETE /api/v1/jobs/{id} — stop/cancel (live: soft-cancel kept in history; batch: hard delete)
**Stopping a live bot cancels its resting orders; it does not close the position.** On shutdown the
bot cancels the entry and exit orders it left at the broker (an unmanaged resting stop is a hazard:
a stop-loss is a sell, and a sell with no position behind it can *open a short*). Whatever is held
is left alone and named in the log as unprotected — closing it is the user's call, in their broker.
Relaunching re-adopts and re-protects the position. Never tell a user that stopping the bot
flattened them.

### Fleet snapshot — save the running live bots, restore them after an outage

A live bot is a container on a runner host. The API and the runner process can each restart without
disturbing one, but a **host reboot or a Docker daemon restart kills every bot** — no container
carries a restart policy — and the user is left re-launching each one by hand. The snapshot is the
manual remedy: save what is running, start it all again in one call afterwards.

**Every operation is explicit. Nothing restores itself, and that is deliberate** — the broker
credentials do not survive a long outage either (a Saxo refresh chain is dead after an hour, and a
logout scrubs the token rows), so an automatic restore would fire N doomed launches at dead
credentials. Reconnecting the broker is what makes a restore able to succeed, and only the user
knows when that has happened. **Tell the user to reconnect their broker before restoring.**

There is exactly **one snapshot per user — the last one saved.** Saving replaces it wholesale.

#### GET /api/v1/jobs/live/snapshot
```
taken_at                datetime | null   (null = never saved)
entries[]               one per saved bot
  job_id                uuid
  strategy_name         string | null
  symbol                string | null
  tickers               string[] | null   (basket / portfolio bots)
  timeframe             string | null
  broker                string | null
  status                current job status, or "missing" if the row is gone
  error_message         string | null     (why it last stopped)
  restorable            bool              (row exists and is not already up)
restorable              int               (how many a restore would start)
running_not_in_snapshot int               (live bots running that this snapshot does NOT hold)
```
`running_not_in_snapshot > 0` means the snapshot is **stale** — the user has launched bots since
saving. It is never refreshed silently; tell them to save again if they want those included.

#### POST /api/v1/jobs/live/snapshot — save/replace from what is running now
Returns the same shape as `GET`. Captures every live job in `running`/`pending`.

**Saving with nothing running stores an empty snapshot, and that is how a snapshot is discarded**
(stop the bots, then save). There is no delete endpoint — one rule, not two. It is also the only
destructive call here, and the page looks exactly like that right after an outage, so **confirm
with the user before saving an empty snapshot over a non-empty one.**

#### POST /api/v1/jobs/live/snapshot/restore — start everything in it that is not already up
```
restored         int
already_running  int
failed           int
results[]
  job_id   uuid
  symbol   string | null
  outcome  "restored" | "already_running" | "missing" | "skipped" | "failed"
  message  string | null   (the real reason — a disconnected broker, an offline runner, the quota)
```
**A partial restore is the normal outcome** — report `results[]` per bot, never just the totals. A
bot skipped on the plan's concurrent-job limit reads `skipped`; a broker that has not been
reconnected reads `failed` with the broker's own error.

It **relaunches the original job rows** (same job ids), so each bot keeps its output directory,
rotated broker tokens and fills ledger. Restoring resets that job's auto-restart counter, clears
its stored error, and books a `restarted` bot event. Restoring twice is safe — the second call
reports `already_running`.

Every strategy in the snapshot is **write-protected** while it is saved there (see `locked` on
`StrategyResponse`).
### POST /api/v1/jobs/{id}/analyse — AI (descriptive) analysis of results
Request: `{ "provider": "gemini"|"mistral"|null }` (null = the server default; see
`GET /api/ai/providers`). The job must be `completed`.
Response: `{ "analysis": "<markdown>", "model": "<model id>" }`.
### POST /api/v1/jobs/compare/analyse — AI (descriptive) analysis comparing several jobs
Request: `{ "ids": [uuid, …], "provider": string|null }` — **2 to 6** ids, each completed and
yours (`400 "Expected 2–6 job IDs"` otherwise). Same response shape as `/analyse`.
**A string override is applied only when the input declares an `options` list and the value is a
member of it.** That is what makes it safe to splice into Pine source: the emitted text comes from
a fixed set already in the script, so nothing user-supplied reaches the engine. A string input
without `options` stays un-overridable.

**An override that does not land is reported, never silent.** A key naming no input, a string
outside its `options`, or any other unrepresentable value leaves the strategy's own default in
place and returns a warning from `patch-preview`. Previously all of these were skipped without a
word, so the job ran on the authored defaults and reported success — and since the defaults are
always a complete, plausible configuration, nothing in the result looked wrong.

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
| `massive` | US equities (a 10-year window on our plan) | Also the source behind Pine's `request.financial` fundamentals and the corporate-action series; Yahoo is merged in as a second fundamentals source per series. Server-side rate-limited, so a long intraday range is paced across many paged calls — a fetch can take minutes without anything being wrong. Vendor rejections are surfaced verbatim rather than as a generic failure. |
| `ibkr` | Equities | Needs IBKR configured (TWS/Gateway). |
| `bitstamp` | Crypto (USD + EUR pairs) + a few FX pairs | **No account or key needed** — public endpoint. Timeframes `1m 5m 15m 30m 60m 1D` only. |
| `wikipedia` | Attention, anything with an article | **Non-price.** Daily pageviews from 2015-07-01. No account needed. `1D` only. |
| `reddit` | Attention, US retail names | **Non-price.** Daily post counts across r/wallstreetbets, r/stocks and r/investing, from 2005. `1D` only. |
| `secform4` | Insider dealing, **US-registered issuers only** | **Non-price.** SEC Form 4 open-market purchases and sales in dollars, from 2003. A foreign private issuer files a 20-F and is exempt from Section 16, so it never files one at any date. `1D` only. |
| `google` | Attention, any search term | **Non-price, and NOT REPRODUCIBLE** — see the warning below. Daily search interest. `1D` only. |

**Use `bitstamp` for deep intraday crypto history.** It is the only source that reaches it:
Yahoo cuts intraday off at 730 days and Alpaca's crypto data begins in 2021, while Bitstamp's
public series goes back to **2011** and quotes real BTC/USD (not USDT). A multi-year hourly
Bitcoin backtest — the window the MCPT literature uses — is only reproducible from this source.

### Non-price series (attention and disclosure)

Four sources carry something other than a price. They are ordinary catalog datasets — fetched the
same way, cached the same way, read with `request.security` — but they live on their **own symbols**,
prefixed by the publisher:

```pine
views = request.security("WIKI:ASML",     "1D", pageviews)
posts = request.security("REDDIT:GME",    "1D", mentionCount)
flow  = request.security("SECFORM4:MSFT", "1D", insiderFlowUsd)
si    = request.security("GOOGLE:NVDA",   "1D", searchInterest)
```

The prefix names **who published the number**, never what it measures. A single `ATTENTION:`
namespace could not say whether a value came from Wikipedia or Google, and those are not
interchangeable measurements: research comparing search-based and Wikipedia-based attention finds
they carry different information and diverge most in stressed markets.

**Every dataset is stored as five columns, because that is what a bar is.** A non-price series has
no prices to put in them, so each column carries a different figure under a name that says what it
holds. **Do not write `open` or `close` on these symbols.**

| Column | `WIKI:` | `REDDIT:` | `SECFORM4:` | `GOOGLE:` |
|---|---|---|---|---|
| 1 | `pageviews` | `mentionCount` | `insiderFlowUsd` | `searchInterest` |
| 2 | `mobileViews` | `mentionScore` | `transactionShares` | `searchInterestRaw` |
| 3 | `desktopViews` | `mentionComments` | `transactionPricePerShare` | *(none)* |
| 4 | `spiderViews` | `topAuthorPosts` | `insiderSharesHeld` | *(none)* |
| 5 | `pageEdits` | `distinctAuthors` | `transactionCount` | `windowsCovering` |

`GOOGLE:` has three names rather than five because the source provides one number per day. The two
unused columns are deliberately **not addressable** — a strategy can name what exists and cannot
name what does not.

**Reddit's author fields are the reason to use that series.** A post count cannot separate
coordinated posting from genuine interest. Counting who posted can:

```pine
posts   = request.security("REDDIT:GME", "1D", mentionCount)
authors = request.security("REDDIT:GME", "1D", distinctAuthors)
topper  = request.security("REDDIT:GME", "1D", topAuthorPosts)

breadth       = authors / posts    // 1.0 = everyone posted once
concentration = topper  / posts    // high = one account is doing the talking
```

**`insiderFlowUsd` is cumulative.** It only moves on filing days and never resets, so the
information is in the difference between two points, not the level: `flow - flow[90]` is the net
dollars filed over the window. Only open-market transactions are counted (Form 4 codes `P` and `S`);
grants, option exercises and shares withheld to pay tax on a vesting are compensation mechanics, not
decisions, and counting them is what produces the "insiders are dumping" headline every time a grant
vests.

**All four are publication-shifted by one day.** A day's figure is complete only once the day is
over, so a bar is dated the session its number could first be acted on. You do not need `[1]`.

> **`GOOGLE:` is not reproducible, and that is a property of the source.** Google scales every
> response 0-100 against the maximum in the *requested window* and serves a sample, so an identical
> request later returns slightly different history — worst on low-volume queries, and a ticker is a
> low-volume query. Daily resolution is also only available for windows of eight months or less, so
> a long history is stitched from overlapping windows rescaled onto each other, and values are no
> longer capped at 100. Every other dataset on the platform returns the same numbers when
> re-fetched; this one does not. Treat a result built on it as one draw rather than a measurement.

> A prefixed symbol never falls back to the bare ticker, and a field name is checked against the
> symbol it is read from. `WIKI:ABN` where no such symbol exists is an error rather than a quiet
> substitution of ABN's price series, and `pageviews` read from a price symbol is refused rather
> than silently returning that stock's close. Both would otherwise run, report plausible numbers,
> and measure the wrong quantity.

### GET /api/v1/data/symbols → array
```
id, tv_symbol, tv_full_symbol, display_name, index_name,
yahoo_ticker|null, massive_ticker|null, ibkr_symbol|null, alpaca_us_symbol|null,
saxo_uic|null, bitstamp_pair|null,
wikipedia_article|null, reddit_query|null, sec_cik|null, google_trends_query|null,
live_tradable  bool,
strategy_profile  string|null
```

The last four gate the non-price sources exactly as the broker tickers gate the price ones: a
`null` means that source is not offered for the symbol. None is derivable from the ticker, which
is why each is stored rather than inferred — `GME` is `GameStop` on Wikipedia, Microsoft's CIK is
`0000789019`, and `NVDA` and `NVDA stock` are materially different Google Trends series.

`strategy_profile` is a **snapshot** keyed by `"<source>:<timeframe>"`, computed once when a
dataset is downloaded. It goes stale the moment that dataset is extended. For a reading that
always matches the bars it describes, use `strategy_fit` on `GET /api/v1/data/structure`, which is
measured live.

**`live_tradable = false` means data-only.** A cash index (DAX 40, CAC 40, …) has a price series
but is not an instrument anyone can hold, so it backtests and sweeps normally and is refused at
live launch: `400 "<sym> is a data-only symbol — an index level, not an instrument you can hold.
… Trade a tracking ETF instead (e.g. DAXEX or DBXD for the DAX, CACC for the CAC 40)."` Check the
flag before offering a symbol for a live bot; the same rejection applies to every entry of a
`symbols` basket. (This is deliberate: those rows are mapped at the broker's *index CFD* for their
bars, so treating them as tradable would have opened a leveraged CFD position from a strategy
backtested on the unleveraged cash index.)
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

### GET /api/v1/data/structure — measured serial dependence of a cached dataset
Query params: `symbol`, `timeframe`, `data_source` (required), `from_date`, `to_date` (optional
`YYYY-MM-DD`, gate the measurement to the window you will actually test). `400` if the dataset is
not cached — fetch it first.

Answers two questions: **what did this market do, before any strategy touched it?** and **is
shuffling this series a fair thing to do?** Run it before `POST /jobs/robustness` and pass the
`suggested_block` it returns as `block_size`.

```
bars                      int      bars measured, after the date gate
variance_ratio            float|null  Lo-MacKinlay VR(2). <1 reverting, 1 random walk, >1 trending
vr_z                      float|null  heteroskedasticity-robust z against the random-walk null
vr_p                      float|null  two-sided p for vr_z
price_structure           string   "mean_reverting" | "trending" | "random_walk" | "unknown".
                                     THE FIELD TO LABEL ON — see the note below
price_structure_label     string   the same verdict as display text
price_structure_detail    string   one sentence including the sample size it rests on, and — when
                                     the verdict is "random_walk" — the smallest effect these bars
                                     could have resolved at all
acf                       [float]  return autocorrelation at lags 1..N
acf_band                  [float]  per-lag 95% band. Per-lag, and wider than the textbook
                                     1.96/sqrt(n), because it is robust to volatility
                                     clustering — which otherwise inflates the ACF's sampling
                                     variance and invents dependence that is not there
acf_band_nominal          float    the textbook 1.96/sqrt(n), for comparison. The gap between
                                     this and acf_band[0] is how heteroskedastic the series is
acf_abs                   [float]  autocorrelation of |returns| — volatility clustering, which
                                     the return ACF and Hurst/VR are all blind to
acf_abs_band              float    1.96/sqrt(n) band for acf_abs
acf_lag1                  float    \ the headline numbers, and whether each clears its band
acf_lag1_significant      bool     /
vol_acf_lag1              float    \
vol_acf_lag1_significant  bool     /
acf_decay_lag             int|null lag from which the ACF stays inside its band; null = the
                                     dependence outruns the measurable window
vol_acf_decay_lag         int|null the same for |returns|
suggested_block           int      the block size the measurement implies. 1 = no detectable
                                     return memory, so single-bar permutation IS the right
                                     (and strictest) null here — not a fallback
summary                   string   one-line plain-English verdict
history                   [obj]    the SAME headline metrics on a walking window, oldest first.
                                     Empty when the series is too short for one full window
strategy_fit              obj|null archetype scores for these bars (see below)
```

Each `history` entry:

```
end_ms                    int      ms since epoch of the window's LAST bar. A point describes the
                                     window ENDING there, never one centred there — nothing uses
                                     bars from its own future
n                         int      returns in the window (constant except possibly at the tail)
variance_ratio, vr_z, acf_lag1, acf_lag1_significant, price_structure   as above, per window
```

**Read `price_structure` / `variance_ratio`, not `hurst`.** The R/S Hurst exponent measures
long-range dependence over many horizons, which is a *different property* from the one-step
reversion a mean-reversion strategy trades. Measured on series whose character is known by
construction, a Hurst threshold cannot return "mean reverting" for any realistic market: an AR(1)
with rho = -0.09 is mean reverting by definition and reports Hurst 0.573 ("trending"), and even
rho = -0.30 only reaches 0.502 ("random walk"). The variance ratio separates those cases cleanly,
and its z is what distinguishes VR = 0.970 on 30,000 bars (real) from VR = 0.89 on 300 bars
(noise). Hurst is still reported where it appears; it is not the field to branch on.

**A `"random_walk"` verdict is a statement about the MEASUREMENT as much as the market.** Read
`price_structure_detail` before concluding the market has no structure — a short window simply
cannot resolve a small effect, and the sentence says how small an effect it could have detected.

**One number over a whole history can hide a regime change**, which is what `history` is for. Real
example: NASDAQ:FLWS 1m reports an emphatic z = -17.37 overall, while the walk splits 30 windows
mean-reverting / 31 random walk. Neighbouring windows share most of their bars, so treat it as a
moving average — a turn means something once it persists for about a window's width.

`strategy_fit` scores how well these bars suit each trading archetype, 0-100:

```
scores.mean_reversion / trend_following / momentum / breakout / swing_trading   int|null
scores.scalping                                        null (needs live spread + depth)
detail.vr_z                                            float|null  the z the scores used
detail.structure_resolved                              bool|null   false = |z| < 1.96
detail.<sub-metric>                                    the raw inputs behind each score
```

**When `detail.structure_resolved` is `false`, ignore `mean_reversion` and `momentum`.** Those two
rest on the variance-ratio z, and inside +/-1.96 these bars cannot tell reverting from trending at
all — the numbers are the limit of the measurement, not a finding. The web UI dims them for this
reason. The other archetypes rest on descriptive measures (ADX, ATR, ranges) without this
particular power problem.

These are descriptive market statistics, **not a recommendation to trade anything**. They are
measured live from the same bars as the rest of this response, so they always describe the same
window as the verdict above them.

Sized from **return** memory only. Volatility clustering runs for hundreds of lags, and letting it
drive the block would leave a handful of blocks to shuffle — the "null distribution" would collapse
to a few dozen arrangements and the test would lose all resolution. Clustering is reported
separately so it can inform the reading instead of wrecking the null.

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
| **Prop firm** | yes | `POST /propfirm/credentials` — firm id + login + the app id/cid/secret the firm issued |
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

### Prop firm — `GET /api/v1/propfirm/firms` → array
```
id       uuid
name     string   (the firm, e.g. "Apex")
gateway  string   ("tradovate")
env      string   ("demo"|"live")
```
The firm is **data**, the gateway is code: onboarding a new firm is a row, not a release. Pick an
`id` from this list for the connect call.

### POST /api/v1/propfirm/credentials → `204`
```
firm_id   uuid    required  (from /propfirm/firms)
username  string  required  (your gateway login)
password  string  required
app_id    string  required  \
cid       string  required   > issued BY THE FIRM together with the account — we mint nothing
sec       string  required  /
```
Verified against the venue before anything is stored, so a bad credential fails here rather than
at bot launch. Two failure modes that do not look like failures are handled for you: a rejected
login answers **HTTP 200** with an `errorText` body, and too many attempts answers 200 with a
rate-limit penalty rather than a token (that is a lockout, not a wrong password). The account must
have **API market data enabled** — its absence is failed at connect, because otherwise no bars can
be fetched and the bot would die at warmup days later.

`GET /api/v1/propfirm/status` → `{ connected, firm, gateway, env, account }`.
`DELETE /api/v1/propfirm/disconnect` → `204`.

**Live trading only.** There is no `propfirm` data source: gateway market data is entitled per
account and non-redistributable, so it is never cached into the shared catalog. Backtest a futures
strategy from another source, then trade it here.

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
hidden_brokers      string[]        (broker cards hidden in the web UI — cosmetic only)
created_at          datetime
dedicated_vps       object | null   ({ subdomain, status }; Dedicated tier only)
```

### PATCH /api/v1/auth/me — update profile fields → the same profile object
All fields optional: `email`, `phone`, `telegram_handle`, `telegram_bot_token`,
`telegram_chat_id`, `github_linked_repo`.

**Three different update semantics, which is easy to get wrong.**

- `telegram_handle`, `telegram_bot_token`, `telegram_chat_id` — **absent keeps, `""` CLEARS.**
  Sending an empty string writes a real `NULL`, which is how you remove a stored credential.
- `github_linked_repo` and `email` are *coalesced*: omitting them **or** sending `""` leaves the
  stored value alone, so they cannot be cleared here.
- `phone` is written **unconditionally**: omitting it **erases** your stored phone number. Send it
  back on every PATCH unless you mean to clear it. A non-empty phone must start with `+` and hold
  7–15 digits.

**`telegram_bot_token` / `telegram_chat_id` are Pro.** Setting a non-empty value on `free` is
`400 "Telegram notifications are included in Pro. …"`. Clearing them (`""`) stays open on every
plan, so a downgraded account can always remove a credential it is no longer using — and a free
account's live bots simply launch without notifications rather than being refused.
`telegram_handle` is **not** gated: it is a contact field on the profile and nothing sends to it.

Changing `github_linked_repo` also moves the sync webhook: it is deleted from the old repo and
registered on the new one.

### PUT /api/v1/auth/me/brokers — which broker cards the web UI shows → the stored array
`{ "hidden": ["lightspeed", "ibkr"] }`. Ids: `saxo`, `alpaca`, `bitstamp`, `propfirm`, `ibkr`,
`ibkr_web`, `lightspeed`; an unknown id is a `400`. The list is sorted and de-duplicated before
storing, and `[]` shows every card again.

**Cosmetic.** Hiding a broker does not disconnect it, does not revoke its credentials and does
not affect a running bot — a hidden broker's live jobs keep trading, and its own endpoints
(`/api/v1/saxo/*`, `/api/v1/alpaca/*`, …) keep working. This is a separate route rather than a
`PATCH /me` field because that call erases an omitted `phone`.

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
  | `dedicated` | 1 | 5 | unlimited |

  (The numbers are server-configurable defaults, so treat the error message as authoritative.)
- **Plan-gated features.** Two shapes, and the shape tells you where the gate is:

  **`403 forbidden` (no message)** — the whole endpoint is above your plan:

  | Requires | Endpoints |
  |---|---|
  | **Pro** or higher | `/jobs/portfolio-backtest`, `/jobs/portfolio-sweep`, and `/jobs/live` with `symbols` (≥2) or `mode: "basket"`/`"portfolio"` |
  | **Premium** or higher | `/jobs/robustness`, `/jobs/stress`, `/models/*`, `/jobs/hmm-train`, `/jobs/clf-train`, `/jobs/prf-train` |

  **`400` with a message naming the plan** — the endpoint is open but one FIELD is not, so the
  message says which and what it costs:

  | Requires | Field |
  |---|---|
  | **Pro** | `webhook_url` on `/jobs/live`; a non-empty `telegram_bot_token` / `telegram_chat_id` on `PATCH /auth/me` |
  | **Premium** | `htf_timeframe`, or any strategy whose own `request.security` reads a second timeframe, on `/backtest`, `/sweep`, `/robustness` and `/jobs/live` |

  `admin` passes everything. `dedicated` counts as **Premium** for entitlement while keeping the
  free tier's *quota* — on the shared platform that plan is a billing marker, because the
  customer's compute runs on their own instance.

  Backtest, sweep and **plain single-symbol live trading are open to every plan, free included** —
  a rejection there naming the concurrent-job limit is the quota, not the tier. The Premium job
  types are also the most expensive on the platform (a permutation test is N full backtests), so
  those gates double as cost control.
- **A disabled or deleted account is `401`, not `403`** — the plan is re-read from the database on
  every single request, so revoking access takes effect immediately, mid-session and mid-key.
- **Sizes:** request bodies are capped at 1 MiB (`413`); strategy source at 256 KiB (`400`).
- **Symbols** are plain tickers (e.g. `AAPL`) — no `/`, `\`, or `..`; invalid symbols return `400`.
- **Errors** are `{ "error": string }` with the HTTP status; `401` = missing/invalid/expired key.

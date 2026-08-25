---
name: pineconex-api
description: >-
  Drive a PineconeX account over its web API — create/validate Pine Script v6
  strategies, run backtests, parameter sweeps, robustness (permutation
  significance) tests, launch and monitor live trading bots, and fetch results.
  Use whenever the user wants to operate PineconeX programmatically (not through
  the web UI) with an API key.
  On a Dedicated VPS instance the account owner is a box admin, so this also covers
  the /api/admin operator surface: users, plans, symbols, the runner fleet, runtime
  images, health and the Dedicated-VPS registry.
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
fresh one. **429** means a per-user rate limit (validation or job launches) — back off before
retrying.

**403** means the endpoint is above the account's plan:

- **Pro or higher** — `/jobs/portfolio-backtest`, `/jobs/portfolio-sweep`, and `/jobs/live` with
  `symbols` (≥2) or `mode: "basket"`/`"portfolio"`.
- **Premium or higher** — `/jobs/robustness`, `/jobs/stress`, `/models/*` and the three trainers
  `/jobs/{hmm,clf,prf}-train`. (`dedicated` counts as Premium here; `admin` passes everything.)

Some gates are a **400 with a message naming the plan** instead, because the endpoint is open and
only one field is not: `webhook_url` on a live bot and a non-empty `telegram_bot_token` are
**Pro**; multi-timeframe is **Premium** — which includes a strategy whose own `request.security`
reads a second timeframe, not just an explicit `htf_timeframe`.

Plain single-symbol backtest, sweep and **live trading are available on every plan**, including
free. A **400 naming the concurrent-job limit** is the quota, not the tier — read the message
before telling a user to upgrade.

## Guardrails — read before acting

- **Live bots trade real broker accounts with real or paper money.** Always confirm the symbol,
  broker, and intent with the user before `POST /api/v1/jobs/live` or before stopping a running bot.
- **Respect quotas and rate limits.** Plans cap concurrent jobs and strategies (`403` when hit); the
  API also rate-limits validation (~20/min) and job launches (~30/min) per user (`429`). On either,
  tell the user the limit and back off — don't retry in a tight loop.
- **Auth goes in the header, never the URL.** Always `Authorization: Bearer <key>` (the `auth` helper
  above) — API keys are not accepted as a `?token=` query parameter on any endpoint, including SSE logs.
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
4. **WAIT** for it, then **fetch results**. Do not poll:
   ```bash
   auth "$PINECONEX_API_URL/api/v1/jobs/<job_id>/wait?timeout=300"   # blocks until done
   auth "$PINECONEX_API_URL/api/v1/jobs/<job_id>/results"            # metrics JSON
   ```
   `/wait` is `GET /jobs/{id}` that blocks — same response shape — and returns within
   milliseconds of the job finishing. If the `status` it returns is still non-terminal, the wait
   timed out: call it again. That is the whole protocol, and it is the entire loop.

   `timeout` is seconds, default 60, max 300. A forty-minute sweep is therefore about eight
   calls, each carrying a real answer, instead of several hundred that say "still running".

   **Watching several jobs? One call, not N.** A fan-out (a multi-symbol scan, a set of
   robustness runs) should use the bulk form rather than one wait per job:
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/v1/jobs/wait" -H "Content-Type: application/json" \
     -d '{"ids":["<id1>","<id2>"],"mode":"all","timeout":300}'
   ```
   `mode` is `any` (default — returns as soon as one is done) or `all`. Up to 100 ids. The
   response is every requested job's current state, so one call is a complete picture.

   Terminal statuses: `completed`, `failed`, `cancelled`, `timeout`. Non-terminal: `queued`,
   `pending`, `running`. Plain `GET /jobs/{id}` still works if you need a single snapshot without
   blocking — just don't build a polling loop out of it.

   A launch answers **`202`** with `status: "queued"` when no runner has room right now. That is
   not an error and must not be retried — the job is accepted and starts on its own. `/wait`
   handles it with no change on your side; `queue_reason` and `queue_eta_secs` say why it is
   waiting and roughly how long, and **`?on=start`** returns as soon as it leaves the queue if you
   want to distinguish "not started" from "not finished". `DELETE /api/v1/jobs/{id}` cancels a
   queued job for free. Live bots are never queued: `POST /api/v1/jobs/live` still refuses with
   `400` at the concurrent limit.

Sweeps (`/api/v1/jobs/sweep`), robustness (`/api/v1/jobs/robustness`) and stress
(`/api/v1/jobs/stress`) follow the same launch → poll → results shape. Robustness runs a
permutation (Monte Carlo significance) test — it returns a `p_value` on whether the strategy's
edge is real price structure or luck (low = real), plus `hurst`/`variance_ratio` describing the
instrument (branch on the variance ratio, not Hurst — see the structure endpoint below). Stress maps where a config *breaks* across synthetic markets — it is neither an
optimizer nor a significance test. Its `model` chooses the family of markets to sweep
(`ou_jump` mean-reverting, `trend_ar1` momentum, `both` a signed axis through a random walk,
`auto` to measure the instrument and pick); the results report which family the instrument is
actually in as `calibration.family`, and reading the grid without it can describe a market the
instrument is not in. Stress takes two separate scoring settings: `metric` scores each path,
while `survival` (default `net_pnl_pct > 0`, e.g. `net_pnl_pct > 0 and max_dd_pct < 20`) is the
pass-or-fail line behind `frac_profitable` — so on a non-default criterion that layer is not a
count of profitable paths, and the results echo the criterion. Robustness and stress require a
**Premium** plan.
See `references/api-reference.md` for their request fields.

**Portfolio (book) jobs.** `/api/v1/jobs/portfolio-backtest` runs ONE strategy across N symbols
against ONE shared account, and `/api/v1/jobs/portfolio-sweep` sweeps its `//@sweep` inputs the same
way (each trial is a full book run). Both take the strategy's own `request.security` basket as the
universe, so they have **no `symbol` field** — instead `initial_capital` and `leverage` configure
the shared book. portfolio-sweep uses the same `mode`/`trials`/`metric`/`min_trades` as `/sweep`
(no `perm_seed`). **Pro plan and above** (`403` on free) — a book job is N instruments of work per
launch. See `references/api-reference.md`.

**Sweep modes are `grid`, `random`, `rbf`, `monte_carlo`, `successive_halving`.** `rbf`,
`monte_carlo` and `successive_halving` read `metric`; grid enumerates every cell and random samples
blindly regardless of the objective. `bayesian` was removed 2026-07-13 after failing a blind-random control at equal budget,
along with the simulated annealing then named `monte_carlo` (it scored *worse* than random).
`monte_carlo` now means the **cross-entropy method**, added 2026-08-02: it samples a population per
round, keeps the best fifth, refits toward them, and repeats. It evaluates a round in parallel
(`rbf` cannot) and stops early once it only proposes parameter sets it has already tried, so it may
return fewer trials than the budget asked for.

**Do not tell a user a steered mode beats random — that is not established.** On 5 instruments x 3
seeds at equal budget, `rbf` was +0.68 and `monte_carlo` +0.38 net PnL % over blind `random`, and
neither margin is statistically distinguishable from zero. Recommend `random` as a perfectly
respectable default, `monte_carlo` when they want steering at random's wall-clock, and `grid` when
the space is small enough to enumerate. All five work on `/jobs/portfolio-sweep` too.

**`successive_halving` is a cheaper way to run a search, not a better one.** It scores every
parameter set on a short run, drops the worst half, doubles the run for the survivors, and repeats,
so the ladder costs a fraction of evaluating everything at full length and an exhaustive grid
becomes affordable on a space you would otherwise sample. Recommend it when the user wants the grid
but the grid is too slow. Four things to get right when reporting one:

- **Omitting `trials` selects the full grid**; setting it makes the parameter sets random samples.
  `rounds` (default 4) is the ladder depth, and the engine shortens it when the range or the
  parameter-set count cannot support the request. It says so in the job log.
- **Only the last round is comparable.** Every trial carries `rung`; filter to the maximum before
  ranking, or you will report a winner measured on a fraction of the data. Earlier rounds ship so
  the elimination can be audited.
- **The resource differs by route.** `/jobs/sweep` ladders the trade gate (the data is never
  truncated, so warmup is unaffected); `/jobs/portfolio-sweep` ladders the book's legs, where an
  early round is a filter on hopeless parameter sets and not an estimate of their scores.
- **Never present it as reducing overfitting.** It buys compute, and by making a wider search
  affordable it raises the multiple-testing bar rather than lowering it. `total_trials` is every
  parameter set started, which is what MinBTL wants.

**Two settings users will not think to ask for, and should:**
* **`metric`** — the objective. Defaults to `net_pnl_pct`, which will happily buy return with
  drawdown. `return_over_dd` and `sharpe` are the risk-adjusted ones. If a user says they care
  about drawdown or consistency, set it. For anything the built-ins can't express, pass a custom
  formula: `"metric": "expr: net_pnl_pct - 0.5 * max_dd_pct + 0.1 * trades"` — variables
  `net_pnl_pct`, `max_dd_pct`, `trades`, `win_rate`, `profit_factor`, `expectancy`, `sharpe`,
  `return_over_dd`, `bars_in_trade`, `time_in_market_pct`, `effective_exposure_pct`; operators
  `+ - * / ( )`. It is MAXIMISED as written (penalties get a minus
  sign; drawdown is a positive %). Works on `/jobs/sweep` (rbf, monte_carlo) and `/jobs/robustness`.
  To price holding time, total bars held is `bars_in_trade * trades`:
  `"expr: net_pnl_pct - 0.02 * bars_in_trade * trades"`. On a **portfolio** sweep
  `effective_exposure_pct` is always 0 and `time_in_market_pct` saturates at 100, so use
  `bars_in_trade` on a book.
* **`min_trades`** (default 5) — a trial with fewer trades can never win. Do **not** set it to 0
  with `profit_factor` or `win_rate`: a config that never trades has no losses, so its profit
  factor is `+inf`, and the search will converge on a strategy that refuses to trade.

**Both endpoints 400 if the strategy has no `//@sweep` parameters.** There is nothing to search.

**`//@sweep` is a marker with no arguments — the swept range is the input's own
`minval`/`maxval`/`step`.** Writing a range after the marker parses fine and is ignored; the bounds on a
swept input ARE its search space. An input with no `minval`/`maxval` covers its whole range and the
launch is refused with `Grid would require N combinations (limit 100000)`.

**The p-value trap — read this before running a significance test on a swept winner.**
`/jobs/robustness` defaults to `search_mode: "fixed"`, which re-runs the strategy's *authored*
input defaults on every permutation. That null — *"what this one rule scores on noise"* — is only
valid if the parameters were chosen **without looking at the data**.

If the parameters came from a sweep, `fixed` is optimistically biased: it cannot see that N
candidates were tried, and the best of N noise draws is high by construction. So when a user asks
you to sweep a strategy and then check whether the result is significant, pass the **same** search
you actually ran — and the **same** `metric`:

```
POST /api/v1/jobs/robustness   { ..., "search_mode": "rbf", "metric": "sharpe" }
```

On robustness, `metric` is both the reported statistic **and** the objective the search hill-climbs
inside every permutation. That is deliberate: the null is only valid if the procedure re-run on the
permuted bars is the procedure you ran on the real bars. Set it to whatever your sweep optimised.

It re-runs that optimizer inside every permutation, so the null becomes *"the best score this
strategy family can be fitted to noise"*. Cost scales with it (`fixed` = 1 backtest per
permutation, `grid` = the whole grid), and `permutations x search size` is capped — over the limit
you get a 400 with the number.

**The other p-value trap: do not set `block_size` unless you mean it.** Permutation is exact only
if the bars are **exchangeable**, and serial dependence breaks that. `block_size` defaults to
`"auto"`, which measures the series being permuted and sizes the block from it — so **leave it
alone**. It is a property of the instrument, not a preference, and a number you pick is a number
you guessed.

## Non-price series: attention and insider dealing

Four `data_source` values carry something other than a price, on their own prefixed symbols:

```
wikipedia -> WIKI:<ticker>      pageviews, 2015+
reddit    -> REDDIT:<ticker>    post and author counts, 2005+
secform4  -> SECFORM4:<ticker>  SEC insider open-market dealing in dollars, 2003+, US issuers only
google    -> GOOGLE:<ticker>    search interest (NOT reproducible, see below)
```

Fetch them exactly like price data (`POST /api/v1/data/fetch` with that `source`), then read them
in Pine **by named field**, never `open`/`close` — a non-price series has no prices, so each of the
five stored columns carries a different figure:

```pine
posts   = request.security("REDDIT:GME",    "1D", mentionCount)
authors = request.security("REDDIT:GME",    "1D", distinctAuthors)
topper  = request.security("REDDIT:GME",    "1D", topAuthorPosts)
flow    = request.security("SECFORM4:MSFT", "1D", insiderFlowUsd)
```

The full field table is in the API reference. Four things to tell a user before they act on one:

- **`insiderFlowUsd` is cumulative** — the signal is `flow - flow[90]`, not the level.
- **Only Form 4 codes `P`/`S` count.** Grants, option exercises and tax withholding are
  compensation mechanics; counting them is what produces the "insiders are dumping" headline.
- **Reddit's author fields are the point.** A post count cannot separate coordinated posting from
  interest; `topAuthorPosts / mentionCount` can.
- **`GOOGLE:` is not reproducible.** Google rescales each response to the requested window and
  samples, so re-fetching returns different history. Every other dataset here does not do this.
  A backtest over it is one draw, not a measurement — say so rather than reporting it like the rest.

A symbol only offers a source it is mapped for (`wikipedia_article`, `reddit_query`, `sec_cik`,
`google_trends_query` on `GET /api/v1/data/symbols`); a null means that source is unavailable for
it. `SECFORM4:` is **US-registered issuers only** — a foreign private issuer files a 20-F and is
exempt from Section 16, so ASML has no Form 4 at any date.

```
POST /api/v1/jobs/robustness   { ... }                        # auto: correct by default
POST /api/v1/jobs/robustness   { ..., "block_size": 5 }        # explicit override, honoured as-is
GET  /api/v1/data/structure?symbol=…&timeframe=…&data_source=…  # inspect an instrument directly
```

Results always report the **resolved** `block_size` plus `block_auto`, `suggested_block` and a
plain-English `block_advice`. A resolved block of 1 means the series has no measurable return
memory, so single-bar permutation **is** the correct and strictest null — it is not a fallback and
must not be "upgraded" to a block. If you do override and shuffle smaller than the data implies, the
permutations destroy structure the market really has, the null becomes an easier opponent than
reality, and the `p_value` is **optimistically biased** — significance the strategy did not earn.
Compare `block_size` against `suggested_block` before reporting any p-value, and say so when they
disagree.

`GET /api/v1/data/structure` also answers "what did this market do, before any strategy touched
it?" — and there is one field to branch on and one trap to avoid.

**Label on `price_structure` / `variance_ratio`, never on `hurst`.** R/S Hurst measures long-range
dependence over many horizons, a *different property* from the one-step reversion a mean-reversion
strategy trades. Measured on series of known character, a Hurst threshold cannot return "mean
reverting" for any realistic market: an AR(1) with rho = -0.09 is mean reverting by construction
and reports Hurst 0.573 ("trending"). The variance ratio separates those cases, and its `vr_z`
distinguishes a real effect from a small sample.

**A `"random_walk"` verdict is as much about the measurement as the market.** Read
`price_structure_detail` before saying an instrument has no structure — it states the smallest
effect these bars could have detected at all. Reporting "no structure" when the honest answer is
"this window cannot tell" is the most common way to misuse this endpoint.

**One number can hide a regime change.** `history[]` re-measures the same metrics on a walking
window (each point dated at the END of its window, so none uses bars from its own future).
Measured: NASDAQ:FLWS 1m reports an emphatic z = -17.37 overall while the walk splits 30 windows
reverting / 31 random walk. Neighbouring windows overlap heavily — read it as a moving average.

`strategy_fit` scores the bars against each trading archetype 0-100. **When
`strategy_fit.detail.structure_resolved` is `false`, do not report `mean_reversion` or
`momentum`** — inside |z| < 1.96 these bars cannot tell reverting from trending, so those two are
the limit of the measurement, not a finding. Descriptive statistics about a market, never a
recommendation to trade it.

Job results do **not** carry a price-structure block (removed 2026-08-03) — it described the price
series, so it was identical for every strategy ever run on that dataset. Ask this endpoint about
the dataset instead.

Volatility clustering is a **separate axis** and is deliberately not allowed to size the block
(it runs for hundreds of lags, which would leave too few blocks to shuffle). A series can have no
return memory at all while volatility clusters for dozens of bars: irrelevant to a directional
edge, but for a volatility edge — breakout, ATR sizing, vol filters — a block below
`vol_acf_decay_lag` leaves the null a calmer market than the real one.

Two related things worth telling users:
* **Out-of-sample needs no special mode** — it is a date range. Sweep on the training window, then
  run this test on the unseen window with the parameters fixed. `fixed` is the *correct* null
  there, because no search touched those bars.
* **`perm_seed` on `/jobs/sweep`** runs a sweep against a permuted copy of the series, so a client
  can build the same null itself by re-submitting one sweep under N seeds.

## Core workflow: live bot

1. Confirm with the user (broker, symbol, paper vs. live).
2. Ensure the broker is connected: `auth "$PINECONEX_API_URL/api/v1/alpaca/status"` (or `saxo`,
   `ibkr`, `lightspeed`, `bitstamp`, `propfirm`). If not connected, the user must connect it in the
   web UI (OAuth) — except **Bitstamp** and **prop-firm** accounts, which take credentials and can
   be connected over the API (`POST /api/v1/bitstamp/credentials`,
   `POST /api/v1/propfirm/credentials`).
3. Launch:
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/v1/jobs/live" \
     -H "Content-Type: application/json" -d '{
       "strategy_id":"<uuid>", "symbol":"AAPL", "timeframe":"1H",
       "broker":"alpaca", "webhook_url":"https://example.com/hook"
     }'
   ```
   `webhook_url` is **Pro**: on a free account this whole call comes back `400 "webhook signals
   are included in Pro…"`. Drop the field to launch the bot without it rather than reporting the
   launch as impossible.
4. Monitor: `GET /api/v1/jobs/<id>` for status, `GET /api/v1/jobs/<id>/logs` (SSE — stream it with the
   `Authorization: Bearer` header, e.g. `curl -N`). Stop with `DELETE /api/v1/jobs/<id>`.

**Stopping is not flattening.** `DELETE` cancels the bot's resting orders and leaves the position
open at the broker — nothing in the platform closes a position. If the user wants out of the
market, they must do it in their broker's own interface. Say so plainly rather than implying the
stop got them flat.

**Every order carries a client reference** — `pcx-<strategy-slug>-<job-id first 8>-<run><seq>`, on
Alpaca's `client_order_id`, Saxo's `ExternalReference` and Bitstamp's `client_order_id`. Useful
when a user is reading their broker's order history and asks which bot placed what. Only 8
characters of the job id fit, so it narrows an order to a job rather than proving which one.

**Realized per-bot performance is not on the API yet.** Closed round trips, win rate and
risk-adjusted edge are collected from the bots and shown under **Live → Performance** in the web
UI; the endpoint behind it is deliberately outside the `/api/v1` contract while its shape settles.
Point the user at the page rather than inventing a figure — and note the platform stores no live
P&L at all, so the broker account remains the source of truth for money.

**Three launch options worth raising, because users don't know they exist:**
* **`symbols`: a basket in one bot.** Two or more entries trade the whole basket in one process on
  one shared timeframe, counting as **one** job against the quota. **Pro plan and above** (`403` on
  free — `mode: "portfolio"` is on the same gate), and not available on `ibkr`, `lightspeed` or
  `propfirm` (`400`). One bot per symbol is
  still the more controllable shape — each has its own log, position and stop.
* **`mode: "portfolio"`: run the strategy's basket as one book.** Worth raising whenever a user
  sends a basket with `percent_of_equity` sizing, because a basket gives EVERY leg the full
  account to size against — three legs at 100% try to deploy 300% of the account, and the broker
  rejects whatever does not fit. A portfolio caps the legs' combined gross exposure at
  `leverage x account equity` (default 1.0) instead. **Send no symbols**: the legs are the
  strategy's own `request.security` universe (`GET /strategies/{id}/universe` — show the user that
  list before launching), 2–5 of them. Refused if the broker reports no equity; `leverage > 1.0`
  refused on Bitstamp (spot cannot borrow).
* **`options_routing`: let a model pick shares, calls or puts per signal.** **Alpaca only** (`400`
  elsewhere), and it forces `execute_orders=false` because the options runtime places every order.
  Tune it per bot with `options_params` (`capital`, `risk_frac`, `min_dte`/`max_dte` — floored at 1,
  so 0DTE is unreachable — `horizon_days`, `allow_short`, `dry_run`, `auto_roll` which defaults on).
  Only suggest it for fast directional strategies: on a slow or mean-reverting one the premium
  decays to nothing while it waits for an exit signal, where the shares would have sat flat.
  `dry_run: true` scores and logs without placing — the honest first step.
* **`execute_orders: false`** — signals and webhooks, no broker orders. The way to watch a strategy
  live without risking anything.

**Prop-firm futures (`broker: "propfirm"`) is live-only and unusually sharp.** The firm's daily
loss limit, trailing drawdown and flat-by time are enforced on the *firm's* side and are invisible
to the bot — a breach flattens every position and locks the account mid-session. There is no
`propfirm` data source, so backtest from another source.

**Futures expire, and that is the one launch where the instrument has an end.** Two venues trade
them: `propfirm` (CME) and `saxo` (Eurex / Euronext); every other broker refuses a futures symbol,
and both futures venues refuse a cash one. The bot resolves the product root to the live delivery
month at launch, re-checks hourly, and warns in the job log from 7 days out. **`futures_auto_roll`
defaults to `true`**: at the roll it closes at the market in the expiring contract and re-opens the
same side and size in the new one. Tell the user what that costs before they leave it on — two real
market orders, the spread twice, the price gap between months, none of it modelled by any backtest,
and any level the strategy holds in a Pine `var` still refers to the old contract. `false` warns and
then holds a contract that stops trading, which is a different risk rather than a safer one.

### Fleet snapshot — the fix for "the server restarted and all my bots are gone"

A live bot is a container on a runner host. A **host reboot or Docker daemon restart kills every
one of them** (no container carries a restart policy), and the user comes back to an empty page.
`POST /api/v1/jobs/live/snapshot` saves what is running; `POST /api/v1/jobs/live/snapshot/restore`
starts it all again. One snapshot per user, the last one saved. `GET` the same path to read it.

Three things to get right:

* **Nothing restores automatically, and you must not simulate that.** After a real outage the
  broker credentials are gone too (a Saxo refresh chain dies after an hour; a logout scrubs the
  token rows), so a restore before the user reconnects their broker fails on every bot. **Ask them
  to reconnect first**, then restore.
* **A partial restore is normal.** Read `results[]` per bot and report the reasons — a plan's
  concurrency limit shows as `skipped`, a broker or runner problem as `failed` with the real error.
  "Restored 3 of 5" without the two reasons is not a useful answer.
* **Saving with nothing running saves an EMPTY snapshot**, which is how one is discarded. There is
  no delete endpoint. Confirm before doing that over a non-empty snapshot — the saved fleet cannot
  be recovered.

A strategy is **write-protected** while it is running live or held in the snapshot: `PUT` code,
`PUT` params and `DELETE` all return `400`, and `StrategyResponse.locked` says why. That is not an
obstacle to route around — a bot runs the code it was launched with, so editing it could never have
reached the running bot anyway. Stop the bot, or re-snapshot without it.

## Crypto and Bitstamp

Crypto is tradeable on **Bitstamp** (USD + EUR spot pairs) and **Alpaca** (US-dollar pairs only),
and backtestable from `yahoo`, `alpaca`, or `bitstamp`. Symbols live under `index_name`
`"Crypto (USD)"` / `"Crypto (EUR)"` and are addressed by `tv_symbol` (`BTCUSD`, `ETHEUR`) —
the API maps that to each venue's own name, so never hand-build `BTC/USD` or `btcusd`.

**Three things worth telling a user before they launch a crypto bot, because Pine hides them:**

* **A Bitstamp stop-loss is not a broker order.** Bitstamp spot has *no* native stop, TP, or OCO
  — it accepts `stop_price`, answers `200 OK` with an order id, and creates **nothing**. So
  `strategy.exit`'s stop on Bitstamp is **synthetic**: the bot checks it at bar close, on a 24/7
  market. A gap through the stop while the bot is between bars is not protected against. Never
  tell a user their Bitstamp bot "has a stop at the broker" — it does not.
* **Alpaca crypto rests one exit, and it is the stop.** Crypto refuses OCO, and the first resting
  exit reserves the whole coin balance, so the take-profit is bot-managed (bar-close, then cancel
  the stop and market-close). Alpaca *equity* gets a real broker-side OCO; the two behave
  differently on the same broker.
* **Bitstamp is spot: long-only.** A short entry is refused. And a bot will refuse to adopt coins
  it cannot price (deposited coins have no purchase price in the API) — fund by **buying**.

For historical data, prefer **`bitstamp`** for anything intraday and older than ~2 years: Yahoo
refuses intraday beyond 730 days and Alpaca's crypto history starts in 2021, while Bitstamp's
public series reaches back to 2011 (no key needed; `1m 5m 15m 30m 60m 1D`).

## Endpoint catalog

| Area | Endpoints |
|---|---|
| Strategies | `GET/POST /api/v1/strategies`, `GET/PUT/DELETE /api/v1/strategies/{id}`, `GET /api/v1/strategies/{id}/inputs`, `GET/PUT /api/v1/strategies/{id}/params`, `POST /api/v1/strategies/{id}/share` |
| Validate | `POST /api/v1/validate` |
| Jobs | `GET /api/v1/jobs`, `POST /api/v1/jobs/{backtest,sweep,robustness,stress,live}`, `GET /api/v1/jobs/{id}`, `GET /api/v1/jobs/{id}/wait`, `POST /api/v1/jobs/wait`, `GET /api/v1/jobs/{id}/results`, `GET /api/v1/jobs/{id}/logs` (SSE), `DELETE /api/v1/jobs/{id}`, `POST /api/v1/jobs/{id}/analyse` |
| Fleet snapshot | `GET/POST /api/v1/jobs/live/snapshot`, `POST /api/v1/jobs/live/snapshot/restore` |
| Data | `GET /api/v1/data/symbols`, `GET /api/v1/data/catalog`, `GET /api/v1/data/structure`, `POST /api/v1/data/fetch` (price **and** the non-price attention/insider sources) |
| ML models (Premium) | `GET/POST /api/v1/models`, `DELETE /api/v1/models/{id}` |
| Train a model (Premium) | `POST /api/v1/jobs/hmm-train`, `/jobs/clf-train`, `/jobs/prf-train` |
| Brokers | `GET /api/v1/{alpaca,saxo,ibkr,lightspeed,bitstamp,propfirm}/status`, `POST /api/v1/bitstamp/credentials`, `POST /api/v1/alpaca/keys`, `POST /api/v1/lightspeed/credentials`, `POST /api/v1/ibkr/settings`, `GET /api/v1/propfirm/firms`, `POST /api/v1/propfirm/credentials`, `DELETE /api/v1/{…}/disconnect` |
| GitHub | `GET /api/v1/auth/github/{repos,files,file}`, `POST /api/v1/auth/github/sync-webhook` (linking itself is browser-only) |
| Account | `GET/PATCH/DELETE /api/v1/auth/me`, `POST /api/v1/auth/telegram/test`, `GET/PUT /api/v1/newsletter/me` (newsletter opt-in/out), `GET/POST /api/v1/auth/keys` (key mgmt is session-only, not via key), `DELETE /api/v1/auth/keys/{id}` |

Two of those are **irreversible and must never be called without the user asking for it in
those words**: `DELETE /api/v1/auth/me` erases the whole account (GDPR Art. 17 — every strategy,
job and stored broker credential, no undo, no confirmation step), and `DELETE /api/v1/models/{id}`
drops a model version permanently. Neither is a cleanup step.

Saxo cannot be connected from here at all (OAuth redirect — browser only), and `PATCH
/api/v1/auth/me` has a trap: `phone` is written unconditionally, so **omitting it erases the
stored number**. Send it back unless the user wants it cleared.

Full request/response field shapes: **`references/api-reference.md`** (read it when you need
exact field names, optional vs. required, or the `inputs`/`params` formats for parameter overrides).

**`params_json5`** (the strategy's saved parameter file, `GET/PUT …/params`) is **JSON5** — an
**array** of `{ symbol, configs: [ { tf, …overrides } ] }`. Each config runs a separate backtest;
keys other than `tf` map to Pine **input variable names** and override their defaults. It's a
live/backtest deployment override only — **sweep and significance ignore it**.

```json5
[ { symbol: "AAPL", configs: [ { tf: "1D", sma_len: 20, use_filter: true } ] } ]
```

## Writing strategies

Strategy `code` is **Pine Script v6 for the PineconeX headless interpreter**, which diverges from
TradingView (chart/UI calls are silently ignored; alert primitives are repurposed). If you are
authoring or editing the Pine source itself, prefer the dedicated Pine authoring guidance rather
than assuming TradingView semantics.

Two `strategy()` arguments have API-side consequences worth knowing:

* **`use_bar_magnifier = true`** resolves a bar that touches both the stop and the take-profit from
  finer data instead of optimistically booking the take-profit — but on backtest and sweep it needs
  an **`intrabar_timeframe`** on the request. Without one it is a silent no-op (warning in the log).
  Robustness and stress need no series. Magnified runs score lower; that is the bias leaving.
* **`margin_long` / `margin_short` default to 100** — full cash cover, no leverage — and an entry
  the account cannot pay for is not opened. So `default_qty_type=strategy.percent_of_equity` with a
  value above 100 no longer silently borrows. Leverage must be asked for explicitly (`margin_long =
  25` is 4×). This applies to the backtest engine too, so results changed for scripts that were
  implicitly over-leveraging.

Two **PineconeX-exclusive namespaces** have no TradingView equivalent, so a plain Pine generator
won't know them:

* **`ml.*`** — run a pre-trained ONNX model in-strategy (`ml.predict(name, features)` →
  `series float`). Upload the model on the Models page; **Premium** feature. Needs
  `//@runtime=2026.07.16-ml` or newer.
* **`gex.*`** — dealer **gamma exposure**: `gex.net` (regime sign), `gex.flip` (zero-gamma pivot),
  `gex.pin` (magnet strike), `gex.call_wall`/`gex.put_wall` (± walls), `gex.g1..g5` (ranked strikes) —
  all `series float`. Needs `//@runtime=2026.08.06-gex`+. **Availability caveat:** live GEX is fetched
  from the options chain on **Saxo** (Eurex) today; a **backtest reads `na`** (historical options
  chains aren't retained) so a GEX strategy backtests to no trades — validate + **paper-trade live**,
  don't backtest it. `gex.*` is data for your own strategy, never a pushed buy/sell signal.

## Admin surface — `/api/admin/*`

This build of the skill also covers the **operator** API: users, plans, the symbol universe, the
runner fleet, runtime images, health, statistics and the Dedicated-VPS registry.

**You only have it if the key's account is on the `admin` plan.** The gate reads the plan, not how
you authenticated, so an admin's `pcx_live_…` key is a *full-admin* key — there is no read-only
admin scope. On a **Dedicated VPS** instance (`<subdomain>.pineconex.com`) the account owner is
admitted as a box admin, which is the normal case for this skill.

Check before you plan any admin work — this is one request and it saves proposing a workflow that
will only 403:

```bash
auth "$PINECONEX_API_URL/api/v1/auth/me" | jq -r .plan   # "admin" → the endpoints below exist
```

`/api/admin/*` is **not** under `/api/v1` and is not part of the versioned public contract — it can
change without a major-version bump. Full field shapes: **`references/admin-reference.md`**.

### Guardrails — these are different from the user surface

The user API acts on one account. This one acts on **other people's** accounts and on the
infrastructure, and several calls are not undoable by any subsequent call.

- **Never call a destructive admin endpoint on your own initiative.** Read-only endpoints
  (`GET`) are fine to explore. Anything that writes needs the user to have asked for that specific
  change. In particular, always confirm first:

  | Call | What actually happens |
  |---|---|
  | `PATCH /api/admin/vps/{id}` with `status: "deprovisioned"` | **destroys the customer's server.** Use `"suspended"` to cut access without killing the box |
  | `PATCH /api/admin/users/{id}/plan` off `pro`/`premium` | **cancels their Stripe subscription immediately** — no refund, no proration |
  | `DELETE /api/admin/users/{id}` | hard-deletes their strategies, bot events and parse errors |
  | `DELETE /api/admin/vps/{id}` | removes the tracking row **without** touching the box — orphans a running server nothing records any more |
  | `DELETE /api/admin/jobs/{id}` on a live job | cancels resting broker orders but **leaves the open position**, now unmanaged |

- **`PATCH /api/admin/symbols/{id}` is a full replace, not a partial update.** An omitted field is
  written as `NULL`. Always `GET /api/admin/symbols`, edit the one object, and send the whole thing
  back. A sparse PATCH silently erases every ticker mapping you left out.
  (`/runners`, `/vps` and `/runtime-versions` *are* partial; `PATCH /api/admin/settings` requires
  **all thirteen** fields or it is rejected `422`. Three endpoints, three semantics — check, don't
  assume.)
- **A `204` is not proof anything existed.** `DELETE /runners/{id}`, `DELETE /vps/{id}`,
  `DELETE /runtime-versions/{v}` and `PATCH /vps/{id}` all return success for ids that never
  existed. Verify with a follow-up `GET` when it matters.
- **Admin reads contain other users' PII** — emails, IP addresses, login history. Summarise; do
  not dump user tables into output the user did not ask for, and never into a file or a webhook.

### Common operator workflows

**Promote a new engine version.** Registering is not deploying — the image must already be on
every runner, or jobs pinned there fail on that host only:

```bash
auth "$PINECONEX_API_URL/api/admin/runtime-availability"        # is it present on every runner?
auth -X POST "$PINECONEX_API_URL/api/admin/runtime-versions" \
  -H "Content-Type: application/json" -d '{"version":"2026.08.06-gex","notes":"gex namespace"}'
auth -X POST "$PINECONEX_API_URL/api/admin/runtime-versions/2026.08.06-gex/default"
```
The default is what unpinned jobs **and every live bot** run — live bots ignore user pins entirely,
so this call is how an engine fix reaches live trading. Deactivating a version auto-demotes it, so
never deactivate the current default without promoting a replacement first.

**Drain a runner before maintenance.** `PATCH /api/admin/runners/{id}` with
`{"is_active": false}` stops new dispatch while running jobs finish. The `DELETE` refuses while
work remains (`400 "runner has N active job(s)"`), which is the check, not an obstacle.

**"Why is nothing running while jobs are queued?"** `GET /api/admin/runners` answers it, and the
job count is the wrong column to read. Scheduling is by MEMORY —
`mem_total_mb - mem_reserve_mb - committed_mem_mb` — so a runner at 2 of 14 jobs can be genuinely
full if those two are portfolio books at 6 GB each. `GET /api/admin/jobs` now includes `queued`
rows with their `cost_mem_mb`, which is the figure admission is tested against and the reason a
large job can sit behind smaller ones that arrived later.

Three things to check before concluding something is stuck: `mem_total_mb: null` means the runner
has not reported capacity, so it is being scheduled on the job count alone; `mem_reserve_mb`
(default 2048) is memory deliberately withheld from jobs for the host, and raising it above what
is free stops admission without stopping anything already running; and a job whose `cost_mem_mb`
exceeds `mem_total_mb - mem_reserve_mb` will never start on that runner at all — it needs capacity
added, and the user is told exactly that in their own `queue_reason`.

**Cancelling a queued job is free** — `DELETE /api/admin/jobs/{id}` drops the row, since no
container exists.

**Diagnose "my data is stale".** `GET /api/admin/data-delay?symbol_id=…&timeframe=…` asks every
source at once. Read it carefully: a failing source reports in its own `error` field, so a `200`
with five errors is a normal response; **Massive ignores the timeframe** and always reports the
previous trading day, so its large delay is by design; and the IBKR row's price comes from Yahoo
(IBKR is only probed for reachability).

**Spot a validator abuser.** `GET /api/admin/validator-stats` — the signal is `crash_rate`, not
`fail_ratio`. Bad Pine *fails validation*; it does not crash the validator, so a legitimate user's
crash rate is ≈0 no matter how bad their code. A high `fail_ratio` alone is just a beginner.

### Admin endpoint catalog

| Area | Endpoints |
|---|---|
| Users | `GET /api/admin/users`, `PATCH /api/admin/users/{id}/plan`, `DELETE /api/admin/users/{id}` |
| Settings | `GET/PATCH /api/admin/settings` (all thirteen fields required on PATCH) |
| Symbols | `GET/POST /api/admin/symbols`, `PATCH/DELETE /api/admin/symbols/{id}`, `POST /api/admin/symbols/refresh-ticks` |
| Options / GEX | `GET/POST /api/admin/options-overrides`, `DELETE /api/admin/options-overrides?tv_symbol=`, `GET /api/admin/gex/preview` |
| Fleet | `GET/POST /api/admin/runners`, `PATCH/DELETE /api/admin/runners/{id}` |
| Runtimes | `GET/POST /api/admin/runtime-versions`, `PATCH/DELETE /api/admin/runtime-versions/{v}`, `POST /api/admin/runtime-versions/{v}/default`, `GET /api/admin/runtime-availability` |
| Dedicated VPS | `GET/POST /api/admin/vps`, `PATCH/DELETE /api/admin/vps/{id}` |
| Jobs | `GET /api/admin/jobs`, `DELETE /api/admin/jobs/{id}` |
| Health | `GET /api/admin/health`, `GET /api/admin/health/history`, `GET /api/admin/container-stats` |
| Statistics | `GET /api/admin/{parse,runtime,validator,rate-limit}-stats` |
| Data delay | `GET /api/admin/data-delay?symbol_id=&timeframe=` |
| Security | `GET /api/admin/login-events`, `GET/POST /api/admin/blacklist`, `DELETE /api/admin/blacklist/{id}`, `GET /api/admin/newsletter` |

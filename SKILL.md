---
name: pineconex-api
description: >-
  Drive a PineconeX account over its web API — create/validate Pine Script v6
  strategies, run backtests, parameter sweeps, robustness (permutation
  significance) tests, launch and monitor live trading bots, and fetch results.
  Use whenever the user wants to operate PineconeX programmatically (not through
  the web UI) with an API key.
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
fresh one. **403** means either a plan quota (concurrent jobs/strategies) was hit, or the endpoint
requires a higher plan — the only plan-gated endpoints are **robustness** and **stress**, which
need **Premium**. Backtest, sweep and **live trading are available on every plan**, including
free; what the tier buys is capacity (concurrent jobs), not access. **429** means a per-user rate
limit (validation or job launches) — back off before retrying.

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
4. **Poll** until terminal, then **fetch results**:
   ```bash
   auth "$PINECONEX_API_URL/api/v1/jobs/<job_id>"          # .status
   auth "$PINECONEX_API_URL/api/v1/jobs/<job_id>/results"  # metrics JSON
   ```
   Terminal statuses: `completed`, `failed`, `cancelled`, `timeout`. Non-terminal: `pending`,
   `running`. Poll every few seconds; don't hammer.

Sweeps (`/api/v1/jobs/sweep`), robustness (`/api/v1/jobs/robustness`) and stress
(`/api/v1/jobs/stress`) follow the same launch → poll → results shape. Robustness runs a
permutation (Monte Carlo significance) test — it returns a `p_value` on whether the strategy's
edge is real price structure or luck (low = real), plus `hurst`/`variance_ratio` describing the
instrument. Stress maps where a config *breaks* across synthetic markets — it is neither an
optimizer nor a significance test. Robustness and stress require a **Premium** plan.
See `references/api-reference.md` for their request fields.

**Sweep modes are `grid`, `random`, `rbf`.** `bayesian` and `monte_carlo` were removed 2026-07-13
after failing a blind-random control at equal budget (monte_carlo scored *worse* than random;
bayesian tied it exactly). Only `rbf` steers, so only `rbf` reads `metric` — grid enumerates every
cell and random samples blindly regardless of the objective.

**Two settings users will not think to ask for, and should:**
* **`metric`** — the objective. Defaults to `net_pnl_pct`, which will happily buy return with
  drawdown. `return_over_dd` and `sharpe` are the risk-adjusted ones. If a user says they care
  about drawdown or consistency, set it. For anything the built-ins can't express, pass a custom
  formula: `"metric": "expr: net_pnl_pct - 0.5 * max_dd_pct + 0.1 * trades"` — variables
  `net_pnl_pct`, `max_dd_pct`, `trades`, `win_rate`, `profit_factor`, `expectancy`, `sharpe`,
  `return_over_dd`; operators `+ - * / ( )`. It is MAXIMISED as written (penalties get a minus
  sign; drawdown is a positive %). Works on `/jobs/sweep` (rbf) and `/jobs/robustness`.
* **`min_trades`** (default 5) — a trial with fewer trades can never win. Do **not** set it to 0
  with `profit_factor` or `win_rate`: a config that never trades has no losses, so its profit
  factor is `+inf`, and the search will converge on a strategy that refuses to trade.

**Both endpoints 400 if the strategy has no `//@sweep` parameters.** There is nothing to search.

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

Two related things worth telling users:
* **Out-of-sample needs no special mode** — it is a date range. Sweep on the training window, then
  run this test on the unseen window with the parameters fixed. `fixed` is the *correct* null
  there, because no search touched those bars.
* **`perm_seed` on `/jobs/sweep`** runs a sweep against a permuted copy of the series, so a client
  can build the same null itself by re-submitting one sweep under N seeds.

## Core workflow: live bot

1. Confirm with the user (broker, symbol, paper vs. live).
2. Ensure the broker is connected: `auth "$PINECONEX_API_URL/api/v1/alpaca/status"` (or `saxo`,
   `ibkr`, `lightspeed`, `bitstamp`). If not connected, the user must connect it in the web UI
   (OAuth) — except **Bitstamp**, which takes an API key and can be connected over the API
   (`POST /api/v1/bitstamp/credentials`).
3. Launch:
   ```bash
   auth -X POST "$PINECONEX_API_URL/api/v1/jobs/live" \
     -H "Content-Type: application/json" -d '{
       "strategy_id":"<uuid>", "symbol":"AAPL", "timeframe":"1H",
       "broker":"alpaca", "webhook_url":"https://example.com/hook"
     }'
   ```
4. Monitor: `GET /api/v1/jobs/<id>` for status, `GET /api/v1/jobs/<id>/logs` (SSE — stream it with the
   `Authorization: Bearer` header, e.g. `curl -N`). Stop with `DELETE /api/v1/jobs/<id>`.

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
| Jobs | `GET /api/v1/jobs`, `POST /api/v1/jobs/{backtest,sweep,robustness,stress,live}`, `GET /api/v1/jobs/{id}`, `GET /api/v1/jobs/{id}/results`, `GET /api/v1/jobs/{id}/logs` (SSE), `DELETE /api/v1/jobs/{id}`, `POST /api/v1/jobs/{id}/analyse` |
| Data | `GET /api/v1/data/symbols`, `GET /api/v1/data/catalog`, `POST /api/v1/data/fetch` |
| Brokers | `GET /api/v1/{alpaca,saxo,ibkr,lightspeed,bitstamp}/status`, `POST /api/v1/bitstamp/credentials`, `DELETE /api/v1/bitstamp/disconnect` |
| Account | `GET/PATCH /api/v1/auth/me`, `GET/PUT /api/v1/newsletter/me` (newsletter opt-in/out), `GET/POST /api/v1/auth/keys` (key mgmt is session-only, not via key), `DELETE /api/v1/auth/keys/{id}` |

Full request/response field shapes: **`references/api-reference.md`** (read it when you need
exact field names, optional vs. required, or the `inputs`/`params` formats for parameter overrides).

## Writing strategies

Strategy `code` is **Pine Script v6 for the PineconeX headless interpreter**, which diverges from
TradingView (chart/UI calls are silently ignored; alert primitives are repurposed). If you are
authoring or editing the Pine source itself, prefer the dedicated Pine authoring guidance rather
than assuming TradingView semantics.

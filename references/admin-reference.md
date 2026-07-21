# PineconeX Admin API — endpoint reference

The operator surface: `/api/admin/*`. It runs the platform itself — users, plans, the symbol
universe, the runner fleet, runtime images, health and the Dedicated-VPS registry.

**Who this is for.** On the shared platform at `pineconex.com` these endpoints are staff-only. On a
**Dedicated VPS** instance (`<subdomain>.pineconex.com`) the admitted owner is admitted as a *box
admin*, so this is the API for running your own single-tenant instance.

---

## Auth

Same `Authorization: Bearer …` header as the user API — either a session JWT or a `pcx_live_…`
API key. The gate is one line:

```
plan == "admin"   →  allowed
anything else     →  403 { "error": "forbidden" }
```

Three consequences worth stating plainly:

- **An admin's API key is a full-admin key.** The gate reads the *plan*, not how you authenticated.
  There is no narrower admin scope, no per-endpoint permission, and no way to mint a read-only
  admin key. Treat an admin `pcx_live_…` as equivalent to the account password.
- **The plan is re-read from the database on every request.** Demoting an admin takes effect on
  their very next call — no token expiry to wait out.
- **An API key in a URL never authenticates.** The `?token=` fallback (which exists for browser
  `EventSource`) discards anything starting with `pcx_`.

`401` = no/invalid credential, disabled account, deleted account. `403` = authenticated but not
admin. Errors are always `{ "error": string }`.

**Not versioned.** Admin lives at `/api/admin/*` only — there is no `/api/v1/admin`. It is
deliberately outside the public contract, so it may change without a major-version bump. Pin
nothing to it that you cannot fix quickly.

---

## Read this before you write anything

Three PATCH endpoints, three different semantics. This is the single most common way to lose data
here:

| Endpoint | Semantics | Omitting a field means |
|---|---|---|
| `PATCH /symbols/{id}` | **full replace** | the column is set to **NULL** — you erase it |
| `PATCH /settings` | **all fields mandatory** | `422` — the request is rejected |
| `PATCH /runners/{id}`, `/vps/{id}`, `/runtime-versions/{v}` | partial (`COALESCE`) | leave it unchanged |

So: `GET` the symbol, edit the object, `PATCH` the whole thing back. Never PATCH a symbol with a
sparse body.

And three endpoints have consequences outside the database:

- `PATCH /vps/{id}` with `status: "deprovisioned"` **destroys the Hetzner server**.
- `PATCH /users/{id}/plan` off a paid plan **cancels their Stripe subscription immediately**, no
  refund and no proration.
- `DELETE /users/{id}` **hard-deletes** that user's strategies, bot events and parse errors.

Several deletes return `204` for ids that never existed (`/runners`, `/vps`, `/runtime-versions`,
and `PATCH /vps`). A `204` is not proof anything was found.

---

## Users

### GET /api/admin/users → array, newest first
```
id              uuid
name            string     ("" if the per-user key cannot decrypt it)
email           string     ("—" on decrypt failure)
plan            string
strategy_count  int
job_count       int
active_jobs     int        (pending + running)
created_at      datetime
```

### PATCH /api/admin/users/{id}/plan → `204`
Body: `{ "plan": "free"|"pro"|"premium"|"dedicated"|"admin"|"disabled" }`, required.
Anything else → `400 "invalid plan value"`. Unknown user → `404`.

**Moving a user off `pro`/`premium` cancels their Stripe subscription** (immediate, no refund) and
clears the stored subscription id. A Stripe failure is logged but does **not** roll back the plan
change — the two can diverge, so check Stripe if the call was slow or errored.

`disabled` is the lockout: that account gets `401` on every request from the next call onward.

### DELETE /api/admin/users/{id} → `204`
Soft-delete + PII scrub, in this order: kill their running jobs on the runner → hard-delete their
`strategies`, `bot_events`, `parse_errors` → anonymise the `users` row (`deleted_at` set, email
overwritten, name/phone/telegram/GitHub fields nulled).

The `jobs` rows themselves are **kept**. Self-deletion is refused
(`400 "Cannot delete your own account via admin"`).

---

## Plans and settings

### GET /api/admin/settings → the twelve platform settings
```
support_telegram        string   ("")
banner_message          string   ("")   — shown site-wide to every user
wiki_repo               string   ("")
trial_days              int      (30)
free_max_strategies     int      (5)
free_max_jobs           int      (1)
pro_max_jobs            int      (5)
max_max_jobs            int      (10)   — the Premium concurrent-job cap
parquet_retention_days  int      (90)   — 0 disables the retention watchdog
job_log_max_mb          int      (10)   — 0 = uncapped
backtest_timeout_secs   int      (600)  — 0 = no timeout
sweep_timeout_secs      int      (1800) — also applies to robustness; 0 = no timeout
```
(Defaults in brackets, used when the key is absent or unparseable.)

### PATCH /api/admin/settings → `204`
**All twelve fields are required** — this is a whole-form replace, and a partial body is rejected
`422`. `GET` first, mutate, send it all back. No range validation is performed: a zero or negative
value is stored as given, and the three `0`-means-disabled cases above are the intended use of that.

`GET /api/settings` (no admin needed) exposes the user-visible subset — `support_telegram`,
`banner_message`, `wiki_repo`, the trial/quota numbers, and `github_webhook_enabled`.

---

## Symbols

The tradeable universe. Each row maps one canonical `tv_symbol` onto every data source and broker;
a `null` ticker means "this source cannot serve this symbol" and a job asking for it gets `400`.

### GET /api/admin/symbols → array, ordered by index then name
```
id, display_name, index_name, tv_symbol,
saxo_uic (int|null), saxo_symbol, saxo_asset_type,
yahoo_ticker, massive_ticker, ibkr_symbol, ibkr_exchange,
alpaca_us_symbol, alpaca_eu_symbol, bitstamp_pair,
mintick (float|null), currency, enabled (bool)
```

### POST /api/admin/symbols → `201` · PATCH /api/admin/symbols/{id} → `200`
Both take the same body. Required: `display_name`, `index_name`, `tv_symbol`. Everything else is
optional — **and on PATCH, optional means "set to NULL if absent"** (see the warning above).

Defaults on an absent field: `saxo_asset_type` → `"Stock"`, `enabled` → `true`.
`bitstamp_pair` is trimmed and **lowercased** (Bitstamp's OHLC path is case-sensitive: `btceur`
works, `BTCEUR` 404s); an empty string becomes NULL.

### DELETE /api/admin/symbols/{id} → `204` (`404` if unknown)

### POST /api/admin/symbols/refresh-ticks → `{ updated, failed, details[] }`
Pulls the real tick size and quote currency from Saxo for every symbol with a `saxo_uic`, and
writes `mintick` / `currency` — the values that become `syminfo.mintick` / `syminfo.currency`
inside running jobs. Tick comes from `TickSize`, falling back to `TickSizeScheme.DefaultTickSize`.
A null from Saxo leaves the stored value alone rather than clearing it.

Uses **your own** Saxo session and its `sim`/`live` environment, so it needs Saxo connected on the
calling admin account (`400 "Saxo not connected: …"` otherwise). `details[]` is one line per
symbol, e.g. `"AAPL: tick=0.01 currency=USD"` or `"XYZ: HTTP 404"`.

---

## Options-chain overrides (GEX symbology)

A **sparse exceptions table** for the `gex.*` namespace. A symbol that is *absent* resolves by
identity (its `yahoo_ticker` / `alpaca_us_symbol` is used as the options underlying). A symbol
that is *present* with a NULL provider column means "options are **unavailable** on that provider"
— the fetcher returns `na` rather than guessing. Only add a row when the identity mapping is wrong.

### GET /api/admin/options-overrides → array
`tv_symbol`, `display_name` (joined from symbols), `yahoo_options_symbol`,
`alpaca_options_symbol`, `note` — the last three nullable.

### POST /api/admin/options-overrides → `200` — create **or** edit (upsert on `tv_symbol`)
There is no PATCH; re-POST the whole row. `tv_symbol` must already exist in `symbols`
(`400 "No symbol with tv_symbol '…'"`). A blank provider field is stored as NULL, i.e. "no options
here".

### DELETE /api/admin/options-overrides?tv_symbol=… → `204` (`404` if unknown)
The key is a **query parameter**, not a path segment, because `tv_symbol` contains colons
(`NASDAQ:INTC`).

### GET /api/admin/gex/preview?symbol=…&source=… → snapshot
Wiring smoke test — resolves the underlying, fetches a bounded chain (45-day expiry horizon),
computes dealer gamma exposure, and returns it with chain diagnostics. Same fetch+compute path the
recorder uses, so a clean preview means the pipeline works for that symbol.

`source` ∈ `alpaca` (default) · `massive` · `yahoo` · `saxo`. **An unrecognised value silently
falls back to `alpaca`** rather than erroring. `alpaca` and `saxo` use the *calling admin's* broker
connection; `massive` uses the server's key.

```
symbol, source, underlying, spot,
net, flip, pin, call_wall, call_wallv, put_wall, put_wallv,   (nullable numbers)
top_walls: [ { strike, gamma } ]        (max 6, ranked by |gamma|; + = call/resistance,
                                         − = put/support)
chain:     { contracts_seen, with_oi, with_mid, legs }
```
`chain` is the diagnostic that matters: a chain with many `contracts_seen` but near-zero `with_oi`
or `with_mid` produced a GEX number out of almost nothing — treat it as unusable, not as a signal.
Every failure is a `400` naming the stage (`"spot: …"`, `"chain: …"`, `"X has no Saxo UIC"`, …).

---

## Runner fleet

### GET /api/admin/runners → array
```
id (int), name, url, max_capacity (int),
active_jobs (int)      — live count of that runner's running+pending jobs
is_active (bool),
last_seen_at datetime|null   — stamped by the 60s health watchdog
created_at
```
`last_seen_at` going stale is the signal that a runner is gone; `is_active` is the *intent* flag
you control. Dispatch picks the active runner with the most headroom and returns `503` when every
one is full.

### POST /api/admin/runners → `201`
`{ name, url, max_capacity? }` — name and url required, capacity defaults to `5`.
(`active_jobs` is reported as `0` in this response regardless of reality.)

### PATCH /api/admin/runners/{id} → `200`
Optional `name`, `url`, `max_capacity`, `is_active`. Partial — omitted fields are untouched.
Setting `is_active: false` drains a runner: no new dispatch, existing jobs keep running.

### DELETE /api/admin/runners/{id} → `204`
**Guarded**: refuses while the runner has work —
`400 "runner has N active job(s) — stop them first"`. Deleting an unknown id is a silent `204`.

---

## Runtime versions

The allow-list behind the `//@runtime=<version>` pin, plus the promoted fleet default.
Unpinned jobs and **all live bots** run the default — never a rolling `:latest` — so an engine fix
reaches live trading by *promotion*, not by rebuilding an image.

### GET /api/admin/runtime-versions → array, newest first
```
version, notes, is_active (bool), is_default (bool), created_at,
usage (int)   — jobs whose config pinned this version
```

### POST /api/admin/runtime-versions → `201`
`{ version, notes?, is_active? }`. Idempotent: re-registering an existing version overwrites its
notes and active flag. `version` must start alphanumeric and use only letters, digits, `.`, `_`
or `-`, max 64 chars. `is_active` defaults to `true`.

Registering does **not** build or distribute anything — the image must already exist on each
runner (`RUNTIME_VERSION=<v> RUNNER_HOSTS="…" ./build-runner.sh`). Check with
`/runtime-availability` before promoting.

### PATCH /api/admin/runtime-versions/{version} → `200`
Optional `notes`, `is_active`. **Deactivating a version automatically demotes it** if it was the
default — so deactivating the current default leaves the fleet with *no* default. Promote a
replacement first.

### DELETE /api/admin/runtime-versions/{version} → `204`
Removes the registry entry only. Already-dispatched jobs keep the image they resolved, and no
Docker image is deleted on any runner. Unknown version → silent `204`.

### POST /api/admin/runtime-versions/{version}/default → `204`
Promote to fleet default. Transactional: demote-all + promote run together, so a failed promote
rolls back and the previous default survives. An unknown or **inactive** version →
`400 "runtime version 'X' not found or inactive — cannot make it the default"`.

### GET /api/admin/runtime-availability
Cross-references the registry against what each active runner actually has on disk — the check
that catches "registered but missing on runner 2", where a pinned job would fail on one host and
work on another. Fans out with a 3s timeout per runner; an unreachable runner degrades to
`reachable: false` instead of failing the request.

```
registered: [ "2026.07.20", … ]
runners:    [ { runner, reachable (bool), images: [ { tag, id } ] } ]
```

---

## Dedicated VPS instances

`dedicated_instances` is the authoritative subdomain registry for the Dedicated tier: one
single-tenant box per customer at `<subdomain>.pineconex.com`, with its own API, runner and
database. In the manual flow an admin adds the row after provisioning.

`region` ∈ `eu` · `us`.
`status` ∈ `pending` · `provisioning` · `active` · `suspended` · `deprovisioned`.

### GET /api/admin/vps → array, newest first
```
id (int), user_id, email, stripe_subscription_id|null,
region, provider, subdomain, vps_ip|null, hetzner_server_id (int|null),
dns_record_created (bool), status, last_seen_at|null, created_at
```

### POST /api/admin/vps → `201 { "id": int }`
`{ user_id, subdomain, region?, stripe_subscription_id? }`. `subdomain` is lowercased; `region`
defaults to `"eu"` and must be `eu`/`us`. Returns **only the id**, not the row.

### PATCH /api/admin/vps/{id} → `204`
Optional `subdomain`, `region`, `status`, `vps_ip`, `hetzner_server_id`, `dns_record_created`,
`stripe_subscription_id`. Partial.

> **Setting `status: "deprovisioned"` deletes the Hetzner server.** The handler re-reads
> `hetzner_server_id` and calls the provider's delete API. It is best-effort: the row records the
> new status whether or not the box actually died, so verify at Hetzner rather than trusting the
> `204`. To suspend a customer without destroying their machine and data, use
> `status: "suspended"`.

Unknown id is a silent `204` — nothing was updated and you are not told.

### DELETE /api/admin/vps/{id} → `204`
**Removes the tracking row only — it does not touch the box.** The teardown path is
`status: "deprovisioned"` above. Deleting the row first orphans the server: nothing left records
its id, so it must then be found and killed by hand at Hetzner.

### The user-facing half of this tier
- `GET /api/dedicated/sso` (session auth, on the shared platform) mints a 60-second, subdomain-bound
  handoff link to the caller's active instance.
- `GET /api/auth/sso?token={jwt}` on the instance is its **only** login path — no OAuth, no login
  page. It verifies against the per-instance secret, checks the email allowlist, and admits the
  user as a box admin.

---

## Jobs (all users)

### GET /api/admin/jobs → every user's active jobs
`running` + `pending` only, ordered by runner. Enriched with live container CPU/memory by fanning
out to every active runner (3s timeout; a slow runner just yields nulls).
```
id, user_id, user_email, container_id|null,
strategy_name|null   (null when the strategy has been deleted)
job_type, status, auto_restart, created_at, finished_at|null,
config               (the persisted job config — credentials are stripped before storage)
cpu_pct|null, mem_pct|null      (null when no runner reported that container)
runner_id|null, runner_name|null  (null = legacy pre-fleet job)
```

### DELETE /api/admin/jobs/{id} → `204`
Cancel any user's job. **Live jobs are soft-cancelled** — the row is kept with
`status: "cancelled"` and a `stopped` bot event, so it stays in the user's history. **Every other
job type is hard-deleted**, along with its result files.

Cancelling a live job **cancels its resting broker orders but does not close the position** — the
user still holds it, now unmanaged. That is deliberate (an open position is an asset; closing it
would be trading on their behalf), but it means killing a live job is not "making it safe" and the
user should be told.

---

## Health

### GET /api/admin/health → live probe of every tier
```
api:      ServiceHealth
runners:  [ { name, health: ServiceHealth } ]
postgres: ServiceHealth
redis:    ServiceHealth
```
`ServiceHealth`: `status` (`"ok"`|`"error"`|`"unreachable"`), plus whichever apply — `uptime_s`,
`mem_rss_mb`, `cpu_pct`, `disk_used_gb`, `disk_total_gb` (API and runners), `latency_ms`
(Postgres and Redis), `containers_running` (runners), `version` (Postgres/Redis), and `build`
(`{ version, git_sha, built_at }`, API and runners — the way to spot a runner left on an old
binary).

`"unreachable"` means the request failed; `"error"` means it answered but unusably. The API's own
status is always `"ok"` — if the handler ran, the API is up. **This request always takes ≥200ms**:
CPU is sampled by reading `/proc/self/stat` twice, 200ms apart. Don't poll it tightly.

### GET /api/admin/health/history → array, oldest first
A rolling in-memory ring buffer sampled by a background task — **not persisted, so it resets on
every API restart** (an empty history right after a crash means the restart, not a healthy past).
```
ts (unix ms), api_cpu, api_mem, api_disk_used,
runner_cpu|null    (mean across reachable runners)
runner_mem|null    (sum), runner_containers|null (sum)
runner_disks: [ { name, used_gb, total_gb } ]
pg_ok (bool), pg_ms|null, redis_ok (bool), redis_ms|null
```

### GET /api/admin/container-stats → per-user container load
`{ runner_count, stats: [ { user_id, cpu_pct, mem_pct } ] }` — summed across all of that user's
running containers on all runners. A user whose container no runner reports is **omitted from
`stats` entirely** rather than appearing with zeros.

---

## Statistics

All four windows are fixed at 30 days (with a 7-day subtotal) and cannot be parameterised.
Shared shapes: `DailyCount = { date, count }`, `TopError = { message, count, last_seen }`,
`TopUser = { user_id, count, last_seen }`. Timestamps in these payloads are **unix milliseconds**.

### GET /api/admin/parse-stats — Pine parse failures
`total_30d`, `total_7d`, `daily[]`, `top_errors[]` (max 20), `top_users[]` (max 10).
The top errors are the product feedback loop: a message climbing this list is usually a Pine
feature people expect and the parser does not have.

### GET /api/admin/runtime-stats — job failures + live-bot crashes
`total_30d`, `total_7d`, `daily[]`, `top_errors[]` (max 20, grouped by job error message),
`top_strategies[]` (`{ strategy_name, job_type, count, last_seen }`, max 20),
`top_users[]` (max 10), and `recent_crashes[]` (`{ job_id, user_id, message, created_at }`, max 50)
drawn from `crashed` / `max_restarts` bot events. A bot at `max_restarts` has a genuinely dead
credential — a restart cannot fix it, the user must reconnect the broker.

### GET /api/admin/validator-stats — the fuzzing-detection panel
`total_30d`, `total_7d`, `crashes_30d`, `timeouts_30d`, `daily[]`, and `top_users[]`:
```
user_id, name, crashes, last_seen,
validations_passed, validations_failed   (lifetime counters)
crash_rate    crashes / (passed + failed)   ← the primary signal
fail_ratio    failed  / (passed + failed)
```
The validator runs untrusted Pine as a subprocess on the API host, so a user with a **non-zero
`crash_rate`** is the thing to look at: a legitimate user's is ≈0 no matter how bad their Pine is,
because bad Pine fails validation, it does not crash the validator. A high `fail_ratio` alone is
just a beginner — do not read the two the same way.

### GET /api/admin/rate-limit-stats — who is hammering the API
```
since (unix ms — API process start; the window these counts cover)
total, validate_hits, job_hits,
top: [ { user_id|null, name, category ("validate"|"job"), hits, last_seen } ]   (max 50)
```
Counted **in memory**, so this resets on restart and `since` is the honest window. It catches a
scripted hammer that never reaches the crash tables, because a request rejected `429` never runs.

---

## Data delay

### GET /api/admin/data-delay?symbol_id={uuid}&timeframe={tf}
Asks **every** data source for its newest bar for one symbol and reports how stale each one is.
This is what makes the *broker → Massive → Yahoo* source priority an informed choice rather than a
guess — and cross-checking `last_close` between vendors is the cheap way to catch one of them
serving a bad print.

Returns five fixed keys — `yahoo`, `massive`, `saxo`, `ibkr`, `alpaca` — each:
```
last_ts_ns     int|null    (newest bar timestamp, NANOSECONDS)
delay_minutes  int|null    (floored, never negative)
last_close     float|null
error          string|null (when set, the other three are null)
```

**A failing source never fails the request** — it fails into its own `error` field, so a `200` with
five errors is a normal response. Sources are queried **sequentially**, and Saxo/Alpaca/IBKR use
the calling admin's own broker credentials, so this call can be slow and it returns *your* view of
the data, not the platform's.

Per-source quirks that will otherwise look like bugs:

- **Massive ignores `timeframe`.** It uses the previous-trading-day endpoint (the delayed plan
  cannot sort descending), so its `delay_minutes` is large by design. Its error strings also still
  say `POLYGON_API_KEY` / `"no Polygon ticker"` — same thing, older name.
- **IBKR is a hybrid probe.** It TCP-connects to your configured TWS/Gateway to prove reachability,
  then reads the *price from Yahoo*. IBKR is never asked for bars, so `last_close` on that row is
  not evidence about IBKR's data.
- **Saxo** uses the server's `SAXO_ENV`, unlike `refresh-ticks` and `gex/preview` which use your
  per-account environment. The two can disagree.
- The timeframe mapping table here is a **deliberate duplicate** of the one in the data routes, so
  a timeframe added there and not here silently stops appearing.

---

## Security

Every mutation below re-syncs the edge proxy's blocklist file. The database is the source of
truth; a proxy sync failure is logged and the database change still stands, so a rule can be
recorded but not yet enforced.

### GET /api/admin/login-events?user_id={uuid}&limit={n}
Successful logins with the real client IP. Both parameters optional; `limit` defaults to **200**
and is clamped to 1..1000. The IP is the proxy's observed peer, not a client-supplied header, so it
cannot be spoofed by the caller.
```
id, user_id, name, email, ip, provider, created_at
```

### GET /api/admin/blacklist → array
`id, ip, reason, added_by|null, created_at, expires_at|null`.

### POST /api/admin/blacklist → `200` (upsert on `ip`)
```
ip                string  required  (IP or CIDR; invalid → 400)
reason            string  optional  ("")
expires_in_hours  int     optional  (> 0 → temporary; omitted or ≤ 0 → PERMANENT)
```
Re-posting an existing IP replaces its reason and expiry. Two self-lockout guards refuse the
request `400`: blocking a loopback address, and blocking **your own current IP**.
(`added_by` comes back `null` in this response even though the row records you.)

### DELETE /api/admin/blacklist/{id} → `204` (`404` if unknown)

### GET /api/admin/newsletter → `[ { email, created_at } ]`
Marketing signups, plaintext by design (not account-linked PII) — the list to paste into a BCC
field. Account-level opt-in/out lives on the user API at `/api/v1/newsletter/me`.

---

## Endpoint index

| Method | Path | → |
|---|---|---|
| GET | `/api/admin/users` | 200 |
| PATCH | `/api/admin/users/{id}/plan` | 204 |
| DELETE | `/api/admin/users/{id}` | 204 |
| GET | `/api/admin/settings` | 200 |
| PATCH | `/api/admin/settings` | 204 |
| GET · POST | `/api/admin/symbols` | 200 · 201 |
| PATCH · DELETE | `/api/admin/symbols/{id}` | 200 · 204 |
| POST | `/api/admin/symbols/refresh-ticks` | 200 |
| GET · POST | `/api/admin/options-overrides` | 200 |
| DELETE | `/api/admin/options-overrides?tv_symbol=` | 204 |
| GET | `/api/admin/gex/preview?symbol=&source=` | 200 |
| GET · POST | `/api/admin/runners` | 200 · 201 |
| PATCH · DELETE | `/api/admin/runners/{id}` | 200 · 204 |
| GET · POST | `/api/admin/runtime-versions` | 200 · 201 |
| PATCH · DELETE | `/api/admin/runtime-versions/{version}` | 200 · 204 |
| POST | `/api/admin/runtime-versions/{version}/default` | 204 |
| GET | `/api/admin/runtime-availability` | 200 |
| GET · POST | `/api/admin/vps` | 200 · 201 |
| PATCH · DELETE | `/api/admin/vps/{id}` | 204 |
| GET | `/api/admin/jobs` | 200 |
| DELETE | `/api/admin/jobs/{id}` | 204 |
| GET | `/api/admin/health` | 200 |
| GET | `/api/admin/health/history` | 200 |
| GET | `/api/admin/container-stats` | 200 |
| GET | `/api/admin/parse-stats` | 200 |
| GET | `/api/admin/runtime-stats` | 200 |
| GET | `/api/admin/validator-stats` | 200 |
| GET | `/api/admin/rate-limit-stats` | 200 |
| GET | `/api/admin/data-delay?symbol_id=&timeframe=` | 200 |
| GET | `/api/admin/login-events` | 200 |
| GET · POST | `/api/admin/blacklist` | 200 |
| DELETE | `/api/admin/blacklist/{id}` | 204 |
| GET | `/api/admin/newsletter` | 200 |

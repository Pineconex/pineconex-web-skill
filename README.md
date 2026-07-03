# PineconeX API skill for Claude

A [Claude](https://claude.com) skill that lets Claude Code (or any agent that reads skills)
drive a [PineconeX](https://pineconex.com) account over its web API — create and validate Pine
Script v6 strategies, run backtests / sweeps / walk-forward analysis, and launch and monitor
live trading bots.

## Install

Copy this skill into your Claude skills directory:

```bash
git clone https://github.com/Pineconex/pineconex-web-skill
mkdir -p ~/.claude/skills/pineconex-api
cp -r pineconex-web-skill/SKILL.md pineconex-web-skill/references ~/.claude/skills/pineconex-api/
```

## Configure

The skill reads two environment variables:

- `PINECONEX_API_KEY` — a personal access token (`pcx_live_…`). Create one in the PineconeX web
  app under **Account → API keys** (shown once).
- `PINECONEX_API_URL` — base URL, default `https://pineconex.com`.

Then ask Claude to, e.g., "backtest my strategy on AAPL daily for 2020–2024 via PineconeX."

## Contents

- [`SKILL.md`](./SKILL.md) — the skill definition (auth, workflows, guardrails).
- [`references/api-reference.md`](./references/api-reference.md) — full endpoint reference.

Full API docs: [https://github.com/Pineconex/pineconex-wep-api](https://github.com/Pineconex/pineconex-wep-api).

## License

[CC BY 4.0](./LICENSE) — © 2026 PineconeX.

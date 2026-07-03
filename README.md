# PineconeX API skill

A skill that lets an AI assistant drive a [PineconeX](https://pineconex.com) account over its
web API — create and validate Pine Script v6 strategies, run backtests / sweeps / walk-forward
analysis, and launch and monitor live trading bots. Works with **Claude** and **ChatGPT**.

First, get an API key: in the PineconeX web app go to **Account → API keys** and create a
personal access token (it looks like `pcx_live_…` and is shown only once — copy it now).

## Install in Claude

**Claude Code (CLI / IDE).** Drop the skill into your Claude skills folder:

```bash
git clone https://github.com/Pineconex/pineconex-web-skill
mkdir -p ~/.claude/skills/pineconex-api
cp -r pineconex-web-skill/SKILL.md pineconex-web-skill/references ~/.claude/skills/pineconex-api/
```

Set your key in the environment so the skill can authenticate:

```bash
export PINECONEX_API_KEY=pcx_live_your_key_here
export PINECONEX_API_URL=https://pineconex.com   # optional; this is the default
```

Then just ask, e.g. *"backtest my strategy on AAPL daily for 2020–2024 via PineconeX."*

**Claude.ai / Claude Desktop.** Create a **Project**, paste the contents of `SKILL.md` into the
project's custom instructions, and upload `references/api-reference.md` as project knowledge.
Give Claude your API key in chat when it asks (or run the `curl` commands it produces yourself).

## Install in ChatGPT (Custom GPT)

1. In ChatGPT, open **Explore GPTs → Create**, then the **Configure** tab.
2. Paste the contents of `SKILL.md` into **Instructions**, and upload
   `references/api-reference.md` under **Knowledge**.
3. To let the GPT call PineconeX **directly**, add an **Action**:
   - **Schema** → import the bundled [`references/openapi.yaml`](./references/openapi.yaml)
     (paste its contents, or host it and paste the URL). It already targets `https://pineconex.com`.
   - **Authentication** → *API Key*, **Auth Type** *Bearer*, and paste your `pcx_live_…` key.
   - Without an Action the GPT still works — it will hand you the exact `curl` commands to run.

Keep your API key private; anyone with it can act on your PineconeX account. Revoke a key any
time under **Account → API keys**.

## Contents

- [`SKILL.md`](./SKILL.md) — the skill definition (auth, workflows, guardrails).
- [`references/api-reference.md`](./references/api-reference.md) — full endpoint reference.
- [`references/openapi.yaml`](./references/openapi.yaml) — OpenAPI 3.1 spec for ChatGPT Actions.

Full API docs: [https://github.com/Pineconex/pineconex-web-api](https://github.com/Pineconex/pineconex-web-api).

## License

[CC BY 4.0](./LICENSE) — © 2026 PineconeX.

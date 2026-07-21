# AI Skill

Let an AI assistant or agent drive your PineconeX account for you. The **PineconeX API skill** is a
small, open package that teaches **Claude**, **ChatGPT**, or **OpenClaw** how to create and validate Pine
Script v6 strategies, run backtests, parameter sweeps and robustness analysis, and launch
and monitor live trading bots — all over the [web API](/app/api-docs), using an API key you
control.

The skill is published in the public repo
[Pineconex/pineconex-web-skill](https://github.com/Pineconex/pineconex-web-skill) under **CC BY 4.0**. Everything below links straight to it on GitHub.

> Running a **Dedicated VPS** instance? You are an admin of your own box, and the
> [`vps` branch](../../tree/vps) carries the same skill plus the `/api/admin` operator surface.

## 1. Get an API key

Create a **personal access token** under [Account → API keys](/app/account). A key looks like
`pcx_live_…` and is shown **once** — copy it then. Keep it private: anyone with it can act on
your account. Revoke a key any time from the same page.

## 2. Download the skill

Grab the whole repo as a zip, or clone it:

- **[Download .zip](https://github.com/Pineconex/pineconex-web-skill/archive/refs/heads/main.zip)** — the complete skill (SKILL.md + references).
- Or clone:

```bash
git clone -b main https://github.com/Pineconex/pineconex-web-skill.git
```

Individual files (direct HTTPS download from GitHub):

| File | Purpose | Download |
|---|---|---|
| `SKILL.md` | The skill definition — auth, workflows, guardrails. | [raw](https://raw.githubusercontent.com/Pineconex/pineconex-web-skill/main/SKILL.md) · [view](https://github.com/Pineconex/pineconex-web-skill/blob/main/SKILL.md) |
| `references/api-reference.md` | Full endpoint reference. | [raw](https://raw.githubusercontent.com/Pineconex/pineconex-web-skill/main/references/api-reference.md) · [view](https://github.com/Pineconex/pineconex-web-skill/blob/main/references/api-reference.md) |
| `references/openapi.yaml` | OpenAPI 3.1 spec for ChatGPT Actions. | [raw](https://raw.githubusercontent.com/Pineconex/pineconex-web-skill/main/references/openapi.yaml) · [view](https://github.com/Pineconex/pineconex-web-skill/blob/main/references/openapi.yaml) |


**Which files do I need?** For **Claude**, use `SKILL.md` + `references/api-reference.md`
(skip the openapi.yaml). For **ChatGPT / OpenAI**, use the same two files, **plus**
`references/openapi.yaml` if you want the GPT to call PineconeX directly via an Action.

## 3. Install in Claude

**Claude Code (CLI / IDE).** Drop the skill into your Claude skills folder:

```bash
git clone -b main https://github.com/Pineconex/pineconex-web-skill
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

## 4. Install in ChatGPT (Custom GPT)

1. In ChatGPT, open **Explore GPTs → Create**, then the **Configure** tab.
2. Paste the contents of `SKILL.md` into **Instructions**, and upload
   `references/api-reference.md` under **Knowledge**.
3. To let the GPT call PineconeX **directly**, add an **Action**:
   - **Schema** → import the bundled `references/openapi.yaml` (paste its contents, or host it
     and paste the URL). It already targets `https://pineconex.com`.
   - **Authentication** → *API Key*, **Auth Type** *Bearer*, and paste your `pcx_live_…` key.
   - Without an Action the GPT still works — it will hand you the exact `curl` commands to run.

## 5. Install in OpenClaw

[OpenClaw](https://openclaw.ai) is a local, open-source AI assistant. Install the skill straight
from GitHub with its CLI:

```bash
openclaw skills install git:Pineconex/pineconex-web-skill
```

This adds it under `~/.openclaw/skills/` (shared across your agents; append `--global` to force
that scope, or install into a workspace's `skills/` folder for a single agent).

Give it your API key. Because OpenClaw runs locally, a shell export works:

```bash
export PINECONEX_API_KEY=pcx_live_your_key_here
```

Or wire it into `~/.openclaw/openclaw.json` so it's injected on every run:

```json5
{
  skills: {
    entries: {
      "pineconex-api": {
        enabled: true,
        env: { PINECONEX_API_KEY: "pcx_live_your_key_here" },
      }
    }
  }
}
```

Then just ask, e.g. *"backtest my strategy on AAPL daily for 2020–2024 via PineconeX."*

## Safety

- **The skill acts as you.** Anything you can do in the web app, the assistant can do with your
  key — including launching and stopping **live bots that trade real broker accounts**. Confirm
  symbol, broker, and intent before it goes live.
- **Never paste your key into a shared prompt or public GPT.** Prefer the environment variable
  (`PINECONEX_API_KEY`) so the key stays out of chat transcripts.
- Revoke any key immediately from [Account → API keys](/app/account) if it may have leaked.


---

Prefer to call the API yourself? See the [API reference](/app/api-docs). Full public API docs
live at [github.com/Pineconex/pineconex-web-api](https://github.com/Pineconex/pineconex-web-api).

## License

Documentation (this README, `SKILL.md`, `references/api-reference.md`) is under
[CC BY 4.0](./LICENSE) — © 2026 PineconeX. The OpenAPI spec
[`references/openapi.yaml`](./references/openapi.yaml) is a functional file and is instead under
the permissive [MIT license](./references/openapi.yaml.LICENSE), so you can embed it in your own
code or a ChatGPT Action without attribution friction.

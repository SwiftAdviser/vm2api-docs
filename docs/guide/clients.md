# Connect clients

Point Claude Code, Codex CLI, or an SDK at this gateway. The request contract is [Call the API](./api). Issue the key on [Keys](./console/keys).

## What you need

| Piece | Value |
| --- | --- |
| Base URL | `https://<host>` behind TLS, or `http://<host>:8787` on the listen port. No trailing slash |
| Protocol key | `sk-vm-…` from [Keys](./console/keys). The plaintext is shown once |

Do not put the master `VM2API_API_KEY` in a client. The master key can change the panel. A protocol key can call `/v1/*` only.

Send either header:

```http
Authorization: Bearer sk-vm-…
x-api-key: sk-vm-…
```

Claude Code and Codex each append their own path. Set the host for Claude Code. Set the host plus `/v1` for Codex and the OpenAI SDK.

## Claude Code

Claude Code posts to `{ANTHROPIC_BASE_URL}/v1/messages`. The variable is the host, not `/v1`.

```bash
export ANTHROPIC_BASE_URL=https://<host>
export ANTHROPIC_AUTH_TOKEN=sk-vm-…
claude
```

Direct listen, no TLS:

```bash
export ANTHROPIC_BASE_URL=http://<host>:8787
export ANTHROPIC_AUTH_TOKEN=sk-vm-…
```

`ANTHROPIC_AUTH_TOKEN` is sent as `Authorization: Bearer`. That matches this gateway. `ANTHROPIC_API_KEY` would send `x-api-key`, which also works. Use one, not both.

To persist the same values for every project, put them in the `env` block of `~/.claude/settings.json`:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://<host>",
    "ANTHROPIC_AUTH_TOKEN": "sk-vm-…"
  }
}
```

That user file applies to you in every project. Do not put the token in a project's `.claude/settings.json`; that file is committed.

A settings `env` value replaces the same variable inherited from the shell. Run `/status` in Claude Code: **Anthropic base URL** should show this host, and **Auth token** should name `ANTHROPIC_AUTH_TOKEN`.

Claude Code needs a schedulable Claude slot. Pick a Claude id from `GET /v1/models`, for example `claude-sonnet-5`.

## Codex CLI

Codex talks the OpenAI Responses wire. This gateway serves that at `POST /v1/responses`. The provider `base_url` must include `/v1`.

User config is `~/.codex/config.toml`. Add a custom provider. Do not reuse the reserved ids `openai`, `ollama`, or `lmstudio`.

```toml
model = "gpt-5.4"
model_provider = "vm2api"

[model_providers.vm2api]
name = "vm2api"
base_url = "https://<host>/v1"
env_key = "VM2API_KEY"
wire_api = "responses"
```

Direct listen:

```toml
base_url = "http://<host>:8787/v1"
```

Then in the shell that launches `codex`:

```bash
export VM2API_KEY=sk-vm-…
```

`env_key` is the name of that variable, not the key itself. Codex sends it as `Authorization: Bearer`. `wire_api = "responses"` is required; the chat wire is gone.

The key must be allowed to see OpenAI models (`all`, or `openai`). `GET /v1/models` lists what that key can call. Use an id from that list. Codex needs a schedulable OpenAI slot.

## OpenAI and Anthropic SDKs

The OpenAI SDK appends `/chat/completions` and `/responses` to its `base_url`, so that value includes `/v1`:

```python
from openai import OpenAI

client = OpenAI(api_key="sk-vm-…", base_url="https://<host>/v1")
```

The Anthropic SDK appends `/v1/messages` itself, so `base_url` is the host:

```python
from anthropic import Anthropic

client = Anthropic(api_key="sk-vm-…", base_url="https://<host>")
```

TypeScript uses `apiKey` and `baseURL` with the same URL shapes. The Anthropic SDK `api_key` is sent as `x-api-key`. Pass `auth_token` instead if you want `Authorization: Bearer`. This gateway accepts both.

## Check with curl

Replace the host and key, then:

```bash
curl -sS https://<host>/v1/models \
  -H "Authorization: Bearer $KEY"
```

Direct listen: `http://<host>:8787/v1/models`. A JSON catalog means the URL and key work. An unknown model later still means auth succeeded.

```bash
curl -sS https://<host>/v1/messages \
  -H "Authorization: Bearer $KEY" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-5",
    "max_tokens": 128000,
    "messages": [{"role": "user", "content": "hello"}]
  }'
```

A visible `content` block is the first real call. Sticky turns use `x-session-id`; see [Call the API](./api).

## If it fails

| What you see | Where to look |
| --- | --- |
| `401 missing_api_key` | No header, or the wrong one. Send Bearer or `x-api-key` |
| `401` after a rotate | The old `sk-vm-…` is dead. Issue a new key on [Keys](./console/keys) |
| `403 forbidden` on `/api/panel/*` | A protocol key cannot open the panel. That split is in the [FAQ](../reference/faq) |
| Empty or Claude-only catalog in Codex | The key's group does not include OpenAI models, or there is no OpenAI slot |
| `400` unknown model | The id is not in `GET /v1/models` for this key |
| `502 incomplete_response` | Slot egress or DNS, not the client snippet. [FAQ](../reference/faq) |
| curl to `:8787` fails on Docker Desktop or WSL | Host loopback. [FAQ](../reference/faq) |

The rest of the error body is in [Call the API](./api).

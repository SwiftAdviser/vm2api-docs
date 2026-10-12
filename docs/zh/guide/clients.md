# 客户端接入

把 Claude Code、Codex CLI 或 SDK 指到这台网关。请求契约在 [调用 API](./api)。密钥在 [密钥](./console/keys) 签发。

## 你需要什么

| 东西 | 取值 |
| --- | --- |
| Base URL | 有 TLS 时用 `https://<主机>`，直连监听端口用 `http://<主机>:8787`。不要末尾斜杠 |
| 协议密钥 | [密钥](./console/keys) 页签发的 `sk-vm-…`。明文只显示一次 |

不要把 Master `VM2API_API_KEY` 填进客户端。Master 能改面板。协议密钥只能调 `/v1/*`。

两个头任选其一：

```http
Authorization: Bearer sk-vm-…
x-api-key: sk-vm-…
```

Claude Code 和 Codex 会自己拼路径。Claude Code 填主机。Codex 和 OpenAI SDK 填主机加 `/v1`。

## Claude Code

Claude Code 请求的是 `{ANTHROPIC_BASE_URL}/v1/messages`。这个变量是主机，不要带 `/v1`。

```bash
export ANTHROPIC_BASE_URL=https://<主机>
export ANTHROPIC_AUTH_TOKEN=sk-vm-…
claude
```

没有 TLS、直连监听端口：

```bash
export ANTHROPIC_BASE_URL=http://<主机>:8787
export ANTHROPIC_AUTH_TOKEN=sk-vm-…
```

`ANTHROPIC_AUTH_TOKEN` 会发成 `Authorization: Bearer`，和这台网关一致。`ANTHROPIC_API_KEY` 会发 `x-api-key`，也能用。两个里只用一个。

要让每个项目都生效，写进 `~/.claude/settings.json` 的 `env`：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://<主机>",
    "ANTHROPIC_AUTH_TOKEN": "sk-vm-…"
  }
}
```

这是用户级文件，对你的所有项目生效。不要把 token 写进项目里的 `.claude/settings.json`，那份会进 git。

settings 里的 `env` 会覆盖 shell 里同名变量。在 Claude Code 里跑 `/status`：**Anthropic base URL** 应是这台主机，**Auth token** 应写着 `ANTHROPIC_AUTH_TOKEN`。

Claude Code 需要可调度的 Claude 槽。模型 id 从 `GET /v1/models` 里挑，例如 `claude-sonnet-5`。

## Codex CLI

Codex 走 OpenAI Responses 协议。这台网关的入口是 `POST /v1/responses`。所以 provider 的 `base_url` 必须带 `/v1`。

用户配置在 `~/.codex/config.toml`。加一个自定义 provider。不要占用保留 id：`openai`、`ollama`、`lmstudio`。

```toml
model = "gpt-5.4"
model_provider = "vm2api"

[model_providers.vm2api]
name = "vm2api"
base_url = "https://<主机>/v1"
env_key = "VM2API_KEY"
wire_api = "responses"
```

直连监听端口：

```toml
base_url = "http://<主机>:8787/v1"
```

在启动 `codex` 的那个 shell 里：

```bash
export VM2API_KEY=sk-vm-…
```

`env_key` 是变量名，不是密钥本身。Codex 会把它发成 `Authorization: Bearer`。`wire_api = "responses"` 必填，chat 线已经去掉了。

这把密钥必须能看见 OpenAI 模型（`all` 或 `openai`）。`GET /v1/models` 列出它能调的 id，从里面选。Codex 需要可调度的 OpenAI 槽。

## OpenAI 和 Anthropic SDK

OpenAI SDK 会在 `base_url` 后面拼 `/chat/completions` 和 `/responses`，所以这里要带 `/v1`：

```python
from openai import OpenAI

client = OpenAI(api_key="sk-vm-…", base_url="https://<主机>/v1")
```

Anthropic SDK 自己拼 `/v1/messages`，所以 `base_url` 是主机：

```python
from anthropic import Anthropic

client = Anthropic(api_key="sk-vm-…", base_url="https://<主机>")
```

TypeScript 参数名是 `apiKey` 和 `baseURL`，URL 形状相同。Anthropic SDK 的 `api_key` 发的是 `x-api-key`。想发 `Authorization: Bearer` 就改用 `auth_token`。这台网关两个都收。

## 用 curl 检查

换上主机和密钥：

```bash
curl -sS https://<主机>/v1/models \
  -H "Authorization: Bearer $KEY"
```

直连监听端口用 `http://<主机>:8787/v1/models`。返回 JSON 目录说明地址和密钥是对的。后面即使模型 id 不对，也已经证明鉴权过了。

```bash
curl -sS https://<主机>/v1/messages \
  -H "Authorization: Bearer $KEY" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-5",
    "max_tokens": 128000,
    "messages": [{"role": "user", "content": "hello"}]
  }'
```

看见 `content` 就是第一次真实调用。粘性轮次用 `x-session-id`，见 [调用 API](./api)。

## 失败时

| 现象 | 先看哪 |
| --- | --- |
| `401 missing_api_key` | 没带头，或头不对。发 Bearer 或 `x-api-key` |
| 换新之后仍 401 | 旧的 `sk-vm-…` 已经作废。到 [密钥](./console/keys) 再签一把 |
| 打 `/api/panel/*` 得到 `403 forbidden` | 协议密钥打不开管理台。这个分开写在 [常见问题](../reference/faq) |
| Codex 里目录是空的，或只有 Claude | 密钥分组不含 OpenAI 模型，或者还没有 OpenAI 槽 |
| `400` 未知模型 | 这个 id 不在这把密钥的 `GET /v1/models` 里 |
| `502 incomplete_response` | 槽出口或 DNS，不是这段客户端配置。[常见问题](../reference/faq) |
| Docker Desktop 或 WSL 上 curl `:8787` 失败 | 宿主回环。[常见问题](../reference/faq) |

错误体的其余字段在 [调用 API](./api)。

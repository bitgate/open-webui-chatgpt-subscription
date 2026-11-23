# Open WebUI with a ChatGPT subscription

Point Open WebUI's OpenAI connection at `https://api.plan2api.com/v1` with a `p2a_` key from [Plan2API](https://plan2api.com/integrations/open-webui), and every chat runs on your ChatGPT Plus or Pro plan instead of a pay-per-token OpenAI API key. Plan2API signs in to ChatGPT with a one-time device code on OpenAI's own page, then serves your plan's Codex models (`openai/gpt-6.1-sol`, `openai/gpt-6-astra`, `openai/gpt-6-luna`) to Open WebUI as a standard OpenAI-compatible API.

Connect Claude Pro or Max, Google AI Pro or Kimi Code to the same Plan2API account and `claude/claude-sonnet-5`, `gemini/gemini-3.8-flash-high` and `kimi/k3` land in the same model picker.

**[Get a free `p2a_` key](https://plan2api.com/connect?utm_source=github&utm_medium=readme&utm_campaign=open-webui-chatgpt-subscription)** | [Open WebUI guide on plan2api.com](https://plan2api.com/integrations/open-webui) | [API reference](https://plan2api.com/docs)

## 1. Turn your ChatGPT plan into an API key

1. Open [plan2api.com/connect](https://plan2api.com/connect?utm_source=github&utm_medium=readme&utm_campaign=open-webui-chatgpt-subscription) and sign up with Google.
2. Connect ChatGPT with a one-time device code: open the link Plan2API shows you, sign in to ChatGPT on OpenAI's own page and enter the code. Device code login has to be on in your ChatGPT security settings (or in workspace permissions, if an admin runs your Business workspace).
3. Copy the `p2a_` key you get after connecting. It is shown once; create more on the Keys page (`plan2api.com/dashboard/keys`).

Claude connects the same way: sign in with your Claude account on Anthropic's own page.

## 2a. Fresh install with Docker Compose

```bash
git clone https://github.com/bitgate/open-webui-chatgpt-subscription.git
cd open-webui-chatgpt-subscription
cp .env.example .env
```

Put your `p2a_` key and a secret (`openssl rand -hex 32`) into `.env`, then start it:

```bash
docker compose up -d
```

Open http://localhost:3000, create the first account (it becomes the admin) and pick `openai/gpt-6.1-sol` in the model picker.

What [`docker-compose.yml`](docker-compose.yml) sets:

| Variable | Value | Why |
| --- | --- | --- |
| `OPENAI_API_BASE_URL` | `https://api.plan2api.com/v1` | The OpenAI-compatible endpoint backed by your subscription |
| `OPENAI_API_KEY` | your `p2a_` key, read from `.env` | Sent as `Authorization: Bearer` |
| `TASK_MODEL_EXTERNAL` | `openai/gpt-6-luna` | Title, tag and follow-up generation run on the light Codex model instead of eating your main model's allowance |
| `ENABLE_OLLAMA_API` | `false` | No local Ollama needed |
| `WEBUI_SECRET_KEY` | from `.env` | Signs login sessions |

Prefer plain `docker run`:

```bash
docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  -e WEBUI_SECRET_KEY=your-secret-key \
  -e OPENAI_API_BASE_URL=https://api.plan2api.com/v1 \
  -e OPENAI_API_KEY=p2a_your_key \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

## 2b. Existing Open WebUI: add the connection in the admin panel

`ENABLE_PERSISTENT_CONFIG` defaults to `True`, so on an install that already ran once, connection env vars are ignored in favor of saved settings. Add Plan2API in the UI instead:

1. Open **Admin Panel > Settings > Connections**.
2. Next to **Manage OpenAI API Connections**, click **+** to add a connection.
3. Fill in:

| Field | Value |
| --- | --- |
| Connection Type | `External` |
| URL | `https://api.plan2api.com/v1` |
| API Key | your `p2a_` key |
| API Type | `Chat Completions` |
| Model IDs | the models you connected, for example `openai/gpt-6.1-sol` and `openai/gpt-6-luna` |
| Prefix ID | optional, for example `p2a` |

4. Click **Verify Connection** (it calls `GET /v1/models`), then **Save**.

Keep **API Type** on `Chat Completions`: Plan2API does not serve the Responses API. Fill **Model IDs** with the models of the subscriptions you actually connected; left empty, Open WebUI lists every provider Plan2API supports, and models from providers you haven't connected answer `403 no_credentials`. If you set a Prefix ID, use the prefixed name in `TASK_MODEL_EXTERNAL` too.

## Models

| Model id | Runs on | Context window |
| --- | --- | --- |
| `openai/gpt-6.1-sol` | ChatGPT Plus, Pro or Business (Codex) | listed by `GET /v1/models` |
| `openai/gpt-6-astra` | ChatGPT Plus and higher | listed by `GET /v1/models` |
| `openai/gpt-6.1-terra` | ChatGPT Plus, Pro, Business, Enterprise or Edu | listed by `GET /v1/models` |
| `openai/gpt-6-luna` | ChatGPT Plus and higher, the light and fast one | listed by `GET /v1/models` |
| `claude/claude-sonnet-5` | Claude Pro or Max | 1,000,000 |
| `claude/claude-opus-5-5` | Claude Pro or Max | 1,000,000 |
| `claude/claude-haiku-4-5` | Claude Pro or Max | 200,000 |
| `gemini/gemini-3.8-flash-high` | Google AI Pro or Ultra | 1,048,576 |
| `kimi/k3` | Kimi Code membership | 1,048,576 |

Plan2API reads the Codex model list live from OpenAI, so new GPT models show up without a config change. `codex/<model>` works as an alias for `openai/<model>`.

## Image generation with gpt-image-2

On Plan2API Pro and Scale, Open WebUI generates images on your ChatGPT plan. Open **Admin Panel > Settings > Images**, set **Image Generation Engine** to `Default (Open AI)` and the model to `gpt-image-2`, or uncomment `ENABLE_IMAGE_GENERATION=true` and `IMAGE_GENERATION_MODEL=gpt-image-2` in the compose file. The image base URL and key default to your chat connection. The Free plan answers image requests with `403 plan_limit`.

## Troubleshooting

| What you see | Why | Fix |
| --- | --- | --- |
| **Verify Connection** fails with `401` `invalid_api_key` | Key missing, mistyped or revoked | Paste the `p2a_` key again, or create a new one on the Keys page |
| `404` `not_found` | The URL is missing `/v1`, or **API Type** is set to `Responses` | URL `https://api.plan2api.com/v1`, API Type `Chat Completions` |
| `403` `no_credentials` | The model belongs to a provider you haven't connected | Connect it at plan2api.com, or list only connected models under **Model IDs** |
| `400` `model_not_found` | Typo or missing prefix | Use the exact id, for example `openai/gpt-6.1-sol` |
| `429` `subscription_quota_exhausted` | Your ChatGPT plan's Codex limit is used up for now | Wait for the `Retry-After` time, keep background tasks on `openai/gpt-6-luna`, or connect a second ChatGPT account for automatic failover |
| `429` `rate` or `inflight` | Several users or background tasks at once on the Free plan (1 request per second, 4 at once) | Move to Pro (10 per second, 16 at once) |
| Env var changes do nothing | `ENABLE_PERSISTENT_CONFIG=True` keeps the saved settings | Edit the connection in the admin panel, or set `ENABLE_PERSISTENT_CONFIG=False` |
| Device code sign-in is refused | Device code login is off in your ChatGPT security settings | Turn it on, or ask your Business workspace admin to allow it |

## FAQ

### Can I use my ChatGPT Plus subscription in Open WebUI?

Yes. Connect ChatGPT Plus to Plan2API, then add `https://api.plan2api.com/v1` as an OpenAI connection with your `p2a_` key. Chats run on your plan's Codex allowance instead of OpenAI API billing.

### How do I use ChatGPT in Open WebUI without an API key?

You never create an OpenAI API key. Plan2API signs in to ChatGPT with a device code on OpenAI's own page and gives Open WebUI a `p2a_` key, which goes in the connection's API Key field.

### What URL do I enter in Open WebUI?

`https://api.plan2api.com/v1`, with Connection Type `External` and API Type `Chat Completions`.

### Should the API Type be Chat Completions or Responses?

Chat Completions. Plan2API serves `/v1/chat/completions`, `/v1/models` and the image endpoints; `/v1/responses` returns `404`.

### Do title and tag generation use my subscription?

Yes. Title, tag and follow-up generation are on by default and each one is an extra request. This repo's compose file sends them to `openai/gpt-6-luna`; you can also switch them off under **Admin Panel > Settings > Interface**.

### Can I add Claude to Open WebUI the same way?

Yes. Connect Claude Pro or Max in Plan2API and `claude/claude-sonnet-5` appears under the same connection, next to your GPT models.

### Which ChatGPT plans work?

Plans with Codex access: Plus, Pro and Business. Plan2API serves whatever Codex models your plan includes.

### What does it cost?

Plan2API Free is $0 with one connected subscription and no card. Pro is $12 a month for 3 subscriptions, higher rate limits and image generation; Scale is $29 a month for 25. See [pricing](https://plan2api.com/pricing).

## More guides

- [Open WebUI + Plan2API guide](https://plan2api.com/integrations/open-webui)
- [Use your ChatGPT subscription as an API](https://plan2api.com/chatgpt-subscription-api)
- [Codex usage limits by plan](https://plan2api.com/codex-usage-limits)
- [GPT-6 models](https://plan2api.com/models/gpt-6)
- [Use your Claude subscription as an API](https://plan2api.com/claude-subscription-api)
- [Plan2API API reference](https://plan2api.com/docs)

Same setup, other clients:

- [SillyTavern with a Claude subscription](https://github.com/bitgate/sillytavern-claude-subscription)
- [Janitor AI Claude proxy](https://github.com/bitgate/janitor-ai-claude-proxy)
- [Cline, Roo Code and Kilo Code with a Claude subscription](https://github.com/bitgate/cline-claude-subscription)
- [OpenCode with a Claude Pro or Max subscription](https://github.com/bitgate/opencode-claude-subscription)

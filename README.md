# RegoloProxy

OpenAI-compatible proxy for [Regolo AI](https://regolo.ai) with **multi-account** support. Zero external dependencies.

## Features

- **OpenAI-Compatible API** — Drop-in for `/v1/chat/completions` and `/v1/models`
- **Multi-Account** — Multiple Regolo accounts with independent SSO login, API keys, and per-account usage tracking
- **Combined + Per-Account Dashboard** — AGNES-style glass box slideshow showing combined tokens across all accounts + per-account breakdown
- **Model Search & Add** — Search the Regolo.ai catalog from the dashboard and add models with one click
- **Response Caching** — LRU cache for non-streaming responses
- **Multi-Key Rotation** — Round-robin API key rotation with session pinning
- **Retry Logic** — Automatic retry on model-unavailable errors
- **Dashboard** — Liquid glass UI with account cards, key management, usage slideshow
- **SSO Login** — Log in with Regolo credentials per account to track daily usage
- **Bing Wallpaper** — Daily rotating backgrounds from Bing

## Quick Start

### Prerequisites

- Node.js 18+ or Bun

### 1. Get a Regolo API Key

1. Go to **[dashboard.regolo.ai](https://dashboard.regolo.ai)** and sign up or log in.
2. Navigate to **API Keys** section in the dashboard.
3. Click **Create API Key** — you'll get a key starting with `sk-...`.
4. Copy this key — it will be used by the proxy.

### 2. Configure the Proxy

Edit `.config/config.json` and set your API key:

```json
{
  "API_KEY": "sk-your-regolo-api-key-here"
}
```

Or via environment variable:

```cmd
set REGOLO_API_KEY=sk-your-regolo-api-key-here
```

### 3. Start the Proxy

```bash
node proxy.js
```

Or on Windows, double-click `start.cmd`.

### 4. Add Models

Open the dashboard at **http://localhost:8082**:

1. Use the **Model Search** section to browse the Regolo catalog.
2. Type a query (e.g. `llama`, `qwen`, `deepseek`) or filter by model family.
3. Click **Add** on any model to enable it.
4. Enabled models appear in the **Enabled Models** list and are immediately available via the API.

### 5. Use with Any OpenAI Client

```javascript
import OpenAI from 'openai';

const client = new OpenAI({
  apiKey: 'sk-your-regolo-api-key',
  baseURL: 'http://localhost:8082/v1'
});

const response = await client.chat.completions.create({
  model: 'Llama-3.3-70B-Instruct', // must be in your enabled models list
  messages: [{ role: 'user', content: 'Hello!' }]
});
```

## Dashboard

Open **http://localhost:8082** in your browser.

### Model Management
- **Model Search** — Search the Regolo catalog by name or family (Llama, Qwen, DeepSeek, Mistral, Gemma, etc.). Results show model ID and whether already added.
- **Enabled Models** — View all enabled models. Click a model name to edit its display name inline. Click **X** to remove it.

### API Key Management
- Set your **Regolo API key** (`sk-...`) in the API Key section.
- Manage multiple keys for rotation via the **Key Manager** modal (add, edit, delete).
- Enable **Proxy API Keys** to restrict access to the proxy itself (auth required on requests).

### SSO Login + Multi-Account (Usage Tracking)

The proxy supports **multiple Regolo accounts**. Configure them in `.config/config.json`:
```json
"ACCOUNTS": [
  {
    "email": "user1@example.com",
    "password": "***",
    "key": "sk-...",
    "accessToken": "",
    "refreshToken": ""
  },
  {
    "email": "user2@example.com",
    "password": "***",
    "key": "sk-...",
    "accessToken": "",
    "refreshToken": ""
  }
]
```

**What you see in the dashboard:**
- **Usage Slideshow** — Glass box with combined slide (total across all accounts) + per-account slides with dot navigation
  - **Daily tokens** from Regolo API's `user_info.spend` (at $0.55/M rate) against 20M limit
  - **Lifetime tokens** from each account's key spend (at $0.15/M rate)
- **Account Cards** — One card per account with online status, usage % bar, and logout
- **Auto-refresh** — Every 30 minutes via fresh SSO login per account

### Other Dashboard Features
- **Response Cache** — View cache hits/misses/size, clear cache
- **Test Chat** — Inline chat interface to test models directly from the dashboard
- **Proxy Configuration** — View current runtime configuration
- **SS Mode** — Toggle token blur (privacy mode)
- **Bing Wallpaper** — Toggle daily Bing background images
- **Quick Actions** — Validate API key, health check, platform login, save config

## Configuration

`.config/config.json` supports:

| Field | Description | Default |
|---|---|---|
| `LISTEN_ADDR` | Proxy listen address | `127.0.0.1:8082` |
| `UPSTREAM_BASE_URL` | Regolo API URL | `https://api.regolo.ai/v1` |
| `API_KEY` | Regolo API key (`sk_*`) | — |
| `REQUEST_TIMEOUT` | Upstream request timeout | `15m` |
| `CACHE_TTL` | Response cache TTL | `60s` |
| `CACHE_MAX_SIZE` | Max cached responses | `100` |
| `CACHE_ENABLED` | Enable/disable cache | `true` |
| `ENABLED_MODELS` | Array of model IDs to expose | `[]` |
| `MODEL_DISPLAY_NAMES` | Custom display names per model | `{}` |
| `KEYS` | Array of `{name, key}` objects for rotation | `[{name, key}]` |
| `API_KEYS` | Array of allowed proxy API keys (auth) | `[]` |
| `ACCOUNTS` | Array of `{email, password, key, accessToken, refreshToken, lastLogin}` | `[]` |

### Environment Variables

All config fields can be overridden via environment variables:

```cmd
set REGOLO_API_KEY=sk-...
set LISTEN_ADDR=0.0.0.0:8082
set REQUEST_TIMEOUT=30m
set CACHE_TTL=120s
set CACHE_MAX_SIZE=200
set CACHE_ENABLED=false
set API_KEYS=key1,key2
```

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/healthz` | Proxy health check |
| `GET` | `/v1/models` | OpenAI-format model list |
| `POST` | `/v1/chat/completions` | OpenAI-format chat completions |
| `GET` | `/api/config` | Get proxy configuration |
| `POST` | `/api/config` | Update proxy configuration |
| `GET` | `/api/validate` | Validate API key |
| `GET` | `/api/models` | List enabled models |
| `GET` | `/api/models/search?q=...` | Search Regolo.ai model catalog |
| `GET` | `/api/models/families` | Available model families |
| `POST` | `/api/models/add` | Add models to enabled list |
| `POST` | `/api/models/remove` | Remove models from enabled list |
| `GET` | `/api/keys` | List API keys |
| `POST` | `/api/keys` | Add/update/delete API keys |
| `GET` | `/api/cache` | Cache stats |
| `DELETE` | `/api/cache` | Clear cache |
| `POST` | `/api/regolo/login` | SSO login with Regolo credentials |
| `GET` | `/api/regolo/user` | Regolo login status |
| `GET` | `/api/regolo/usage` | Combined + per-account daily + lifetime token usage |
| `GET` | `/api/regolo/accounts-usage` | Raw per-account usage cache |
| `POST` | `/api/regolo/logout` | Logout from Regolo SSO |
| `POST` | `/api/regolo/dashboard-cookie` | Save dashboard.regolo.ai cookie for spend data |
| `GET` | `/api/regolo/dashboard-data` | Fetch spend data via dashboard cookie |
| `GET` | `/api/bg` | Daily Bing wallpaper |

## Opencode Integration

The proxy automatically configures itself as an Opencode provider. After starting the proxy, you can select `regolo` as your provider in Opencode — models will be populated from your enabled models list with any custom display names you've set.

## Usage Tips

- **Find available models**: Use the dashboard search or `curl "http://localhost:8082/api/models/search?q=llama"`
- **Add a model via API**: `curl -X POST http://localhost:8082/api/models/add -H "Content-Type: application/json" -d '{"models":["Llama-3.3-70B-Instruct"]}'`
- **Multi-account monitoring**: Configure `ACCOUNTS[]` in config for per-account SSO login & usage tracking. The dashboard shows a combined + paged slideshow with daily tokens (from `user_info.spend`) and lifetime tokens (from key spend)
- **Multiple API keys**: Add keys via the Key Manager for round-robin rotation with session pinning. Keys are auto-populated from ACCOUNTS
- **Secure the proxy**: Set `API_KEYS` in config — all requests must then include `x-api-key` or `Authorization: Bearer <key>`
- **Check per-account usage**: `curl http://localhost:8082/api/regolo/accounts-usage`

## License

MIT

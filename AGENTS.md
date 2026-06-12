# RegoloProxy — Developer Guide

## Project Structure

```
REGOLO-PROXY/
├── proxy.js              # Main proxy implementation + request router
├── dashboard.html        # Liquid glass dashboard with model search, key management, usage tracking
├── .config/
│   ├── config.json       # Runtime configuration (keys + ACCOUNTS)
│   └── usage.json        # Daily token usage tracking (auto-created)
├── .cache/               # Cached assets (auto-created)
│   └── wallpaper.jpg     # Cached Bing wallpaper
├── package.json          # Project metadata (MIT, no deps)
├── start.cmd             # Auto-detect launcher (Bun preferred, Node fallback)
├── start-node.cmd        # Node.js-only launcher
├── skills.md             # Opencode provider configuration reference
├── README.md             # User documentation
└── AGENTS.md             # This file
```

## Key Components

### 1. Constants & Config (proxy.js lines 1-200)

- `REGOLO_API_BASE` — `https://api.regolo.ai/v1`
- `API_KEY_ENV_VAR` — `REGOLO_API_KEY`
- `REGOLO_USAGE_LIMIT` — 20M daily token limit per account
- `REGOLO_AVG_COST_PER_TOKEN` — $0.15/M for key spend → total lifetime tokens
- `REGOLO_AVG_COST_PER_TOKEN_DAILY` — $0.55/M for user_info spend → daily dashboard tokens
- `loadConfig()` — Loads `.config/config.json`, builds `keys[]` from `API_KEY` + `ACCOUNTS[].key`
- `saveConfig()` — Writes config back to `.config/config.json`
- `parseDuration()` — Parses duration strings like `15m`, `6h`, `30s`

### 2. Multi-Account (ACCOUNTS config, proxy.js lines 132-162, 279-358)

Config format:
```json
"ACCOUNTS": [
  {
    "email": "user@example.com",
    "password": "***",
    "key": "sk-...",
    "accessToken": "sso-jwt...",
    "refreshToken": "...",
    "lastLogin": "ISO-date"
  }
]
```

- Keys auto-generated from `ACCOUNTS[].key` if not already in `keys[]`
- Each account has its own SSO credentials + API key
- `loadConfig()` migrates old `REGOLO_USERNAME/PASSWORD` into `ACCOUNTS` array on startup

### 3. Usage Tracking (proxy.js lines 279-360, 1997-2027)

- `fetchAccountUsage(account)` — Calls `/user/info` with account's API key
- `refreshAllAccountUsage()` — Iterates all accounts, re-logins via SSO, fetches usage
- `accountUsageCache` — Per-account cached usage, refreshed every 30 min
- `GET /api/regolo/usage` — Returns `{ combined: {used,limit,percent}, accounts: [...] }`
- `GET /api/regolo/accounts-usage` — Returns raw `{ accounts: [...] }` cache
- Daily tokens from `user_info.spend` at $0.55/M rate
- Total lifetime from `keys[0].spend` at $0.15/M rate

### 4. UpstreamClient (proxy.js lines 400-452)

- `headers(stream)` — Returns Bearer token + Content-Type/Accept headers
- `getUserInfo()` — `GET /models` with 10s timeout to validate API key
- `chatCompletions(body)` — `POST /chat/completions` (streaming-aware)

### 5. Model Management (proxy.js lines 455-488)

- `searchRegoloModels(query, filters)` — Searches Regolo catalog by fetching `/v1/models` and filtering locally
- `GET /api/models/search?q=...&family=...` — Search Regolo catalog via proxy
- `POST /api/models/add` — Add models to enabled list
- `POST /api/models/remove` — Remove models from enabled list

### 6. HTTP Handlers (proxy.js lines 1087-1784)

- `authorized(req)` — Checks `x-api-key` or `Authorization: Bearer` against `config.apiKeys`
- `handleHealthz(req, res)` — Returns uptime, token_state (per-key status), models count
- `handleModels(req, res)` — OpenAI-format model list
- `handleChatCompletions(req, res)` — Parses body, calls `proxyChatRequest`
- `proxyChatRequest(res, payload, model, ...)` — Clone payload, normalize tools, forward to upstream

### 7. Request Router (proxy.js lines 1430-1784)

Routes by pathname:
- `/` or `/dashboard` → Serve `dashboard.html`
- `/api/config` (GET/POST) → Config read/write
- `/api/validate` (GET) → Validate API key
- `/api/models` (GET) → Model list
- `/api/models/search` (GET) → Search Regolo catalog
- `/api/models/add` (POST) → Add models to enabled list
- `/api/models/remove` (POST) → Remove models from enabled list
- `/api/bg` (GET) → Bing wallpaper via peapix.com
- `/api/keys` (GET/POST) → Multi-key CRUD
- `/api/cache` (GET/DELETE) → Response cache stats/clear
- `/api/models/families` (GET) → Available model families
- `/api/regolo/login` (POST) → SSO login (saves to ACCOUNTS)
- `/api/regolo/user` (GET) → Login status + accountCount
- `/api/regolo/usage` (GET) → Combined + per-account daily + lifetime tokens
- `/api/regolo/accounts-usage` (GET) → Raw per-account usage cache
- `/api/regolo/logout` (POST) → Logout
- `/api/regolo/dashboard-cookie` (POST) → Save dashboard.regolo.ai cookie
- `/api/regolo/dashboard-data` (GET) → Fetch spend data via cookie
- `/healthz` → Health check + token_state
- `/v1/models` → OpenAI models
- `/v1/chat/completions` → OpenAI chat

### 8. Opencode Config (proxy.js lines 1787-1833)

- `setupOpencodeConfig()` — Writes provider config with display names from `modelDisplayNames`
- Creates `openconfig.b4regolo.json` backup before first edit
- Provider key: `regolo`, using `@ai-sdk/openai-compatible`

### 9. Dashboard (dashboard.html)

- **Liquid Glass Engine** — Canvas-generated displacement maps with refraction profiles
- **Usage Slideshow** — AGNES-style combined + paged per-account glass box with dot navigation
  - Slide 0: Combined (daily tokens across all accounts)
  - Slides 1..N: Per-account breakdown (daily tokens + lifetime from key spend)
- **Account Cards** — One card per account with status, % bar, and logout
- **Key Cards** — One card per key with masked token, edit/delete
- **Key Manager Modal** — Account(s): and Key(s): sections with per-account cards
- **Model Search UI** — Search Regolo catalog with family filter, rich cards
- **SS Mode** — `token-blurred` CSS class (blur on hover)
- **Test Chat** — Inline chat interface to test models directly
- **Auto-refresh** — Health check every 15s, usage every 5s
- **Collapsible Sections** — Available Models, API Key, Quick Actions, Environment, Proxy Configuration
- **Bing Wallpaper** — Daily rotating backgrounds toggle

## Request Lifecycle

```
Client request arrives
    ↓
Check API key authorization (if apiKeys configured)
    ↓
Route by pathname → handler
    ↓
Parse + validate request body
    ↓
Detect session signal (fingerprint first user msg)
    ↓
  ├─ Known fingerprint → pin to same token (sticky)
  └─ New fingerprint → rotate to next key, store mapping
    ↓
Clone payload, normalize tool schemas
    ↓
Forward to upstream api.regolo.ai/v1
    ↓
Success → pipe/buffer response (stream or JSON), log done
Error   → parse upstream error, return formatted response
```

## Startup Sequence

1. `loadConfig()` — Load `.config/config.json` + env var overrides, build `keys[]` from `ACCOUNTS[]`
2. `UpstreamClient` — Initialize HTTP client
3. `validateApiKey()` — Verify via `/models`
4. Migrate old `REGOLO_USERNAME/PASSWORD` into `ACCOUNTS` if not present
5. `syncUsageFromApi()` — Fetch all-time tokens + refresh multi-account usage
6. `setInterval(syncUsageFromApi, 30min)` — Periodic re-sync (fetches usage per-account)
7. `setupOpencodeConfig()` — Write/update opencode provider config
8. `http.createServer(handleRequest).listen(port)` — Start HTTP server on port 8082

## Testing

```bash
# Syntax check
node --check proxy.js

# Start proxy
node proxy.js

# Test endpoints
curl http://localhost:8082/healthz
curl http://localhost:8082/v1/models
curl http://localhost:8082/api/models
curl http://localhost:8082/api/regolo/accounts-usage

# Test model search
curl "http://localhost:8082/api/models/search?q=llama"

```

## Dependencies

No external npm dependencies — uses Node.js built-in modules only: `fs`, `path`, `os`, `http`, `https`, `url`, `crypto`, `child_process`.

## Data Storage

- `.config/config.json` — Proxy configuration (API keys, enabled models, display names, ACCOUNTS)
- `.config/usage.json` — Daily token usage tracking (auto-created)
- `.cache/wallpaper.jpg` — Cached Bing wallpaper

## Response Caching

Configurable LRU cache for non-streaming LLM responses.

- **Key**: MD5 hash of `(model + stream_flag + system + messages + tools)`
- **TTL**: Configurable via `CACHE_TTL` (env or config, default `60s`)
- **Max size**: Configurable via `CACHE_MAX_SIZE` (env or config, default 100 entries)
- **Disable**: Set `CACHE_ENABLED=false`
- **Stats**: `GET /api/cache` — hits, misses, evictions, size
- **Clear**: `DELETE /api/cache`

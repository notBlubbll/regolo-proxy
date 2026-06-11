# RegoloProxy — Developer Guide

## Project Structure

```
REGOLO-PROXY/
├── proxy.js              # Main proxy implementation + request router
├── dashboard.html        # Liquid glass dashboard with model search, key management, usage tracking
├── .config/
│   ├── config.json       # Runtime configuration
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

### 1. Constants & Config (proxy.js lines 1-170)

- `REGOLO_API_BASE` — `https://api.regolo.ai/v1`
- `API_KEY_ENV_VAR` — `REGOLO_API_KEY`
- `loadConfig()` — Loads `.config/config.json` with env var overrides
- `saveConfig()` — Writes config back to `.config/config.json`
- `parseDuration()` — Parses duration strings like `15m`, `6h`, `30s`

### 2. UpstreamClient (proxy.js lines 170-230)

- `headers(stream)` — Returns Bearer token + Content-Type/Accept headers
- `getUserInfo()` — `GET /models` with 10s timeout to validate API key
- `chatCompletions(body)` — `POST /chat/completions` (streaming-aware)

### 3. Model Management (proxy.js lines 232-400)

- `searchRegoloModels(query, filters)` — Searches Regolo catalog by fetching `/v1/models` and filtering locally
- `GET /api/models/search?q=...&family=...` — Search Regolo catalog via proxy
- `POST /api/models/add` — Add models to enabled list
- `POST /api/models/remove` — Remove models from enabled list
- `modelDisplayNames` — Custom display names persisted to config and opencode

### 4. HTTP Handlers (proxy.js lines 500-800)

- `authorized(req)` — Checks `x-api-key` header or `Authorization: Bearer` against `config.apiKeys`
- `readBody(req)` — Buffers incoming request body to string
- `writeJSON(res, statusCode, payload)` — JSON response with error-safe write
- `handleHealthz(req, res)` — Returns uptime, API key validity, models count, runtime info
- `handleModels(req, res)` — OpenAI-format model list
- `handleChatCompletions(req, res)` — Parses body, calls `proxyChatRequest`
- `proxyChatRequest(res, payload, model, ...)` — Core proxy: clone payload, normalize tools, forward to upstream

### 5. Request Router (proxy.js lines 1550-1820)

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
- `/api/regolo/login` (POST) → SSO login with Regolo credentials
- `/api/regolo/user` (GET) → Regolo login status
- `/api/regolo/usage` (GET) → Daily token usage + limit + countdown
- `/api/regolo/logout` (POST) → Logout from Regolo SSO
- `/api/regolo/dashboard-cookie` (POST) → Save dashboard.regolo.ai cookie
- `/api/regolo/dashboard-data` (GET) → Fetch spend data via dashboard cookie
- `/healthz` → Health check
- `/v1/models` → OpenAI models
- `/v1/chat/completions` → OpenAI chat

### 6. Opencode Config (proxy.js lines 1830-1880)

- `setupOpencodeConfig()` — Writes provider config with display names from `modelDisplayNames`
- Creates `openconfig.b4regolo.json` backup before first edit
- Provider key: `regolo`, using `@ai-sdk/openai-compatible`

### 7. Dashboard (dashboard.html)

- **Liquid Glass Engine** — Canvas-generated displacement maps with refraction profiles
- **Model Search UI** — Search Regolo catalog with family filter, rich cards with model ID
- **Enabled Models** — Click to edit display name (inline input), remove with X
- **Key Manager Modal** — Inline add/edit/delete for multiple API keys
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

1. `loadConfig()` — Load `.config/config.json` + env var overrides
2. `UpstreamClient` — Initialize HTTP client
3. `validateApiKey()` — Verify via `/models` (optional, warns if missing)
4. `syncUsageFromApi()` — Initialize daily token count from Regolo API spend
5. `setInterval(syncUsageFromApi, 30min)` — Periodic usage re-sync
6. `setupOpencodeConfig()` — Write/update opencode provider config
7. `http.createServer(handleRequest).listen(port)` — Start HTTP server on port 8082

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

# Test model search
curl "http://localhost:8082/api/models/search?q=llama"

```

## Dependencies

No external npm dependencies — uses Node.js built-in modules only: `fs`, `path`, `os`, `http`, `https`, `url`, `crypto`, `child_process`.

## Data Storage

- `.config/config.json` — Proxy configuration (API keys, enabled models, display names)
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

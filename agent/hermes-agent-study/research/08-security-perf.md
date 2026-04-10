# Security & Performance Analysis: Hermes Agent

## 1. Secrets Management

### Approach: Environment Variables + `.env` File

**Pattern:** API keys stored in `~/.hermes/.env`, loaded via `python-dotenv`.

**Evidence:**
- `.env.example` documents all supported API keys (lines 1-322)
- Secrets stored in `~/.hermes/.env` with owner-only permissions (0600)
- `hermes_cli/config.py`:
  - `_secure_file()` sets 0600 on `.env` (line 106)
  - `_secure_dir()` sets 0700 on `~/.hermes/` (line 97)
  - `save_env_value()` atomic writes via temp file + `os.replace()` (lines 1549-1561)

**Security Controls:**
- Secrets never written to `config.yaml` — only `.env`
- `.env` excluded from version control (`.gitignore`)
- Sanitization of corrupted `.env` entries (concatenated keys, stale placeholders) — `sanitize_env_file()` (lines 1467-1509)
- Environment variable expansion in config: `${VAR}` syntax supported (`_expand_env_vars()`, lines 1209-1226)

**Supported Providers:**
| Provider | Env Var |
|----------|---------|
| OpenRouter | `OPENROUTER_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` / `ANTHROPIC_TOKEN` |
| OpenAI | `VOICE_TOOLS_OPENAI_KEY` |
| Firecrawl | `FIRECRAWL_API_KEY` |
| Parallel | `PARALLEL_API_KEY` |
| Tavily | `TAVILY_API_KEY` |
| Browserbase | `BROWSERBASE_API_KEY` |
| FAL.ai | `FAL_KEY` |

**Files:**
- `hermes_cli/config.py` — env loading, secure file ops
- `.env.example` — documented secrets template
- `hermes_cli/env_loader.py` — `.env` loading with override precedence

---

## 2. Authentication / Authorization

### Multiple Messaging Platform Auth

**Pattern:** Platform-specific token-based authentication via environment variables.

**Evidence:**
- `gateway/config.py` — `PlatformConfig` dataclass with `token` and `api_key` fields (lines 133-177)
- Token validation on startup — warns if enabled platform has empty token (lines 564-581)
- Supported platforms with auth:
  | Platform | Auth Method |
  |----------|-------------|
  | Telegram | Bot Token (`TELEGRAM_BOT_TOKEN`) |
  | Discord | Bot Token (`DISCORD_BOT_TOKEN`) |
  | Slack | Bot Token + App Token |
  | WhatsApp | Baileys bridge (pairing-based) |
  | Signal | HTTP Gateway + Account |
  | Mattermost | Personal Access Token |
  | Matrix | Access Token / Password |
  | Email | IMAP/SMTP credentials |
  | SMS | Twilio Account SID + Auth Token |

### User Allowlisting

**Pattern:** Per-platform user ID allowlists.

**Evidence:**
- `GATEWAY_ALLOW_ALL_USERS=false` — deny-by-default gate (`.env.example` line 238)
- Platform-specific allowlists:
  - `TELEGRAM_ALLOWED_USERS` — comma-separated Telegram user IDs
  - `DISCORD_ALLOWED_USERS` — comma-separated Discord user IDs
  - `SLACK_ALLOWED_USERS`
  - `SIGNAL_ALLOWED_USERS` / `SIGNAL_GROUP_ALLOWED_USERS`
  - `EMAIL_ALLOWED_USERS`
  - `WHATSAPP_ALLOWED_USERS`
  - `MATTERMOST_ALLOWED_USERS`
  - `MATRIX_ALLOWED_USERS`
- `gateway/platforms/base.py` — `allowed_user_ids` checked before processing messages

### OAuth Support

**Pattern:** Nous Portal OAuth + OpenAI Codex OAuth.

**Evidence:**
- `hermes_cli/auth.py` — OAuth token management
- `agent/auxiliary_client.py`:
  - `_read_nous_auth()` reads `~/.hermes/auth.json` for Nous Portal tokens (lines 436-455)
  - `_read_codex_access_token()` reads Codex OAuth tokens with JWT expiry check (lines 468-495)
  - Nous auth validated for `active_provider == "nous"` with `agent_key` or `access_token`

### SSH Remote Execution Security

**Pattern:** Agent code stays local, commands execute remotely via SSH.

**Evidence (`.env.example` lines 119-133):**
- `TERMINAL_SSH_HOST`, `TERMINAL_SSH_USER`, `TERMINAL_SSH_KEY`
- Agent cannot read `.env` file on remote — API keys protected
- Agent cannot modify its own code — isolated sandbox

---

## 3. Input Validation

### Pydantic Usage

**Pattern:** Pydantic v2 for structured data validation.

**Evidence:**
- `pyproject.toml` — `pydantic >= 2.12.5, < 3` (line 33)
- `gateway/config.py`:
  - `HomeChannel`, `SessionResetPolicy`, `PlatformConfig`, `GatewayConfig` dataclasses (lines 62-249)
  - `_coerce_bool()` — safe bool coercion from strings (lines 24-32)
  - `_normalize_unauthorized_dm_behavior()` — enum validation (lines 35-41)
- `hermes_cli/config.py`:
  - `DEFAULT_CONFIG` — validated structure for all config fields
  - `load_config()` — merges user config with defaults, validates structure (lines 1246-1270)

### Pre-Execution Security Scanning (Tirith)

**Pattern:** Binary scanning for command-level threats before execution.

**Evidence (`tools/tirith_security.py`):**
- Scans for: homograph URLs, pipe-to-interpreter, terminal injection
- Exit codes: 0=allow, 1=block, 2=warn
- Auto-install from GitHub releases with SHA-256 verification (lines 281-371)
- Cosign provenance verification when available (lines 216-253)
- Configurable fail-open/fail-closed: `tirith_fail_open` (default: True)
- Background thread installation so startup never blocks (lines 478-589)
- 24-hour disk cache of install failures to avoid repeated network attempts

**Config (from `hermes_cli/config.py` lines 396-407):**
```yaml
security:
  redact_secrets: True
  tirith_enabled: True
  tirith_path: "tirith"
  tirith_timeout: 5
  tirith_fail_open: True
```

### URL Safety

**Pattern:** Allowlist-based URL access control.

**Evidence (`tools/url_safety.py`, `tools/website_policy.py`):**
- `is_safe_url()` — checks against blocklists
- `check_website_access()` — policy enforcement for web tools
- Optional blocklist configuration:
```yaml
security:
  website_blocklist:
    enabled: False
    domains: []
    shared_files: []
```

### Supply Chain Security (CI)

**Evidence (`.github/workflows/supply-chain-audit.yml`):**
Scans PRs for:
- `.pth` files (auto-execute on Python startup)
- base64 decode + exec/eval patterns
- subprocess with encoded commands
- marshal/pickle/compile usage

---

## 4. Performance Optimization Patterns

### Context Compression

**Pattern:** Automatic summarization of conversation history when context approaches limit.

**Evidence (`hermes_cli/config.py` lines 189-197):**
```yaml
compression:
  enabled: True
  threshold: 0.50        # compress at 50% of context limit
  target_ratio: 0.20    # preserve 20% of threshold as recent tail
  protect_last_n: 20    # minimum recent messages uncompressed
  summary_model: ""      # empty = use main model
```

**Implementation:** `trajectory_compressor.py` — middleware turn compression using auxiliary LLM client.

### Smart Model Routing

**Pattern:** Route simple queries to cheap/fast model, complex to strong model.

**Evidence (`hermes_cli/config.py` lines 198-203):**
```yaml
smart_model_routing:
  enabled: False
  max_simple_chars: 160
  max_simple_words: 28
  cheap_model: {}
```

**Implementation:** `agent/smart_model_routing.py` — char/word count heuristics.

### Client Caching

**Pattern:** Thread-safe client caching with loop-aware async management.

**Evidence (`agent/auxiliary_client.py` lines 1202-1316):**
- `_client_cache: Dict[tuple, tuple>` — keyed by (provider, async_mode, base_url, api_key, loop_id)
- Event loop identity in cache key prevents cross-loop reuse (lines 1274-1282)
- Stale async clients detected and discarded when loop is closed (lines 1288-1295)
- `shutdown_cached_clients()` — clean teardown before event loop close (lines 1227-1251)

### Lazy Initialization

**Pattern:** Clients created on first use, not at import time.

**Evidence:**
- `_firecrawl_client = None` — initialized in `_get_firecrawl_client()` (`tools/web_tools.py` line 95)
- Multiple provider detection in `agent/auxiliary_client.py` — `_resolve_auto()` tries providers in order

---

## 5. Rate Limiting & Resilience

### Retry Logic (Tenacity)

**Pattern:** Configurable retry with exponential backoff via `tenacity`.

**Evidence (`pyproject.toml` line 39):** `tenacity >= 9.1.4, < 10`

**Usage locations:**
- `tools/skills_hub.py` — GitHub API retries
- `tools/mcp_tool.py` — MCP server connection retries
- `tools/mixture_of_agents_tool.py` — MoA aggregation retries
- `gateway/run.py` — platform adapter reconnection
- `run_agent.py` — LLM API transient error retries

### Timeout Configuration

**Pattern:** Per-operation timeouts with sensible defaults.

**Evidence:**
| Operation | Default | Config |
|-----------|---------|--------|
| LLM API call | 30s | `timeout` param in `call_llm()` |
| Auxiliary tasks | 30s | `timeout` param |
| Terminal command | 180s | `terminal.timeout` in config.yaml |
| Browser command | 30s | `browser.command_timeout` |
| Browser session | 300s | `BROWSER_SESSION_TIMEOUT` |
| Browser inactivity | 120s | `BROWSER_INACTIVITY_TIMEOUT` |
| Tirith scan | 5s | `tirith_timeout` |
| Terminal lifetime | 300s | `TERMINAL_LIFETIME_SECONDS` |

**Files:**
- `agent/auxiliary_client.py` lines 1404-1413 — `timeout` parameter in `_build_call_kwargs()`
- `hermes_cli/config.py` lines 143-156 — terminal backend resource limits
- `tools/tirith_security.py` lines 615-626 — subprocess timeout

### Fallback Provider Chain

**Pattern:** Automatic failover when primary provider is unavailable.

**Evidence (`agent/auxiliary_client.py` lines 734-744):**
```python
def _resolve_auto():
    # Full auto-detection chain: OpenRouter → Nous → custom → Codex → API-key → None
    for try_fn in (_try_openrouter, _try_nous, _try_custom_endpoint,
                   _try_codex, _resolve_api_key_provider):
        client, model = try_fn()
        if client is not None:
            return client, model
```

**Triggers:**
- Rate limits (HTTP 429)
- Overload (HTTP 529)
- Service errors (HTTP 503)
- Connection failures

### Graceful Degradation

**Pattern:** Fail-open for optional security features, fail-closed for required.

**Evidence:**
- Tirith: `tirith_fail_open=True` — if scanner unavailable, allow command
- SSL certs: `_ensure_ssl_certs()` auto-detects system certs, falls back to certifi bundle (lines 36-72 of `gateway/run.py`)
- Vision backend: Falls back through provider list — OpenRouter → Nous → Codex → Anthropic → custom

---

## 6. Resource Management

### Container Isolation

**Pattern:** Code execution in containers with resource limits.

**Evidence (`hermes_cli/config.py` lines 156-164):**
```yaml
terminal:
  container_cpu: 1
  container_memory: 5120    # MB (5GB)
  container_disk: 51200      # MB (50GB)
  container_persistent: True # Persist filesystem across sessions
```

**Backends:** Docker, Singularity, Modal, Daytona, SSH

### Session Lifecycle

**Pattern:** Automatic session reset policies prevent unbounded memory growth.

**Evidence (`gateway/config.py` lines 90-130):**
```python
@dataclass
class SessionResetPolicy:
    mode: str = "both"        # "daily", "idle", "both", or "none"
    at_hour: int = 4          # Hour for daily reset (0-23)
    idle_minutes: int = 1440  # 24 hours
    notify: bool = True
```

### Process Registry

**Pattern:** Track and clean up child processes.

**Evidence (`tools/process_registry.py`):**
- Registers all spawned processes
- Cleanup on exit / timeout
- Prevents zombie processes

---

## Summary Table

| Category | Pattern | Key Files |
|----------|---------|-----------|
| Secrets | `.env` + env vars, 0600 perms | `hermes_cli/config.py` |
| Auth | Token-based per platform + OAuth | `gateway/config.py`, `hermes_cli/auth.py` |
| Allowlisting | Per-platform user ID lists | `gateway/platforms/base.py` |
| Input Validation | Pydantic dataclasses | `gateway/config.py` |
| Security Scanning | Tirith pre-exec binary scan | `tools/tirith_security.py` |
| URL Safety | Blocklist + policy checks | `tools/url_safety.py`, `tools/website_policy.py` |
| Compression | Auto-summarize at 50% threshold | `trajectory_compressor.py` |
| Caching | Thread-safe client cache with loop awareness | `agent/auxiliary_client.py` |
| Retries | Tenacity with exponential backoff | Multiple tools |
| Timeouts | Per-operation configurable | `agent/auxiliary_client.py`, `hermes_cli/config.py` |
| Failover | Multi-provider auto-detection | `agent/auxiliary_client.py` |
| Container Limits | CPU, memory, disk quotas | `hermes_cli/config.py` |
| Session Reset | Daily/idle/both policies | `gateway/config.py` |

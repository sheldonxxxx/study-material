# Security: Hermes Agent

## Secrets Management

### Approach: Environment Variables + `.env` File

API keys stored in `~/.hermes/.env`, loaded via `python-dotenv`.

**Security Controls:**
- Secrets stored in `~/.hermes/.env` with owner-only permissions (0600)
- `_secure_file()` sets 0600 on `.env` (`hermes_cli/config.py` line 106)
- `_secure_dir()` sets 0700 on `~/.hermes/` (`hermes_cli/config.py` line 97)
- `save_env_value()` atomic writes via temp file + `os.replace()` (lines 1549-1561)
- Secrets never written to `config.yaml` — only `.env`
- `.env` excluded from version control (`.gitignore`)
- Sanitization of corrupted `.env` entries — `sanitize_env_file()` (lines 1467-1509)
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

---

## Authentication / Authorization

### Multiple Messaging Platform Auth

Platform-specific token-based authentication via environment variables.

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

Per-platform user ID allowlists with deny-by-default.

**Pattern:** `GATEWAY_ALLOW_ALL_USERS=false` — deny-by-default gate

Platform-specific allowlists:
- `TELEGRAM_ALLOWED_USERS` — comma-separated Telegram user IDs
- `DISCORD_ALLOWED_USERS` — comma-separated Discord user IDs
- `SLACK_ALLOWED_USERS`
- `SIGNAL_ALLOWED_USERS` / `SIGNAL_GROUP_ALLOWED_USERS`
- `EMAIL_ALLOWED_USERS`
- `WHATSAPP_ALLOWED_USERS`
- `MATTERMOST_ALLOWED_USERS`
- `MATRIX_ALLOWED_USERS`

Implementation: `gateway/platforms/base.py` — `allowed_user_ids` checked before processing messages.

### OAuth Support

- **Nous Portal OAuth** — reads `~/.hermes/auth.json` for tokens (`agent/auxiliary_client.py` lines 436-455)
- **OpenAI Codex OAuth** — reads Codex OAuth tokens with JWT expiry check (lines 468-495)

### SSH Remote Execution Security

Agent code stays local, commands execute remotely via SSH.

- `TERMINAL_SSH_HOST`, `TERMINAL_SSH_USER`, `TERMINAL_SSH_KEY`
- Agent cannot read `.env` file on remote — API keys protected
- Agent cannot modify its own code — isolated sandbox

---

## Input Validation

### Pydantic Usage

Pydantic v2 for structured data validation in configuration.

**Evidence:**
- `gateway/config.py`:
  - `HomeChannel`, `SessionResetPolicy`, `PlatformConfig`, `GatewayConfig` dataclasses
  - `_coerce_bool()` — safe bool coercion from strings
  - `_normalize_unauthorized_dm_behavior()` — enum validation
- `hermes_cli/config.py`:
  - `DEFAULT_CONFIG` — validated structure for all config fields
  - `load_config()` — merges user config with defaults, validates structure

### Pre-Execution Security Scanning (Tirith)

Binary scanning for command-level threats before execution.

**Features (`tools/tirith_security.py`):**
- Scans for: homograph URLs, pipe-to-interpreter, terminal injection
- Exit codes: 0=allow, 1=block, 2=warn
- Auto-install from GitHub releases with SHA-256 verification
- Cosign provenance verification when available
- Configurable fail-open/fail-closed: `tirith_fail_open` (default: True)
- Background thread installation so startup never blocks
- 24-hour disk cache of install failures

**Config:**
```yaml
security:
  redact_secrets: True
  tirith_enabled: True
  tirith_path: "tirith"
  tirith_timeout: 5
  tirith_fail_open: True
```

### URL Safety

Allowlist-based URL access control.

- `is_safe_url()` — checks against blocklists
- `check_website_access()` — policy enforcement for web tools

### Supply Chain Security (CI)

Scans PRs for:
- `.pth` files (auto-execute on Python startup)
- base64 decode + exec/eval patterns
- subprocess with encoded commands
- marshal/pickle/compile usage

---

## Rate Limiting & Resilience

### Retry Logic (Tenacity)

Configurable retry with exponential backoff via `tenacity` library.

**Usage locations:**
- `tools/skills_hub.py` — GitHub API retries
- `tools/mcp_tool.py` — MCP server connection retries
- `tools/mixture_of_agents_tool.py` — MoA aggregation retries
- `gateway/run.py` — platform adapter reconnection
- `run_agent.py` — LLM API transient error retries

### Timeout Configuration

Per-operation timeouts with sensible defaults.

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

### Fallback Provider Chain

Automatic failover when primary provider is unavailable.

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

Fail-open for optional security features, fail-closed for required.

- **Tirith:** `tirith_fail_open=True` — if scanner unavailable, allow command
- **SSL certs:** `_ensure_ssl_certs()` auto-detects system certs, falls back to certifi bundle
- **Vision backend:** Falls back through provider list — OpenRouter → Nous → Codex → Anthropic → custom

---

## Resource Management

### Container Isolation

Code execution in containers with resource limits.

```yaml
terminal:
  container_cpu: 1
  container_memory: 5120    # MB (5GB)
  container_disk: 51200      # MB (50GB)
  container_persistent: True # Persist filesystem across sessions
```

**Backends:** Docker, Singularity, Modal, Daytona, SSH

### Session Lifecycle

Automatic session reset policies prevent unbounded memory growth.

```python
@dataclass
class SessionResetPolicy:
    mode: str = "both"        # "daily", "idle", "both", or "none"
    at_hour: int = 4          # Hour for daily reset (0-23)
    idle_minutes: int = 1440  # 24 hours
    notify: bool = True
```

### Process Registry

Track and clean up child processes.

- Registers all spawned processes
- Cleanup on exit / timeout
- Prevents zombie processes

---

## Security Summary

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

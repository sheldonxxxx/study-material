# CoPaw Security and Performance Analysis

## Executive Summary

CoPaw demonstrates a well-structured security posture with defense-in-depth mechanisms including tool guards, skill scanning, and authentication. The project uses standard Python libraries for security-critical operations, minimizing dependency attack surface. Performance patterns include async/await throughout the FastAPI backend, embedding cache support, retry logic with exponential backoff, and memory management with compaction.

---

## 1. Secrets Management Approach

### Storage Architecture

CoPaw uses a two-layer secret storage system:

| Layer | Location | Purpose |
|-------|----------|---------|
| `envs.json` | `~/.copaw/.secret/` | Persisted environment variables |
| `auth.json` | `~/.copaw/.secret/` | Authentication credentials and JWT secrets |

**Key files:**
- `src/copaw/envs/store.py` - Environment variable persistence
- `src/copaw/app/auth.py` - Authentication and JWT management
- `src/copaw/constant.py` - Directory path configuration

### Security Controls

**File permissions (defense: filesystem access control)**
```python
# envs/store.py
_chmod_best_effort(path.parent, 0o700)  # Secret directory: owner only
_chmod_best_effort(path, 0o600)          # Secret files: owner read/write

# auth.py
_chmod_best_effort(AUTH_FILE, 0o600)     # auth.json restricted to owner
```

**Secret directory isolation**
```python
# constant.py
SECRET_DIR = Path(os.getenv("COPAW_SECRET_DIR", f"{WORKING_DIR}.secret"))
WORKING_DIR = Path(os.getenv("COPAW_WORKING_DIR", "~/.copaw"))
```

**Protected bootstrap keys**
```python
# envs/store.py - Keys that should not be persisted to envs.json
_PROTECTED_BOOTSTRAP_KEYS = frozenset({
    "COPAW_WORKING_DIR",
    "COPAW_SECRET_DIR",
})
```

**JWT secret rotation on password change**
```python
# auth.py
def update_credentials(...):
    # Rotate JWT secret to invalidate all existing sessions
    data["jwt_secret"] = secrets.token_hex(32)
```

### Authentication Implementation

**Password hashing (stdlib only - no external dependencies)**
```python
# auth.py
def _hash_password(password: str, salt: Optional[str] = None) -> tuple[str, str]:
    if salt is None:
        salt = secrets.token_hex(16)
    h = hashlib.sha256((salt + password).encode("utf-8")).hexdigest()
    return h, salt

def verify_password(password: str, stored_hash: str, salt: str) -> bool:
    h, _ = _hash_password(password, salt)
    return hmac.compare_digest(h, stored_hash)  # Constant-time comparison
```

**Token format: HMAC-SHA256 signed payload**
```python
# auth.py
def create_token(username: str) -> str:
    payload = json.dumps({
        "sub": username,
        "exp": int(time.time()) + TOKEN_EXPIRY_SECONDS,  # 7 days
        "iat": int(time.time()),
    })
    payload_b64 = base64.urlsafe_b64encode(payload.encode()).decode()
    sig = hmac.new(secret.encode(), payload_b64.encode(), hashlib.sha256).hexdigest()
    return f"{payload_b64}.{sig}"
```

**Token extraction methods**
- Authorization header: `Bearer <token>`
- WebSocket query parameter: `?token=<token>` (upgrade requests only)
- Regular query parameter: `?token=<token>`

**Public paths bypass authentication**
```python
_PUBLIC_PATHS = frozenset({
    "/api/auth/login",
    "/api/auth/status",
    "/api/auth/register",
    "/api/version",
})
_PUBLIC_PREFIXES = ("/assets/", "/logo.png", "/copaw-symbol.svg")
```

**Localhost bypass for CLI**
```python
# auth.py - AuthMiddleware._should_skip_auth()
client_host = request.client.host if request.client else ""
return client_host in ("127.0.0.1", "::1")  # CLI access unaffected
```

### Environment Variable Handling

**Priority-based loading**
```python
# config/constant.py
# envs/store.py - Bootstrap envs excluded from persistence injection
bootstrap_envs = {
    key: value for key, value in envs.items()
    if key not in _PROTECTED_BOOTSTRAP_KEYS
}
_apply_to_environ(bootstrap_envs, overwrite=False)  # Runtime vars take precedence
```

---

## 2. Authentication/Authorization Implementation

### Authentication Modes

**Disabled by default (opt-in)**
```python
# auth.py
def is_auth_enabled() -> bool:
    env_flag = os.environ.get("COPAW_AUTH_ENABLED", "").strip().lower()
    return env_flag in ("true", "1", "yes")
```

**Single-user model with auto-registration support**
```python
# auth.py
def auto_register_from_env() -> None:
    # Creates admin account from COPAW_AUTH_USERNAME and COPAW_AUTH_PASSWORD
    # Useful for Docker/Kubernetes automated deployments
```

### Middleware Protection

**FastAPI AuthMiddleware (starlette BaseHTTPMiddleware)**
```python
# auth.py
class AuthMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        if self._should_skip_auth(request):
            return await call_next(request)

        token = self._extract_token(request)
        if not token:
            return Response(status_code=401)
        user = verify_token(token)
        if user is None:
            return Response(status_code=401)

        request.state.user = user
        return await call_next(request)
```

### Web Authentication Flow

1. First visit when auth enabled -> Registration page (no user exists)
2. Subsequent visits -> Login page
3. Post-login -> Token stored in browser localStorage (7-day validity)
4. Token expiry -> Auto-redirect to login

---

## 3. Input Validation Patterns

### Configuration Validation (Pydantic)

**AgentRunningConfig validators**
```python
# config/config.py
@model_validator(mode="after")
def validate_llm_retry_backoff(self) -> "AgentsRunningConfig":
    if self.llm_backoff_cap < self.llm_backoff_base:
        raise ValueError("llm_backoff_cap must be >= llm_backoff_base")
    return self
```

**Field constraints**
```python
max_iters: int = Field(default=100, ge=1)           # Greater than or equal
llm_backoff_base: float = Field(default=1.0, ge=0.1)
memory_compact_ratio: float = Field(default=0.75, ge=0.3, le=0.9)
history_max_length: int = Field(default=10000, ge=1000)
```

**MCP transport validation**
```python
# config/config.py
@model_validator(mode="after")
def _validate_transport_config(self):
    if self.transport == "stdio":
        if not self.command.strip():
            raise ValueError("stdio MCP client requires non-empty command")
    if not self.url.strip():
        raise ValueError(f"{self.transport} MCP client requires non-empty url")
```

### Shell Command Path Extraction

**File path validation in FilePathToolGuardian**
```python
# security/tool_guard/guardians/file_guardian.py
def _looks_like_path_token(token: str) -> bool:
    if token.startswith("-"):
        return False
    if lowered.startswith(("http://", "https://", "ftp://", "data:")):
        return False
    if token.startswith(("~", "/", "./", "../")):
        return True
    if "/" in token:
        return True
    return False
```

**Shell redirect operator parsing**
```python
_SHELL_REDIRECT_OPERATORS = frozenset({">", ">>", "1>", "1>>", "2>", "2>>", "&>", "&>>", "<", "<<", "<<<"})

def _extract_paths_from_shell_command(command: str) -> list[str]:
    tokens = shlex.split(command, posix=True)  # Proper shell parsing
    # Handles: `cat a > out.txt`, `>out.txt`, `2>err.log`
```

---

## 4. Security Measures

### Tool Guard System

**Architecture: Guardian pattern with pluggable analyzers**

```
ToolGuardEngine
    ├── FilePathToolGuardian (always_run=True)
    │   └── Blocks access to sensitive file paths
    └── RuleBasedToolGuardian
        └── Regex pattern matching via YAML rules
```

**Key files:**
- `src/copaw/security/tool_guard/engine.py` - Orchestrator
- `src/copaw/security/tool_guard/guardians/file_guardian.py` - Path protection
- `src/copaw/security/tool_guard/guardians/rule_guardian.py` - Rule-based detection
- `src/copaw/security/tool_guard/rules/dangerous_shell_commands.yaml` - Built-in rules

**Tool guard configuration**
```python
# config/config.py
class ToolGuardConfig(BaseModel):
    enabled: bool = True
    guarded_tools: Optional[List[str]] = None  # None = guard all
    denied_tools: List[str] = Field(default_factory=list)
    custom_rules: List[ToolGuardRuleConfig] = Field(default_factory=list)
    disabled_rules: List[str] = Field(default_factory=list)
```

**Priority: env var > config > default (True)**
```python
# security/tool_guard/engine.py
def _guard_enabled() -> bool:
    env_val = os.environ.get("COPAW_TOOL_GUARD_ENABLED")
    if env_val is not None:
        return env_val.lower() in {"true", "1", "yes"}
    return cfg.security.tool_guard.enabled
```

### File Guard (Path-Based Protection)

**Default protected paths**
```python
# security/tool_guard/guardians/file_guardian.py
_DEFAULT_DENY_DIRS: list[str] = [str(SECRET_DIR) + "/"]
```

**File path checking**
```python
def _is_sensitive(self, abs_path: str) -> bool:
    if abs_path in self._sensitive_files:
        return True
    return any(
        path_obj.is_relative_to(Path(dir_path))
        for dir_path in self._sensitive_dirs
    )
```

**Tool-specific parameter mapping**
```python
_TOOL_FILE_PARAMS: dict[str, tuple[str, ...]] = {
    "read_file": ("file_path",),
    "write_file": ("file_path",),
    "edit_file": ("file_path",),
    "execute_shell_command": ("command",),  # Paths extracted from command
}
```

### Rule-Based Guardian

**Rule format (YAML)**
```yaml
# security/tool_guard/rules/dangerous_shell_commands.yaml
- id: SHELL_PIPE_TO_EXEC
  tool: execute_shell_command
  params: [command]
  category: command_injection
  severity: HIGH
  patterns:
    - "curl.*\\|.*(?:sh|bash)"
    - "wget.*\\|.*(?:sh|bash)"
  exclude_patterns:
    - "^#"
  description: "Piping downloaded content directly to a shell"
  remediation: "Download to a file first and inspect before execution"
```

**Guard finding model**
```python
@dataclass
class GuardFinding:
    id: str
    rule_id: str
    category: GuardThreatCategory
    severity: GuardSeverity  # CRITICAL, HIGH, MEDIUM, LOW
    title: str
    description: str
    tool_name: str
    param_name: str
    matched_value: str
    matched_pattern: str
    snippet: str
    remediation: str
    guardian: str
```

### Skill Scanner

**Architecture: PatternAnalyzer with extensible design**

```
SkillScanner
    ├── PatternAnalyzer (YAML regex signatures)
    │   ├── command_injection
    │   ├── data_exfiltration
    │   ├── hardcoded_secrets
    │   └── prompt_injection
    └── Cache (mtime-based, 64 entry LRU)
```

**Key files:**
- `src/copaw/security/skill_scanner/__init__.py` - Main scanner
- `src/copaw/security/skill_scanner/scanner.py` - Orchestrator
- `src/copaw/security/skill_scanner/analyzers/pattern_analyzer.py` - Pattern matching
- `src/copaw/security/skill_scanner/rules/signatures/*.yaml` - Threat signatures

**Scanner modes**
```python
# security/skill_scanner/__init__.py
_VALID_MODES = {"block", "warn", "off"}  # Default: "warn"

def _get_scan_mode(cfg=None) -> str:
    env = os.environ.get("COPAW_SKILL_SCAN_MODE")
    if env is not None and env in _VALID_MODES:
        return env
    return cfg.mode if cfg else "block"
```

**Scan caching (mtime-based)**
```python
_MAX_CACHE_ENTRIES = 64
_scan_cache: dict[str, tuple[float, ScanResult]] = {}

def _get_cached_result(skill_dir: Path) -> ScanResult | None:
    cached_mtime, cached_result = _scan_cache.get(key)
    if current_mtime == cached_mtime:  # Unchanged = use cache
        return cached_result
    return None
```

**Content hash for whitelist validation**
```python
def compute_skill_content_hash(skill_dir: Path) -> str:
    h = hashlib.sha256()
    for p in sorted(skill_dir.rglob("*")):
        if p.is_file() and not p.is_symlink():
            h.update(p.read_bytes())
    return h.hexdigest()
```

**Scan timeout protection**
```python
with futures.ThreadPoolExecutor(max_workers=1) as executor:
    future = executor.submit(scanner.scan_skill, resolved, skill_name=skill_name)
    try:
        result = future.result(timeout=effective_timeout)  # Default 30s
    except futures.TimeoutError:
        logger.warning("Security scan timed out after %.0fs", effective_timeout)
        return None
```

### Security Configuration (Pydantic models)

```python
# config/config.py
class SecurityConfig(BaseModel):
    tool_guard: ToolGuardConfig = Field(default_factory=ToolGuardConfig)
    file_guard: FileGuardConfig = Field(default_factory=FileGuardConfig)
    skill_scanner: SkillScannerConfig = Field(default_factory=SkillScannerConfig)

class FileGuardConfig(BaseModel):
    enabled: bool = True
    sensitive_files: List[str] = Field(default_factory=list)

class SkillScannerConfig(BaseModel):
    mode: Literal["block", "warn", "off"] = "warn"
    timeout: int = Field(default=30, ge=5, le=300)
    whitelist: List[SkillScannerWhitelistEntry] = Field(default_factory=list)
```

---

## 5. Performance Patterns

### Async/Await Architecture

**FastAPI with full async pipeline**

```python
# providers/provider_manager.py
async def list_provider_info(self) -> List[ProviderInfo]:
    tasks = [provider.get_info() for provider in self.builtin_providers.values()]
    tasks += [provider.get_info() for provider in self.custom_providers.values()]
    provider_infos = await asyncio.gather(*tasks)
    return list(provider_infos)

async def fetch_provider_models(self, provider_id: str) -> List[ModelInfo]:
    models = await provider.fetch_models()  # Async HTTP calls
    provider.extra_models = models
```

**Async channel implementations**
```python
# app/channels/dingtalk/channel.py
async def handle_event(self, event: Dict[str, Any]) -> None:
    async with self._session.post(...) as response:
        ...

# app/channels/feishu/channel.py
async def send_message(self, channel: str, msg: Message) -> None:
    await self.client.send_message(...)
```

### LLM Retry with Exponential Backoff

**RetryChatModel wrapper**

```python
# providers/retry_chat_model.py
RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504, 529}

def _compute_backoff(attempt: int, retry_config: RetryConfig) -> float:
    return min(
        retry_config.backoff_cap,
        retry_config.backoff_base * (2 ** max(0, attempt - 1)),
    )
```

**Configuration**
```python
# constant.py
LLM_MAX_RETRIES = EnvVarLoader.get_int("COPAW_LLM_MAX_RETRIES", 3, min_value=0)
LLM_BACKOFF_BASE = EnvVarLoader.get_float("COPAW_LLM_BACKOFF_BASE", 1.0, min_value=0.1)
LLM_BACKOFF_CAP = EnvVarLoader.get_float("COPAW_LLM_BACKOFF_CAP", 10.0, min_value=0.5)
```

**Streaming retry support**
```python
async def _wrap_stream(self, stream, call_args, call_kwargs, current_attempt, max_attempts):
    try:
        async for chunk in stream:
            yield chunk
    except Exception as exc:
        if not _is_retryable(exc) or current_attempt >= max_attempts:
            raise
        await asyncio.sleep(_compute_backoff(current_attempt, self._retry_config))
        # Retry entire request
```

### Memory Management

**Memory compaction with configurable ratio**

```python
# config/config.py
class AgentsRunningConfig(BaseModel):
    memory_compact_ratio: float = Field(default=0.75, ge=0.3, le=0.9)
    memory_reserve_ratio: float = Field(default=0.1, ge=0.05, le=0.3)
    max_input_length: int = Field(default=128 * 1024, ge=1000)  # 128K tokens

    @property
    def memory_compact_threshold(self) -> int:
        return int(self.max_input_length * self.memory_compact_ratio)

    @property
    def memory_compact_reserve(self) -> int:
        return int(self.max_input_length * self.memory_reserve_ratio)
```

**Tool result compaction**
```python
class AgentsRunningConfig(BaseModel):
    tool_result_compact_recent_n: int = Field(default=1, ge=1, le=10)
    tool_result_compact_old_threshold: int = Field(default=1000, ge=100)
    tool_result_compact_recent_threshold: int = Field(default=30000, ge=1000)
    tool_result_compact_retention_days: int = Field(default=3, ge=1, le=10)
```

### Embedding Cache

```python
# config/config.py
class EmbeddingConfig(BaseModel):
    enable_cache: bool = Field(default=True)
    max_cache_size: int = Field(default=3000)
    max_input_length: int = Field(default=8192)
    max_batch_size: int = Field(default=10)
```

### Provider Directory Permission Hardening

```python
# providers/provider_manager.py
def _prepare_disk_storage(self):
    for path in [self.root_path, self.builtin_path, self.custom_path]:
        path.mkdir(parents=True, exist_ok=True)
        try:
            os.chmod(path, 0o700)  # Restrict permissions
        except Exception:
            pass
```

---

## 6. Rate Limiting

**No explicit rate limiting found in codebase.**

However, retry logic handles transient 429 (rate limit) responses:

```python
# providers/retry_chat_model.py
RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504, 529}

def _is_retryable(exc: Exception) -> bool:
    status = getattr(exc, "status_code", None)
    if status is not None and status in RETRYABLE_STATUS_CODES:
        return True
    return False
```

The `429` status code triggers automatic retry with exponential backoff, effectively handling upstream rate limiting.

---

## 7. Overall Security Posture Assessment

### Strengths

| Area | Implementation |
|------|---------------|
| **Defense in depth** | Tool Guard + File Guard + Skill Scanner layered approach |
| **Stdlib-only crypto** | Uses hashlib, hmac, secrets (no PyJWT dependency) |
| **Constant-time comparison** | `hmac.compare_digest` for password/token verification |
| **File permissions** | 0o600 for secrets, 0o700 for secret directories |
| **Secret isolation** | Separate SECRET_DIR from WORKING_DIR |
| **JWT rotation** | Secret rotated on password change, invalidating sessions |
| **Localhost bypass** | CLI access exempt from auth (127.0.0.1/::1) |
| **Skill scanning** | Pre-activation security scan with mtime-based caching |
| **Path traversal protection** | File Guardian blocks access to sensitive paths |
| **Shell injection detection** | Regex rules for dangerous patterns in shell commands |
| **Config validation** | Pydantic models with field constraints |
| **Graceful failure** | Guardians log warnings but don't block on failure |

### Areas for Improvement

| Issue | Current State | Recommendation |
|-------|---------------|----------------|
| **No rate limiting on API** | 429 retry only | Add per-IP or per-token rate limiting middleware |
| **No CSRF protection** | Single-page app with localStorage tokens | Consider CSRF tokens for browser-based attacks |
| **Token storage** | localStorage (XSS vulnerable) | Consider httpOnly cookie alternative |
| **Password entropy** | SHA-256 + salt (acceptable but slow) | Consider argon2 or bcrypt for better KDF |
| **No MFA support** | Single factor | Add TOTP-based two-factor authentication |
| **Audit logging** | Basic logger statements | Structured audit log for security events |

### Security Documentation

Comprehensive documentation exists at `website/public/docs/security.en.md` covering:
- Tool Guard configuration
- File Guard configuration
- Skill Scanner modes
- Web authentication setup
- Password reset procedures

---

## 8. Performance Documentation

### Async Patterns

- **Channel handlers**: Fully async for DingTalk, Feishu, Discord, Telegram, QQ, etc.
- **Provider API calls**: Async model listing, probing, and fetching
- **Background tasks**: `asyncio.create_task` for non-blocking multimodal probing

### Caching Strategy

| Cache | Mechanism | Size Limit | Invalidation |
|-------|-----------|------------|--------------|
| Skill scan results | Mtime-based | 64 entries | On file change |
| Embedding cache | Configurable | 3000 entries | TTL-based |
| Provider models | Disk persistence | N/A | On provider update |

### Memory Management

- **Compaction ratio**: 75% default (configurable 30-90%)
- **Reserve ratio**: 10% of context window
- **Tool result retention**: 3 days default
- **Character thresholds**: Recent 30K, Old 1K characters

---

## Key Files Reference

| Category | File Path |
|----------|-----------|
| Authentication | `src/copaw/app/auth.py` |
| Secrets Store | `src/copaw/envs/store.py` |
| Config Validation | `src/copaw/config/config.py` |
| Constants | `src/copaw/constant.py` |
| Tool Guard Engine | `src/copaw/security/tool_guard/engine.py` |
| File Guardian | `src/copaw/security/tool_guard/guardians/file_guardian.py` |
| Rule Guardian | `src/copaw/security/tool_guard/guardians/rule_guardian.py` |
| Skill Scanner | `src/copaw/security/skill_scanner/__init__.py` |
| LLM Retry | `src/copaw/providers/retry_chat_model.py` |
| Provider Manager | `src/copaw/providers/provider_manager.py` |
| Memory Manager | `src/copaw/agents/memory/memory_manager.py` |
| Security Docs | `website/public/docs/security.en.md` |

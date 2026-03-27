# CoPaw Security Analysis

## Executive Summary

CoPaw implements defense-in-depth security with layered mechanisms including Tool Guards, File Guardians, and Skill Scanners. Authentication uses standard library crypto (hashlib, hmac, secrets) with JWT-style tokens. Secrets are stored in isolated directories with restricted filesystem permissions.

---

## Secrets Management

### Storage Architecture

| Layer | Location | Purpose |
|-------|----------|---------|
| `envs.json` | `~/.copaw/.secret/` | Persisted environment variables |
| `auth.json` | `~/.copaw/.secret/` | Authentication credentials and JWT secrets |

### Security Controls

**File permissions (filesystem access control):**
```python
# envs/store.py
_chmod_best_effort(path.parent, 0o700)  # Secret directory: owner only
_chmod_best_effort(path, 0o600)          # Secret files: owner read/write

# auth.py
_chmod_best_effort(AUTH_FILE, 0o600)     # auth.json restricted to owner
```

**Secret directory isolation:**
```python
SECRET_DIR = Path(os.getenv("COPAW_SECRET_DIR", f"{WORKING_DIR}.secret"))
WORKING_DIR = Path(os.getenv("COPAW_WORKING_DIR", "~/.copaw"))
```

**Protected bootstrap keys:**
```python
_PROTECTED_BOOTSTRAP_KEYS = frozenset({
    "COPAW_WORKING_DIR",
    "COPAW_SECRET_DIR",
})
```

---

## Authentication Implementation

### Password Hashing (Stdlib Only)

```python
def _hash_password(password: str, salt: Optional[str] = None) -> tuple[str, str]:
    if salt is None:
        salt = secrets.token_hex(16)
    h = hashlib.sha256((salt + password).encode("utf-8")).hexdigest()
    return h, salt

def verify_password(password: str, stored_hash: str, salt: str) -> bool:
    h, _ = _hash_password(password, salt)
    return hmac.compare_digest(h, stored_hash)  # Constant-time comparison
```

### Token Format

**HMAC-SHA256 signed payload:**
```python
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

### Token Extraction

| Method | Use Case |
|--------|----------|
| Authorization header | `Bearer <token>` |
| WebSocket query param | `?token=<token>` (upgrade requests) |
| Query parameter | `?token=<token>` |

### Public Path Bypass

```python
_PUBLIC_PATHS = frozenset({
    "/api/auth/login",
    "/api/auth/status",
    "/api/auth/register",
    "/api/version",
})
_PUBLIC_PREFIXES = ("/assets/", "/logo.png", "/copaw-symbol.svg")
```

### Localhost Bypass

```python
# auth.py - CLI access unaffected
client_host = request.client.host if request.client else ""
return client_host in ("127.0.0.1", "::1")
```

### JWT Secret Rotation

```python
def update_credentials(...):
    # Rotate JWT secret to invalidate all existing sessions
    data["jwt_secret"] = secrets.token_hex(32)
```

---

## Tool Guard System

### Architecture

```
ToolGuardEngine
    ├── FilePathToolGuardian (always_run=True)
    │   └── Blocks access to sensitive file paths
    └── RuleBasedToolGuardian
        └── Regex pattern matching via YAML rules
```

### File Guardian

**Default protected paths:**
```python
_DEFAULT_DENY_DIRS: list[str] = [str(SECRET_DIR) + "/"]
```

**Path validation:**
```python
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

### Rule-Based Guardian

**Rule format (YAML):**
```yaml
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

### Configuration

```python
class ToolGuardConfig(BaseModel):
    enabled: bool = True
    guarded_tools: Optional[List[str]] = None  # None = guard all
    denied_tools: List[str] = Field(default_factory=list)
    custom_rules: List[ToolGuardRuleConfig] = Field(default_factory=list)
    disabled_rules: List[str] = Field(default_factory=list)
```

**Priority:** env var > config > default (True)

---

## Skill Scanner

### Architecture

```
SkillScanner
    ├── PatternAnalyzer (YAML regex signatures)
    │   ├── command_injection
    │   ├── data_exfiltration
    │   ├── hardcoded_secrets
    │   └── prompt_injection
    └── Cache (mtime-based, 64 entry LRU)
```

### Scanner Modes

```python
_VALID_MODES = {"block", "warn", "off"}  # Default: "warn"
```

### Scan Caching

```python
_max_cache_entries = 64
_scan_cache: dict[str, tuple[float, ScanResult]] = {}

def _get_cached_result(skill_dir: Path) -> ScanResult | None:
    cached_mtime, cached_result = _scan_cache.get(key)
    if current_mtime == cached_mtime:
        return cached_result
    return None
```

### Content Hash

```python
def compute_skill_content_hash(skill_dir: Path) -> str:
    h = hashlib.sha256()
    for p in sorted(skill_dir.rglob("*")):
        if p.is_file() and not p.is_symlink():
            h.update(p.read_bytes())
    return h.hexdigest()
```

---

## Input Validation

### Pydantic Models

**Field constraints:**
```python
max_iters: int = Field(default=100, ge=1)
llm_backoff_base: float = Field(default=1.0, ge=0.1)
memory_compact_ratio: float = Field(default=0.75, ge=0.3, le=0.9)
history_max_length: int = Field(default=10000, ge=1000)
```

**Cross-field validation:**
```python
@model_validator(mode="after")
def validate_llm_retry_backoff(self) -> "AgentsRunningConfig":
    if self.llm_backoff_cap < self.llm_backoff_base:
        raise ValueError("llm_backoff_cap must be >= llm_backoff_base")
    return self
```

---

## Performance Patterns

### Async/Await Architecture

**FastAPI async pipeline:**
```python
async def list_provider_info(self) -> List[ProviderInfo]:
    tasks = [provider.get_info() for provider in self.builtin_providers.values()]
    provider_infos = await asyncio.gather(*tasks)
    return list(provider_infos)
```

### LLM Retry with Exponential Backoff

```python
RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504, 529}

def _compute_backoff(attempt: int, retry_config: RetryConfig) -> float:
    return min(
        retry_config.backoff_cap,
        retry_config.backoff_base * (2 ** max(0, attempt - 1)),
    )
```

**Configuration:**
```python
LLM_MAX_RETRIES = EnvVarLoader.get_int("COPAW_LLM_MAX_RETRIES", 3, min_value=0)
LLM_BACKOFF_BASE = EnvVarLoader.get_float("COPAW_LLM_BACKOFF_BASE", 1.0, min_value=0.1)
LLM_BACKOFF_CAP = EnvVarLoader.get_float("COPAW_LLM_BACKOFF_CAP", 10.0, min_value=0.5)
```

---

## Security Posture Assessment

### Strengths

| Area | Implementation |
|------|---------------|
| Defense in depth | Tool Guard + File Guard + Skill Scanner layered approach |
| Stdlib-only crypto | Uses hashlib, hmac, secrets (no PyJWT dependency) |
| Constant-time comparison | `hmac.compare_digest` for password/token verification |
| File permissions | 0o600 for secrets, 0o700 for secret directories |
| Secret isolation | Separate SECRET_DIR from WORKING_DIR |
| JWT rotation | Secret rotated on password change |
| Localhost bypass | CLI access exempt from auth |
| Skill scanning | Pre-activation security scan with caching |
| Path traversal protection | File Guardian blocks sensitive paths |
| Config validation | Pydantic models with field constraints |

### Areas for Improvement

| Issue | Current State | Recommendation |
|-------|---------------|----------------|
| No rate limiting on API | 429 retry only | Add per-IP or per-token rate limiting |
| No CSRF protection | localStorage tokens | Consider CSRF tokens |
| Token storage | localStorage (XSS vulnerable) | Consider httpOnly cookie alternative |
| Password hashing | SHA-256 + salt | Consider argon2 or bcrypt |
| No MFA support | Single factor | Add TOTP-based 2FA |
| Audit logging | Basic logger statements | Structured audit log |

---

## Key Security Files

| Category | File Path |
|----------|-----------|
| Authentication | `src/copaw/app/auth.py` |
| Secrets Store | `src/copaw/envs/store.py` |
| Config Validation | `src/copaw/config/config.py` |
| Tool Guard Engine | `src/copaw/security/tool_guard/engine.py` |
| File Guardian | `src/copaw/security/tool_guard/guardians/file_guardian.py` |
| Rule Guardian | `src/copaw/security/tool_guard/guardians/rule_guardian.py` |
| Skill Scanner | `src/copaw/security/skill_scanner/__init__.py` |
| Security Docs | `website/public/docs/security.en.md` |

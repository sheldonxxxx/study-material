# Action Items for Building a Similar Project

## Priority 1: Architecture & Design

### 1.1 Use Dependency Injection Instead of Singletons
**Why:** Singleton patterns like `ProviderManager.get_instance()` make testing difficult and create global state.

**Action:** Design services to accept dependencies via constructors. Use a DI container or factory pattern for wiring.

**Example:**
```python
# Instead of
manager = ProviderManager.get_instance()

# Do
manager = ProviderManager(provider_registry, config, secrets_store)
```

---

### 1.2 Split Large Files from Day One
**Why:** `channels_cmd.py` (35KB), `provider_manager.py` (1130 lines), and `Chat/index.tsx` (640 lines) became unmaintainable.

**Action:** Enforce a maximum file size limit (e.g., 200 lines / 15KB). Add pre-commit hooks to warn on oversized files.

**Example:** When a file exceeds limits, extract related code into a module:
- `channels_cmd.py` -> `channels/list.py`, `channels/config.py`, `channels/enable.py`
- `Chat/index.tsx` -> `Chat/useIMEEffect.ts`, `Chat/useMultimodal.ts`, `Chat/useSessionSync.ts`

---

### 1.3 Implement Structured Error Types
**Why:** All features use `try/except` with logging but no structured error types. Debugging requires reading logs.

**Action:** Define error hierarchies per subsystem:

```python
# errors.py
class CoPawError(Exception):
    """Base error for all CoPaw exceptions."""
    code: str

class ChannelError(CoPawError):
    """Base for channel-related errors."""

class TelegramError(ChannelError):
    """Telegram-specific errors."""
    code = "TELEGRAM_ERROR"

class ProviderError(CoPawError):
    """LLM provider errors."""
```

---

### 1.4 Avoid Reflection-Based Attribute Access
**Why:** `getattr(obj, "attr", None)` and `hasattr` scattered 10+ times in channels/ indicates runtime schema flexibility that could be centralized.

**Action:** Define clear Pydantic models for all payloads. Use discriminated unions for variants:

```python
class Request(BaseModel):
    type: Literal["text", "image", "audio"]
    content: Union[TextContent, ImageContent, AudioContent]

# Instead of
if hasattr(request.input[0], "model_copy"):
    request.input[0] = request.input[0].model_copy(...)
```

---

## Priority 2: Security

### 2.1 Use Modern Password Hashing (Argon2 or bcrypt)
**Why:** SHA-256 + salt is fast and vulnerable to GPU cracking. CoPaw uses this.

**Action:** Replace with argon2-cffi:

```python
from argon2 import PasswordHasher
ph = PasswordHasher()

def hash_password(password: str) -> str:
    return ph.hash(password)

def verify_password(password: str, hash: str) -> bool:
    return ph.verify(hash, password)
```

---

### 2.2 Move Tokens to httpOnly Cookies
**Why:** localStorage tokens are vulnerable to XSS. CoPaw uses localStorage.

**Action:** Store JWT in httpOnly, Secure cookie:

```python
# auth.py
response.set_cookie(
    key="auth_token",
    value=token,
    httponly=True,
    secure=True,
    samesite="lax",
    max_age=7 * 24 * 60 * 60  # 7 days
)
```

---

### 2.3 Add API Rate Limiting
**Why:** CoPaw has no rate limiting on API endpoints.

**Action:** Add slowapi or custom middleware:

```python
from slowapi import Limiter
limiter = Limiter(key_func=get_remote_address)

@router.post("/chat")
@limiter.limit("60/minute")
async def chat(request: Request):
    ...
```

---

### 2.4 Run Security Scans on Skills Directory
**Why:** Skills are excluded from linting/type checks, creating a security blind spot.

**Action:** Add skills to pre-commit security hooks:

```yaml
# pre-commit-config.yaml
- repo: local
  hooks:
    - id: scan-skills
      name: Security scan skills/
      entry: python -m copaw.security.scan_skills
      files: ^skills/
```

---

### 2.5 Add MFA Support
**Why:** Single-factor authentication is weak for a web-facing application.

**Action:** Add TOTP support via pyotp:

```python
import pyotp

def setup_mfa(user_id: str) -> str:
    secret = pyotp.random_base32()
    totp = pyotp.TOTP(secret)
    return totp.provisioning_uri(user_id, issuer_name="CoPaw")

def verify_mfa(token: str, secret: str) -> bool:
    return pyotp.TOTP(secret).verify(token)
```

---

## Priority 3: Performance

### 3.1 Implement Auto-Update for Desktop App
**Why:** Users must manually download new versions. No notification of available updates.

**Action:** Use github-release-checker:

```python
import checks for updates from GitHub releases
def check_for_updates():
    latest = get_latest_release("owner/repo")
    if Version(latest) > current_version:
        notify_user(latest)
```

---

### 3.2 Add Structured Logging (Audit Trail)
**Why:** Security events are logged with basic logger statements, not structured audit logs.

**Action:** Use structlog for structured security logging:

```python
import structlog
log = structlog.get_logger()

log.security(
    "tool_execution_blocked",
    tool="execute_shell_command",
    pattern="curl.*|.*bash",
    user=request.state.user,
)
```

---

### 3.3 Profile CLI Startup Time
**Why:** CoPaw has timing instrumentation but no visible profiling. Lazy loading is used but not measured.

**Action:** Add startup profiling to CI:

```bash
copaw --profile-output /tmp/profile.json agents list
# Analyze with snakeviz or similar
```

---

## Priority 4: Code Quality

### 4.1 Reduce Disabled Linting Rules
**Why:** 30+ Pylint rules disabled in pre-commit. May hide real issues.

**Action:** Audit disabled rules and re-enable where safe. Track technical debt explicitly:

```python
# TODO(code-quality): Re-enable consider-using-with after fixing resource leaks
# pylint: disable=consider-using-with
```

---

### 4.2 Add Frontend Tests
**Why:** React components with complex state (Chat, 640 lines) have no tests.

**Action:** Add Vitest + React Testing Library:

```typescript
import { render, screen } from '@testing-library/react'
import { Chat } from './Chat'

test('renders message input', () => {
  render(<Chat />)
  expect(screen.getByPlaceholderText('Type a message...')).toBeInTheDocument()
})
```

---

### 4.3 Fix MLX Structured Output
**Why:** MLX backend falls back to prompt-based JSON instead of true structured output.

**Action:** Track MLX library for native JSON mode support. Document the limitation prominently. Consider fallback to llama.cpp for structured output needs.

---

## Priority 5: Maintainability

### 5.1 Remove Directory Naming Hacks
**Why:** `discord_/` with trailing underscore is confusing.

**Action:** Rename to `copaw_discord` or `channel_discord`:

```bash
mv discord_/ copaw_discord/
# Update registry.py
```

---

### 5.2 Implement Centralized Schema Definitions
**Why:** Reflection-based attribute access throughout channels/ indicates missing centralized schemas.

**Action:** Define Pydantic models for all message types:

```python
# schemas/messages.py
class IncomingMessage(BaseModel):
    session_id: str
    content: List[ContentPart]
    sender: Sender
    timestamp: datetime

class OutgoingMessage(BaseModel):
    content: List[ContentPart]
    metadata: Optional[MessageMetadata]
```

---

### 5.3 Document Platform-Specific Quirks
**Why:** DingTalk special-casing in `manager._process_batch()` indicates undocumented platform quirks.

**Action:** Create `PLATFORM_QUIRKS.md`:

```markdown
## DingTalk
- Image + caption arrive as separate payloads (use no-text debouncing)
- Webhook signature verification required
- Rate limits: 60 requests/minute
```

---

### 5.4 Add Backup/Restore for Manifests
**Why:** Manifest corruption in local_models silently returns fresh manifest, losing user config.

**Action:** Implement backup rotation:

```python
def _load_manifest(path: Path) -> Manifest:
    if path.exists():
        backup_path = path.with_suffix('.bak')
        shutil.copy(path, backup_path)  # Always have backup

    try:
        return Manifest.parse_file(path)
    except JSONDecodeError:
        if backup_path.exists():
            logger.warning("Manifest corrupted, restoring from backup")
            return Manifest.parse_file(backup_path)
        raise
```

---

## Priority 6: Developer Experience

### 6.1 Create Architecture Decision Records (ADRs)
**Why:** No documentation explaining why architectural choices were made.

**Action:** Add `docs/adr/` directory:

```markdown
# ADR-001: Singleton ProviderManager

## Status
Accepted

## Context
We need global access to providers across the application.

## Decision
Use singleton pattern with get_instance() method.

## Consequences
- Testing becomes harder; use dependency injection in tests.
```

---

### 6.2 Add Local Development Docker Compose
**Why:** Setting up local dev environment is complex (multiple channels, LLM providers).

**Action:** Provide `docker-compose.dev.yml`:

```yaml
services:
  copaw:
    build: .
    volumes:
      - .:/app
    environment:
      - COPAW_WORKING_DIR=/app/.copaw
    ports:
      - "8000:8000"
```

---

### 6.3 Document Error Codes
**Why:** No centralized error code documentation for API consumers.

**Action:** Create `docs/errors.md`:

```markdown
## API Error Codes

| Code | Meaning | Resolution |
|------|---------|------------|
| AUTH_001 | Invalid credentials | Check username/password |
| AUTH_002 | Token expired | Re-authenticate |
| CHANNEL_001 | Channel not configured | Run `copaw channels config` |
```

---

## Summary: Top 5 Immediate Actions

1. **Switch to dependency injection** - Remove global state, enable testing
2. **Add frontend tests** - Even minimal coverage for Chat component
3. **Fix password hashing** - Replace SHA-256 with argon2
4. **Split large files** - Enforce size limits, extract modules
5. **Add rate limiting** - Protect API from abuse

---

## Timeline Recommendations

| Phase | Duration | Focus |
|-------|----------|-------|
| Phase 1 | Week 1-2 | Security fixes (2.1, 2.2, 2.4) |
| Phase 2 | Week 3-4 | Architecture (1.1, 1.2, 1.4) |
| Phase 3 | Week 5-6 | Code quality (4.1, 4.2, 5.2) |
| Phase 4 | Week 7-8 | Polish (3.1, 3.2, 5.4) |

---

*This document is intended as guidance for building a similar project. Not all items may apply to your specific context - evaluate each based on your project's scope, threat model, and resources.*

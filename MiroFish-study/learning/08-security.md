# MiroFish Security Analysis

## Critical Security Issues

### 1. NO AUTHENTICATION ON ANY ENDPOINT - CRITICAL

All API endpoints are publicly accessible with no authentication or authorization checks.

| Endpoint Category | Risk Level | Notes |
|------------------|------------|-------|
| Project CRUD | CRITICAL | Anyone can create/delete projects |
| Simulation control | CRITICAL | Start/stop simulations freely |
| File download | HIGH | Download any simulation script |
| Zep graph access | HIGH | Read/write to memory graphs |

**No evidence of:**
- JWT or session-based authentication
- API key authentication
- Role-based access control (RBAC)
- Rate limiting per user/IP

**Before any production deployment, authentication MUST be implemented.**

---

### 2. Path Traversal Vulnerability - HIGH SEVERITY

**Location**: `simulation.py` lines 1331-1367 (script download endpoint)

```python
# Line 1347 - No validation of script_name
script_path = os.path.join(scripts_dir, script_name)
# An attacker could request ../../../etc/passwd
```

The `script_name` parameter is not validated or sanitized. An attacker could use path traversal (e.g., `../../etc/passwd`) to read arbitrary files accessible to the backend process.

**Fix**: Whitelist allowed script names or validate path components.

---

### 3. Full Tracebacks in Error Responses - HIGH SEVERITY

**Location**: Multiple error handlers (simulation.py lines 88-89, 235, etc.)

```python
"error": str(e),
"traceback": traceback.format_exc()
```

API responses leak full stack traces including:
- Internal file paths
- Library versions
- Code structure
- Potential exploit paths

**Fix**: Return generic error messages in production; log details server-side only.

---

## Secrets Management

### Environment Variables
- Uses `python-dotenv` to load from `.env` file
- `.env.example` provided with placeholder values

### Issues Found

| Pattern | Status | Details |
|---------|--------|---------|
| Environment variables | OK | Properly loaded via dotenv |
| Secrets in code | WARNING | Hardcoded fallback `SECRET_KEY = 'mirofish-secret-key'` |
| .env.example provided | OK | Template exists |
| load_dotenv override | RISK | `load_dotenv(project_root_env, override=True)` can override system env vars |

### Hardcoded Secret Issue
```python
# config.py line 24
SECRET_KEY = os.environ.get('SECRET_KEY', 'mirofish-secret-key')
```

The fallback default is insecure for production. If `.env` is missing or `SECRET_KEY` is unset, the application uses a known default.

---

## Input Validation

### Good Patterns
- Required field validation (e.g., `project_id`, `simulation_id`)
- Enum validation for `platform` parameter (twitter/reddit/parallel)
- Type conversion for query parameters with defaults

### Gaps
- No schema validation using Pydantic (despite being in dependencies)
- No sanitization of string inputs before passing to LLM or external services
- No file content validation (only extension checking for uploads)

---

## CORS Configuration

**Status**: UNCLEAR

`flask-cors` is in dependencies but no explicit CORS configuration found. Default behavior may allow cross-origin requests from any domain.

---

## File Upload Security

**Location**: `config.py` lines 38-41

```python
MAX_CONTENT_LENGTH = 50 * 1024 * 1024  # 50MB
ALLOWED_EXTENSIONS = {'pdf', 'md', 'txt', 'markdown'}
```

**Concerns:**
- No file content validation (only extension checking)
- No virus scanning
- Upload directory likely writable by the application

---

## Debug Mode

**Issue**: `FLASK_DEBUG` defaults to `True` in config.py line 25

This enables debug features which can expose sensitive information in production.

---

## Summary: Security Posture

| Issue | Severity | Location |
|-------|----------|----------|
| No authentication on any endpoint | CRITICAL | All API routes |
| Full tracebacks in error responses | HIGH | simulation.py:88-89, 235, etc. |
| Path traversal in script download | HIGH | simulation.py:1347 |
| Debug mode defaulting to True | MEDIUM | config.py:25 |
| No rate limiting | MEDIUM | All endpoints |
| CORS configuration unclear | MEDIUM | app/__init__.py |
| Hardcoded SECRET_KEY fallback | MEDIUM | config.py:24 |

---

## Recommendations

1. **Implement authentication** - Add JWT or session-based auth before any production deployment
2. **Remove tracebacks from responses** - Use generic error messages in production
3. **Validate script_name parameter** - Whitelist allowed scripts
4. **Set CORS explicitly** - Configure allowed origins
5. **Disable debug mode** - Default to False in production
6. **Add rate limiting** - Especially for LLM-heavy endpoints
7. **Remove hardcoded SECRET_KEY** - Require it to be set in environment
8. **Add file content validation** - Validate uploaded file magic bytes, not just extensions

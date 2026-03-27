# MiroFish Security and Performance Analysis

## Overview

MiroFish is a Flask-based swarm intelligence engine with Vue 3 frontend. This analysis examines security and performance patterns across the backend API, configuration management, and external service integrations.

---

## Security Analysis

### 1. Secrets Management

**Location:** `backend/app/config.py`, `.env.example`

**Findings:**

| Pattern | Status | Details |
|---------|--------|---------|
| Environment variables | OK | Uses `python-dotenv` to load from `.env` file |
| Secrets in code | WARNING | Hardcoded fallback `SECRET_KEY = 'mirofish-secret-key'` |
| .env.example provided | OK | Template exists with placeholder values |
| load_dotenv override | RISK | `load_dotenv(project_root_env, override=True)` - production vars can be overridden |

```python
# config.py line 14
load_dotenv(project_root_env, override=True)

# config.py line 24
SECRET_KEY = os.environ.get('SECRET_KEY', 'mirofish-secret-key')
```

**Concern:** The `override=True` means if a `.env` file exists in the project root, it will override any environment variables set in the system. This is problematic for production deployments.

---

### 2. Authentication and Authorization

**Finding: NO AUTHENTICATION IMPLEMENTED**

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

---

### 3. Input Validation

**Status:** PARTIAL

**Good patterns observed:**
- Required field validation (e.g., `project_id`, `simulation_id`)
- Enum validation for `platform` parameter (lines 1517-1520, 2216-2220)

```python
# simulation.py line 1517-1520
if platform not in ['twitter', 'reddit', 'parallel']:
    return jsonify({"error": f"无效的平台类型: {platform}"}), 400
```

- Type conversion for query parameters with defaults

```python
# simulation.py line 907
limit = request.args.get('limit', 20, type=int)
```

**Concerns:**
- No schema validation using Pydantic (despite being in dependencies)
- No sanitization of string inputs before passing to LLM or external services
- Path traversal vulnerability in script download endpoint (lines 1331-1367)

```python
# simulation.py line 1347 - No validation of script_name
script_path = os.path.join(scripts_dir, script_name)
# An attacker could request ../../../etc/passwd
```

---

### 4. Error Handling and Information Disclosure

**Status:** RISKY

**Concern:** Full tracebacks exposed in API responses

```python
# simulation.py line 88-89
"error": str(e),
"traceback": traceback.format_exc()
```

This leaks:
- Internal file paths
- Library versions
- Code structure
- Potential exploit paths

---

### 5. CORS Configuration

**Status:** UNCLEAR

`flask-cors` is in dependencies but no explicit CORS configuration found in the codebase. Default behavior may allow cross-origin requests from any domain.

---

### 6. File Upload Security

**Location:** `config.py` lines 38-41

```python
MAX_CONTENT_LENGTH = 50 * 1024 * 1024  # 50MB
ALLOWED_EXTENSIONS = {'pdf', 'md', 'txt', 'markdown'}
```

**Concerns:**
- No file content validation (only extension checking)
- No virus scanning
- Upload directory likely writable by the application

---

## Performance Patterns

### 1. Caching

**Status:** MINIMAL

**Found:**
- Preparation cache check (`_check_simulation_prepared`) to avoid re-generating configs

```python
# simulation.py line 428-443
if not force_regenerate:
    is_prepared, prepare_info = _check_simulation_prepared(simulation_id)
    if is_prepared:
        return jsonify({"already_prepared": True, ...})
```

**Missing:**
- No HTTP cache headers
- No in-memory caching (e.g., Redis)
- No CDN usage for static assets

---

### 2. Pagination

**Status:** GOOD

| Endpoint | Pagination | Default |
|----------|------------|---------|
| `GET /simulation/history` | `limit` | 20 |
| `GET /simulation/actions` | `limit`, `offset` | 100, 0 |
| `GET /simulation/posts` | `limit`, `offset` | 50, 0 |
| `GET /simulation/comments` | `limit`, `offset` | 50, 0 |

---

### 3. Async Processing

**Status:** BASIC (Threading-based)

**Pattern observed:**

```python
# simulation.py line 606-607
thread = threading.Thread(target=run_prepare, daemon=True)
thread.start()
```

**Concerns:**
- Uses daemon threads (cannot cleanly shutdown)
- No proper async/await pattern (Flask is sync by default)
- Long-running LLM calls block the request or require fire-and-forget

---

### 4. External API Handling (LLM)

**Location:** `backend/app/utils/llm_client.py`

**Patterns:**

| Pattern | Status | Details |
|---------|--------|---------|
| Retry with backoff | OK | Decorator-based with jitter |
| Timeout handling | OK | Configurable via `timeout` parameter |
| Response parsing | OK | Cleans markdown code blocks from JSON |
| Error handling | OK | Specific exception types caught |

**Retry configuration** (`retry.py` lines 15-23):
```python
@retry_with_backoff(max_retries=3, initial_delay=1.0, max_delay=30.0, backoff_factor=2.0, jitter=True)
```

---

### 5. External API Handling (Zep)

**Location:** `backend/app/services/zep_graph_memory_updater.py`

**Performance optimizations:**

| Pattern | Value | Purpose |
|---------|-------|---------|
| BATCH_SIZE | 5 | Batch activities before sending |
| SEND_INTERVAL | 0.5s | Rate limiting |
| MAX_RETRIES | 3 | Resilience |
| Thread-based queue | Yes | Non-blocking activity collection |

**Good pattern:**
```python
# Lines 359-389: Worker loop with batch accumulation
# Lines 390-427: Batch send with retry
# Lines 429-452: Flush remaining on shutdown
```

---

### 6. Database (SQLite)

**Status:** SINGLE-CONNECTION PER REQUEST

```python
# simulation.py line 2019
conn = sqlite3.connect(db_path)
# ... use connection ...
conn.close()  # Line 2039
```

**Concerns:**
- No connection pooling
- No query optimization visible
- Synchronous I/O blocks request

---

### 7. Text Processing

**Configuration** (`config.py` lines 43-45):
```python
DEFAULT_CHUNK_SIZE = 500  # For RAG/summarization
DEFAULT_CHUNK_OVERLAP = 50
```

---

## Anti-Patterns and Concerns Summary

### Security

| Issue | Severity | Location |
|-------|----------|----------|
| No authentication on any endpoint | CRITICAL | All API routes |
| Full tracebacks in error responses | HIGH | simulation.py:88-89, 235, etc. |
| Path traversal in script download | HIGH | simulation.py:1347 |
| Debug mode defaulting to True | MEDIUM | config.py:25 |
| No rate limiting | MEDIUM | All endpoints |
| CORS configuration unclear | MEDIUM | app/__init__.py |

### Performance

| Issue | Severity | Location |
|-------|----------|----------|
| Synchronous LLM calls | HIGH | llm_client.py |
| SQLite per-request connections | MEDIUM | simulation.py:2019 |
| Daemon threads (no clean shutdown) | MEDIUM | simulation.py:606 |
| No HTTP caching headers | LOW | All endpoints |
| No in-memory caching | LOW | - |

---

## Recommendations

### Security

1. **Implement authentication** - Add JWT or session-based auth before any production deployment
2. **Remove tracebacks from responses** - Use generic error messages in production
3. **Validate script_name parameter** - Whitelist allowed scripts
4. **Set CORS explicitly** - Configure allowed origins
5. **Disable debug mode** - Default to False in production
6. **Add rate limiting** - Especially for LLM-heavy endpoints

### Performance

1. **Consider async framework** - Move from Flask to FastAPI or add Celery for background tasks
2. **Add Redis caching** - Cache frequently accessed data (entity lists, config)
3. **Connection pooling** - Use SQLAlchemy with connection pooling
4. **Add cache headers** - ETags and Last-Modified for GET endpoints
5. **Batch LLM calls** - Where possible, parallelize multiple LLM invocations

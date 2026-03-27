# My Action Items from MiroFish Study

## Priority Framework

These items are organized by priority based on:
1. **Day 1**: Essential for any project, including prototypes
2. **Before V1**: Needed before first public deployment
3. **Future**: Nice to have for scale/maintainability

---

## Day 1 Essentials

### [1] Add Authentication from Day One

**Source**: MiroFish has ZERO authentication -- CRITICAL security gap

**Action**: Even for a prototype, implement JWT-based auth:
```python
# Use PyJWT, not rolling our own
from datetime import datetime, timedelta
import jwt

SECRET_KEY = os.environ.get('SECRET_KEY')  # NO DEFAULT
ALGORITHM = "HS256"

def create_token(user_id: str) -> str:
    payload = {
        "user_id": user_id,
        "exp": datetime.utcnow() + timedelta(hours=24)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
```

**Verification**: All API endpoints return 401 without valid token

---

### [2] Configure Linting and Formatting BEFORE Writing Code

**Source**: MiroFish has no linting configured; style will diverge from day one

**Action**: Set up both backend and frontend tooling:

```bash
# Backend (Python)
uv add --dev ruff black isort
echo '[tool.ruff]
line-length = 100
select = ["E", "F", "I"]  # pycodestyle errors, pyflakes, isort
' > ruff.toml

# Frontend (JavaScript/TypeScript)
npm install --save-dev eslint prettier
npx eslint --init
```

**Verification**: Pre-commit hook runs linting; CI fails on lint errors

---

### [3] Write Tests BEFORE Code

**Source**: MiroFish has zero automated tests; refactoring is impossible

**Action**: Use TDD for core business logic:
```bash
# Backend
uv add --dev pytest pytest-asyncio
mkdir tests && touch tests/__init__.py

# Example test structure
# tests/
#   test_api/
#     test_auth.py
#     test_projects.py
#   test_services/
#     test_simulation_manager.py
```

**Verification**: `pytest` runs in CI; must achieve >80% coverage on services

---

### [4] Set TypeScript Strict Mode

**Source**: MiroFish lists TypeScript in package.json but never uses it

**Action**: Add TypeScript to frontend with strict mode:
```bash
cd frontend
npm install --save-dev typescript @types/node @types/react @types/react-dom
npx tsc --init

# tsconfig.json essentials:
# {
#   "compilerOptions": {
#     "strict": true,
#     "noImplicitAny": true,
#     "strictNullChecks": true,
#     "moduleResolution": "bundler"
#   }
# }
```

**Verification**: `npx tsc --noEmit` passes with zero errors

---

### [5] Implement Structured Error Responses

**Source**: MiroFish leaks tracebacks in error responses

**Action**: Never expose internals to clients:
```python
class AppError(Exception):
    def __init__(self, message: str, status_code: int = 500):
        self.message = message
        self.status_code = status_code

@app.errorhandler(AppError)
def handle_app_error(error: AppError):
    return jsonify({
        "success": False,
        "error": error.message  # User-safe message only
    }), error.status_code

# For unexpected errors, log full traceback but return generic message
@app.errorhandler(Exception)
def handle_generic_error(error):
    logger.exception("Unexpected error")
    return jsonify({
        "success": False,
        "error": "An unexpected error occurred"
    }), 500
```

**Verification**: Error responses contain no file paths, no tracebacks, no library names

---

### [6] Implement Progress Callbacks for Async Operations

**Source**: MiroFish's polling-based async updates work well (2-3 second intervals)

**Action**: Every async operation >5 seconds gets progress callback:
```python
def long_operation(progress_callback=None):
    total_steps = 10
    for step in range(total_steps):
        # Do work
        if progress_callback:
            progress_callback(
                step=step,
                total=total_steps,
                message=f"Processing step {step + 1}..."
            )
```

**Verification**: Frontend shows progress bar; status endpoint returns current step

---

## Before V1 Deployment

### [7] Remove Path Traversal Vulnerabilities

**Source**: MiroFish has unsanitized script_name parameter

**Action**: Whitelist allowed values:
```python
ALLOWED_SCRIPTS = {'run_reddit.py', 'run_twitter.py', 'run_parallel.py'}

@app.route('/api/simulation/scripts/<script_name>')
def download_script(script_name):
    if script_name not in ALLOWED_SCRIPTS:
        abort(400, "Invalid script name")
    # Proceed with file serving
```

**Verification**: Requesting `../../../etc/passwd` returns 400

---

### [8] Add Rate Limiting

**Source**: MiroFish has no rate limiting; LLM endpoints vulnerable to abuse

**Action**: Implement per-user/per-IP rate limits:
```python
from flask_limiter import Limiter

limiter = Limiter(
    key_func=get_user_identifier,
    default_limits=["200 per day", "50 per hour"],
    storage_uri="memory://"
)

@limiter.limit("10 per minute")
@app.route('/api/llm/generate')
def generate():
    ...
```

**Verification**: Exceeding rate limit returns 429; response includes `Retry-After` header

---

### [9] Configure CORS Explicitly

**Source**: MiroFish's CORS configuration is unclear

**Action**: Explicitly configure allowed origins:
```python
from flask_cors import CORS

CORS(app, resources={
    r"/api/*": {
        "origins": ["https://myapp.com"],
        "methods": ["GET", "POST", "PUT", "DELETE"],
        "allow_headers": ["Authorization", "Content-Type"]
    }
})
```

**Verification**: OPTIONS preflight returns proper Access-Control-Allow-Origin header

---

### [10] Split Monolithic Files

**Source**: MiroFish has 2,712-line `simulation.py`, 1,764-line `runner.py`

**Action**: Split by responsibility BEFORE adding features:
```
backend/app/api/simulation/
├── __init__.py        # Blueprint registration
├── create.py          # POST /api/simulation/create
├── prepare.py         # POST /api/simulation/prepare
├── start.py           # POST /api/simulation/start
├── status.py          # GET /api/simulation/<id>/status
└── history.py         # GET /api/simulation/history
```

**Verification**: No single file exceeds 500 lines; imports are clean (no circular deps)

---

### [11] Add Input Validation with Pydantic

**Source**: MiroFish uses Pydantic for config but not for API validation

**Action**: Validate all API inputs:
```python
from pydantic import BaseModel, Field

class CreateSimulationRequest(BaseModel):
    project_id: str = Field(..., min_length=1, max_length=100)
    graph_id: str = Field(..., min_length=1, max_length=100)
    enable_twitter: bool = True
    enable_reddit: bool = True
    platform: Literal["twitter", "reddit", "parallel"] = "parallel"

@app.route('/api/simulation/create', methods=['POST'])
def create_simulation():
    data = CreateSimulationRequest(**request.json)
    # data is validated; no manual checking needed
```

**Verification**: Invalid input returns 422 with clear field-level error messages

---

### [12] Implement Graceful Shutdown

**Source**: MiroFish uses daemon threads that cannot cleanly shutdown

**Action**: Use proper process management:
```python
import signal
import sys

_shutdown_event = threading.Event()

def signal_handler(signum, frame):
    _shutdown_event.set()

signal.signal(signal.SIGTERM, signal_handler)
signal.signal(signal.SIGINT, signal_handler)

# In worker loop
while not _shutdown_event.is_set():
    # Do work
    if _shutdown_event.wait(timeout=1):
        break  # Received shutdown signal

# Cleanup
cleanup_resources()
```

**Verification**: `docker stop` completes within 30 seconds with clean logs

---

### [13] Add Health Check Endpoint

**Source**: MiroFish has basic `/health` but no deep health checks

**Action**: Verify dependencies are reachable:
```python
@app.route('/health')
def health():
    checks = {
        "status": "ok",
        "checks": {
            "database": check_db(),
            "llm": check_llm(),
            "memory_service": check_zep()
        }
    }
    if not all(checks["checks"].values()):
        checks["status"] = "degraded"
        return jsonify(checks), 503
    return jsonify(checks)
```

**Verification**: `curl /health` returns component status; degraded returns 503

---

## Future / Scale

### [14] Add Structured Logging with Request IDs

**Source**: MiroFish's logging is good but lacks request correlation

**Action**: Add request ID to all log entries:
```python
import uuid
from functools import wraps

@app.before_request
def add_request_id():
    g.request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))

def get_logger(name: str):
    logger = logging.getLogger(name)
    # Add request_id to all log entries
    old_factory = logging.getLogRecordFactory()
    def record_factory(*args, **kwargs):
        record = old_factory(*args, **kwargs)
        record.request_id = g.get('request_id', '-')
        return record
    logging.setLogRecordFactory(record_factory)
    return logger
```

**Verification**: Every log line includes request_id for correlation

---

### [15] Implement Circuit Breaker for External Services

**Source**: LLM or Zep failures can cascade

**Action**: Add circuit breaker pattern:
```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
def call_llm(messages):
    return llm_client.chat(messages)

# When circuit is open, calls fail fast with CircuitBreakerException
```

**Verification**: After 5 failures, calls return cached response or default; logs show "Circuit OPEN"

---

### [16] Add Database Connection Pooling

**Source**: MiroFish uses file-based JSON; real database needs pooling

**Action**: When adding PostgreSQL:
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True  # Verify connections work
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

**Verification**: Under load, connection count stays below pool_size; no "too many connections" errors

---

### [17] Create Governance Documentation

**Source**: MiroFish has 43K stars but no CODEOWNERS, CONTRIBUTING.md

**Action**: Even for small projects:
```
.github/
├── CODEOWNERS                    # Who owns what
├── CONTRIBUTING.md              # How to contribute
├── ISSUE_TEMPLATE.md            # Structured bug reports
└── PULL_REQUEST_TEMPLATE.md     # PR checklist
```

**Verification**: New contributors can find contribution guidelines in <5 minutes

---

## Decision Checklist for My Project

Based on MiroFish lessons, I will:

| Decision | My Choice | MiroFish's Choice |
|----------|-----------|-------------------|
| Authentication | JWT from Day 1 | None (AVOID) |
| Database | PostgreSQL from v0.1 | File-based JSON |
| Async jobs | Celery + Redis | Daemon threads (AVOID) |
| API validation | Pydantic | None |
| Rate limiting | Flask-Limiter | None |
| Error responses | Generic + logged details | Tracebacks exposed (AVOID) |
| Testing | pytest + pytest-asyncio | None |
| Type safety | TypeScript strict + mypy | Not enforced |
| File size limit | Max 500 lines per file | Monolithic (AVOID) |
| Deployment | Docker + explicit CORS | Single image, unclear CORS |

---

## Top 5 Immediate Actions

1. **Add JWT auth middleware** -- Without this, nothing else matters for production
2. **Set up ruff + black + isort** -- Code quality from day one
3. **Write first test** -- Even one test makes TDD the norm, not the exception
4. **Configure CORS explicitly** -- Ship with this correct, not assume defaults
5. **Split simulation.py** -- Before adding feature #2, clean up feature #1

---

## References

- MiroFish Security Analysis: `.planning/research/08-security-perf.md`
- MiroFish Code Quality: `.planning/research/07-code-quality.md`
- MiroFish Architecture: `.planning/research/06-architecture.md`
- MiroFish Feature Batches: `.planning/research/05a-d-features-batch-*.md`

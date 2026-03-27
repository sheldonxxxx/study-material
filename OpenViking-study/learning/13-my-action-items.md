# My Action Items: Building an OpenViking-Inspired Project

**Study Date:** 2026-03-27
**Source:** `/Users/sheldon/Documents/claw/reference/OpenViking/`
**Purpose:** Concrete next steps for building a similar context database for AI agents

---

## Priority 1: Foundation (Start Here)

### 1.1 Implement Tiered Context Loading from Day One

**Why:** OpenViking's L0/L1/L2 system is fundamental to their token optimization. Without it, context management becomes expensive.

**Action:**
1. Define context levels in core types:
```python
class ContextLevel(int, Enum):
    ABSTRACT = 0  # ~100 tokens
    OVERVIEW = 1  # ~2KB
    DETAIL = 2    # Unlimited
```

2. Create abstract/overview generators that run on content ingestion
3. Store `.abstract.md` and `.overview.md` alongside content files
4. Vector search indexes at L0, reranking uses L1, L2 loaded on demand

**Reasoning:** This is OpenViking's core innovation. Building it later means retrofitting every piece of content.

### 1.2 Design Virtual Filesystem URI Scheme

**Why:** Matches agent mental model (files/directories) and provides unified access pattern.

**Action:**
1. Define URI format: `agent://{scope}/{path}`
2. Scopes: `memory`, `resource`, `skill`, `session`, `system`
3. Implement URI validation (prevent traversal attacks)
4. Create filesystem-like operations: ls, tree, read, write, mkdir, rm

**Reasoning:** OpenViking's `viking://` scheme enables filesystem metaphors for all context types.

### 1.3 Build Telemetry with Disabled Mode

**Why:** OpenViking's `OperationTelemetry` has zero overhead when disabled - essential for production safety.

**Action:**
```python
class OperationTelemetry:
    def __init__(self, operation: str, enabled: bool = False):
        self.enabled = enabled
        self._counters: Dict[str, float] = {}
        self._gauges: Dict[str, Any] = {}

    def count(self, key: str, delta: float = 1):
        if not self.enabled:
            return
        self._counters[key] += delta

    @contextmanager
    def measure(self, key: str):
        if not self.enabled:
            yield
            return
        start = time.perf_counter()
        yield
        self.add_duration(key, (time.perf_counter() - start) * 1000)
```

Use `contextvars` for request-scoped binding.

**Reasoning:** Observability without performance penalty. Always-on metrics in development, zero-cost in production.

### 1.4 Use Service Composition Pattern

**Why:** OpenViking's `OpenVikingService` facade scales better than inheritance hierarchies.

**Action:**
```python
class ContextService:
    def __init__(self):
        self._memory_service = MemoryService()
        self._resource_service = ResourceService()
        self._skill_service = SkillService()
        self._search_service = SearchService()

    @property
    def memory(self):
        return self._memory_service

    @property
    def resources(self):
        return self._resource_service
```

**Reasoning:** Each service can evolve independently. Wiring happens once at initialization.

---

## Priority 2: Security (Non-Negotiable)

### 2.1 Implement Envelope Encryption

**Why:** OpenViking's pattern of per-file keys encrypted with account keys is the right security model.

**Action:**
1. Generate random 32-byte File Key per file
2. Derive Account Key from master key via HKDF-SHA256
3. Encrypt File Key with Account Key using AES-256-GCM
4. Store encrypted File Key alongside file metadata

**Reasoning:** Compromise of one file doesn't expose entire dataset.

### 2.2 Hash API Keys with Argon2id

**Why:** Argon2id is the state-of-the-art password/key hashing. OpenViking uses it correctly.

**Action:**
```python
ARGON2_TIME_COST = 3
ARGON2_MEMORY_COST = 65536  # 64 MB
ARGON2_PARALLELISM = 2
ARGON2_HASH_LENGTH = 32

def hash_api_key(key: str) -> str:
    return argon2.PasswordHasher(
        time_cost=ARGON2_TIME_COST,
        memory_cost=ARGON2_MEMORY_COST,
        parallelism=ARGON2_PARALLELISM,
        hash_len=ARGON2_HASH_LENGTH,
    ).hash(key)

def verify_api_key(key: str, hash: str) -> bool:
    return argon2.PasswordHasher().verify(hash, key)
```

**Reasoning:** Protection against GPU/ASIC brute force.

### 2.3 Multi-Tenant Isolation via Account Scoping

**Why:** OpenViking's automatic account filter injection prevents data leakage.

**Action:**
1. Every request carries account context
2. Vector store queries automatically include account filter
3. File operations verify account ownership
4. No cross-account data access possible at query level

**Reasoning:** Defense in depth. Application logic errors don't expose tenant data.

### 2.4 Default CORS to Closed

**Why:** OpenViking's default `["*"]` is a production risk.

**Action:**
```python
class ServerConfig(BaseModel):
    cors_origins: List[str] = []  # Empty = no CORS
    # Or explicit allowlist only
```

**Reasoning:** Fail secure. Operators must explicitly enable cross-origin requests.

---

## Priority 3: Code Quality (Build In, Don't Bolt On)

### 3.1 Enable Strict Type Checking from Day One

**Why:** OpenViking's lenient mypy settings allow untyped code to accumulate.

**Action:**
```toml
[tool.mypy]
python_version = "3.10"
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
warn_return_any = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
```

**Reasoning:** Technical debt compounds. Strict types catch errors at write time, not runtime.

### 3.2 Don't Disable Lint Rules

**Why:** OpenViking ignores E722 (bare except) and B006 (mutable defaults) - common bug sources.

**Action:**
```toml
[tool.ruff.lint]
ignore = [
    # E501 handled by formatter
    # Keep everything else enabled
]
```

If a rule causes issues, fix the code, don't ignore the rule.

**Reasoning:** Each ignored rule is a potential bug waiting to happen.

### 3.3 Run Clippy on Rust Code in CI

**Why:** OpenViking's Rust CLI has no lint gate.

**Action:**
```yaml
# .github/workflows/rust.yml
- name: Clippy
  run: cargo clippy --all-targets -- -D warnings
```

**Reasoning:** Catch common Rust mistakes before they ship.

### 3.4 Add ESLint + Prettier for TypeScript

**Why:** OpenViking's bot framework has zero JavaScript validation.

**Action:**
```bash
npm init -y
npm install -D eslint prettier @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

**Reasoning:** TypeScript without linting is not TypeScript.

### 3.5 Keep Files Small (< 500 lines)

**Why:** OpenViking's 35KB main.rs and 900-line memory_extractor.py are unmaintainable.

**Action:**
- Split by feature, not by layer
- If a file exceeds 500 lines, refactor
- Use dependency injection to manage complexity

**Reasoning:** Small files are easier to understand, test, and review.

---

## Priority 4: Testing (Enable, Don't Disable)

### 4.1 Keep Full Test Suite Enabled

**Why:** OpenViking disabled their 12-config matrix test suite.

**Action:**
1. Write tests alongside code (TDD for business logic)
2. Keep CI fast enough that tests always run
3. If tests fail, fix them - don't disable them

**Reasoning:** Disabled tests are worse than no tests - they give false confidence.

### 4.2 Test Async Code Properly

**Why:** Async bugs are subtle and hard to catch manually.

**Action:**
```python
import pytest
from pytest_asyncio import fixture

@pytest.mark.asyncio
async def test_session_commit():
    session = await Session.create()
    await session.add_message("user", "Hello")
    result = await session.commit_async()
    assert result["archived"] == True
```

**Reasoning:** Async code requires async tests.

### 4.3 Add Integration Tests for External Services

**Why:** OpenViking has extensive integration test infrastructure.

**Action:**
```python
# tests/integration/test_vector_store.py
@pytest.mark.asyncio
async def test_vector_search_with_filter():
    backend = await create_test_backend()
    await backend.upsert([record1, record2])
    results = await backend.query(
        query_vector=embedding,
        filter=Eq("account_id", "test"),
        limit=10
    )
    assert len(results) == 2
```

**Reasoning:** Mocking external services hides integration bugs.

---

## Priority 5: Architecture Decisions

### 5.1 Use Adapter Pattern for Vector Stores

**Why:** OpenViking's CollectionAdapter enables pluggable backends.

**Action:**
```python
class VectorStore(ABC):
    @abstractmethod
    async def upsert(self, records: List[Record]) -> None: ...

    @abstractmethod
    async def query(self, vector: List[float], limit: int) -> List[Record]: ...

class LocalVectorStore(VectorStore):
    async def upsert(self, records: List[Record]) -> None:
        # Store locally
        ...

class CloudVectorStore(VectorStore):
    async def upsert(self, records: List[Record]) -> None:
        # Call cloud API
        ...
```

**Reasoning:** Swap backends without changing call sites.

### 5.2 Implement Dual Client Modes

**Why:** OpenViking's embedded vs HTTP modes serve different use cases.

**Action:**
```python
class BaseClient(ABC):
    @abstractmethod
    async def read(self, uri: str) -> bytes: ...

class LocalClient(BaseClient):
    async def read(self, uri: str) -> bytes:
        return open(uri).read()

class HTTPClient(BaseClient):
    async def read(self, uri: str) -> bytes:
        return await self._request("GET", uri)
```

**Reasoning:** Local for development, HTTP for production scaling.

### 5.3 Use Two-Phase Commit for Sessions

**Why:** OpenViking's Phase 1 (sync, fast) + Phase 2 (async, background) pattern keeps responses fast.

**Action:**
```python
async def commit(self):
    # Phase 1: Lock-protected, fast
    async with self._lock:
        if not self._messages:
            return
        self._archive_messages()
        self._clear_messages()

    # Phase 2: Background, slow
    asyncio.create_task(self._extract_memories())
```

**Reasoning:** User doesn't wait for LLM-based memory extraction.

### 5.4 Design Score Propagation for Retrieval

**Why:** OpenViking's hierarchical retriever propagates scores from parent to child.

**Action:**
```python
def propagate_score(parent_score: float, child_score: float, alpha: float = 0.5) -> float:
    return alpha * child_score + (1 - alpha) * parent_score
```

**Reasoning:** High-scoring directories boost all their children.

---

## Priority 6: Operational Excellence

### 6.1 Implement Health Checks

**Why:** OpenViking's `/api/v1/system/health` endpoint enables load balancer integration.

**Action:**
```python
@router.get("/health")
async def health():
    return {
        "status": "healthy",
        "version": VERSION,
        "dependencies": {
            "vector_store": await check_vector_store(),
            "file_system": await check_fs(),
        }
    }
```

**Reasoning:** Kubernetes needs health checks for rolling deployments.

### 6.2 Add Prometheus Metrics

**Why:** OpenViking's telemetry exports to Prometheus.

**Action:**
```python
from prometheus_client import Counter, Histogram

requests_total = Counter(
    'context_requests_total',
    'Total requests',
    ['method', 'endpoint', 'status']
)

request_duration = Histogram(
    'context_request_duration_seconds',
    'Request duration',
    ['method', 'endpoint']
)
```

**Reasoning:** Standard observability for production systems.

### 6.3 Implement Graceful Shutdown

**Why:** OpenViking's server handles shutdown signals and cleans up child processes.

**Action:**
```python
import signal

class Server:
    def __init__(self):
        self._running = True
        signal.signal(signal.SIGTERM, self._shutdown)
        signal.signal(signal.SIGINT, self._shutdown)

    async def _shutdown(self, signum, frame):
        self._running = False
        await self._cleanup()
```

**Reasoning:** Prevent requests from being dropped during deployment.

### 6.4 Document Multi-Language Contributing Guides

**Why:** OpenViking has English, Chinese, and Japanese contributing docs.

**Action:**
1. CONTRIBUTING.md (English)
2. CONTRIBUTING_*.md for other languages as needed
3. Include: setup, testing, PR process, code style

**Reasoning:** Open source projects need accessible contribution paths.

---

## Priority 7: Start Small, Iterate Fast

### 7.1 Core MVP First

**Why:** OpenViking tried to do everything (Python + Rust + C++ + Go + TypeScript).

**Action - Start with:**
1. Python core only
2. Local vector store (SQLite)
3. Simple HTTP server
4. Basic tiered context (L0/L2 only)

**Add later:**
- Rust CLI for performance
- C++ vector engine extensions
- Multi-provider LLM routing
- TypeScript bot framework

**Reasoning:** Get to working prototype before adding complexity.

### 7.2 Validate Before Building

**Why:** OpenViking's 19K stars suggest strong market fit - but validate your specific niche.

**Action:**
1. Talk to 5-10 potential users
2. Understand their current context management pain points
3. Build only what they need
4. Iterate based on feedback

**Reasoning:** Avoid building features nobody uses.

### 7.3 Set Realistic Scope

**Why:** OpenViking's 17 CI workflows and polyglot architecture requires significant maintenance.

**Action - For a small team:**
- 1-2 languages max at start
- Simple CI (build + test + lint)
- Incremental complexity

**Reasoning:** Maintenance burden scales with codebases.

---

## Summary: Critical Path

```
Phase 1 (Week 1-2):
  [ ] Define ContextLevel enum (L0/L1/L2)
  [ ] Implement envelope encryption
  [ ] Build local vector store adapter
  [ ] Create filesystem URI scheme

Phase 2 (Week 3-4):
  [ ] Implement tiered context generators
  [ ] Add multi-tenant scoping
  [ ] Build dual client modes
  [ ] Enable strict mypy

Phase 3 (Week 5-6):
  [ ] Add comprehensive telemetry
  [ ] Implement two-phase session commit
  [ ] Build hierarchical retrieval
  [ ] Add Prometheus metrics

Phase 4 (Later):
  [ ] Rust CLI (if needed for performance)
  [ ] Multi-provider LLM routing
  [ ] TypeScript bot framework
  [ ] C++ vector engine (if benchmarking shows need)
```

**Remember:** OpenViking has 19K stars because it solves a real problem. Build to solve problems you understand, not to replicate everything they did.

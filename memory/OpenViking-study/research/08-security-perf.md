# OpenViking Security and Performance Patterns Analysis

## Executive Summary

OpenViking implements a multi-layered security model with strong encryption at rest, API key authentication with Argon2id hashing, and role-based access control. Performance is optimized through C++ vector database extensions, async/await patterns throughout, caching layers, and telemetry-based observability.

---

## 1. Secrets Management

### Configuration File Loading

OpenViking uses a **four-level configuration resolution chain** (`openviking_cli/utils/config/config_loader.py`):

```
1. Explicit path (--config CLI argument)
2. Environment variable (OPENVIKING_CONFIG_FILE)
3. ~/.openviking/<config_file>
4. /etc/openviking/<config_file>
```

**Environment Variable Expansion**: The `load_json_config()` function uses `os.path.expandvars()` to support environment variable substitution in JSON config files:
```python
raw = os.path.expandvars(raw)
```

### API Key Management

API keys are managed via `openviking/server/api_keys.py` with the following security measures:

**Key Generation**: Uses `secrets.token_hex(32)` for cryptographically secure 64-character hex keys.

**Argon2id Hashing**: When encryption is enabled, API keys are hashed with Argon2id parameters:
- Time cost: 3 iterations
- Memory cost: 65536 KB (64 MB)
- Parallelism: 2
- Hash length: 32 bytes

```python
ARGON2_TIME_COST = 3
ARGON2_MEMORY_COST = 65536
ARGON2_PARALLELISM = 2
ARGON2_HASH_LENGTH = 32
```

**Prefix Index for Fast Lookup**: Keys are indexed by first 8 characters (`_get_key_prefix()`) to enable O(1) candidate selection without exposing the full key.

**HMAC Constant-Time Comparison**: Uses `hmac.compare_digest()` for all key comparisons to prevent timing attacks:
```python
if hmac.compare_digest(api_key, self._root_key):
```

### Encryption at Rest

**Envelope Encryption Pattern** (`openviking/crypto/encryptor.py`):
- Each file has an independent random File Key (32 bytes)
- File Key is encrypted with Account Key
- Account Key is derived from Root Key via HKDF-SHA256

**Three Key Provider Options**:

| Provider | Root Key Storage | Use Case |
|----------|------------------|----------|
| `LocalFileProvider` | File system (0600 permissions) | Development |
| `VaultProvider` | HashiCorp Vault transit engine | Production |
| `VolcengineKMSProvider` | Volcengine KMS | Cloud production |

**AES-256-GCM Encryption**: All file content encrypted with AES-GCM using 12-byte random IVs.

---

## 2. Authentication and Authorization

### Authentication Modes

**Two Auth Modes** (`openviking/server/config.py`):

1. **`api_key` mode**: Standard API key authentication via `X-Api-Key` header or `Bearer` token
2. **`trusted` mode**: Trust external identity headers (`X-OpenViking-Account`, `X-OpenViking-User`) - requires gateway

### Role-Based Access Control

**Three Roles**:
- `ROOT`: Full administrative access
- `ADMIN`: Account-level administration
- `USER`: Standard user access

### Multi-Tenant Isolation

**Tenant Scoping**: Every request must include `X-OpenViking-Account` and `X-OpenViking-User` headers. ROOT requests to tenant-scoped APIs require explicit headers.

**Allowed Paths for ROOT without Tenant Context**:
```python
_ROOT_IMPLICIT_TENANT_ALLOWED_PATHS = {
    "/api/v1/system/status",
    "/api/v1/system/wait",
    "/api/v1/debug/health",
}
_ROOT_IMPLICIT_TENANT_ALLOWED_PREFIXES = (
    "/api/v1/admin",
    "/api/v1/observer",
)
```

### Security Validation

`validate_server_config()` in `config.py` enforces:
- Non-localhost binding without auth is a fatal error
- Trusted mode on non-localhost generates a warning

---

## 3. Input Validation and Sanitization

### Pydantic Validation

Comprehensive validation using Pydantic v2 with `field_validator` and `model_validator`:

**Collection Schema Validation** (`openviking/storage/vectordb/utils/validation.py`):
```python
@field_validator("FieldName")
@classmethod
def validate_fieldname(cls, v):
    return validate_name_str(v)  # Alphanumeric + underscore, 1-128 chars

@field_validator("Dim")
@classmethod
def validate_dim(cls, v):
    if v is not None:
        if v % 4 != 0:
            raise ValueError(f"dimension must be a multiple of 4, got {v}")
    return v
```

**Name Validation Rules**:
- Length: 1-128 characters
- Must start with a letter
- Alphanumeric + underscore only

**Field Type Validation**:
```python
REQUIRED_COLLECTION_FIELD_TYPE_CHECK = {
    "int64": ([int], None, 0),
    "float32": ([int, float], None, 0.0),
    "string": ([str], lambda l: len(l) <= 1024, "default"),
    "vector": ([list], lambda l: all(isinstance(item, (int, float)) for item in l), []),
    # ... more types
}
```

### Index Meta Validation

Validates scalar indexes against field metadata:
```python
if model.ScalarIndex:
    unknown_fields = set(model.ScalarIndex) - set(field_meta_dict.keys())
    if unknown_fields:
        raise ValidationError(f"scalar index contains unknown fields: {list(unknown_fields)}")
```

---

## 4. Performance Patterns

### C++ Vector Database Engine

**SIMD Optimizations** (`src/CMakeLists.txt`):

| Platform | Variants | Flags |
|----------|----------|-------|
| x86_64 | SSE3, AVX2, AVX512 | `-msse3`, `-mavx2`, `-mavx512f` |
| ARM64 | Native + KRL | ARM-specific vector extensions |

```cmake
if(OV_VARIANT STREQUAL "avx512")
    foreach(FLAG -mavx512f -mavx512bw -mavx512dq -mavx512vl)
        list(APPEND OV_FLAGS ${FLAG})
    endforeach()
endif()
```

**Build Output**: Python abi3 stable ABI extensions (Python 3.10+):
```cmake
target_compile_definitions(
    ${MODULE_TARGET}
    PRIVATE
        Py_LIMITED_API=0x030A0000
)
```

### Async/Await Throughout

The codebase uses `asyncio` extensively for I/O-bound operations:
- `async def` for all service methods
- `asyncio.Semaphore` for concurrency control
- `asyncio.Lock` for critical sections

**Example - Path Locking** (`openviking/storage/transaction/path_lock.py`):
```python
async def _create_lock_file(
    self, lock_path: str, owner_id: str, lock_type: str = LOCK_TYPE_POINT
) -> None:
    token = _make_fencing_token(owner_id, lock_type)
    self._agfs.write(lock_path, token.encode("utf-8"))
```

### Caching Patterns

**In-Memory Schema Cache** (`openviking/session/memory/schema_model_generator.py`):
```python
self._model_cache: Dict[str, Type[BaseModel]] = {}

def get_model(self, memory_type: MemoryType) -> Type[BaseModel]:
    cache_key = memory_type.memory_type
    if cache_key in self._flat_data_models:
        return self._flat_data_models[cache_key]
```

**Tool Description Cache** (`openviking/session/memory_extractor.py`):
```python
self._tool_desc_cache: dict[str, str] = {}
self._tool_desc_cache_ready: bool = False
```

### Pagination and Lazy Loading

**Vector Search with Limit** (`openviking/session/memory_deduplicator.py`):
```python
results = await self.vikingdb.search_similar_memories(
    owner_space=owner_space,
    category_uri_prefix=category_uri_prefix,
    query_vector=query_vector,
    limit=5,  # Only fetch top 5 similar memories
    ctx=ctx,
)
```

**Max Prompt Similar Memories**: Limits LLM context to 5 candidates:
```python
MAX_PROMPT_SIMILAR_MEMORIES = 5  # Number of similar memories sent to LLM
```

### Background Scheduler

`APScheduler.BackgroundScheduler` for periodic maintenance tasks:
```python
self.scheduler = BackgroundScheduler(
    executors={"default": {"type": "threadpool", "max_workers": 1}}
)
```

Configurable intervals via environment variables:
- `VECTORDB_TTL_CLEANUP_SECONDS`: TTL expiration cleanup
- `VECTORDB_INDEX_MAINTENANCE_SECONDS`: Index maintenance

---

## 5. Telemetry and Observability

### Custom Telemetry Framework

Located in `openviking/telemetry/`:

**Core Components**:
- `operation.py`: `OperationTelemetry` - operation-scoped collector with counters, gauges, durations
- `context.py`: Context variable-based telemetry binding
- `runtime.py`: Global telemetry runtime management

**OperationTelemetry Features**:
```python
class OperationTelemetry:
    def count(self, key: str, delta: float = 1) -> None
    def set(self, key: str, value: Any) -> None
    def add_duration(self, key: str, duration_ms: float) -> None
    def add_token_usage(self, input_tokens: int, output_tokens: int) -> None
    def measure(self, key: str) -> Iterator[None]:  # Context manager
```

**Disabled Mode for Zero Overhead**: When `enabled=False`, all methods are no-ops:
```python
def count(self, key: str, delta: float = 1) -> None:
    if not self.enabled:
        return
```

### Metric Categories

**Token Metrics**:
- `tokens.llm.input`, `tokens.llm.output`, `tokens.llm.total`
- `tokens.embedding.total`

**Vector Search Metrics**:
- `vector.searches`, `vector.scored`, `vector.passed`
- `vector.scanned`, `vector.returned`

**Memory Extraction Metrics**:
- `memory.extracted`
- `memory.extract.stage.*.duration_ms` (12 stages tracked)

### Context-Based Binding

Uses Python `contextvars` for request-scoped telemetry:
```python
_CURRENT_TELEMETRY: contextvars.ContextVar[OperationTelemetry] = contextvars.ContextVar(
    "openviking_operation_telemetry",
    default=_NOOP_TELEMETRY,
)

@contextmanager
def bind_telemetry(handle: OperationTelemetry) -> Iterator[OperationTelemetry]:
    token = _CURRENT_TELEMETRY.set(handle)
    try:
        yield handle
    finally:
        _CURRENT_TELEMETRY.reset(token)
```

### Prometheus Observer

Optional Prometheus metrics export via `openviking/storage/observers/prometheus_observer.py`.

---

## 6. Concurrency Control

### Path Locks with Fencing Tokens

Prevents split-brain in distributed scenarios with fencing tokens:
```python
def _make_fencing_token(owner_id: str, lock_type: str = LOCK_TYPE_POINT) -> str:
    return f"{owner_id}:{time.time_ns()}:{lock_type}"
```

**Lock Types**:
- `P`: Point lock (single file)
- `S`: Subtree lock (directory tree)

### Semaphore-Based LLM Concurrency

`SemanticDagExecutor` limits concurrent LLM calls:
```python
self._llm_sem = asyncio.Semaphore(max_concurrent_llm)
```

---

## 7. Security Observations

### Strengths

1. **Strong Encryption**: AES-256-GCM with envelope encryption and HKDF key derivation
2. **Secure Key Storage**: Argon2id for API key hashing, 0600 file permissions
3. **Constant-Time Comparison**: HMAC compare_digest for all secret comparisons
4. **Multi-Tenant Isolation**: Explicit tenant headers required for tenant-scoped operations
5. **Pydantic Validation**: Comprehensive input validation at API boundaries

### Areas for Consideration

1. **Rate Limiting**: No rate limiting infrastructure found in core codebase
2. **Audit Logging**: Telemetry captures metrics but explicit security audit logs not evident
3. **Secret Scanning**: `.pr_agent.toml` suggests PR agent for security scanning, but no custom secret scanning in CI
4. **CORS Configuration**: Default `cors_origins: ["*"]` - may need restriction in production

---

## 8. Performance Observations

### Strengths

1. **Native Extensions**: C++ vector engine with SIMD optimizations (SSE3/AVX2/AVX512)
2. **Async Throughout**: Consistent async/await for I/O operations
3. **Caching Layers**: Schema model cache, tool description cache
4. **Pagination**: Vector search limits prevent unbounded results
5. **Background Maintenance**: Scheduled TTL cleanup and index maintenance
6. **Memory Deduplication**: LLM-assisted with vector pre-filtering (only top 5 candidates to LLM)

### Architecture Patterns

1. **Vertical Slice by Concern**: Each phase has distinct model/API/UI components
2. **Envelope Encryption**: Consistent encryption pattern across all providers
3. **Context Variables**: Thread-safe request-scoped state
4. **Observer Pattern**: Prometheus observer for metrics export

---

## 9. Files Examined

| Category | Files |
|----------|-------|
| Auth | `openviking/server/auth.py`, `openviking/server/api_keys.py` |
| Config | `openviking/server/config.py`, `openviking_cli/utils/config/config_loader.py` |
| Encryption | `openviking/crypto/encryptor.py`, `openviking/crypto/providers.py` |
| Validation | `openviking/storage/vectordb/utils/validation.py` |
| Telemetry | `openviking/telemetry/*.py` (7 files) |
| C++ Build | `src/CMakeLists.txt` |
| Concurrency | `openviking/storage/transaction/path_lock.py` |
| Memory | `openviking/session/memory_deduplicator.py`, `openviking/session/memory_extractor.py` |

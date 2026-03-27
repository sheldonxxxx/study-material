# OpenViking Security Analysis

## Overview

OpenViking implements a **multi-layered security model** with strong encryption at rest, API key authentication with Argon2id hashing, and role-based access control. Key security components include envelope encryption, constant-time comparisons, and multi-tenant isolation.

---

## 1. Secrets Management

### Configuration File Loading

OpenViking uses a **four-level configuration resolution chain**:

```
1. Explicit path (--config CLI argument)
2. Environment variable (OPENVIKING_CONFIG_FILE)
3. ~/.openviking/<config_file>
4. /etc/openviking/<config_file>
```

**Environment Variable Expansion:** The `load_json_config()` function supports environment variable substitution:
```python
raw = os.path.expandvars(raw)
```

### API Key Management

API keys are managed via `openviking/server/api_keys.py` with comprehensive security measures:

**Key Generation:** Uses `secrets.token_hex(32)` for cryptographically secure 64-character hex keys.

**Argon2id Hashing:** When encryption is enabled, API keys are hashed with strong parameters:
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

**Prefix Index for Fast Lookup:** Keys are indexed by first 8 characters to enable O(1) candidate selection without exposing the full key.

**HMAC Constant-Time Comparison:** Uses `hmac.compare_digest()` for all key comparisons to prevent timing attacks:
```python
if hmac.compare_digest(api_key, self._root_key):
```

---

## 2. Encryption at Rest

### Envelope Encryption Pattern

Implemented in `openviking/crypto/encryptor.py`:

```
Each File
    |
    +-- File Key (32 bytes random, per-file)
           |
           +-- Encrypted with Account Key
                  |
                  +-- Derived from Root Key via HKDF-SHA256
```

### Three Key Provider Options

| Provider | Root Key Storage | Use Case |
|----------|------------------|----------|
| `LocalFileProvider` | File system (0600 permissions) | Development |
| `VaultProvider` | HashiCorp Vault transit engine | Production |
| `VolcengineKMSProvider` | Volcengine KMS | Cloud production |

### Encryption Details

- **Algorithm:** AES-256-GCM
- **IV:** 12-byte random nonce per encryption operation
- **Key Derivation:** HKDF-SHA256 from Root Key to Account Key

---

## 3. Authentication and Authorization

### Authentication Modes

**Two Auth Modes** (`openviking/server/config.py`):

1. **`api_key` mode:** Standard API key authentication via `X-Api-Key` header or `Bearer` token
2. **`trusted` mode:** Trust external identity headers (`X-OpenViking-Account`, `X-OpenViking-User`) - requires gateway

### Role-Based Access Control

Three predefined roles:

| Role | Access Level |
|------|--------------|
| `ROOT` | Full administrative access |
| `ADMIN` | Account-level administration |
| `USER` | Standard user access |

### Multi-Tenant Isolation

**Tenant Scoping:** Every request must include `X-OpenViking-Account` and `X-OpenViking-User` headers.

**Allowed Paths for ROOT without Tenant Context:**
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

`validate_server_config()` enforces:
- Non-localhost binding without auth is a **fatal error**
- Trusted mode on non-localhost generates a **warning**

---

## 4. Input Validation and Sanitization

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

**Name Validation Rules:**
- Length: 1-128 characters
- Must start with a letter
- Alphanumeric + underscore only

**Field Type Validation:**
```python
REQUIRED_COLLECTION_FIELD_TYPE_CHECK = {
    "int64": ([int], None, 0),
    "float32": ([int, float], None, 0.0),
    "string": ([str], lambda l: len(l) <= 1024, "default"),
    "vector": ([list], lambda l: all(isinstance(item, (int, float)) for item in l), []),
}
```

---

## 5. Concurrency Control

### Path Locks with Fencing Tokens

Prevents split-brain in distributed scenarios with fencing tokens:

```python
def _make_fencing_token(owner_id: str, lock_type: str = LOCK_TYPE_POINT) -> str:
    return f"{owner_id}:{time.time_ns()}:{lock_type}"
```

**Lock Types:**
- `P`: Point lock (single file)
- `S`: Subtree lock (directory tree)

### Semaphore-Based LLM Concurrency

`SemanticDagExecutor` limits concurrent LLM calls:

```python
self._llm_sem = asyncio.Semaphore(max_concurrent_llm)
```

---

## 6. Telemetry Security Considerations

### Custom Telemetry Framework

Located in `openviking/telemetry/`:

**Core Components:**
- `operation.py`: Operation-scoped collector with counters, gauges, durations
- `context.py`: Context variable-based telemetry binding
- `runtime.py`: Global telemetry runtime management

**Zero-Overhead Disabled Mode:**
```python
def count(self, key: str, delta: float = 1) -> None:
    if not self.enabled:
        return
```

---

## 7. Security Strengths

1. **Strong Encryption**: AES-256-GCM with envelope encryption and HKDF key derivation
2. **Secure Key Storage**: Argon2id for API key hashing, 0600 file permissions
3. **Constant-Time Comparison**: HMAC compare_digest for all secret comparisons
4. **Multi-Tenant Isolation**: Explicit tenant headers required for tenant-scoped operations
5. **Pydantic Validation**: Comprehensive input validation at API boundaries
6. **Structured Error Handling**: No information leakage in error messages

---

## 8. Areas for Consideration

### Rate Limiting

**Status:** No rate limiting infrastructure found in core codebase.

### Audit Logging

**Status:** Telemetry captures metrics but explicit security audit logs not evident.

### Secret Scanning

**Status:** `.pr_agent.toml` suggests PR agent for security scanning, but no custom secret scanning in CI.

### CORS Configuration

**Observation:** Default `cors_origins: ["*"]` - may need restriction in production.

---

## 9. Performance Patterns (Security-Relevant)

### SIMD Vector Engine

C++ extensions with platform-specific optimizations:

| Platform | Variants | Flags |
|----------|----------|-------|
| x86_64 | SSE3, AVX2, AVX512 | `-msse3`, `-mavx2`, `-mavx512f` |
| ARM64 | Native + KRL | ARM-specific vector extensions |

### Caching with Security Implications

**In-Memory Schema Cache:**
```python
self._model_cache: Dict[str, Type[BaseModel]] = {}
```

**Tool Description Cache:**
```python
self._tool_desc_cache: dict[str, str] = {}
self._tool_desc_cache_ready: bool = False
```

### Pagination

Vector search limits prevent unbounded results:
```python
MAX_PROMPT_SIMILAR_MEMORIES = 5  # Number of similar memories sent to LLM
```

---

## 10. Files Examined

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

---

## Summary

OpenViking demonstrates a mature security posture with:

- **Strong cryptographic foundations** (AES-256-GCM, Argon2id, HKDF)
- **Defense in depth** (envelope encryption, key providers, constant-time comparisons)
- **Multi-tenant isolation** (RBAC, tenant-scoped operations)
- **Comprehensive input validation** (Pydantic v2)

Areas warranting attention include implementing rate limiting, adding explicit audit logging, and restricting CORS configuration for production deployments.

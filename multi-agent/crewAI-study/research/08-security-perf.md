# crewAI Security and Performance Patterns Analysis

## Executive Summary

This document analyzes security and performance patterns in the crewAI codebase. The project demonstrates reasonable security practices in several areas but has notable concerns in others, particularly around SQL injection vulnerabilities, code execution sandboxing, and some subprocess handling patterns.

---

## 1. Secrets and API Key Management

### 1.1 Environment Variable Handling

**Location:** `.env.test`, `lib/crewai/src/crewai/llm.py`

crewAI uses `python-dotenv` for environment variable loading with `load_dotenv()` at module initialization:

```python
# lib/crewai/src/crewai/llm.py:97
load_dotenv()
```

API keys are passed through environment variables and accessed via `os.environ.get()` or directly as constructor parameters:

```python
# Pattern used across providers
api_key: str | None = None  # Can be passed explicitly
# or
api_key = os.environ.get("OPENAI_API_KEY")
```

### 1.2 API Key Configuration

**Location:** `lib/crewai-tools/src/crewai_tools/tools/stagehand_tool/.env.example`

```bash
ANTHROPIC_API_KEY="your_anthropic_api_key"
OPENAI_API_KEY="your_openai_api_key"
MODEL_API_KEY="your_model_api_key"
BROWSERBASE_API_KEY="your_browserbase_api_key"
BROWSERBASE_PROJECT_ID="your_browserbase_project_id"
```

### 1.3 Token Manager (CLI Authentication)

**Location:** `lib/crewai/src/crewai/cli/authentication/main.py`

The CLI authentication uses OAuth2 Device Code flow with JWT token validation:

```python
# lib/crewai/src/crewai/cli/authentication/main.py:163-178
def _validate_and_save_token(self, token_data: dict[str, Any]) -> None:
    jwt_token = token_data["access_token"]
    issuer = self.oauth2_provider.get_issuer()
    jwt_token_data = {
        "jwt_token": jwt_token,
        "jwks_url": self.oauth2_provider.get_jwks_url(),
        "issuer": issuer,
        "audience": self.oauth2_provider.get_audience(),
    }
    decoded_token = validate_jwt_token(**jwt_token_data)
    expires_at = decoded_token.get("exp", 0)
    self.token_manager.save_tokens(jwt_token, expires_at)
```

**Observation:** JWT tokens are stored via `TokenManager` but the storage mechanism is not reviewed here. Ensure tokens are stored securely (e.g., system keyring, not plaintext files).

### 1.4 Telemetry Privacy

**Location:** `lib/crewai/src/crewai/telemetry/telemetry.py:1-7`

The telemetry module explicitly states privacy commitments:

```python
"""Telemetry module for CrewAI.

This module provides anonymous telemetry collection for development purposes.
No prompts, task descriptions, agent backstories/goals, responses, or sensitive
data is collected. Users can opt-in to share more complete data using the
`share_crew` attribute.
"""
```

Telemetry can be disabled via environment variables:
- `CREWAI_DISABLE_TELEMETRY=true`
- `OTEL_SDK_DISABLED=true`

---

## 2. Input Validation and Sanitization

### 2.1 Tool Name Sanitization

**Location:** `lib/crewai/src/crewai/utilities/string_utils.py:26-54`

Tool names are sanitized to prevent injection into LLM providers:

```python
def sanitize_tool_name(name: str, max_length: int = _MAX_TOOL_NAME_LENGTH) -> str:
    name = unicodedata.normalize("NFKD", name)
    name = name.encode("ascii", "ignore").decode("ascii")
    name = _CAMEL_UPPER_LOWER.sub(r"\1_\2", name)
    name = _CAMEL_LOWER_UPPER.sub(r"\1_\2", name)
    name = name.lower()
    name = _QUOTE_PATTERN.sub("", name)  # Removes quotes
    name = _DISALLOWED_CHARS_PATTERN.sub("_", name)
    name = _DUPLICATE_UNDERSCORE_PATTERN.sub("_", name)
    name = name.strip("_")
    # Truncates with hash suffix if too long
```

### 2.2 String Interpolation Validation

**Location:** `lib/crewai/src/crewai/utilities/string_utils.py:79-154`

The `interpolate_only` function validates template variables and type-checks input values:

```python
def interpolate_only(
    input_string: str | None,
    inputs: dict[str, str | int | float | dict[str, Any] | list[Any]],
) -> str:
    # Validates only {variable_name} pattern allowed
    # Restricts to str, int, float, bool, dict, list types
    # Raises ValueError for unsupported types
```

### 2.3 File Path Validation

**Location:** `lib/crewai-files/src/crewai_files/core/sources.py:176`

Symlink traversal is blocked:

```python
if os.path.islink(full_path):
    raise ValueError(f"Symlink escapes allowed directory: {self.path}")
```

### 2.4 Pydantic Model Validation

The codebase extensively uses Pydantic v2 for input validation across:
- `lib/crewai/src/crewai/mcp/config.py` - MCP server configurations
- `lib/crewai/src/crewai/task.py` - Task definitions
- `lib/crewai/src/crewai/crew.py` - Crew configurations
- `lib/crewai-files/src/crewai_files/processing/validators.py` - File validation

---

## 3. SQL Injection Vulnerability

### 3.1 NL2SQL Tool - CRITICAL

**Location:** `lib/crewai-tools/src/crewai_tools/tools/nl2sql/nl2sql_tool.py:58-63`

**VULNERABILITY:** The `_fetch_all_available_columns` method uses f-string interpolation for table names, creating a SQL injection vulnerability:

```python
def _fetch_all_available_columns(
    self, table_name: str
) -> list[dict[str, Any]] | str:
    return self.execute_sql(
        f"SELECT column_name, data_type FROM information_schema.columns WHERE table_name = '{table_name}';"  # noqa: S608
    )
```

The `# noqa: S608` comment indicates bandit (security linter) flags this as a SQL injection risk.

**Impact:** An attacker who controls the `table_name` parameter could inject arbitrary SQL. While the immediate impact is limited by the context (the table_name comes from the database's own information_schema), this is still a code smell and potential risk.

**Recommendation:** Use parameterized queries:
```python
def _fetch_all_available_columns(self, table_name: str) -> list[dict[str, Any]] | str:
    return self.execute_sql(
        "SELECT column_name, data_type FROM information_schema.columns WHERE table_name = :table_name;",
        {"table_name": table_name}
    )
```

### 3.2 User-Provided SQL Execution

**Location:** `lib/crewai-tools/src/crewai_tools/tools/nl2sql/nl2sql_tool.py:65-75`

The `_run` method executes user-provided SQL queries directly:

```python
def _run(self, sql_query: str) -> list[dict[str, Any]] | str:
    try:
        data = self.execute_sql(sql_query)
    except Exception as exc:
        data = (
            f"Based on these tables {self.tables} and columns {self.columns}, "
            "you can create SQL queries to retrieve data from the database."
            f"Get the original request {sql_query} and the error {exc} and create the correct SQL query."
        )
    return data
```

**Note:** While user SQL is executed directly, SQLAlchemy's `text()` wrapper (line 87) does provide some protection. However, the database user should have minimal privileges (principle of least privilege).

---

## 4. Shell Command Execution

### 4.1 Subprocess Usage with Shell=False

**Location:** Multiple files

The codebase uses `subprocess.run` with `shell=False` (the safe default) in most cases:

```python
# lib/crewai-tools/src/crewai_tools/tools/code_interpreter_tool/code_interpreter_tool.py:286
subprocess.run(  # noqa: S603
    ["pip", "install", "--quiet"] + requirements,
    capture_output=True,
    check=True,
)
```

However, bandit suppression comments (`# noqa: S603`, `# noqa: S607`) appear throughout, indicating the security linter flags these as potentially dangerous.

### 4.2 Code Interpreter Sandbox - KNOWN LIMITATION

**Location:** `lib/crewai-tools/src/crewai_tools/tools/code_interpreter_tool/code_interpreter_tool.py:52-88`

The `SandboxPython` class explicitly documents its insecurity:

```python
class SandboxPython:
    """INSECURE: A restricted Python execution environment with known vulnerabilities.

    WARNING: This class does NOT provide real security isolation and is vulnerable to
    sandbox escape attacks via Python object introspection. Attackers can recover the
    original __import__ function and bypass all restrictions.

    DO NOT USE for untrusted code execution. Use Docker containers instead.
    """

    BLOCKED_MODULES: ClassVar[set[str]] = {
        "os", "sys", "subprocess", "shutil", "importlib",
        "inspect", "tempfile", "sysconfig", "builtins",
    }

    UNSAFE_BUILTINS: ClassVar[set[str]] = {
        "exec", "eval", "open", "compile", "input",
        "globals", "locals", "vars", "help", "dir",
    }
```

**Observation:** The code acknowledges this is insecure and recommends Docker. However, the Docker-based alternative should be verified as properly isolated.

### 4.3 MCP Stdio Transport

**Location:** `lib/crewai/src/crewai/mcp/transports/stdio.py:59-109`

The MCP stdio transport spawns subprocesses with user-provided commands:

```python
async def connect(self) -> Self:
    # ...
    process_env = os.environ.copy()
    process_env.update(self.env)

    server_params = StdioServerParameters(
        command=self.command,
        args=self.args,
        env=process_env if process_env else None,
    )
```

**Risk:** If an attacker can control `command`, `args`, or `env`, they could execute arbitrary code. This is mitigated by:
1. The MCP server configuration is typically controlled by the application developer, not end users
2. The `env` parameter allows passing sensitive data (like API keys) to the subprocess

---

## 5. Authentication and Authorization

### 5.1 OAuth2 Device Code Flow

**Location:** `lib/crewai/src/crewai/cli/authentication/main.py`

The CLI uses OAuth2 Device Authorization Grant flow:

```python
def _get_device_code(self) -> dict[str, Any]:
    device_code_payload = {
        "client_id": self.oauth2_provider.get_client_id(),
        "scope": " ".join(self.oauth2_provider.get_oauth_scopes()),
        "audience": self.oauth2_provider.get_audience(),
    }
    response = httpx.post(
        url=self.oauth2_provider.get_authorize_url(),
        data=device_code_payload,
        timeout=20,
    )
```

### 5.2 Bearer Token Authentication

**Location:** Multiple files

API requests use Bearer token authentication:

```python
# lib/crewai/src/crewai/cli/plus_api.py:32-33
if api_key:
    self.headers["Authorization"] = f"Bearer {api_key}"

# lib/crewai-tools/src/crewai_tools/adapters/enterprise_adapter.py:253
"Authorization": f"Bearer {self.enterprise_action_token}",
```

---

## 6. Performance Patterns

### 6.1 Caching

#### Tool Result Cache

**Location:** `lib/crewai/src/crewai/agents/cache/cache_handler.py`

Thread-safe in-memory cache with read-write lock:

```python
class CacheHandler(BaseModel):
    _cache: dict[str, Any] = PrivateAttr(default_factory=dict)
    _lock: RWLock = PrivateAttr(default_factory=RWLock)

    def add(self, tool: str, input: str, output: Any) -> None:
        with self._lock.w_locked():
            self._cache[f"{tool}-{input}"] = output

    def read(self, tool: str, input: str) -> Any | None:
        with self._lock.r_locked():
            return self._cache.get(f"{tool}-{input}")
```

**Observation:** Uses SHA-like key construction (`f"{tool}-{input}"`). No TTL or size limits implemented.

#### MCP Tool Caching

**Location:** `lib/crewai/src/crewai/mcp/config.py:47-50`

```python
cache_tools_list: bool = Field(
    default=False,
    description="Whether to cache the tool list for faster subsequent access.",
)
```

### 6.2 Rate Limiting

**Location:** `lib/crewai/src/crewai/utilities/rpm_controller.py`

RPM (Requests Per Minute) controller with thread-safe implementation:

```python
class RPMController(BaseModel):
    max_rpm: int | None = Field(
        default=None,
        description="Maximum requests per minute. If None, no limit is applied.",
    )

    def check_or_wait(self) -> bool:
        # Thread-safe RPM checking
        # Resets counter every 60 seconds
```

**Observation:** The rate limiter is a simple in-memory counter. It won't work correctly in multi-process deployments without shared state.

### 6.3 Async Patterns

The codebase uses `asyncio` extensively:
- `lib/crewai/src/crewai/mcp/transports/stdio.py` - async connect/disconnect
- `lib/crewai/src/crewai/llm.py` - async `acall` method with `async for` streaming

### 6.4 Database Connection Handling

**Location:** `lib/crewai/src/crewai/flow/persistence/sqlite.py`

SQLite connections use timeout parameters:

```python
# lib/crewai/src/crewai/flow/persistence/sqlite.py:78
sqlite3.connect(self.db_path, timeout=30) as conn,
```

### 6.5 HTTP Client Configuration

**Location:** `lib/crewai/src/crewai/cli/plus_api.py:49`

```python
with httpx.Client(trust_env=False, verify=verify) as client:
    return client.request(method, url, headers=self.headers, **kwargs)
```

- `trust_env=False` - ignores proxy environment variables (security benefit)
- `verify` parameter - allows disabling SSL verification if needed

---

## 7. MCP Tool Integration Security

### 7.1 Tool Filtering

**Location:** `lib/crewai/src/crewai/mcp/filters.py`

MCP tools can be filtered by name:

```python
class StaticToolFilter:
    def __init__(
        self,
        allowed_tool_names: list[str] | None = None,
        blocked_tool_names: list[str] | None = None,
    ) -> None:
        self.allowed_tool_names = set(allowed_tool_names or [])
        self.blocked_tool_names = set(blocked_tool_names or [])

    def __call__(self, tool: dict[str, Any]) -> bool:
        tool_name = tool.get("name", "")
        if self.blocked_tool_names and tool_name in self.blocked_tool_names:
            return False
        if self.allowed_tool_names:
            return tool_name in self.allowed_tool_names
        return True
```

Dynamic filtering is also supported with context:

```python
class ToolFilterContext(BaseModel):
    agent: Any = Field(..., description="The agent requesting tools.")
    server_name: str = Field(..., description="Name of the MCP server.")
    run_context: dict[str, Any] | None = Field(default=None)
```

### 7.2 HTTP Transport Headers

**Location:** `lib/crewai/src/crewai/mcp/config.py:72-75`

```python
headers: dict[str, str] | None = Field(
    default=None,
    description="Optional HTTP headers for authentication or other purposes.",
)
```

Headers are passed to MCP HTTP servers, allowing Bearer token authentication.

---

## 8. Identified Security Concerns

### 8.1 Critical Issues

| Issue | Location | Description |
|-------|----------|-------------|
| SQL Injection (Low Risk) | `nl2sql_tool.py:62` | F-string interpolation with table_name in SQL query |
| Insecure Sandbox | `code_interpreter_tool.py:52-88` | SandboxPython class is explicitly marked insecure |

### 8.2 Medium Risk Issues

| Issue | Location | Description |
|-------|----------|-------------|
| User SQL Execution | `nl2sql_tool.py:67` | User-provided SQL executed directly (though via SQLAlchemy) |
| Subprocess Bandit Suppressions | Multiple | `# noqa: S603/S607` suppressed security warnings |

### 8.3 Lower Risk Issues

| Issue | Location | Description |
|-------|----------|-------------|
| No TTL on Cache | `cache_handler.py` | In-memory cache has no expiration or size limit |
| Single-Process Rate Limiting | `rpm_controller.py` | RPM controller is in-memory only |
| Token Storage | `TokenManager` | JWT storage mechanism not reviewed |

---

## 9. Security Best Practices Observed

1. **Pydantic Validation** - Extensive use of Pydantic v2 for input validation
2. **Tool Name Sanitization** - LLM tool names are sanitized before use
3. **Environment Variable Pattern** - Uses `python-dotenv` for configuration
4. **OAuth2 Flow** - CLI uses secure OAuth2 Device Code flow
5. **Telemetry Opt-Out** - Users can disable telemetry via environment variables
6. **Symlink Blocking** - File operations check for symlink traversal
7. **SSL Verification Option** - `trust_env=False` and `verify` parameter available
8. **Thread-Safe Primitives** - RWLock for cache, threading.Lock for RPM

---

## 10. Performance Best Practices Observed

1. **In-Memory Caching** - Tool results cached with RWLock for concurrency
2. **MCP Tool List Caching** - Optional caching of MCP tool lists
3. **Async/Await** - Extensive use of asyncio for I/O operations
4. **Streaming Support** - LLM responses support streaming
5. **Connection Timeouts** - Database connections have timeouts (30s for SQLite)
6. **Batch Processing** - Tracing uses batch exporters

---

## 11. Recommendations

### Security

1. **Fix SQL Injection** in NL2SQL tool - use parameterized queries
2. **Audit Token Storage** - verify TokenManager uses secure storage
3. **Review Subprocess Usage** - ensure all subprocess calls are necessary and safe
4. **Document Docker Isolation** - verify code interpreter Docker usage is properly sandboxed
5. **Add Cache TTL** - implement expiration for cached tool results

### Performance

1. **Redis/Memcache Option** - consider optional distributed caching for multi-process deployments
2. **Connection Pooling** - review database connection pooling settings
3. **Lazy Loading** - continue current patterns for heavy imports
4. **Pagination** - ensure large result sets are paginated

---

## 12. References

- Security Policy: `.github/security.md`
- Test Environment: `.env.test`
- Bandit Config: `pyproject.toml` (ruff with S, B, RUF rules)
- CodeQL: `.github/workflows/codeql.yml`

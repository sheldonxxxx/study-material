# crewAI Security Assessment

## Executive Summary

crewAI implements reasonable security practices in several areas including Pydantic-based input validation, tool name sanitization, OAuth2 authentication for CLI, and telemetry opt-out. However, notable concerns exist around a SQL injection vulnerability in the NL2SQL tool, an explicitly insecure code sandbox, and subprocess handling patterns. The project uses bandit (S rules) and has a CodeQL workflow for automated scanning.

---

## 1. Secrets and API Key Management

### Environment Variable Handling

API keys are managed through `python-dotenv` with `load_dotenv()` at module initialization:

```python
# lib/crewai/src/crewai/llm.py:97
load_dotenv()
```

Keys are accessed via `os.environ.get()` or passed as constructor parameters:

```python
api_key: str | None = None  # Can be passed explicitly
# or
api_key = os.environ.get("OPENAI_API_KEY")
```

### Token Manager (CLI Authentication)

The CLI uses **OAuth2 Device Code flow** with JWT token validation:

```python
# lib/crewai/src/crewai/cli/authentication/main.py:163-178
def _validate_and_save_token(self, token_data: dict[str, Any]) -> None:
    jwt_token = token_data["access_token"]
    issuer = self.oauth2_provider.get_issuer()
    # ... JWT validation via validate_jwt_token()
    self.token_manager.save_tokens(jwt_token, expires_at)
```

**Note:** JWT storage mechanism (TokenManager) was not reviewed. Verify tokens are stored in system keyring, not plaintext files.

### Telemetry Privacy

Telemetry is **opt-in by default** with explicit opt-out:

```python
"""Telemetry module for CrewAI.

This module provides anonymous telemetry collection for development purposes.
No prompts, task descriptions, agent backstories/goals, responses, or sensitive
data is collected. Users can opt-in to share more complete data using the
`share_crew` attribute.
"""
```

Disable via environment variables:
- `CREWAI_DISABLE_TELEMETRY=true`
- `OTEL_SDK_DISABLED=true`

---

## 2. Input Validation and Sanitization

### Tool Name Sanitization

Tool names are sanitized before being sent to LLM providers:

```python
# lib/crewai/src/crewai/utilities/string_utils.py:26-54
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

### String Interpolation Validation

Template variable validation restricts input types:

```python
# lib/crewai/src/crewai/utilities/string_utils.py:79-154
def interpolate_only(
    input_string: str | None,
    inputs: dict[str, str | int | float | dict[str, Any] | list[Any]],
) -> str:
    # Validates only {variable_name} pattern allowed
    # Restricts to str, int, float, bool, dict, list types
```

### File Path Validation

Symlink traversal is explicitly blocked:

```python
# lib/crewai-files/src/crewai_files/core/sources.py:176
if os.path.islink(full_path):
    raise ValueError(f"Symlink escapes allowed directory: {self.path}")
```

### Pydantic Model Validation

Extensive use of Pydantic v2 for input validation across:
- `lib/crewai/src/crewai/mcp/config.py` - MCP server configurations
- `lib/crewai/src/crewai/task.py` - Task definitions
- `lib/crewai/src/crewai/crew.py` - Crew configurations
- `lib/crewai-files/src/crewai_files/processing/validators.py` - File validation

---

## 3. Authentication and Authorization

### OAuth2 Device Code Flow

```python
# lib/crewai/src/crewai/cli/authentication/main.py:284-294
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

### Bearer Token Authentication

API requests use Bearer token authentication:

```python
# lib/crewai/src/crewai/cli/plus_api.py:32-33
if api_key:
    self.headers["Authorization"] = f"Bearer {api_key}"
```

---

## 4. SQL Injection Vulnerability

### NL2SQL Tool - CRITICAL

**Location:** `lib/crewai-tools/src/crewai_tools/tools/nl2sql/nl2sql_tool.py:58-63`

**VULNERABILITY:** The `_fetch_all_available_columns` method uses f-string interpolation for table names:

```python
def _fetch_all_available_columns(
    self, table_name: str
) -> list[dict[str, Any]] | str:
    return self.execute_sql(
        f"SELECT column_name, data_type FROM information_schema.columns WHERE table_name = '{table_name}';"  # noqa: S608
    )
```

The `# noqa: S608` comment indicates bandit flags this as SQL injection risk.

**Impact:** Limited by context (table_name comes from database's own information_schema), but still a code smell.

**Recommendation:** Use parameterized queries:
```python
return self.execute_sql(
    "SELECT column_name, data_type FROM information_schema.columns WHERE table_name = :table_name;",
    {"table_name": table_name}
)
```

### User-Provided SQL Execution

```python
# lib/crewai-tools/src/crewai_tools/tools/nl2sql/nl2sql_tool.py:65-75
def _run(self, sql_query: str) -> list[dict[str, Any]] | str:
    try:
        data = self.execute_sql(sql_query)
    except Exception as exc:
        # Returns error guidance to LLM
        data = f"Based on these tables... Get the original request {sql_query} and the error {exc}"
    return data
```

SQLAlchemy's `text()` wrapper provides some protection, but database user should have minimal privileges.

---

## 5. Shell Command Execution

### Subprocess Usage

The codebase uses `subprocess.run` with `shell=False` (safe default) in most cases:

```python
# lib/crewai-tools/src/crewai_tools/tools/code_interpreter_tool/code_interpreter_tool.py:286
subprocess.run(
    ["pip", "install", "--quiet"] + requirements,
    capture_output=True,
    check=True,
)
```

However, bandit suppression comments (`# noqa: S603`, `# noqa: S607`) appear throughout, indicating security linter flags these as potentially dangerous.

### Code Interpreter Sandbox - KNOWN LIMITATION

**Location:** `lib/crewai-tools/src/crewai_tools/tools/code_interpreter_tool/code_interpreter_tool.py:52-88`

The `SandboxPython` class is **explicitly documented as insecure**:

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

**Note:** Docker-based alternative should be verified as properly isolated.

### MCP Stdio Transport

**Location:** `lib/crewai/src/crewai/mcp/transports/stdio.py:59-109`

MCP stdio transport spawns subprocesses with user-controlled commands:

```python
async def connect(self) -> Self:
    process_env = os.environ.copy()
    process_env.update(self.env)

    server_params = StdioServerParameters(
        command=self.command,
        args=self.args,
        env=process_env if process_env else None,
    )
```

**Risk:** If an attacker controls `command`, `args`, or `env`, they could execute arbitrary code. Mitigated by:
1. MCP server configuration is typically controlled by application developer, not end users
2. The `env` parameter allows passing sensitive data (API keys) to subprocess

---

## 6. MCP Integration Security

### Tool Filtering

MCP tools can be filtered by name:

```python
# lib/crewai/src/crewai/mcp/filters.py
class StaticToolFilter:
    def __init__(
        self,
        allowed_tool_names: list[str] | None = None,
        blocked_tool_names: list[str] | None = None,
    ) -> None:
        ...

    def __call__(self, tool: dict[str, Any]) -> bool:
        tool_name = tool.get("name", "")
        if self.blocked_tool_names and tool_name in self.blocked_tool_names:
            return False
        if self.allowed_tool_names:
            return tool_name in self.allowed_tool_names
        return True
```

### HTTP Transport Headers

```python
# lib/crewai/src/crewai/mcp/config.py:72-75
headers: dict[str, str] | None = Field(
    default=None,
    description="Optional HTTP headers for authentication or other purposes.",
)
```

Headers allow Bearer token authentication to MCP servers.

---

## 7. Guardrails Implementation

### Hallucination Guardrail

**Location:** `lib/crewai/src/crewai/tasks/hallucination_guardrail.py`

### LLM Guardrail

**Location:** `lib/crewai/src/crewai/tasks/llm_guardrail.py`

---

## 8. Security Best Practices Observed

1. **Pydantic Validation** - Extensive use of Pydantic v2 for input validation
2. **Tool Name Sanitization** - LLM tool names are sanitized before use
3. **Environment Variable Pattern** - Uses `python-dotenv` for configuration
4. **OAuth2 Flow** - CLI uses secure OAuth2 Device Code flow
5. **Telemetry Opt-Out** - Users can disable telemetry via environment variables
6. **Symlink Blocking** - File operations check for symlink traversal
7. **SSL Verification Option** - `trust_env=False` and `verify` parameter available
8. **Thread-Safe Primitives** - RWLock for cache, threading.Lock for RPM

---

## 9. Identified Security Concerns

### Critical Issues

| Issue | Location | Description |
|-------|----------|-------------|
| SQL Injection (Low Risk) | `nl2sql_tool.py:62` | F-string interpolation with table_name in SQL query |
| Insecure Sandbox | `code_interpreter_tool.py:52-88` | SandboxPython class is explicitly marked insecure |

### Medium Risk Issues

| Issue | Location | Description |
|-------|----------|-------------|
| User SQL Execution | `nl2sql_tool.py:67` | User-provided SQL executed directly (though via SQLAlchemy) |
| Subprocess Bandit Suppressions | Multiple | `# noqa: S603/S607` suppressed security warnings |

### Lower Risk Issues

| Issue | Location | Description |
|-------|----------|-------------|
| No TTL on Cache | `cache_handler.py` | In-memory cache has no expiration or size limit |
| Single-Process Rate Limiting | `rpm_controller.py` | RPM controller is in-memory only |
| Token Storage | `TokenManager` | JWT storage mechanism not reviewed |

---

## 10. Recommendations

### Security

1. **Fix SQL Injection** in NL2SQL tool - use parameterized queries
2. **Audit Token Storage** - verify TokenManager uses secure storage (system keyring)
3. **Review Subprocess Usage** - ensure all subprocess calls are necessary and safe
4. **Document Docker Isolation** - verify code interpreter Docker usage is properly sandboxed
5. **Add Cache TTL** - implement expiration for cached tool results

### Performance

1. **Redis/Memcache Option** - consider optional distributed caching for multi-process deployments
2. **Connection Pooling** - review database connection pooling settings
3. **Lazy Loading** - continue current patterns for heavy imports
4. **Pagination** - ensure large result sets are paginated

---

## 11. References

- Security Policy: `.github/security.md`
- Test Environment: `.env.test`
- Bandit Config: `pyproject.toml` (ruff with S, B, RUF rules)
- CodeQL: `.github/workflows/codeql.yml`

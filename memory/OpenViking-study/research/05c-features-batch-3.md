# OpenViking Feature Research: Batch 3 (Features 7-9)

**Repository:** `/Users/sheldon/Documents/claw/reference/OpenViking/`
**Research Date:** 2026-03-27
**Features Covered:**
- Feature 7: Multi-Provider Model Support
- Feature 8: Rust CLI (ov_cli)
- Feature 9: HTTP Server & REST API

---

## Feature 7: Multi-Provider Model Support (Core Feature)

### Overview

OpenViking implements unified multi-provider support through a layered architecture:
1. **Provider Registry** (`bot/vikingbot/providers/registry.py`) - Single source of truth for provider metadata
2. **LiteLLM Provider** (`bot/vikingbot/providers/litellm_provider.py`) - Uses LiteLLM library for routing
3. **OpenAI-Compatible Provider** (`bot/vikingbot/providers/openai_compatible_provider.py`) - Direct OpenAI SDK for OpenAI-compatible APIs
4. **Base Interface** (`bot/vikingbot/providers/base.py`) - Abstract `LLMProvider` base class

### Provider Registry Architecture

The registry uses a `ProviderSpec` dataclass to define provider metadata:

```python
@dataclass(frozen=True)
class ProviderSpec:
    name: str              # config field name, e.g. "dashscope"
    keywords: tuple[str]   # model-name keywords for matching (lowercase)
    env_key: str          # LiteLLM env var, e.g. "DASHSCOPE_API_KEY"
    litellm_prefix: str   # "dashscope" → model becomes "dashscope/{model}"
    skip_prefixes: tuple  # don't prefix if model already starts with these
    env_extras: tuple     # extra env vars, e.g. (("ZHIPUAI_API_KEY", "{api_key}"),)
    is_gateway: bool      # routes any model (OpenRouter, AiHubMix)
    is_local: bool        # local deployment (vLLM, Ollama)
    detect_by_key_prefix: str  # match api_key prefix, e.g. "sk-or-"
    detect_by_base_keyword: str  # match substring in api_base URL
    default_api_base: str  # fallback base URL
    strip_model_prefix: bool  # strip "provider/" before re-prefixing
    model_overrides: tuple  # per-model param overrides
```

**Supported Providers** (15 total):

| Provider | Keywords | Prefix | Notes |
|----------|----------|--------|-------|
| OpenRouter | openrouter | openrouter | Gateway, detects by `sk-or-` key prefix |
| AiHubMix | aihubmix | openai | Gateway, strips model prefix |
| Anthropic | anthropic, claude | (none) | Native LiteLLM support |
| OpenAI | openai, gpt | (none) | Native LiteLLM support |
| DeepSeek | deepseek | deepseek | |
| VolcEngine | volcengine, volces, ark | volcengine | |
| Gemini | gemini | gemini | |
| Zhipu | zhipu, glm, zai | zai | Mirrors key to ZHIPUAI_API_KEY |
| DashScope | qwen, dashscope | dashscope | |
| Moonshot | moonshot, kimi | moonshot | K2.5 requires temp >= 1.0 |
| MiniMax | minimax | minimax | |
| vLLM | vllm | hosted_vllm | Local deployment |
| Groq | groq | groq | |

### Key Implementation Details

#### Provider Detection Logic

```python
def find_gateway(provider_name, api_key, api_base):
    # 1. Direct match by config key
    if provider_name and spec.is_gateway or spec.is_local:
        return spec
    # 2. Auto-detect by api_key prefix / api_base keyword
    for spec in PROVIDERS:
        if spec.detect_by_key_prefix and api_key.startswith(spec.detect_by_key_prefix):
            return spec
        if spec.detect_by_base_keyword and spec.detect_by_base_keyword in api_base:
            return spec
```

#### Model Resolution

The `_resolve_model` method handles provider-specific model name transformations:

```python
def _resolve_model(self, model: str) -> str:
    if self._gateway:
        # Gateway mode: apply gateway prefix, skip provider-specific prefixes
        prefix = self._gateway.litellm_prefix
        if self._gateway.strip_model_prefix:
            model = model.split("/")[-1]  # Strip to bare model name
        if prefix and not model.startswith(f"{prefix}/"):
            model = f"{prefix}/{model}"
        return model

    # Standard mode: auto-prefix for known providers
    spec = find_by_model(model)
    if spec and spec.litellm_prefix:
        if not any(model.startswith(s) for s in spec.skip_prefixes):
            model = f"{spec.litellm_prefix}/{model}"
    return model
```

#### System Message Handling for MiniMax

MiniMax doesn't support system messages, so they merge into the first user message:

```python
def _handle_system_message(self, model: str, messages):
    if "minimax" in model.lower():
        # Extract system messages
        system_contents = [msg.get("content") for msg in messages if msg.get("role") == "system"]
        cleaned_messages = [msg for msg in messages if msg.get("role") != "system"]
        full_system_prompt = "\n\n".join(system_contents)

        # Merge into first user message
        for msg in cleaned_messages:
            if msg.get("role") == "user":
                msg["content"] = f"{full_system_prompt}\n\n{msg.get('content')}"
                break
        return cleaned_messages
    return messages
```

#### Model-Specific Overrides

Kimi K2.5 requires `temperature >= 1.0`:

```python
model_overrides=(("kimi-k2.5", {"temperature": 1.0}),)
```

### Clever Solutions

1. **Gateway Priority**: Gateways are matched first since they can route any model
2. **Skip Prefixes**: Prevents double-prefixing when models already include provider prefix
3. **Environment Variable Mirroring**: ZhipuAI key is mirrored to satisfy different LiteLLM code paths
4. **API Key Prefix Detection**: OpenRouter detected by `sk-or-` prefix without needing model keywords

### Technical Debt / Shortcuts

1. **LiteLLM + Direct SDK Hybrid**: Both `LiteLLMProvider` and `OpenAICompatibleProvider` exist - potential for consolidation
2. **Provider Detection Complexity**: Multiple detection strategies (by name, key prefix, base URL) may cause edge cases
3. **Hardcoded Model Overrides**: K2.5 temperature override is hardcoded rather than configurable

---

## Feature 8: Rust CLI (ov_cli)

### Overview

The Rust CLI (`crates/ov_cli/`) is a high-performance command-line client alternative to the Python client, featuring:
- **TUI (Terminal UI)** for interactive file browsing
- **Rich output formatting** (table, JSON, simple)
- **Async HTTP client** using `reqwest`
- **18 command categories** with subcommands

### Architecture

```
crates/ov_cli/src/
├── main.rs          # Entry point, command routing (35KB)
├── client.rs        # HTTP client with all API methods (26KB)
├── config.rs        # Config file loading
├── error.rs         # Error types
├── output.rs        # Output formatting
├── commands/        # Individual command handlers
│   ├── admin.rs     # Multi-tenant account management
│   ├── chat.rs      # Chat with vikingbot agent
│   ├── content.rs   # Read/abstract/overview/reindex
│   ├── crypto.rs    # Cryptographic key management
│   ├── filesystem.rs # ls/tree/mkdir/rm/mv/stat
│   ├── observer.rs  # System status commands
│   ├── pack.rs      # Import/export .ovpack
│   ├── relations.rs # Link/unlink resources
│   ├── resources.rs # Add resources/skills
│   ├── search.rs    # find/search/grep/glob
│   ├── session.rs   # Session management
│   └── system.rs    # Health/status/wait
├── tui/             # Terminal UI
│   ├── mod.rs       # TUI initialization, event loop
│   ├── app.rs       # App state management
│   ├── tree.rs      # Directory tree state
│   ├── ui.rs        # Ratatui rendering
│   └── event.rs     # Key event handling
└── utils.rs         # Helpers
```

### HTTP Client Implementation

```rust
pub struct HttpClient {
    http: ReqwestClient,
    base_url: String,
    api_key: Option<String>,
    agent_id: Option<String>,
}

impl HttpClient {
    pub fn new(base_url: &str, api_key: Option<String>, agent_id: Option<String>, timeout_secs: f64) -> Self {
        let http = ReqwestClient::builder()
            .timeout(Duration::from_secs_f64(timeout_secs))
            .build()
            .expect("Failed to build HTTP client");
        // ...
    }

    pub async fn get<T: DeserializeOwned>(&self, path: &str, params: &[(String, String)]) -> Result<T>
    pub async fn post<B: serde::Serialize, T: DeserializeOwned>(&self, path: &str, body: &B) -> Result<T>
    pub async fn put<B: serde::Serialize, T: DeserializeOwned>(&self, path: &str, body: &B) -> Result<T>
    pub async fn delete<T: DeserializeOwned>(&self, path: &str, params: &[(String, String)]) -> Result<T>
}
```

### Directory Zipping for Resource Upload

Local directories are zipped before upload:

```rust
fn zip_directory(&self, dir_path: &Path) -> Result<NamedTempFile> {
    let temp_file = NamedTempFile::new()?;
    let mut zip = zip::ZipWriter::new(File::create(temp_file.path())?);
    let options = FileOptions::default().compression_method(CompressionMethod::Deflated);

    let walkdir = WalkDir::new(dir_path);
    for entry in walkdir.into_iter().filter_map(|e| e.ok()) {
        if entry.path().is_file() {
            let name = path.strip_prefix(dir_path).unwrap_or(path);
            zip.start_file(name.to_string_lossy(), options)?;
            let mut file = File::open(path)?;
            std::io::copy(&mut file, &mut zip)?;
        }
    }
    zip.finish()?;
    Ok(temp_file)
}
```

### Configuration

Config loaded from `~/.openviking/ovcli.conf` (JSON format):

```rust
pub struct Config {
    pub url: String,           // Default: "http://localhost:1933"
    pub api_key: Option<String>,
    pub agent_id: Option<String>,
    pub timeout: f64,          // Default: 60.0 seconds
    pub output: String,         // "table" (default), "json", "simple"
    pub echo_command: bool,     // Echo command before output
}
```

### TUI Implementation

The TUI uses `crossterm` for terminal control and `ratatui` for rendering:

```rust
pub async fn run_tui(client: HttpClient, uri: &str) -> Result<()> {
    enable_raw_mode()?;
    io::stdout().execute(EnterAlternateScreen)?;

    let backend = CrosstermBackend::new(io::stdout());
    let mut terminal = Terminal::new(backend)?;
    let mut app = App::new(client);
    app.init(uri).await;

    loop {
        terminal.draw(|frame| ui::render(frame, &app))?;

        if ct_event::poll(Duration::from_millis(100))? {
            if let Event::Key(key) = ct_event::read()? {
                if key.kind == KeyEventKind::Press {
                    event::handle_key(&mut app, key).await;
                }
            }
        }

        if app.should_quit {
            break;
        }
    }
    // Restore terminal on exit
    disable_raw_mode()?;
    io::stdout().execute(LeaveAlternateScreen)?;
}
```

### App State

```rust
pub struct App {
    pub client: HttpClient,
    pub tree: TreeState,
    pub focus: Panel,          // Tree or Content panel
    pub content: String,        // Current file content
    pub content_scroll: u16,
    pub should_quit: bool,
    pub status_message: String,
    pub vector_state: VectorRecordsState,
    pub showing_vector_records: bool,
    pub current_uri: String,
}
```

### Key Commands

| Command | Description |
|---------|-------------|
| `ov add-resource` | Add local file/URL as resource |
| `ov ls/tree` | List directory contents |
| `ov read/abstract/overview` | Read content at L0/L1/L2 |
| `ov find/search` | Semantic search |
| `ov session new/list/get/delete` | Session management |
| `ov admin create-account` | Multi-tenant account creation |
| `ov tui` | Interactive TUI file explorer |
| `ov chat` | Chat with vikingbot agent |

### Error Handling Pattern

```rust
async fn handle_response<T: DeserializeOwned>(&self, response: reqwest::Response) -> Result<T> {
    let status = response.status();

    // Handle empty response (204 No Content)
    if status == StatusCode::NO_CONTENT || status == StatusCode::ACCEPTED {
        return serde_json::from_value(Value::Null)...;
    }

    let json: Value = response.json().await?;

    // Handle HTTP errors
    if !status.is_success() {
        let error_msg = json.get("error").and_then(|e| e.get("message"))
            .and_then(|m| m.as_str())
            .unwrap_or_else(|| format!("HTTP error {}", status));
        return Err(Error::Api(error_msg));
    }

    // Extract result from wrapped response
    let result = json.get("result").cloned().unwrap_or(json);
    serde_json::from_value(result)?
}
```

### Clever Solutions

1. **Panic Hook for Terminal Restore**: Ensures terminal state is restored even on crash
2. **Path Unescaping**: Handles shell-escaped spaces in paths with helpful suggestions
3. **Machine ID for Session**: Uses `machine_uid` crate for persistent session IDs
4. **Configurable Output Formats**: Table, JSON, simple - with compact mode

### Technical Debt

1. **Large main.rs (35KB)**: Command routing is centralized rather than distributed
2. **No Connection Pooling Visibility**: Client creates one client, unclear if connections are reused optimally
3. **Synchronous ZIP in Async Context**: `zip_directory` is sync but called from async - uses blocking internally

---

## Feature 9: HTTP Server & REST API

### Overview

FastAPI-based HTTP server providing persistent context API for AI agents. Features:
- **Multi-tenant support** with API key authentication
- **16 router categories** covering all operations
- **Bot API proxy** to Vikingbot gateway
- **Prometheus telemetry** integration

### Server Bootstrap (`openviking/server/bootstrap.py`)

```python
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--host", default=None)
    parser.add_argument("--port", type=int, default=None)
    parser.add_argument("--config", type=str, default=None)
    parser.add_argument("--workers", type=int, default=None)
    parser.add_argument("--bot", action="store_true")  # Start vikingbot gateway
    parser.add_argument("--with-bot", action="store_true")  # Enable Bot API proxy
    parser.add_argument("--bot-url", default="http://localhost:18790")

    # Load config from ov.conf
    config = load_server_config(args.config)

    # Start vikingbot gateway subprocess if --with-bot
    if args.with_bot:
        bot_process = _start_vikingbot_gateway(...)

    # Run uvicorn
    if workers > 1:
        uvicorn.run("openviking.server.app:create_app", factory=True, ...)
    else:
        uvicorn.run(app, host=config.host, port=config.port, ...)
```

### FastAPI Application Factory (`openviking/server/app.py`)

```python
def create_app(config: Optional[ServerConfig] = None, service: Optional[OpenVikingService] = None) -> FastAPI:
    @asynccontextmanager
    async def lifespan(app: FastAPI):
        # Initialize OpenVikingService
        service = OpenVikingService()
        await service.initialize()
        set_service(service)

        # Initialize APIKeyManager if api_key auth mode
        if config.auth_mode == "api_key" and config.root_api_key:
            api_key_manager = APIKeyManager(root_key=config.root_api_key, ...)
            await api_key_manager.load()
            app.state.api_key_manager = api_key_manager

        yield

        # Cleanup
        await service.close()

    app = FastAPI(title="OpenViking API", lifespan=lifespan)
    app.add_middleware(CORSMiddleware, ...)
    app.add_middleware(add_timing)  # X-Process-Time header

    # Register routers
    app.include_router(admin_router)      # /api/v1/admin
    app.include_router(resources_router)   # /api/v1/resources
    app.include_router(filesystem_router)  # /api/v1/fs
    app.include_router(search_router)     # /api/v1/search
    app.include_router(sessions_router)   # /api/v1/sessions
    app.include_router(bot_router, prefix="/bot/v1")  # Bot API proxy

    return app
```

### Router Architecture

```
openviking/server/routers/
├── __init__.py      # Exports all routers
├── admin.py         # Account/user management (ROOT/ADMIN roles)
├── bot.py           # Bot API proxy to Vikingbot
├── content.py       # Read/abstract/overview/reindex
├── debug.py         # Health checks
├── filesystem.py    # ls/tree/mkdir/rm/mv/stat
├── metrics.py       # Prometheus metrics
├── observer.py      # System observer status
├── pack.py          # Import/export .ovpack
├── relations.py     # Link relations
├── resources.py     # Add resources, temp upload
├── search.py        # find/search/grep/glob
├── sessions.py      # Session CRUD
├── stats.py         # Statistics
├── system.py        # System status, wait
└── tasks.py         # Task tracking
```

### Server Configuration (`openviking/server/config.py`)

```python
class ServerConfig(BaseModel):
    host: str = "127.0.0.1"
    port: int = 1933
    workers: int = 1
    auth_mode: str = "api_key"  # "api_key" | "trusted"
    root_api_key: Optional[str] = None
    cors_origins: List[str] = ["*"]
    with_bot: bool = False
    bot_api_url: str = "http://localhost:18790"
    encryption_enabled: bool = False
    telemetry: TelemetryConfig = Field(default_factory=TelemetryConfig)
```

### Authentication & Multi-Tenancy

**Three Auth Modes:**

1. **api_key with manager** (production): API keys resolved via `APIKeyManager`
2. **api_key without manager** (dev): Implicit ROOT/default identity
3. **trusted** (gateway): Trust `X-OpenViking-Account/User` headers

```python
async def resolve_identity(request, x_api_key, authorization,
                          x_openviking_account, x_openviking_user, x_openviking_agent):
    auth_mode = _auth_mode(request)

    if auth_mode == "trusted":
        # Validate root API key if configured
        if configured_root_api_key and not hmac.compare_digest(api_key, configured_root_api_key):
            raise UnauthenticatedError("Invalid API Key")
        # Trust headers for identity
        return ResolvedIdentity(role=Role.USER, account_id=x_openviking_account, ...)

    if api_key_manager is None:
        # Dev mode: implicit ROOT
        return ResolvedIdentity(role=Role.ROOT, account_id="default", ...)

    # Production: resolve via APIKeyManager
    identity = api_key_manager.resolve(api_key)
    return identity
```

### Role-Based Access Control

```python
def require_role(*allowed_roles: Role):
    async def _check(ctx: RequestContext = Depends(get_request_context)):
        if ctx.role not in allowed_roles:
            raise PermissionDeniedError(f"Requires role: {allowed_roles}")
        return ctx
    return Depends(_check)

# Usage:
@router.post("/admin/accounts")
async def create_account(ctx: RequestContext = require_role(Role.ROOT)):
    ...
```

### Request Flow Example: Add Resource

```python
# POST /api/v1/resources
class AddResourceRequest(BaseModel):
    path: Optional[str] = None
    temp_path: Optional[str] = None
    to: Optional[str] = None
    parent: Optional[str] = None
    reason: str = ""
    instruction: str = ""
    wait: bool = False
    timeout: Optional[float] = None
    watch_interval: float = 0

@router.post("/resources")
async def add_resource(request: AddResourceRequest, ctx: RequestContext = Depends(get_request_context)):
    service = get_service()
    execution = await run_operation(
        operation="resources.add_resource",
        fn=lambda: service.resources.add_resource(
            path=path, ctx=ctx, to=request.to, ...
        ),
    )
    return Response(status="ok", result=execution.result, telemetry=execution.telemetry)
```

### Response Model

```python
class Response(BaseModel):
    status: str          # "ok" | "error"
    result: Optional[Any] = None
    error: Optional[ErrorInfo] = None
    telemetry: Optional[Dict[str, Any]] = None

ERROR_CODE_TO_HTTP_STATUS = {
    "INVALID_ARGUMENT": 400,
    "NOT_FOUND": 404,
    "PERMISSION_DENIED": 403,
    "UNAUTHENTICATED": 401,
    "RESOURCE_EXHAUSTED": 429,
    "INTERNAL": 500,
    ...
}
```

### Admin Router: Multi-Tenant Account Management

```python
@router.post("/accounts")
async def create_account(body: CreateAccountRequest, ctx: RequestContext = require_role(Role.ROOT)):
    manager = _get_api_key_manager(request)
    user_key = await manager.create_account(body.account_id, body.admin_user_id)

    # Initialize account directories
    service = get_service()
    account_ctx = RequestContext(user=UserIdentifier(body.account_id, body.admin_user_id, "default"), role=Role.ADMIN)
    await service.initialize_account_directories(account_ctx)

    return Response(status="ok", result={"account_id": ..., "user_key": user_key})

@router.delete("/accounts/{account_id}")
async def delete_account(request, account_id: str, ctx: RequestContext = require_role(Role.ROOT)):
    # Cascade delete: AGFS data + VectorDB records + account metadata
    viking_fs = get_viking_fs()
    for prefix in ["viking://user/", "viking://agent/", "viking://session/", "viking://resources/"]:
        await viking_fs.rm(prefix, recursive=True, ctx=cleanup_ctx)

    storage = viking_fs._get_vector_store()
    await storage.delete_account_data(account_id)

    await manager.delete_account(account_id)
```

### Clever Solutions

1. **Factory Mode for Multi-Worker**: Uses import string `"openviking.server.app:create_app"` for worker processes
2. **Bot Process Management**: Spawns vikingbot as subprocess with log file handling
3. **Dependency Injection**: Global `_service` via `set_service()` / `get_service()`
4. **Telemetry Wrapper**: `run_operation()` wraps all operations for metrics
5. **Float Sanitization**: `_sanitize_floats()` replaces inf/nan with 0.0 for JSON compliance

### Technical Debt / Edge Cases

1. **Security: ROOT API Key in Dev Mode**: When no API key configured on non-localhost, server refuses to start (correct)
2. **Trusted Mode Warning**: Logs warning if trusted mode used on non-localhost
3. **Graceful Shutdown**: Uses `finally` block to cleanup bot process, but vikingbot subprocess termination could timeout

---

## Cross-Feature Integration

### Feature 7 (Multi-Provider) → Feature 9 (HTTP API)

The HTTP server exposes search endpoints that use the VikingBot providers for LLM calls:

```
/api/v1/search/find  → service.search.find()
/api/v1/search/search → service.search.search()
```

These internally use the multi-provider system for any LLM-powered operations.

### Feature 8 (Rust CLI) → Feature 9 (HTTP API)

The Rust CLI is a pure HTTP client - it doesn't call OpenViking directly, only through the REST API:

```
ov commands → HTTP client → OpenViking HTTP Server (Feature 9)
```

The CLI can be used to interact with a remotely running OpenViking server.

### Feature 9 (HTTP API) → Feature 8 (Rust CLI)

The Bot API proxy (`/bot/v1`) routes to a separate Vikingbot gateway:

```python
# In app.py
if config.with_bot:
    bot_module.set_bot_api_url(config.bot_api_url)
    # Proxies /bot/v1/* → vikingbot gateway
```

---

## Summary Table

| Aspect | Feature 7: Multi-Provider | Feature 8: Rust CLI | Feature 9: HTTP API |
|--------|---------------------------|--------------------|--------------------|
| **Architecture** | Registry + Provider classes | Rust CLI + TUI | FastAPI + Routers |
| **Key Files** | `providers/registry.py` | `crates/ov_cli/src/main.rs` | `server/app.py`, `server/routers/*` |
| **Size** | ~500 lines registry | 35KB main.rs | 16 router files |
| **Dependencies** | LiteLLM, OpenAI SDK | reqwest, ratatui, clap | FastAPI, uvicorn |
| **Auth Model** | N/A | API key header | API key / Trusted headers |
| **Multi-Tenant** | N/A | N/A | Yes (accounts/users) |
| **Testing Surface** | Unit tests on providers | Integration tests | API integration tests |

# Feature Batch 5 Deep Dive

**Date:** 2026-03-26
**Features:** Feature 13 (Browser Automation), Feature 14 (Extensibility: WASM, GraphQL, Metrics)

---

## Feature 13: Browser Automation

**Crate:** `crates/browser/`
**Priority:** SECONDARY
**Lines of Code:** ~5.1K

### Overview

The browser automation system provides headless Chrome/Chromium control via the Chrome DevTools Protocol (CDP). It supports two execution modes: **host mode** (direct browser on the host machine) and **sandboxed mode** (browser inside Docker/Podman/Apple Container).

### Architecture

```
BrowserManager (manager.rs)
    |
    +-- BrowserPool (pool.rs) - manages multiple browser instances
    |       |
    |       +-- BrowserInstance (pool.rs) - single browser with pages
    |               |
    |               +-- chromiumoxide::Browser (CDP connection)
    |               +-- Page (CDP page handle)
    |               +-- BrowserContainer (container.rs, sandboxed only)
    |
    +-- BrowserAction handlers (manager.rs)
            +-- Navigate, Screenshot, Snapshot, Click, Type, Scroll, Evaluate, Wait, etc.
```

### Key Components

#### 1. BrowserManager (`manager.rs`)

The main entry point for browser operations. Handles:

- **Request routing**: Dispatches `BrowserRequest` to appropriate handler
- **Session management**: Creates/reuses browser sessions by ID
- **Timeout handling**: Wraps operations with configurable timeouts
- **Error classification**: Distinguishes connection errors from other failures
- **Retry logic**: Automatically retries navigation with fresh session on connection death

**Notable behavior:**
- Navigate action has its own retry-with-fresh-session logic to avoid double-cleanup
- Stale connections detected for all non-Navigate actions
- Metrics integration (counters for errors, histograms for navigation duration)

#### 2. BrowserPool (`pool.rs`)

Manages browser instance lifecycle:

- **Instance reuse**: Sessions map to browser instances
- **Memory limits**: Enforces `memory_limit_percent` (default 90%) and `max_instances`
- **Idle cleanup**: Closes instances after `idle_timeout_secs` (default 300s)
- **Hard TTL**: Maximum 30-minute lifetime per instance to prevent Chromium memory leaks
- **Low-memory mode**: Auto-injects `--single-process`, `--renderer-process-limit=1` on constrained systems

**Key structures:**
```rust
struct BrowserInstance {
    browser: Browser,
    pages: HashMap<String, Page>,  // "main" page per session
    last_used: Instant,
    created_at: Instant,  // for hard TTL enforcement
    sandboxed: bool,
    container: Option<BrowserContainer>,
}
```

#### 3. Container Management (`container.rs`)

Supports three container backends with auto-detection priority:
1. **Apple Container** (macOS, VM-isolated, preferred on macOS)
2. **Podman** (daemonless, preferred on Linux)
3. **Docker** (fallback)

Key functions:
- `BrowserContainer::start()` - launches container and waits for Chrome readiness
- `wait_for_ready()` - probes `/json/version` endpoint (not just TCP connectivity)
- `browserless_session_timeout_ms()` - computes container timeout from pool settings

**Chrome launch args:**
- `--window-size=<width>,<height>`
- `--user-data-dir=<profile>` (if persistent profile enabled)
- `--disable-dev-shm-usage` (Apple Container only)
- Low-memory flags when `low_memory_threshold_mb` exceeded

#### 4. DOM Snapshot System (`snapshot.rs`)

Extracts interactive elements with numeric references:

```javascript
// Tags: a, button, input, select, textarea, [role="*"], [onclick], [tabindex]
// Each element gets: ref_, tag, role, text, href, placeholder, value, aria_label, bounds
// Stored as data-moltis-ref attribute for later retrieval
```

Key functions:
- `extract_snapshot()` - extracts full DOM snapshot with element references
- `find_element_by_ref()` - gets element center coordinates by ref
- `focus_element_by_ref()` - focuses input element by ref
- `scroll_element_into_view()` - scrolls element into view

**Security design:** No CSS selectors exposed to the model - elements identified by ref number only.

#### 5. Browser Detection (`detect.rs`)

Auto-detects Chromium-based browsers across platforms:

**Supported browsers:**
- Chrome, Chromium, Edge, Brave, Opera, Vivaldi, Arc

**Detection order:**
1. Custom path from config
2. `CHROME` environment variable
3. Platform-specific paths (macOS app bundles, Windows registry)
4. PATH lookup for known executable names

**Auto-install support:**
- macOS: `brew install --cask <browser>`
- Linux: `apt-get`, `dnf`, `pacman` with package names
- Windows: `winget install --id <package-id>`
- 180-second timeout per install attempt

#### 6. Error Handling (`error.rs`)

```rust
enum Error {
    BrowserNotAvailable,
    Io(std::io::Error),
    LaunchFailed(String),
    NavigationFailed(String),
    ElementNotFound(u32),
    InvalidSelector(String),
    JsEvalFailed(String),
    ScreenshotFailed(String),
    Timeout(String),
    PoolExhausted,
    BrowserClosed,
    ConnectionClosed(String),
    Cdp(String),
    InvalidAction(String),
    Other(Box<dyn StdError + Send + Sync>),
}
```

**Connection error detection:**
- Explicit: `BrowserClosed`, `ConnectionClosed`
- Pattern matching on message strings: "receiver is gone", "oneshot canceled", "Request timed out", "Connection closed", "AlreadyClosed"

### Key Data Structures

**BrowserRequest:**
```rust
struct BrowserRequest {
    session_id: Option<String>,
    action: BrowserAction,
    timeout_ms: u64,
    sandbox: Option<bool>,       // None = use host mode
    browser: Option<BrowserPreference>,
}
```

**BrowserResponse:**
```rust
struct BrowserResponse {
    success: bool,
    session_id: String,
    sandboxed: bool,
    error: Option<String>,
    screenshot: Option<String>,        // base64 PNG data URI
    screenshot_scale: Option<f64>,
    snapshot: Option<DomSnapshot>,
    result: Option<serde_json::Value>,  // JS eval result
    url: Option<String>,
    title: Option<String>,
    duration_ms: u64,
}
```

**BrowserConfig:**
```rust
struct BrowserConfig {
    enabled: bool,
    chrome_path: Option<String>,
    headless: bool,
    viewport_width: u32,         // default: 2560
    viewport_height: u32,       // default: 1440
    device_scale_factor: f64,   // default: 2.0 (Retina)
    max_instances: usize,       // 0 = unlimited
    memory_limit_percent: u8,   // default: 90
    idle_timeout_secs: u64,      // default: 300
    navigation_timeout_ms: u64,  // default: 30000
    chrome_args: Vec<String>,
    sandbox_image: String,      // default: "browserless/chrome"
    container_prefix: String,   // default: "moltis-browser"
    allowed_domains: Vec<String>,
    low_memory_threshold_mb: u64,// default: 2048
    persist_profile: bool,      // default: true
    profile_dir: Option<String>,
    container_host: String,    // default: "127.0.0.1"
}
```

### Notable Technical Decisions

1. **Element references over selectors**: Stable numeric refs prevent breakage from page updates
2. **Data URI for screenshots**: `data:image/png;base64,...` format allows UI display while sanitizer can strip for LLM context
3. **Profile persistence**: Default enabled, stored in `~/.moltis/browser/profile/`
4. **Container profile permissions**: World-writable (0o777) for container uid 999 access
5. **Hard TTL on instances**: 30-minute max prevents Chromium memory leak accumulation
6. **Navigate retry logic**: Automatic retry with fresh session on connection death during navigation
7. **URL validation**: Rejects URLs with suspicious LLM garbage patterns ("}}}", "assistant to=", "functions.")

### Edge Cases

- **Connection recovery**: Stale connections detected via message pattern matching, session cleaned up and user notified to navigate again
- **Memory pressure**: Checks memory before creating new instances, cleans up idle instances first
- **Redirect loop detection**: Web fetch tracks visited URLs to prevent redirect loops
- **UTF-8 boundary truncation**: Content truncated at valid character boundaries, not byte boundaries
- **Missing Chrome**: Clear error message with platform-specific install instructions

---

## Feature 14: Extensibility (WASM, GraphQL, Metrics)

### 14A: WASM Tools

**Crate:** `crates/wasm-tools/` (plus `wit/` for WIT definitions)

#### Architecture

```
crates/wasm-tools/
    calc/src/lib.rs          - Pure arithmetic expression evaluator
    web-fetch/src/lib.rs     - HTTP fetch with content extraction
    web-search/src/lib.rs    - Brave Search API integration
    (wit/                    - WebAssembly Interface Type definitions)
```

#### WIT World: `pure-tool` and `http-tool`

The `wit/` directory defines WASM component interfaces:

**Key types:**
```wit
record tool-result {
    value: option<string>,
    error: option<tool-error>,
}

record tool-error {
    code: string,
    message: string,
}

variant tool-value {
    json(string),
    text(string),
}
```

#### 14A.1: Calc Tool (`calc/src/lib.rs`)

Pure arithmetic expression evaluator. Only compiles for `target_arch = "wasm32"`.

**Supported operations:**
- Addition, Subtraction, Multiplication, Division, Modulo, Power
- Unary plus/minus
- Parentheses for precedence

**Safety limits:**
```rust
const MAX_EXPRESSION_CHARS: usize = 512;
const MAX_TOKENS: usize = 256;
const MAX_AST_DEPTH: usize = 64;
const MAX_OPERATIONS: usize = 512;
const MAX_ABS_EXPONENT: f64 = 1024.0;
const MAX_ABS_RESULT: f64 = 1.0e308;
```

**Architecture:**
1. **Tokenizer** (`tokenize()`) - Converts input string to tokens
2. **Normalizer** (`normalize_expression()`) - Produces canonical form without whitespace
3. **Parser** (`Parser`) - Builds AST with precedence handling (power is right-associative)
4. **Evaluator** (`Evaluator`) - Computes result from AST

**Key invariants:**
- Division and modulo by zero caught at evaluation time
- Negative zero normalized to positive zero
- Integer results serialized as JSON integers, floats as JSON floats
- Scientific notation supported in input

#### 14A.2: Web Fetch Tool (`web-fetch/src/lib.rs`)

Fetches web URLs with content extraction. Only compiles for `target_arch = "wasm32"`.

**Parameters:**
```rust
struct WebFetchParams {
    url: String,           // required
    extract_mode: String, // "markdown" or "text", default: "markdown"
    max_chars: usize,      // default: 50,000
}
```

**Features:**
- HTTP/HTTPS only
- Redirect following (max 5 hops, loop detection)
- Content-type detection (JSON, text/plain, HTML)
- HTML to markdown/text extraction (strips tags, scripts, styles)
- UTF-8 boundary-safe truncation
- Timeout: 10 seconds
- Max response: 2MB

**HTML extraction logic:**
```rust
fn extract_content(body: &str, content_type: &str, requested_mode: &str) -> (String, String) {
    // JSON -> pretty print
    // text/plain -> passthrough
    // HTML -> strip tags, collapse whitespace
    // default mode is markdown
}
```

#### 14A.3: Web Search Tool (`web-search/src/lib.rs`)

Brave Search API integration. Only compiles for `target_arch = "wasm32"`.

**Parameters:**
```rust
struct WebSearchParams {
    query: String,
    count: u8,          // 1-10, default 5
    country: Option<String>,
    search_lang: Option<String>,
    ui_lang: Option<String>,
    freshness: Option<String>,  // pd, pw, pm, py
}
```

**Features:**
- Brave Search API (requires API token via host)
- Returns: `{ provider, query, results: [{ title, url, description }] }`
- Timeout: 12 seconds
- Max response: 2MB

### 14B: GraphQL API

**Crate:** `crates/graphql/`
**Priority:** SECONDARY
**Lines of Code:** ~4.8K

#### Architecture

```
crates/graphql/
    lib.rs              - Schema building and exports
    schema.rs           - Schema construction (QueryRoot, MutationRoot, SubscriptionRoot)
    context.rs          - GqlContext bridging to gateway services
    types/mod.rs        - GraphQL output/input types
    mutations/mod.rs    - Mutation resolvers organized by namespace
    queries/mod.rs      - Query resolvers organized by namespace
    subscriptions/mod.rs - Subscription resolvers
    scalars.rs          - Custom scalar types (Json)
    error.rs            - Error handling helpers
```

#### Schema Building

```rust
pub type MoltisSchema = Schema<QueryRoot, MutationRoot, SubscriptionRoot>;

pub fn build_schema(
    services: Arc<Services>,
    broadcast_tx: broadcast::Sender<(String, Value)>,
) -> MoltisSchema {
    let ctx = Arc::new(GqlContext { broadcast_tx, services });
    Schema::build(QueryRoot, MutationRoot, SubscriptionRoot)
        .data(ctx)
        .finish()
}
```

#### Context Structure

```rust
pub struct GqlContext {
    pub broadcast_tx: broadcast::Sender<(String, Value)>,  // For subscriptions
    pub services: Arc<Services>,  // Direct service access, no RPC indirection
}
```

#### Key Design Decisions

1. **Direct service access**: Resolvers call domain services directly via `services!` macro, no RPC string dispatch
2. **Shared types with services**: Types defined in `types/mod.rs` use `#[derive(SimpleObject)]` for GraphQL and `#[derive(Deserialize)]` for service JSON
3. **Broadcast for subscriptions**: Real-time events via tokio broadcast channel
4. **Json scalar**: Dynamic JSON wrapped in custom scalar type

#### Example Mutation (Browser)

```rust
pub struct BrowserMutation;

#[Object]
impl BrowserMutation {
    async fn request(&self, ctx: &Context<'_>, input: Json) -> Result<Json> {
        let s = services!(ctx);
        // Returns browser response with dynamic content
        from_service_json(s.browser.request(input.0).await)
    }
}
```

#### Types Coverage

The GraphQL types module defines ~50+ output types including:
- Sessions, Chat, Cron jobs, Projects
- Channels (Telegram, Discord, etc.)
- Providers, Models, Skills
- MCP servers and tools
- Voice config, TTS/STT status
- Memory status, Hooks, Logs
- Usage, Cost tracking

### 14C: Metrics System

**Crate:** `crates/metrics/`
**Priority:** SECONDARY

#### Architecture

```
crates/metrics/
    lib.rs              - Exports and re-exports
    definitions.rs       - Metric names and label keys
    recorder.rs         - MetricsHandle and initialization
    snapshot.rs         - MetricSnapshot for historical data
    store.rs            - SqliteMetricsStore (sqlite feature)
    tracing_integration.rs - OpenTelemetry tracing integration
```

#### Key Concepts

1. **Facade pattern**: Uses `metrics` crate facade (allows switching backends)
2. **Prometheus export**: When `prometheus` feature enabled, exports Prometheus format
3. **Feature-gated**: No prometheus feature = no-op recorder
4. **Histogram buckets**: Custom buckets per metric type

#### Metrics Definitions

Comprehensive coverage across domains:

| Domain | Metrics |
|--------|---------|
| HTTP | requests_total, request_duration_seconds, requests_in_flight |
| WebSocket | connections_total, messages_sent/received |
| LLM | completions_total, completion_duration, tokens, cache, time_to_first_token |
| Session | created_total, active, messages, duration |
| Tool | executions_total, execution_duration, errors |
| Sandbox | command_executions, command_duration |
| MCP | server_connections, tool_calls, resource_reads |
| Channel | messages_received/sent, active |
| Memory | searches, embeddings, documents |
| Plugin | loaded, executions, hook_executions |
| Browser | instances_active, screenshots, navigation_duration |
| Cron | jobs_scheduled, executions, timer_loop_latency |
| Auth | login_attempts, api_key_auth |
| System | uptime_seconds, build_info |

#### Histogram Buckets

```rust
// HTTP: 1ms to 60s
HTTP_DURATION = [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0, 60.0]

// LLM: 100ms to 5 minutes
LLM_DURATION = [0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 15.0, 30.0, 60.0, 120.0, 180.0, 300.0]

// Time to first token: 10ms to 30s
TTFT = [0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.0, 5.0, 10.0, 20.0, 30.0]

// Tokens per second: 1 to 500
TOKENS_PER_SECOND = [1.0, 5.0, 10.0, 20.0, 30.0, 40.0, 50.0, 75.0, 100.0, 150.0, 200.0, 300.0, 500.0]
```

#### Recorder Initialization

```rust
pub struct MetricsRecorderConfig {
    pub enabled: bool,
    pub prefix: Option<String>,
    pub global_labels: Vec<(String, String)>,
}

pub fn init_metrics(config: MetricsRecorderConfig) -> Result<MetricsHandle> {
    // When prometheus feature enabled:
    // - Sets custom histogram buckets for specific metrics
    // - Adds global labels
    // - Returns handle for /metrics endpoint rendering
}
```

#### Usage Pattern

```rust
use moltis_metrics::{counter, gauge, histogram};

// Counter
counter!("http_requests_total", "endpoint" => "/api/chat").increment(1);

// Gauge
gauge!("active_sessions").set(42.0);

// Histogram
histogram!("request_duration_seconds").record(0.123);
```

---

## Cross-Feature Integration Points

1. **Browser + GraphQL**: `BrowserMutation.request()` resolver calls `s.browser.request()` service
2. **Browser + Metrics**: Browser actions record `browser_errors_total`, `browser_navigation_duration_seconds`, etc.
3. **WASM Tools + Browser**: WASM tools provide HTTP capability to browser/WASM sandbox
4. **GraphQL + Metrics**: GraphQL endpoint could expose metrics via subscriptions

## Observations

### Browser Automation
- Well-structured with clear separation between host and sandboxed modes
- Comprehensive error handling with connection error detection
- Memory pressure awareness prevents resource exhaustion
- Security-conscious design (no CSS selectors exposed, domain allowlisting)

### WASM Tools
- Pure tool (calc) demonstrates clean separation - pure functions, comprehensive tests
- HTTP tools use host capability for outbound requests (secure WASM sandboxing)
- WIT definitions provide typed interface between host and guest

### GraphQL
- Direct service access pattern avoids RPC indirection
- Comprehensive type coverage for all gateway services
- Strong separation: schema definition vs. resolver implementation

### Metrics
- Facade pattern allows runtime backend selection
- Comprehensive domain coverage
- Custom buckets per metric type (appropriate for LLM vs HTTP timing)
- Integration with tracing via `tracing_integration` module

---

*Generated: 2026-03-26*
*Source: crates/browser/, crates/wasm-tools/, crates/graphql/, crates/metrics/*

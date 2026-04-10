# Deep Dive: Batch 1 Features

**Date:** 2026-03-26
**Features:** AI Gateway/Multi-Provider LLM, Chat Engine & Agent Orchestration, Session Management

---

## Feature 1: AI Gateway / Multi-Provider LLM

### Overview

The AI Gateway provides unified LLM access through a pluggable provider architecture. Multiple providers (OpenAI, Anthropic, GitHub Copilot, Kimi, local models) implement a common `LlmProvider` trait, with automatic failover via `ProviderChain`.

### Core Architecture

#### Provider Trait (`moltis_agents::model::LlmProvider`)

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    fn name(&self) -> &str;
    fn id(&self) -> &str;
    async fn complete(&self, messages: &[ChatMessage], tools: &[serde_json::Value]) -> anyhow::Result<CompletionResponse>;
    fn stream(&self, messages: Vec<ChatMessage>) -> Pin<Box<dyn Stream<Item = StreamEvent> + Send + '_>>;
    fn stream_with_tools(&self, messages: Vec<ChatMessage>, tools: Vec<serde_json::Value>) -> Pin<Box<dyn Stream<Item = StreamEvent> + Send + '_>>;
    fn supports_tools(&self) -> bool { false }
    fn context_window(&self) -> u32 { 200_000 }
    fn supports_vision(&self) -> bool { false }
    fn tool_mode(&self) -> Option<moltis_config::ToolMode> { None }
    fn reasoning_effort(&self) -> Option<ReasoningEffort> { None }
}
```

Key design decisions:
- `Send + Sync` bounds enable sharing providers across async tasks
- Default `context_window` of 200K tokens for unspecified providers
- `tool_mode` allows per-provider tool behavior overrides

#### Stream Events

```rust
pub enum StreamEvent {
    Delta(String),                                    // Text delta
    ProviderRaw(serde_json::Value),                  // Raw for debugging
    ReasoningDelta(String),                          // Extended thinking
    ToolCallStart { id, name, index },              // Tool call begins
    ToolCallArgumentsDelta { index, delta },        // Tool args streaming
    ToolCallComplete { index },                      // Tool args done
    Done(Usage),                                     // Stream finished
    Error(String),
}
```

#### Completion Response

```rust
pub struct CompletionResponse {
    pub text: Option<String>,
    pub tool_calls: Vec<ToolCall>,
    pub usage: Usage,
}

pub struct Usage {
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub cache_read_tokens: u32,
    pub cache_write_tokens: u32,
}
```

### Provider Chain (Failover)

Located in `moltis_agents::provider_chain`, implements the circuit breaker pattern:

```rust
pub struct ProviderChain {
    chain: Vec<ChainEntry>,  // Primary first, then fallbacks
}
```

**Error Classification:**

```rust
pub enum ProviderErrorKind {
    RateLimit,        // 429 — failover
    AuthError,        // 401/403 — failover
    ServerError,     // 5xx — failover
    BillingExhausted, // Quota — failover
    ContextWindow,    // Token limit — NO failover (trigger compaction)
    InvalidRequest,   // 400 — NO failover (will fail everywhere)
    Unknown,          // Try failover
}
```

**Circuit Breaker Implementation:**

```rust
struct ProviderState {
    consecutive_failures: AtomicUsize,
    last_failure: Mutex<Option<Instant>>,
}
// Trips after 3 consecutive failures
// Resets after 60s cooldown
fn is_tripped(&self) -> bool {
    failures >= 3 && last_failure.elapsed() < Duration::from_secs(60)
}
```

For streaming, if primary is tripped, selects first non-tripped provider upfront (cannot retry mid-stream).

### Model ID Namespace Pattern

Providers use namespaced model IDs: `"provider::model-id"` (e.g., `"anthropic::claude-sonnet-4-20250514"`)

Helper functions in `moltis_providers::lib.rs`:
- `namespaced_model_id(provider, model_id)` — creates namespaced ID
- `split_reasoning_suffix(model_id)` — extracts `@reasoning-high` suffix
- `raw_model_id(model_id)` — strips namespace and reasoning suffix
- `capability_model_id(model_id)` — extracts model capability identifier

### Provider Implementations

| Provider | File | Notes |
|----------|------|-------|
| OpenAI | `providers/src/openai.rs` | Full-featured, streaming, tools |
| Anthropic | `providers/src/anthropic.rs` | Claude models, extended thinking |
| OpenAI Codex | `providers/src/openai_codex.rs` | GitHub Copilot |
| GitHub Copilot | `providers/src/github_copilot.rs` | OAuth discovery support |
| Kimi | `providers/src/kimi_code.rs` | Moonshot partner |
| OpenAI Compatible | `providers/src/openai_compat.rs` | OpenRouter, Ollama, etc. |
| Local GGUF | `providers/src/local_gguf/` | Local quantized models |
| Local LLM | `providers/src/local_llm/` | llama.cpp, ollama |

### Key Data Structures

#### ChatMessage (Typed)

```rust
pub enum ChatMessage {
    System { content: String },
    User { content: UserContent },
    Assistant { content: Option<String>, tool_calls: Vec<ToolCall> },
    Tool { tool_call_id: String, content: String },
}
```

#### ToolCall

```rust
pub struct ToolCall {
    pub id: String,
    pub name: String,
    pub arguments: serde_json::Value,
}
```

### Error Handling Patterns

1. **Rate Limit Retry**: Exponential backoff (2s initial, 60s max, 10 retries max)
2. **Server Error Retry**: Fixed 2s delay, 1 retry
3. **Context Window**: Propagate `AgentRunError::ContextWindowExceeded` (caller should compact)
4. **Billing Exhaustion**: No retry (user action required)

### Anthropic Extended Thinking

```rust
fn apply_thinking(&self, body: &mut serde_json::Value) {
    let budget_tokens: u64 = match effort {
        ReasoningEffort::Low => 4096,
        ReasoningEffort::Medium => 10240,
        ReasoningEffort::High => 32768,
    };
    body["thinking"] = serde_json::json!({
        "type": "enabled",
        "budget_tokens": budget_tokens,
    });
}
```

### Notable Implementation Details

1. **Shared HTTP Client**: `providers::shared_http_client()` reuses connection pools
2. **Model Discovery**: Providers can fetch model lists from API (`/v1/models`)
3. **Metrics Integration**: Tracks completions, tokens (input/output/cache), errors by provider/model
4. **Reasoning Effort Suffix**: `"model@reasoning-high"` configures extended thinking per-request

### Edge Cases

- Orphan tool messages filtered by tracking `pending_tool_call_ids`
- Tool name sanitization strips `functions_` prefix, trailing `_\d+` suffix
- Empty tool name triggers retry prompt
- Malformed tool calls (looks like text) triggers retry

---

## Feature 2: Chat Engine & Agent Orchestration

### Overview

The chat engine (`moltis-chat`) orchestrates agent runs, managing conversation context, parallel tool execution, sub-agent delegation, and prompt assembly. The core agent loop lives in `moltis-agents::runner`.

### Core Agent Loop

Located in `run_agent_loop()` (~500 lines), the main loop:

1. Builds message list with system prompt + history + user content
2. Calls LLM via `provider.complete()`
3. On tool calls: executes concurrently, handles results
4. On text only: returns result
5. Handles retries, malformation recovery, context window errors

### Tool Execution Flow

```rust
// Execute tool calls concurrently
let tool_futures: Vec<_> = response.tool_calls.iter().map(|tc| {
    async move {
        // BeforeToolCall hook
        // Execute tool
        // AfterToolCall hook
    }
}).collect();
```

### Tool Registry (`moltis_agents::tool_registry`)

```rust
pub struct ToolRegistry {
    tools: HashMap<String, ToolEntry>,
    activated: ActivatedTools,  // Runtime-activated via lazy tool discovery
}

pub trait AgentTool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, params: serde_json::Value) -> Result<serde_json::Value>;
}
```

### Tool Source Tracking

```rust
pub enum ToolSource {
    Builtin,
    Mcp { server: String },
    Wasm { component_hash: [u8; 32] },
}
```

Enables filtering tools by source (e.g., `clone_without_mcp()` for sub-agents).

### Hook System

Before/After hooks for LLM calls and tool execution:

```rust
HookPayload::BeforeLLMCall { session_key, provider, model, messages, tool_count, iteration }
HookPayload::AfterLLMCall { session_key, provider, model, text, tool_calls, ... }
HookPayload::BeforeToolCall { session_key, tool_name, arguments }
```

Hook actions: `Block(reason)`, `ModifyPayload(new_value)`, `Continue`

### Prompt Assembly

Located in `moltis_agents::prompt` (~1200 lines), builds system prompts with:
- Session runtime context
- Sandbox context
- Node info
- Voice reply suffix
- Session memory injection

### Multimodal Handling

```rust
pub enum UserContent {
    Text(String),
    Multimodal(Vec<ContentPart>),
}

pub enum ContentPart {
    Text(String),
    Image { media_type: String, data: String },
}
```

Data URI parsing: `parse_data_uri("data:image/png;base64,...")` extracts media type and base64 data.

### Tool Result Sanitization

Before feeding tool results back to LLM:

1. **Strip base64 blobs** (>= 200 char payloads) — replace with placeholder
2. **Strip hex blobs** (>= 200 hex chars) — replace with placeholder
3. **Truncate** to max_bytes with char boundary awareness

```rust
pub fn sanitize_tool_result(input: &str, max_bytes: usize) -> String {
    let mut result = strip_base64_blobs(input);
    result = strip_hex_blobs(&result);
    // truncate at char boundary...
}
```

### Vision Provider Support

When provider `supports_vision() == true`, tool results with images become multimodal content:

```rust
pub fn tool_result_to_content(result: &str, max_bytes: usize, supports_vision: bool) -> serde_json::Value {
    if !supports_vision {
        return serde_json::Value::String(sanitize_tool_result(result, max_bytes));
    }
    // Extract images and build multimodal content array
}
```

### Explicit Shell Commands

Detects `/sh ...` commands in user content for forced execution:

```rust
fn explicit_shell_command_from_user_content(user_content: &UserContent) -> Option<String> {
    // Only single-line, < 4096 chars, starts with /sh or /sh@mention
    // Returns the command to execute
}
```

### Sub-Agent Support

`run_agent_loop_with_context()` accepts:
- `tool_context`: Injected into tool call parameters (e.g., `_session_key`)
- `hook_registry`: For per-agent hook enforcement

### Error Recovery

1. **Malformed tool calls**: Retry prompt sent to LLM
2. **Empty tool names**: Retry if no named tools present
3. **Rate limits**: Exponential backoff (2s-60s, max 10 retries)
4. **Server errors**: Fixed 2s delay, 1 retry
5. **Context window**: Propagate `AgentRunError::ContextWindowExceeded`

### Runner Events (Callbacks)

```rust
pub enum RunnerEvent {
    Thinking,
    ThinkingDone,
    ToolCallStart { id, name, arguments },
    ToolCallEnd { id, name, success, error, result },
    ThinkingText(String),
    TextDelta(String),
    Iteration(usize),
    SubAgentStart { task, model, depth },
    SubAgentEnd { task, model, depth, iterations, tool_calls_made },
    RetryingAfterError { error, delay_ms },
}
```

### Notable Patterns

1. **Iteration limit**: Default 25, lazy mode 75 (3x for tool_search discovery)
2. **Message filtering**: Skips orphan tool messages (no matching assistant tool_call_id)
3. **Response sanitization**: `clean_response()` removes think tags, fixes formatting
4. **Parallel tool execution**: All tools in a turn execute concurrently
5. **Tool schema recomputation**: Each iteration re-fetches schemas for dynamic tools

---

## Feature 3: Session Management

### Overview

Session management in `moltis-sessions` provides JSONL-based persistence with SQLite-backed metadata, full-text search, and per-session state storage.

### Storage Architecture

#### JSONL Session Files

Sessions stored at: `<data_dir>/agents/<agentId>/sessions/<sessionKey>.jsonl`

```rust
pub struct SessionStore {
    pub base_dir: PathBuf,
}
```

Each line is a JSON object representing one message. File locking via `fd_lock::RwLock` for concurrent access.

#### Key Operations

```rust
impl SessionStore {
    pub async fn append(&self, key: &str, message: &serde_json::Value) -> Result<()>
    pub async fn read(&self, key: &str) -> Result<Vec<serde_json::Value>>
    pub async fn read_by_run_id(&self, key: &str, run_id: &str) -> Result<Vec<serde_json::Value>>
    pub async fn read_last_n(&self, key: &str, n: usize) -> Result<Vec<serde_json::Value>>
    pub async fn clear(&self, key: &str) -> Result<()>
    pub async fn replace_history(&self, key: &str, messages: Vec<serde_json::Value>) -> Result<()>
    pub async fn search(&self, query: &str, max_results: usize) -> Result<Vec<SearchResult>>
}
```

### Session Key Handling

Keys contain colons (e.g., `"agent:session123"`). Filenames use underscores:

```rust
pub fn key_to_filename(key: &str) -> String {
    key.replace(':', "_")
}
```

### Message Format

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "role", rename_all = "lowercase")]
pub enum PersistedMessage {
    System { content: String, created_at: Option<u64> },
    Notice { content: String, created_at: Option<u64> },  // UI-only
    User {
        content: MessageContent,  // Text or Multimodal
        audio: Option<String>,
        channel: Option<serde_json::Value>,
        seq: Option<u64>,
        run_id: Option<String>,
        created_at: Option<u64>,
    },
    Assistant {
        content: String,
        model: Option<String>,
        provider: Option<String>,
        input_tokens: Option<u32>,
        output_tokens: Option<u32>,
        duration_ms: Option<u64>,
        tool_calls: Option<Vec<PersistedToolCall>>,
        reasoning: Option<String>,
        llm_api_response: Option<serde_json::Value>,
        audio: Option<String>,
        seq: Option<u64>,
        run_id: Option<String>,
        created_at: Option<u64>,
    },
    Tool {
        tool_call_id: String,
        content: String,
        created_at: Option<u64>,
    },
    ToolResult {
        tool_call_id: String,
        tool_name: String,
        arguments: Option<serde_json::Value>,
        success: bool,
        result: Option<serde_json::Value>,
        error: Option<String>,
        reasoning: Option<String>,
        run_id: Option<String>,
        created_at: Option<u64>,
    },
}
```

### Multimodal Content

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum MessageContent {
    Text(String),
    Multimodal(Vec<ContentBlock>),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ContentBlock {
    Text { text: String },
    ImageUrl { image_url: ImageUrl },
}

pub struct ImageUrl {
    pub url: String,  // Data URI
}
```

### Media Storage

Session media files stored separately:

```rust
fn media_dir_for(&self, key: &str) -> PathBuf {
    self.base_dir.join("media").join(Self::key_to_filename(key))
}

pub async fn save_media(&self, key: &str, filename: &str, data: &[u8]) -> Result<String>
// Returns relative path: "media/<key-sanitized>/<filename>"
```

### Search Implementation

```rust
pub async fn search(&self, query: &str, max_results: usize) -> Result<Vec<SearchResult>> {
    // Case-insensitive content search across all sessions
    // Returns one hit per session (first match)
    // Snippet extracted with 40 chars before + 60 chars after query
}
```

### Session State Store (KV)

SQLite-backed key-value store scoped to `(session_key, namespace, key)`:

```rust
pub struct SessionStateStore {
    pool: sqlx::SqlitePool,
}

impl SessionStateStore {
    pub async fn get(&self, session_key: &str, namespace: &str, key: &str) -> Result<Option<String>>
    pub async fn set(&self, session_key: &str, namespace: &str, key: &str, value: &str) -> Result<()>
    pub async fn delete(&self, session_key: &str, namespace: &str, key: &str) -> Result<bool>
    pub async fn list(&self, session_key: &str, namespace: &str) -> Result<Vec<StateEntry>>
    pub async fn delete_all(&self, session_key: &str, namespace: &str) -> Result<u64>
    pub async fn delete_session(&self, session_key: &str) -> Result<u64>
}
```

State table schema:
```sql
CREATE TABLE session_state (
    session_key TEXT NOT NULL,
    namespace   TEXT NOT NULL,
    key         TEXT NOT NULL,
    value       TEXT NOT NULL,
    updated_at  INTEGER NOT NULL,
    PRIMARY KEY (session_key, namespace, key)
)
```

### Session Metadata (SQLite)

Located in `moltis_sessions::metadata`, tracks:
- Session entries with metadata
- Channel sessions (cross-channel continuity)
- Usage analytics

### Notable Patterns

1. **Append-only JSONL**: Messages only appended, never modified (except `replace_history` for compaction)
2. **File locking**: `RwLock` from `fd_lock` crate for safe concurrent reads/writes
3. **Orphan tool handling**: Tool messages without matching assistant tool_call_id are skipped on replay
4. **Run ID filtering**: Can read messages for specific agent run (`read_by_run_id`)
5. **Sequence numbers**: Client-assigned `seq` for ordering diagnostics
6. **Notice messages**: UI-only info (not sent to LLM)

### Conversions

**Session to Agent messages** (`values_to_chat_messages`):
- Converts persisted JSON to typed `ChatMessage`
- Tracks `pending_tool_call_ids` to filter orphan tool results
- `tool_result` role converted to standard `tool` messages

**Session to Chat runtime** (`to_user_content`):
- Converts `MessageContent` to agents `UserContent`
- Parses data URI images back to `ContentPart::Image`

### Compaction

`replace_history()` enables compaction when context window approaches limit. Old messages replaced with summary.

---

## Cross-Feature Integration

### Provider Chain in Chat

When `provider.complete()` fails:
1. `ProviderChain::complete()` catches error
2. Classifies error type
3. If failover-eligible, tries next provider
4. If context window, returns `AgentRunError::ContextWindowExceeded`
5. Chat runtime triggers session compaction

### Session in Chat

1. Chat loads session history via `SessionStore::read()`
2. Converts persisted messages to `ChatMessage` via `values_to_chat_messages()`
3. Appends new messages via `SessionStore::append()`
4. Tool results stored as `PersistedMessage::ToolResult`
5. State stored via `SessionStateStore` for cross-message context

### Tool Registry in Chat

1. Chat builds `ToolRegistry` with all available tools
2. Each tool is `Arc<dyn AgentTool>` for cheap cloning
3. Sub-agents get filtered registry via `clone_without_prefix()` or `clone_without_mcp()`
4. Lazy mode allows runtime tool activation via `ToolSearchTool`

---

## Technical Debt & Observations

### 1. Large Single Files
- `agents/src/runner.rs` (226K) — monolithic agent loop
- `chat/src/lib.rs` (457K) — chat engine
- `providers/src/lib.rs` (145K) — provider registry

### 2. Error Classification via String Matching
- `classify_error()` uses `err.to_string().to_lowercase().contains(...)`
- Fragile to provider error message changes
- Could be improved with structured error types

### 3. Retry Logic Duplication
- `runner.rs` and `provider_chain.rs` both have retry logic
- Different error classification strategies
- `ProviderChain` handles provider-level failover
- `runner` handles within-provider retries

### 4. Session Search Limitations
- Simple substring search, not semantic/vector
- One hit per session limitation
- No ranking by relevance

### 5. Tool Name Sanitization
- Strips `functions_` prefix (OpenAI legacy)
- Strips trailing `_\d+` suffix (Kimi indexing)
- Could conflict with legitimate tool names

---

## Summary

| Aspect | AI Gateway | Chat Engine | Sessions |
|--------|------------|-------------|----------|
| **Core Type** | `LlmProvider` trait | `run_agent_loop()` | `SessionStore` |
| **Key Pattern** | Circuit breaker failover | Tool execution loop | JSONL append |
| **Error Handling** | Classified retries | Context window propagation | N/A |
| **Storage** | Provider config | Tool registry | JSONL + SQLite |
| **Extensibility** | New provider impl | New tool impl | New metadata |
| **Size** | 145K (providers/lib.rs) | 226K (runner.rs) | 28K (store.rs) |

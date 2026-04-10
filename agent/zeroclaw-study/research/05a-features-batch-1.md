# ZeroClaw Feature Deep Dive: Batch 1

> **Project:** ZeroClaw - Personal AI Assistant
> **Source:** Feature index + src/ directory analysis + source code
> **Date:** 2026-03-26
> **Features Covered:**
> 1. Agent Loop & Orchestration
> 2. Multi-Channel Inbox
> 3. Tool System

---

## Feature 1: Agent Loop & Orchestration

### Overview

The agent loop (`src/agent/loop_.rs`, 346KB) is the core orchestration engine that handles tool dispatch, prompt construction, message classification, context management, and memory loading. It implements a classic LLM agent loop pattern with multiple provider abstraction layers.

### Key Components

#### 1.1 Agent Structure (`src/agent/agent.rs`)

The `Agent` struct is the main entry point:

```rust
pub struct Agent {
    provider: Box<dyn Provider>,
    tools: Vec<Box<dyn Tool>>,
    tool_specs: Vec<ToolSpec>,
    memory: Arc<dyn Memory>,
    observer: Arc<dyn Observer>,
    prompt_builder: SystemPromptBuilder,
    tool_dispatcher: Box<dyn ToolDispatcher>,
    memory_loader: Box<dyn MemoryLoader>,
    config: crate::config::AgentConfig,
    model_name: String,
    temperature: f64,
    workspace_dir: std::path::PathBuf,
    identity_config: crate::config::IdentityConfig,
    skills: Vec<crate::skills::Skill>,
    skills_prompt_mode: crate::config::SkillsPromptInjectionMode,
    auto_save: bool,
    memory_session_id: Option<String>,
    history: Vec<ConversationMessage>,
    classification_config: crate::config::QueryClassificationConfig,
    available_hints: Vec<String>,
    route_model_by_hint: HashMap<String, String>,
    allowed_tools: Option<Vec<String>>,
    response_cache: Option<Arc<crate::memory::response_cache::ResponseCache>>,
    tool_descriptions: Option<ToolDescriptions>,
    security_summary: Option<String>,
    autonomy_level: crate::security::AutonomyLevel,
}
```

**Builder Pattern:** The `AgentBuilder` uses a fluent builder pattern with individual setters for each field:

```rust
impl AgentBuilder {
    pub fn new() -> Self { ... }
    pub fn provider(mut self, provider: Box<dyn Provider>) -> Self { ... }
    pub fn tools(mut self, tools: Vec<Box<dyn Tool>>) -> Self { ... }
    pub fn memory(mut self, memory: Arc<dyn Memory>) -> Self { ... }
    pub fn build(self) -> Result<Agent> {
        let mut tools = self.tools.ok_or_else(|| anyhow::anyhow!("tools are required"))?;
        let allowed = self.allowed_tools.clone();
        if let Some(ref allow_list) = allowed {
            tools.retain(|t| allow_list.iter().any(|name| name == t.name()));
        }
        // ... validation and construction
    }
}
```

#### 1.2 The Core Loop (`run_tool_call_loop`)

The main agent loop is defined at line 2783 of `loop_.rs`. Key characteristics:

**Iteration Control:**
```rust
pub(crate) async fn run_tool_call_loop(
    provider: &dyn Provider,
    history: &mut Vec<ChatMessage>,
    tools_registry: &[Box<dyn Tool>],
    // ... many more parameters
) -> Result<String> {
    let max_iterations = if max_tool_iterations == 0 {
        DEFAULT_MAX_TOOL_ITERATIONS  // 10
    } else {
        max_tool_iterations
    };

    for iteration in 0..max_iterations {
        // Tool dispatch, LLM call, result processing
    }
}
```

**Loop Cancellation:** Uses `CancellationToken` for graceful shutdown:
```rust
if cancellation_token.as_ref().is_some_and(CancellationToken::is_cancelled) {
    return Err(ToolLoopCancelled.into());
}
```

**Model Switching:** The loop checks for model switch requests via a global state:
```rust
pub type ModelSwitchCallback = Arc<Mutex<Option<(String, String)>>>>;
static MODEL_SWITCH_REQUEST: LazyLock<Arc<Mutex<Option<(String, String)>>>>> =
    LazyLock::new(|| Arc::new(Mutex::new(None)));

pub fn get_model_switch_state() -> ModelSwitchCallback { ... }
```

When a switch is requested, the loop returns a `ModelSwitchRequested` error:
```rust
return Err(ModelSwitchRequested {
    provider: new_provider.clone(),
    model: new_model.clone(),
}.into());
```

#### 1.3 Tool Dispatcher Abstraction (`src/agent/dispatcher.rs`)

Two dispatcher strategies exist for parsing LLM responses:

**NativeToolDispatcher:** For providers with native function calling (OpenAI, Anthropic):
```rust
impl ToolDispatcher for NativeToolDispatcher {
    fn parse_response(&self, response: &ChatResponse) -> (String, Vec<ParsedToolCall>) {
        let text = response.text.clone().unwrap_or_default();
        let calls = response.tool_calls.iter().map(|tc| ParsedToolCall {
            name: tc.name.clone(),
            arguments: serde_json::from_str(&tc.arguments).unwrap_or_else(|e| {
                tracing::warn!("Failed to parse native tool call arguments as JSON: {e}");
                Value::Object(serde_json::Map::new())
            }),
            tool_call_id: Some(tc.id.clone()),
        }).collect();
        (text, calls)
    }
}
```

**XmlToolDispatcher:** For providers returning XML-style tool calls:
```rust
impl ToolDispatcher for XmlToolDispatcher {
    fn parse_response(&self, response: &ChatResponse) -> (String, Vec<ParsedToolCall>) {
        let text = response.text_or_empty();
        Self::parse_xml_tool_calls(text)
    }
}
```

The XML parser (`parse_xml_tool_calls`) handles multiple formats:
- `<tool_call>{"name": "tool_name", "arguments": {...}}</tool_call>`
- `<toolname><argument1>value1</argument1></toolname>` (nested tags)
- MiniMax format: `<invoke name="shell"><parameter name="command">ls</parameter></invoke>`

#### 1.4 Multi-Format XML Tool Call Parsing

The loop supports 6 different XML tool call formats:
```rust
const TOOL_CALL_OPEN_TAGS: [&str; 6] = [
    "<tool_call>", "<toolcall>", "<tool-call>",
    "<invoke>", "<minimax:tool_call>", "<minimax:toolcall>",
];
```

The parser uses regex-based extraction with balanced brace tracking:
```rust
fn extract_json_values(input: &str) -> Vec<serde_json::Value> {
    let mut values = Vec::new();
    // Attempts direct JSON parse first, then falls back to
    // character-by-character JSON object/array extraction
}
```

#### 1.5 Streaming Support

The loop implements sophisticated streaming with provider fallback:
```rust
let should_consume_provider_stream = on_delta.is_some()
    && provider.supports_streaming()
    && (request_tools.is_none() || provider.supports_streaming_tool_events());

let chat_result = if should_consume_provider_stream {
    consume_provider_streaming_response(...).await
} else {
    // Fallback to non-streaming with optional timeout
    call_provider_chat(...).await
};
```

**Draft Events:** Progress updates sent via channel:
```rust
pub enum DraftEvent {
    Clear,
    Progress(String),  // Status updates like "Thinking..."
    Content(String),   // Actual response content
}
```

#### 1.6 Vision Provider Routing

Handles image inputs by routing to a vision-capable provider when the default does not support vision:
```rust
let vision_provider_box: Option<Box<dyn Provider>> = if image_marker_count > 0
    && !provider.supports_vision()
{
    if let Some(ref vp) = multimodal_config.vision_provider {
        let vp_instance = providers::create_provider(vp, None)?;
        // ... validation
        Some(vp_instance)
    } else {
        return Err(ProviderCapabilityError { ... }.into());
    }
} else {
    None
};
```

#### 1.7 Credential Scrubbing

Tool outputs are scrubbed to prevent credential exfiltration:
```rust
static SENSITIVE_KV_REGEX: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r#"(?i)(token|api[_-]?key|password|secret|user[_-]?key|bearer|credential)["']?\s*[:=]\s*(?:"([^"]{8,})"|'([^']{8,})'|([a-zA-Z0-9_\-\.]{8,}))"#).unwrap()
});

pub(crate) fn scrub_credentials(input: &str) -> String {
    SENSITIVE_KV_REGEX.replace_all(input, |caps: &regex::Captures| {
        // Preserves first 4 chars of value for context
        let prefix = if val.len() > 4 { &val[..4] } else { "" };
        format!("{}: {}*[REDACTED]", key, prefix)
    }).to_string()
}
```

### Notable Patterns & Technical Debt

**Strengths:**
- Clean separation of concerns with dispatcher strategy pattern
- Comprehensive error handling with typed errors
- Graceful degradation with streaming fallbacks
- Security-conscious credential scrubbing

**Concerns:**
- Large file size (346KB) suggests potential for decomposition
- Global static for model switch state (`MODEL_SWITCH_REQUEST`) is not ideal for testability
- Deeply nested iteration logic with many conditional branches

---

## Feature 2: Multi-Channel Inbox

### Overview

The multi-channel inbox (`src/channels/mod.rs`, 415KB) provides a unified messaging infrastructure across 26+ platforms. Each channel implements the `Channel` trait for send, listen, and auxiliary operations.

### Key Architecture

#### 2.1 Channel Trait (`src/channels/traits.rs`)

The `Channel` trait is the contract for all messaging platforms:

```rust
#[async_trait]
pub trait Channel: Send + Sync {
    fn name(&self) -> &str;
    async fn send(&self, message: &SendMessage) -> anyhow::Result<()>;
    async fn listen(&self, tx: tokio::sync::mpsc::Sender<ChannelMessage>) -> anyhow::Result<()>;
    async fn health_check(&self) -> bool { true }
    async fn start_typing(&self, _recipient: &str) -> anyhow::Result<()> { Ok(()) }
    async fn stop_typing(&self, _recipient: &str) -> anyhow::Result<()> { Ok(()) }

    // Draft update support (progressive message editing)
    fn supports_draft_updates(&self) -> bool { false }
    async fn send_draft(&self, _message: &SendMessage) -> anyhow::Result<Option<String>> { Ok(None) }
    async fn update_draft(&self, _recipient: &str, _message_id: &str, _text: &str) -> anyhow::Result<()> { Ok(()) }

    // Streaming support
    fn supports_multi_message_streaming(&self) -> bool { false }
    fn multi_message_delay_ms(&self) -> u64 { 800 }

    // Reactions
    async fn add_reaction(&self, _channel_id: &str, _message_id: &str, _emoji: &str) -> anyhow::Result<()> { Ok(()) }

    // Message management
    async fn pin_message(&self, _channel_id: &str, _message_id: &str) -> anyhow::Result<()> { Ok(()) }
    async fn redact_message(&self, _channel_id: &str, _message_id: &str, _reason: Option<String>) -> anyhow::Result<()> { Ok(()) }
}
```

#### 2.2 Message Types

**Incoming ChannelMessage:**
```rust
pub struct ChannelMessage {
    pub id: String,
    pub sender: String,
    pub reply_target: String,
    pub content: String,
    pub channel: String,
    pub timestamp: u64,
    pub thread_ts: Option<String>,
    pub interruption_scope_id: Option<String>,
    pub attachments: Vec<MediaAttachment>,
}
```

**Outgoing SendMessage:**
```rust
pub struct SendMessage {
    pub content: String,
    pub recipient: String,
    pub subject: Option<String>,
    pub thread_ts: Option<String>,
    pub cancellation_token: Option<CancellationToken>,
}
```

#### 2.3 Channel Building (`build_channel_by_id`)

Factory function that constructs channels from config:

```rust
fn build_channel_by_id(config: &Config, channel_id: &str) -> Result<Arc<dyn Channel>> {
    match channel_id {
        "telegram" => {
            let tg = config.channels_config.telegram.as_ref()
                .context("Telegram channel is not configured")?;
            Ok(Arc::new(
                TelegramChannel::new(tg.bot_token.clone(), tg.allowed_users.clone(), tg.mention_only)
                    .with_ack_reactions(ack)
                    .with_streaming(tg.stream_mode, tg.draft_update_interval_ms)
                    .with_transcription(config.transcription.clone())
                    .with_tts(config.tts.clone())
                    .with_workspace_dir(config.workspace_dir.clone()),
            ))
        }
        "discord" => { /* similar pattern */ }
        "slack" => { /* similar pattern */ }
        other => anyhow::bail!("Unknown channel '{other}'"),
    }
}
```

**Builder Pattern:** Each channel uses a fluent builder for optional configurations:
```rust
TelegramChannel::new(token, users, mention_only)
    .with_ack_reactions(ack)
    .with_streaming(stream_mode, interval_ms)
```

#### 2.4 Channel Startup (`start_channels`)

The `start_channels` function wires up all subsystems:

```rust
pub async fn start_channels(config: Config) -> Result<()> {
    // 1. Create resilient provider with warmup
    let provider: Arc<dyn Provider> = Arc::from(
        create_resilient_provider_nonblocking(&provider_name, ...).await?
    );
    provider.warmup().await;  // Pre-establish connections

    // 2. Build all tools (same as CLI path)
    let (built_tools, delegate_handle, reaction_handle, _, ask_user_handle, escalate_handle) =
        tools::all_tools_with_runtime(...);

    // 3. Wire MCP tools (eager or deferred loading)
    if config.mcp.enabled && !config.mcp.servers.is_empty() {
        let registry = crate::tools::McpRegistry::connect_all(&config.mcp.servers).await?;
        if config.mcp.deferred_loading {
            // Deferred: register tool_search, load on demand
            deferred_section = mcp_deferred::build_deferred_tools_section(&deferred_set);
        } else {
            // Eager: register all MCP tools directly
            for name in registry.tool_names() {
                let wrapper: Arc<dyn Tool> = Arc::new(McpToolWrapper::new(name, def, registry.clone()));
                delegate_handle.write().push(Arc::clone(&wrapper));
                built_tools.push(Box::new(ArcToolRef(wrapper)));
            }
        }
    }

    // 4. Per-sender conversation tracking
    type ConversationHistoryMap = Arc<Mutex<HashMap<String, Vec<ChatMessage>>>>;
    let conversation_history: ConversationHistoryMap = Arc::new(Mutex::new(HashMap::new()));

    // 5. Spawn listener tasks for each configured channel
    // 6. Message processing loop with concurrency control
}
```

#### 2.5 Per-Sender Conversation History

Each sender gets a dedicated conversation history:
```rust
type ConversationHistoryMap = Arc<Mutex<HashMap<String, Vec<ChatMessage>>>>;
const MAX_CHANNEL_HISTORY: usize = 50;

fn add_to_history(history: &mut Vec<ChatMessage>, msg: ChatMessage) {
    history.push(msg);
    if history.len() > MAX_CHANNEL_HISTORY {
        // Proactive compaction before context window exhaustion
        let budget = history.iter().map(|m| m.content.len()).sum::<usize>();
        if budget > PROACTIVE_CONTEXT_BUDGET_CHARS { /* compact */ }
    }
}
```

#### 2.6 Concurrent Message Processing

Channels process messages with configurable parallelism:
```rust
const CHANNEL_PARALLELISM_PER_CHANNEL: usize = 4;
const CHANNEL_MIN_IN_FLIGHT_MESSAGES: usize = 8;
const CHANNEL_MAX_IN_FLIGHT_MESSAGES: usize = 64;
```

#### 2.7 Health Monitoring

```rust
const CHANNEL_HEALTH_HEARTBEAT_SECS: u64 = 30;

async fn health_check_loop(channel: Arc<dyn Channel>, channel_id: &str) {
    loop {
        tokio::time::sleep(Duration::from_secs(CHANNEL_HEALTH_HEARTBEAT_SECS)).await;
        match tokio::time::timeout(Duration::from_secs(5), channel.health_check()).await {
            Ok(true) => { /* healthy */ }
            Ok(false) => { /* unhealthy */ }
            Err(_) => { /* timeout */ }
        }
    }
}
```

### Notable Channels Implementation

#### Telegram (`src/channels/telegram.rs`, 183KB)
- Long polling for updates
- ACK reactions (emoji responses to confirm message receipt)
- Streaming draft mode with progressive updates
- Transcription and TTS integration

#### Slack (`src/channels/slack.rs`, 162KB)
- Event-based webhooks + web API polling hybrid
- Markdown block formatting
- Thread support
- Multi-channel support (channel_ids list)

#### Discord (`src/channels/discord.rs`, 85KB)
- Gateway bot connection
- Guild/server awareness
- Channel and thread navigation
- Embed formatting

### Notable Patterns & Technical Debt

**Strengths:**
- Unified `Channel` trait enables easy addition of new platforms
- Comprehensive feature coverage per platform (typing indicators, reactions, drafts)
- Per-sender conversation isolation
- Graceful reconnection with exponential backoff

**Concerns:**
- 415KB mod.rs is extremely large (likely contains implementation + configuration logic)
- Each channel implementation has significant boilerplate
- Some channels (Telegram, Slack) are much larger than others, suggesting uneven abstraction levels
- Health checking is fire-and-forget without explicit restart logic for unhealthy channels

---

## Feature 3: Tool System

### Overview

The tool system (`src/tools/mod.rs`, 50KB + `src/tools/delegate.rs`, 99KB) provides 70+ tools for the LLM to interact with the world. Tools implement the `Tool` trait and are assembled into registries.

### Key Architecture

#### 3.1 Tool Trait (`src/tools/traits.rs`)

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;
    fn spec(&self) -> ToolSpec {  // Default implementation
        ToolSpec {
            name: self.name().to_string(),
            description: self.description().to_string(),
            parameters: self.parameters_schema(),
        }
    }
}

pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,
}

pub struct ToolSpec {
    pub name: String,
    pub description: String,
    pub parameters: serde_json::Value,
}
```

#### 3.2 Tool Registry (`src/tools/mod.rs`)

**Default Tools (minimal set):**
```rust
pub fn default_tools(security: Arc<SecurityPolicy>) -> Vec<Box<dyn Tool>> {
    default_tools_with_runtime(security, Arc::new(NativeRuntime::new()))
}

pub fn default_tools_with_runtime(...) -> Vec<Box<dyn Tool>> {
    vec![
        Box::new(ShellTool::new(security.clone(), runtime)),
        Box::new(FileReadTool::new(security.clone())),
        Box::new(FileWriteTool::new(security.clone())),
        Box::new(FileEditTool::new(security.clone())),
        Box::new(GlobSearchTool::new(security.clone())),
        Box::new(ContentSearchTool::new(security)),
    ]
}
```

**Full Tool Registry (`all_tools_with_runtime`):**
- Shell, File I/O, Browser control, Git operations
- Web fetch/search, MCP tools
- Jira, Notion, Google Workspace integrations
- Cron scheduling, Memory operations
- Delegation, Swarm orchestration
- Hardware access (ESP32, Arduino, etc.)

#### 3.3 DelegateTool (`src/tools/delegate.rs`, 99KB)

The `DelegateTool` enables multi-agent workflows by spawning sub-agents:

```rust
pub struct DelegateTool {
    agents: Arc<HashMap<String, DelegateAgentConfig>>,
    security: Arc<SecurityPolicy>,
    fallback_credential: Option<String>,
    provider_runtime_options: providers::ProviderRuntimeOptions,
    depth: u32,  // Nesting depth in delegation chain
    parent_tools: Arc<RwLock<Vec<Arc<dyn Tool>>>>,
    multimodal_config: crate::config::MultimodalConfig,
    delegate_config: DelegateToolConfig,
    workspace_dir: PathBuf,
    cancellation_token: CancellationToken,
}
```

**Execution Modes:**

1. **Synchronous (default):** Blocks until sub-agent completes
2. **Background (`background: true`):** Spawns tokio task, returns task_id immediately
3. **Parallel (`parallel: [...]`):** Runs multiple agents concurrently

**Parameters Schema:**
```rust
fn parameters_schema(&self) -> serde_json::Value {
    json!({
        "type": "object",
        "properties": {
            "action": {
                "type": "string",
                "enum": ["delegate", "check_result", "list_results", "cancel_task"],
            },
            "agent": { "type": "string" },
            "prompt": { "type": "string" },
            "context": { "type": "string" },
            "background": { "type": "boolean" },
            "parallel": { "type": "array", "items": { "type": "string" } },
            "task_id": { "type": "string" },
        }
    })
}
```

#### 3.4 Background Task Management

**Result Persistence:**
```rust
fn results_dir(&self) -> PathBuf {
    self.workspace_dir.join("delegate_results")
}
```

Background tasks persist to `workspace/delegate_results/{task_id}.json`:
```rust
pub struct BackgroundDelegateResult {
    pub task_id: String,
    pub agent: String,
    pub status: BackgroundTaskStatus,
    pub output: Option<String>,
    pub error: Option<String>,
    pub started_at: String,
    pub finished_at: Option<String>,
}

pub enum BackgroundTaskStatus {
    Running,
    Completed,
    Failed,
    Cancelled,
}
```

**Result Retrieval:**
```rust
async fn handle_check_result(&self, args: &serde_json::Value) -> anyhow::Result<ToolResult> {
    let task_id = args.get("task_id").and_then(|v| v.as_str())...;
    Self::validate_task_id(task_id)?;  // UUID validation for path safety
    let result_path = self.results_dir().join(format!("{task_id}.json"));
    let content = tokio::fs::read_to_string(&result_path).await?;
    let result: BackgroundDelegateResult = serde_json::from_str(&content)?;
    // ...
}
```

**Task ID Validation (Security):**
```rust
fn validate_task_id(task_id: &str) -> Result<(), String> {
    if uuid::Uuid::parse_str(task_id).is_err() {
        return Err(format!("Invalid task_id '{task_id}': must be a valid UUID"));
    }
    Ok(())
}
```
This prevents path traversal attacks like `../../etc/passwd`.

#### 3.5 Parallel Execution

```rust
async fn execute_parallel(&self, parallel_agents: &[serde_json::Value], args: &serde_json::Value) -> anyhow::Result<ToolResult> {
    // Validate all agents exist before starting any
    for name in &agent_names {
        if !self.agents.contains_key(name) {
            return Ok(ToolResult { success: false, ... });
        }
    }

    // Spawn all agents concurrently
    let mut handles = Vec::with_capacity(agent_names.len());
    for agent_name in &agent_names {
        handles.push(tokio::spawn(async move {
            let inner = DelegateTool { agents, security, ... };
            let result = Box::pin(inner.execute_sync(&agent_name, &prompt, &args_clone)).await;
            (agent_name, result)
        }));
    }

    // Collect all results
    for handle in handles {
        match handle.await {
            Ok((agent_name, Ok(tool_result))) => { /* aggregate */ }
            Ok((agent_name, Err(e))) => { /* record failure */ }
            Err(e) => { /* join error */ }
        }
    }
}
```

#### 3.6 Cancellation Cascade

```rust
pub fn cancel_all_background_tasks(&self) {
    self.cancellation_token.cancel();
}

pub fn with_cancellation_token(mut self, token: CancellationToken) -> Self {
    self.cancellation_token = token;
    self
}
```

Each background task receives a child token that propagates cancellation.

#### 3.7 Enriched System Prompt Building

Sub-agents get an enriched system prompt:
```rust
fn build_enriched_system_prompt(
    &self,
    agent_config: &DelegateAgentConfig,
    sub_tools: &[Box<dyn Tool>],
    workspace_dir: &Path,
) -> Option<String> {
    // Compose: tools instructions, skills, workspace context, datetime, shell policy
}
```

### MCP Integration

**Deferred Loading:** MCP tools can be loaded on-demand via `ToolSearchTool`:
```rust
if config.mcp.deferred_loading {
    let deferred_set = DeferredMcpToolSet::from_registry(registry).await;
    deferred_section = mcp_deferred::build_deferred_tools_section(&deferred_set);
    tools_registry.push(Box::new(ToolSearchTool::new(deferred_set, activated)));
} else {
    // Eager: register all MCP tools
    for name in registry.tool_names() {
        let wrapper: Arc<dyn Tool> = Arc::new(McpToolWrapper::new(name, def, registry.clone()));
        delegate_handle.write().push(Arc::clone(&wrapper));
        tools_registry.push(Box::new(ArcToolRef(wrapper)));
    }
}
```

### Notable Patterns & Technical Debt

**Strengths:**
- Clean `Tool` trait with async execution
- Security-first task ID validation (UUID-only)
- Flexible execution modes (sync, background, parallel)
- Cancellation propagation via token tree
- Credential scrubbing in outputs

**Concerns:**
- 99KB `delegate.rs` handles many concerns (delegation, parallel, background, result retrieval)
- Background result files have no TTL or cleanup mechanism
- No isolation between parallel agent executions (shared `DelegateTool` config)
- Delegate tool creates new instances for each spawned agent but shares mutable state

---

## Cross-Cutting Concerns

### Error Handling

All three features use `anyhow::Result` for ergonomic error handling:
```rust
pub type Result<T> = std::result::Result<T, anyhow::Error>;
```

Typed errors for specific failure modes:
```rust
pub enum ToolLoopCancelled { }
pub struct ModelSwitchRequested { provider: String, model: String }
pub struct ProviderCapabilityError { provider: String, capability: String, message: String }
```

### Observability

Events emitted via `Observer` trait:
```rust
pub trait Observer: Send + Sync {
    fn record_event(&self, event: &ObserverEvent);
    fn record_metric(&self, metric: &ObserverMetric);
    fn flush(&self);
}

pub enum ObserverEvent {
    ToolCallStart { tool: String, arguments: Option<String> },
    LlmRequest { provider: String, model: String, messages_count: usize },
    // ...
}
```

### Security

- Credential scrubbing in tool outputs
- UUID validation for task IDs (path traversal prevention)
- Security policy enforcement in tool execution
- Capability-based tool access control via allowlists

### Async Patterns

- `async_trait` for trait-based async execution
- `tokio::spawn` for concurrent sub-agent execution
- `CancellationToken` for graceful shutdown
- `tokio::sync::mpsc` for message passing

---

## Summary

| Feature | Files | Key Pattern | Size |
|---------|-------|-------------|------|
| Agent Loop | loop_.rs (346KB), agent.rs (62KB), dispatcher.rs | LLM agent loop with dispatcher strategy | ~500KB |
| Multi-Channel | mod.rs (415KB), traits.rs, 26 channel impls | Channel trait + factory + builder | ~600KB+ |
| Tool System | mod.rs (50KB), delegate.rs (99KB), traits.rs | Tool trait + registry + delegation | ~200KB+ |

The codebase demonstrates mature Rust async patterns with strong separation of concerns. The main architectural risks are file size (especially `mod.rs` files) and the use of global statics for model switch state. The delegate tool's multi-execution-mode design is particularly sophisticated, supporting sync, background, and parallel agent execution with proper cancellation propagation.
# ZeroClaw Architecture Analysis

**Repository:** `/Users/sheldon/Documents/claw/reference/zeroclaw/`
**Generated:** 2026-03-26

## Architectural Pattern

ZeroClaw is a **modular monolithic agent framework** with plugin extensibility. It follows a **layered + plugin-based** architecture where:

1. Core orchestration layer (Agent) with dependency injection via traits
2. Pluggable provider abstraction (LLM backends)
3. Pluggable channel abstraction (messaging platforms)
4. Pluggable memory abstraction (storage backends)
5. Pluggable runtime adapter (native vs Docker)
6. Event-driven hooks system for observability

The architecture prioritizes **pluggability** at major boundaries through Rust's `dyn Trait` pattern, allowing runtime selection of implementations.

## Core Abstractions (Trait System)

### Provider Trait (`src/providers/traits.rs`)

Central to the agent is the `Provider` trait - an abstraction over LLM backends:

```rust
#[async_trait]
pub trait Provider: Send + Sync {
    fn capabilities(&self) -> ProviderCapabilities;
    fn convert_tools(&self, tools: &[ToolSpec]) -> ToolsPayload;
    async fn simple_chat(&self, message: &str, model: &str, temperature: f64) -> anyhow::Result<String>;
    async fn chat_with_system(&self, system_prompt: Option<&str>, message: &str, model: &str, temperature: f64) -> anyhow::Result<String>;
    async fn chat(&self, request: ChatRequest) -> anyhow::Result<ChatResponse>;
    async fn stream(&self, request: ChatRequest, options: StreamOptions) -> StreamResult<Box<dyn StreamExt<Item = StreamEvent> + Send>>;
}
```

**Key design decisions:**
- `ProviderCapabilities` flags enable adaptive behavior (native tool calling, vision, prompt caching)
- `ToolsPayload` enum handles provider-specific tool formats (Gemini, Anthropic, OpenAI, PromptGuided fallback)
- `StreamEvent` enum encapsulates structured streaming with explicit tool call signals

**Implementations:**
- `OpenAiProvider` - OpenAI API
- `AzureOpenAiProvider` - Azure OpenAI
- `AnthropicProvider` - Anthropic Claude
- `BedrockProvider` - AWS Bedrock
- `OllamaProvider` - Local Ollama
- `LmStudioProvider` - Local LM Studio
- `ReliableProvider` - Failover wrapper with retry logic

### Tool Trait (`src/tools/traits.rs`)

Tools are the agent's capabilities exposed to the LLM:

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;
    fn spec(&self) -> ToolSpec { ... }
}
```

**Tool system characteristics:**
- 96 tool implementations across the codebase
- Tools are boxed (`Box<dyn Tool>`) and stored in `Vec`
- Tool registry pattern in `HardwareToolRegistry` for hardware tools
- MCP (Model Context Protocol) transport via `McpTransportConn` trait

### Channel Trait (`src/channels/traits.rs`)

Messaging platform abstraction:

```rust
#[async_trait]
pub trait Channel: Send + Sync {
    fn name(&self) -> &str;
    async fn send(&self, message: &SendMessage) -> anyhow::Result<()>;
    async fn listen(&self, tx: tokio::sync::mpsc::Sender<ChannelMessage>) -> anyhow::Result<()>;
    async fn health_check(&self) -> bool;
    // ... draft support, reactions, threading
}
```

**Design patterns observed:**
- **Builder pattern** for `SendMessage` (`.in_thread()`, `.with_cancellation()`)
- **Default trait methods** for optional capabilities (reactions, drafts, threading)
- Draft support with platform-specific message IDs for progressive disclosure
- 45+ channel implementations (Discord, Slack, Telegram, WhatsApp, Matrix, etc.)

### Memory Trait (`src/memory/traits.rs`)

Storage abstraction with multiple backends:

```rust
#[async_trait]
pub trait Memory: Send + Sync {
    async fn insert(&self, entry: MemoryEntry) -> anyhow::Result<()>;
    async fn retrieve(&self, query: &str, limit: usize) -> anyhow::Result<Vec<MemoryEntry>>;
    async fn export(&self, filter: ExportFilter) -> anyhow::Result<Vec<MemoryEntry>>;
    // ... lifecycle, importance, hygiene
}
```

**Memory categories:** `Core`, `Daily`, `Conversation`, `Custom(String)`

### Other Key Traits

| Trait | Location | Purpose |
|-------|----------|---------|
| `RuntimeAdapter` | `runtime/traits.rs` | Platform abstraction (native, docker) |
| `Sandbox` | `security/traits.rs` | OS-level process isolation |
| `HookHandler` | `hooks/traits.rs` | Event interception and modification |
| `Observer` | `observability/traits.rs` | Event emission for monitoring |
| `Peripheral` | `peripherals/traits.rs` | Hardware board abstraction |
| `Transport` | `hardware/transport.rs` | Serial/I2C/SPI communication |
| `SessionBackend` | `channels/session_backend.rs` | Conversation session persistence |
| `Tunnel` | `tunnel/mod.rs` | Network tunneling (Tailscale, Pinggy) |

## Agent Architecture (`src/agent/`)

### Core Components

```
Agent (struct)
├── provider: Box<dyn Provider>
├── tools: Vec<Box<dyn Tool>>
├── memory: Arc<dyn Memory>
├── observer: Arc<dyn Observer>
├── tool_dispatcher: Box<dyn ToolDispatcher>
├── memory_loader: Box<dyn MemoryLoader>
└── prompt_builder: SystemPromptBuilder

AgentBuilder (builder pattern)
├── Fluent API with .provider(), .tools(), .memory(), etc.
└── Returns Agent via .build()
```

### Agent Loop (`src/agent/loop_.rs`)

The agent loop is a complex state machine:

1. **Message ingestion** via channels
2. **Memory loading** - loads relevant memories via `MemoryLoader`
3. **Prompt building** - constructs system prompt via `SystemPromptBuilder`
4. **LLM call** - sends to provider (potentially streaming)
5. **Tool dispatch** - parses tool calls, routes via `ToolDispatcher`
6. **Tool execution** - runs tools, records results
7. **Memory persistence** - saves conversation, updates importance
8. **Response delivery** - sends back via channel

**Tool dispatchers:**
- `NativeToolDispatcher` - uses provider's native function calling
- `XmlToolDispatcher` - parses `<tool_call>` XML tags from text responses
- `NativeToolDispatcher` as fallback

### Submodules

| Module | Files | Purpose |
|--------|-------|---------|
| `agent.rs` | 17 | Agent struct, AgentBuilder, TurnEvent enum |
| `loop_.rs` | - | Main orchestration loop |
| `dispatcher.rs` | - | Tool call parsing and routing |
| `prompt.rs` | - | System prompt construction |
| `context_analyzer.rs` | - | Query analysis and routing |
| `context_compressor.rs` | - | History pruning for context limits |
| `classifier.rs` | - | Query classification |
| `memory_loader.rs` | - | Memory retrieval strategy |
| `personality.rs` | - | Agent persona/identity |
| `thinking.rs` | - | Reasoning/thinking mode |
| `loop_detector.rs` | - | Repetition detection |

## Component Communication Patterns

### 1. Direct Trait Objects (Most Common)

Components communicate via boxed trait references:

```rust
// Agent holds trait objects directly
pub struct Agent {
    provider: Box<dyn Provider>,
    tools: Vec<Box<dyn Tool>>,
    memory: Arc<dyn Memory>,
    observer: Arc<dyn Observer>,
    // ...
}
```

### 2. Arc<dyn Trait> for Shared Access

When multiple components need shared access to the same implementation:

```rust
// Channels module
channels_by_name: Arc<HashMap<String, Arc<dyn Channel>>>
provider: Arc<dyn Provider>
memory: Arc<dyn Memory>
observer: Arc<dyn Observer>
```

### 3. tokio::sync::mpsc for Message Passing

Channels use async message passing for stream-based communication:

```rust
// Channel trait - long-running listener
async fn listen(&self, tx: tokio::sync::mpsc::Sender<ChannelMessage>) -> anyhow::Result<()>;

// Agent emits events
pub enum TurnEvent {
    Chunk { delta: String },
    Thinking { delta: String },
    ToolCall { name: String, args: serde_json::Value },
    ToolResult { name: String, output: String },
}
```

### 4. Hook System (Event-Driven)

The hooks system allows modification and interception:

```rust
pub trait HookHandler: Send + Sync {
    // Void hooks (fire-and-forget)
    async fn on_llm_input(&self, messages: &[ChatMessage], model: &str) {}
    async fn on_after_tool_call(&self, tool: &str, result: &ToolResult, duration: Duration) {}

    // Modifying hooks (can cancel or transform)
    async fn before_tool_call(&self, name: String, args: Value) -> HookResult<(String, Value)>;
    async fn before_llm_call(&self, messages: Vec<ChatMessage>, model: String) -> HookResult<(Vec<ChatMessage>, String)>;
}
```

### 5. task_local for Context

Cost tracking uses `tokio::task_local`:

```rust
tokio::task_local! {
    pub(crate) static TOOL_LOOP_COST_TRACKING_CONTEXT: Option<ToolLoopCostTrackingContext>;
}
```

### 6. tokio::sync::CancellationToken

For interruptible operations (streaming, long-running tool calls):

```rust
pub struct SendMessage {
    pub cancellation_token: Option<CancellationToken>,
}
```

## Design Patterns in Use

### 1. Builder Pattern

`AgentBuilder` provides fluent configuration:

```rust
let agent = AgentBuilder::new()
    .provider(Box::new(provider))
    .tools(tools)
    .memory(Arc::new(memory))
    .observer(Arc::new(observer))
    .build()?;
```

### 2. Factory Pattern

Runtime selection via factory functions:

```rust
// Runtime factory
pub fn create_runtime(config: &RuntimeConfig) -> anyhow::Result<Box<dyn RuntimeAdapter>> {
    match config.kind.as_str() {
        "native" => Ok(Box::new(NativeRuntime::new())),
        "docker" => Ok(Box::new(DockerRuntime::new(config.docker.clone()))),
        // ...
    }
}

// Tunnel factory
pub fn create_tunnel(config: &TunnelConfig) -> Result<Option<Box<dyn Tunnel>>>;

// Provider cache with fallback
pub struct ReliableProvider { providers: Vec<(String, Box<dyn Provider>)> }
```

### 3. Registry Pattern

Hardware device and tool registries:

```rust
// Hardware registry
pub struct DeviceRegistry { tools: HashMap<String, Box<dyn Tool>> }
impl DeviceRegistry {
    pub fn register(&mut self, name: &str, tool: Box<dyn Tool>);
    pub fn get(&self, name: &str) -> Option<&Box<dyn Tool>>;
    pub fn into_tools(self) -> Vec<Box<dyn Tool>>;
}

// Board registry (USB VID/PID lookup)
pub fn lookup_board(vid: u16, pid: u16) -> Option<&'static BoardInfo>;
```

### 4. Strategy Pattern

Tool dispatchers provide interchangeable parsing strategies:

```rust
pub trait ToolDispatcher: Send + Sync {
    async fn dispatch(&self, response: &str, tools: &[Box<dyn Tool>]) -> Vec<ParsedToolCall>;
}

// Multiple implementations
pub struct NativeToolDispatcher;
pub struct XmlToolDispatcher;
```

### 5. Adapter Pattern

Provider implementations adapt different API formats:

```rust
pub struct OpenAiProvider { /* ... */ }
pub struct AnthropicProvider { /* ... */ }
// All implement the same Provider trait
```

### 6. Template Method Pattern

Channel trait with default implementations:

```rust
pub trait Channel: Send + Sync {
    async fn health_check(&self) -> bool { true }  // Default
    async fn start_typing(&self, _recipient: &str) -> Ok(()) {}  // No-op default
    // Override only what you need
}
```

### 7. Decorator/Wrapper Pattern

`ReliableProvider` wraps a primary provider with retry/fallback logic:

```rust
pub struct ReliableProvider {
    providers: Vec<(String, Box<dyn Provider>)>,
    // ...
}
```

### 8. Null Object Pattern

`NoopObserver`, `NoopSandbox`, `NoneTunnel` provide no-op implementations:

```rust
pub struct NoopObserver;
pub struct NoopSandbox;
pub struct NoneTunnel;
```

## Module Map and Responsibilities

### Core Modules (`src/`)

| Module | Files | Responsibility |
|--------|-------|----------------|
| `agent/` | 17 | Orchestration loop, context management, evaluation |
| `providers/` | 21 | LLM backend implementations and routing |
| `channels/` | 45+ | Messaging platform integrations |
| `tools/` | 96 | Tool implementations |
| `memory/` | 26 | Memory storage and retrieval |
| `config/` | 6 | Configuration management and schema |
| `runtime/` | - | Native/Docker execution environments |
| `security/` | 25 | Auth, sandboxing, trust management |
| `hooks/` | - | Event system and hook handlers |
| `observability/` | - | Metrics, tracing, logging |
| `gateway/` | 8 | HTTP/WebSocket gateway server |
| `cron/` | - | Scheduled task execution |
| `hardware/` | 14+ | Hardware discovery, peripheral management |
| `peripherals/` | - | Peripheral trait and implementations |
| `skills/` | - | Skill system and discovery |
| `sop/` | - | Standard operating procedures |
| `identity/` | - | Agent identity and persona |
| `i18n/` | - | Internationalization |
| `rag/` | - | Retrieval-augmented generation |
| `tunnel/` | - | Network tunneling |

### Workspace Crates (`crates/`)

| Crate | Purpose |
|-------|---------|
| `robot-kit` | Robot hardware abstraction layer |
| `aardvark-sys` | Total Phase Aardvark adapter bindings |

### Applications (`apps/`)

| App | Purpose |
|-----|---------|
| `tauri/` | Desktop GUI application |

## Data Flow Example: Message Processing

```
1. Channel (e.g., Telegram)
   └── Receives webhook → ChannelMessage
   └── Sends to orchestrator via mpsc

2. Orchestrator (in channels/mod.rs)
   └── Receives ChannelMessage
   └── Validates/authenticates
   └── Creates/retrieves session
   └── Calls run_agent_session()

3. Agent Loop (agent/loop_.rs)
   └── MemoryLoader.load() → retrieves relevant memories
   └── SystemPromptBuilder.build() → constructs prompt
   └── Provider.chat() → LLM request
   └── If streaming: processes TurnEvent stream
   └── Parses response for tool calls
   └── ToolDispatcher.dispatch() → routes to tools

4. Tools
   └── Execute via tool.execute(args)
   └── Return ToolResult

5. Memory
   └── Save conversation
   └── Update importance scores
   └── Prune if needed

6. Response
   └── Channel.send() → delivers to user
   └── Observer emits events
```

## Configuration Architecture

ZeroClaw uses a layered configuration system:

1. **Config struct** - main configuration (YAML deserialized)
2. **ChannelConfig trait** - channel-specific configuration
3. **Provider configuration** - per-provider settings
4. **Runtime configuration** - native vs docker settings

Example configuration structure:
```yaml
provider:
  default: openai
  openai:
    model: gpt-4o
channels:
  telegram:
    bot_token: ${TELEGRAM_BOT_TOKEN}
gateway:
  host: 127.0.0.1
  port: 8080
runtime:
  kind: native
memory:
  backend: sqlite
```

## Observations

### Strengths

1. **Excellent trait abstraction** - All major components are pluggable via traits
2. **Comprehensive error handling** - `anyhow::Result` throughout with specific error types
3. **Rich streaming support** - StreamEvent with structured tool call signals
4. **Multi-layer retry/fallback** - Provider-level and HTTP-level retries
5. **Security consciousness** - Sandbox abstraction, secrets management, security policies
6. **Observability built-in** - Observer trait with comprehensive event types

### Architectural Concerns

1. **Large monolithic `channels/mod.rs`** - Single 7000+ line file for channel orchestration
2. **Feature-gated complexity** - Many optional features make the architecture harder to follow
3. **Some inconsistent patterns** - Some traits use `Arc<dyn>` while others use `Box<dyn>`

### Notable Design Decisions

1. **Task-local for context** - Using `tokio::task_local!` for request-scoped data (cost tracking, provider fallback info)
2. **CancellationToken propagation** - Allows interruptible streaming and tool calls
3. **Draft message pattern** - Progressive message updates for better UX
4. **Importance-weighted memory** - Memory entries have importance scores for prioritized retrieval

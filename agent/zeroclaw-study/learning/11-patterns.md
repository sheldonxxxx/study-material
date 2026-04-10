# ZeroClaw Design Patterns

**Repository:** `/Users/sheldon/Documents/claw/reference/zeroclaw/`
**Generated:** 2026-03-26

## Pattern Catalog

Patterns observed in ZeroClaw codebase with file references and code evidence.

---

## 1. Builder Pattern

**Purpose:** Fluent configuration with validation before construction

**Evidence:** `src/agent/agent.rs`

```rust
pub struct AgentBuilder {
    provider: Option<Box<dyn Provider>>,
    tools: Vec<Box<dyn Tool>>,
    memory: Option<Arc<dyn Memory>>,
    observer: Option<Arc<dyn Observer>>,
    // ...
}

impl AgentBuilder {
    pub fn provider(&mut self, provider: Box<dyn Provider>) -> &mut Self {
        self.provider = Some(provider);
        self
    }

    pub fn tools(&mut self, tools: Vec<Box<dyn Tool>>) -> &mut Self {
        self.tools = tools;
        self
    }

    pub fn memory(&mut self, memory: Arc<dyn Memory>) -> &mut Self {
        self.memory = Some(memory);
        self
    }

    pub fn build(&self) -> anyhow::Result<Agent> {
        // Validation and construction
        let agent = Agent {
            provider: self.provider.context("Provider not set")?,
            // ...
        };
        Ok(agent)
    }
}
```

**Usage:**
```rust
let agent = AgentBuilder::new()
    .provider(Box::new(provider))
    .tools(tools)
    .memory(Arc::new(memory))
    .observer(Arc::new(observer))
    .build()?;
```

---

## 2. Factory Pattern

**Purpose:** Runtime selection of implementations based on configuration

**Evidence:** `src/runtime/mod.rs`

```rust
pub fn create_runtime(config: &RuntimeConfig) -> anyhow::Result<Box<dyn RuntimeAdapter>> {
    match config.kind.as_str() {
        "native" => Ok(Box::new(NativeRuntime::new())),
        "docker" => Ok(Box::new(DockerRuntime::new(config.docker.clone()))),
        _ => anyhow::bail!("Unknown runtime kind: {}", config.kind),
    }
}
```

**Evidence:** `src/tunnel/mod.rs`

```rust
pub fn create_tunnel(config: &TunnelConfig) -> Result<Option<Box<dyn Tunnel>>>;
```

**Evidence:** `src/providers/router.rs`

```rust
pub struct ReliableProvider {
    providers: Vec<(String, Box<dyn Provider>)>,
    // Tries providers in order with fallback
}
```

---

## 3. Registry Pattern

**Purpose:** Centralized registration and lookup of implementations

**Evidence:** `src/hardware/registry.rs`

```rust
pub struct DeviceRegistry {
    tools: HashMap<String, Box<dyn Tool>>,
}

impl DeviceRegistry {
    pub fn register(&mut self, name: &str, tool: Box<dyn Tool>) {
        self.tools.insert(name.to_string(), tool);
    }

    pub fn get(&self, name: &str) -> Option<&Box<dyn Tool>> {
        self.tools.get(name)
    }

    pub fn into_tools(self) -> Vec<Box<dyn Tool>> {
        self.tools.into_values().collect()
    }
}
```

**Evidence:** `src/hardware/board.rs` (USB VID/PID lookup)

```rust
pub fn lookup_board(vid: u16, pid: u16) -> Option<&'static BoardInfo>;
```

---

## 4. Strategy Pattern

**Purpose:** Interchangeable algorithms for tool call parsing

**Evidence:** `src/agent/dispatcher.rs`

```rust
pub trait ToolDispatcher: Send + Sync {
    async fn dispatch(&self, response: &str, tools: &[Box<dyn Tool>]) -> Vec<ParsedToolCall>;
}

// Multiple implementations
pub struct NativeToolDispatcher;   // Uses provider's native function calling
pub struct XmlToolDispatcher;     // Parses <tool_call> XML tags
```

**Usage in Agent:**
```rust
pub struct Agent {
    tool_dispatcher: Box<dyn ToolDispatcher>,
    // ...
}

// Can swap between NativeToolDispatcher and XmlToolDispatcher
```

---

## 5. Adapter Pattern

**Purpose:** Uniform interface over different provider APIs

**Evidence:** `src/providers/traits.rs`

```rust
pub trait Provider: Send + Sync {
    fn capabilities(&self) -> ProviderCapabilities;
    fn convert_tools(&self, tools: &[ToolSpec]) -> ToolsPayload;
    async fn chat(&self, request: ChatRequest) -> anyhow::Result<ChatResponse>;
    async fn stream(&self, request: ChatRequest, options: StreamOptions) -> StreamResult<...>;
}
```

**Implementations adapting different APIs:**
```rust
pub struct OpenAiProvider { /* adapts OpenAI API */ }
pub struct AnthropicProvider { /* adapts Anthropic API */ }
pub struct BedrockProvider { /* adapts AWS Bedrock API */ }
pub struct OllamaProvider { /* adapts Ollama API */ }
pub struct LmStudioProvider { /* adapts LM Studio API */ }
```

---

## 6. Template Method Pattern

**Purpose:** Default implementations for optional capabilities

**Evidence:** `src/channels/traits.rs`

```rust
pub trait Channel: Send + Sync {
    async fn health_check(&self) -> bool { true }  // Default implementation

    async fn start_typing(&self, _recipient: &str) -> Ok(()) {}  // No-op default

    async fn send_draft(&self, _draft: &DraftMessage) -> anyhow::Result<()>
        { anyhow::bail!("Draft not supported") }  // Default = unsupported

    // Override only what you need
    async fn send(&self, message: &SendMessage) -> anyhow::Result<()>;
    async fn listen(&self, tx: tokio::sync::mpsc::Sender<ChannelMessage>) -> anyhow::Result<()>;
}
```

---

## 7. Decorator/Wrapper Pattern

**Purpose:** Add behavior around a primary implementation

**Evidence:** `src/providers/reliable.rs`

```rust
pub struct ReliableProvider {
    providers: Vec<(String, Box<dyn Provider>)>,  // Primary + fallbacks
    retry_policy: RetryPolicy,
}

impl Provider for ReliableProvider {
    async fn chat(&self, request: ChatRequest) -> anyhow::Result<ChatResponse> {
        // Try primary provider
        // On failure, try fallbacks in order
        // Apply retry policy with exponential backoff
    }
}
```

---

## 8. Null Object Pattern

**Purpose:** Provide default no-op implementations

**Evidence:** Multiple files

```rust
// src/observability/noop.rs
pub struct NoopObserver;

impl Observer for NoopObserver {
    async fn on_event(&self, _event: &Event) {}
}

// src/security/noop.rs
pub struct NoopSandbox;

// src/tunnel/mod.rs
pub struct NoneTunnel;
```

**Usage:**
```rust
// When no observer configured
AgentBuilder::new()
    .observer(Arc::new(NoopObserver))
    .build()
```

---

## 9. Trait Object Pattern

**Purpose:** Runtime polymorphism for plugin systems

**Evidence:** Throughout codebase

```rust
// Agent holds multiple trait objects
pub struct Agent {
    provider: Box<dyn Provider>,           // Single implementation
    tools: Vec<Box<dyn Tool>>,              // Multiple implementations
    memory: Arc<dyn Memory>,                // Shared ownership
    observer: Arc<dyn Observer>,            // Shared ownership
}

// Collection of channels
channels_by_name: Arc<HashMap<String, Arc<dyn Channel>>>
```

---

## 10. Builder for Messages

**Purpose:** Fluent construction of complex messages

**Evidence:** `src/channels/message.rs`

```rust
pub struct SendMessage {
    pub text: String,
    pub thread: Option<ThreadRef>,
    pub cancellation_token: Option<CancellationToken>,
    pub metadata: HashMap<String, String>,
}

impl SendMessage {
    pub fn text<T: Into<String>>(mut self, text: T) -> Self {
        self.text = text.into();
        self
    }

    pub fn in_thread(mut self, thread: ThreadRef) -> Self {
        self.thread = Some(thread);
        self
    }

    pub fn with_cancellation(mut self, token: CancellationToken) -> Self {
        self.cancellation_token = Some(token);
        self
    }
}
```

**Usage:**
```rust
channel.send(
    SendMessage::new()
        .text("Hello")
        .in_thread(thread_ref)
        .with_cancellation(token)
).await?;
```

---

## 11. Resource Acquisition Pattern

**Purpose:** Bundled setup/teardown for resources

**Evidence:** `src/hardware/mod.rs`

```rust
pub struct HardwareSession {
    device: Device,
    // ...
}

impl HardwareSession {
    pub fn new(vid: u16, pid: u16) -> anyhow::Result<Self> {
        let device = Device::open(vid, pid)?;
        Ok(Self { device })
    }
}

impl Drop for HardwareSession {
    fn drop(&mut self) {
        self.device.close();
    }
}
```

---

## 12. State Machine Pattern

**Purpose:** Explicit state transitions for complex flows

**Evidence:** `src/agent/loop_.rs`

```rust
pub enum AgentState {
    Idle,
    AwaitingResponse,
    ProcessingToolCalls,
    Streaming,
    Error,
}

// State transitions managed explicitly
match self.state {
    AgentState::Idle => {
        self.state = AgentState::AwaitingResponse;
        // send request
    }
    AgentState::AwaitingResponse => {
        // handle streaming or blocking response
    }
    // ...
}
```

---

## 13. Event/Observer Pattern

**Purpose:** Decoupled notification for state changes

**Evidence:** `src/observability/traits.rs`

```rust
pub trait Observer: Send + Sync {
    async fn on_event(&self, event: &Event);
}

pub enum Event {
    LlmCall { messages: Vec<ChatMessage>, model: String },
    ToolCall { name: String, args: Value, result: ToolResult },
    TurnComplete { turn_id: String, duration_ms: u64 },
    // ...
}
```

---

## 14. Hook Pattern

**Purpose:** Interception and modification of operations

**Evidence:** `src/hooks/traits.rs`

```rust
pub trait HookHandler: Send + Sync {
    // Void hooks - fire and forget
    async fn on_llm_input(&self, messages: &[ChatMessage], model: &str) {}
    async fn on_after_tool_call(&self, tool: &str, result: &ToolResult, duration: Duration) {}

    // Modifying hooks - can transform or cancel
    async fn before_tool_call(&self, name: String, args: Value) -> HookResult<(String, Value)>;
    async fn before_llm_call(&self, messages: Vec<ChatMessage>, model: String) -> HookResult<(Vec<ChatMessage>, String)>;
}
```

---

## 15. Streaming Iterator Pattern

**Purpose:** Progressive result delivery with structured events

**Evidence:** `src/providers/traits.rs`

```rust
pub enum StreamEvent {
    Chunk { delta: String },
    Thinking { delta: String },
    ToolCall { name: String, args: serde_json::Value },
    ToolResult { name: String, output: String },
    Done,
}

pub struct StreamOptions {
    pub include_usage: bool,
    pub include_stop_reason: bool,
}

pub type StreamResult<T> = anyhow::Result<T>;
```

---

## 16. Capability Flags Pattern

**Purpose:** Runtime feature detection for adaptive behavior

**Evidence:** `src/providers/traits.rs`

```rust
#[derive(Default)]
pub struct ProviderCapabilities {
    pub supports_function_calling: bool,
    pub supports_vision: bool,
    pub supports_prompt_caching: bool,
    pub supports_streaming: bool,
    // ...
}

// Usage
impl Provider for OpenAiProvider {
    fn capabilities(&self) -> ProviderCapabilities {
        ProviderCapabilities {
            supports_function_calling: true,
            supports_vision: self.model.supports_vision(),
            ..Default::default()
        }
    }
}

// Agent adapts behavior based on capabilities
if provider.capabilities().supports_function_calling {
    use_native_dispatcher();
}
```

---

## Pattern Summary Table

| Pattern | Purpose | Key Evidence |
|---------|---------|--------------|
| Builder | Fluent configuration | `AgentBuilder` |
| Factory | Runtime implementation selection | `create_runtime()`, `create_tunnel()` |
| Registry | Centralized lookup | `DeviceRegistry`, `lookup_board()` |
| Strategy | Interchangeable algorithms | `ToolDispatcher` trait + implementations |
| Adapter | Uniform interface | `Provider` trait implementations |
| Template Method | Default implementations | `Channel` trait defaults |
| Decorator | Behavior wrapping | `ReliableProvider` |
| Null Object | Default no-op | `NoopObserver`, `NoopSandbox` |
| Trait Object | Runtime polymorphism | `Box<dyn Provider>`, `Arc<dyn Memory>` |
| Builder (Messages) | Fluent message construction | `SendMessage` builder |
| RAII | Resource lifecycle | `HardwareSession` with `Drop` |
| State Machine | Explicit transitions | `AgentState` enum |
| Observer | Decoupled notification | `Observer` trait + `Event` enum |
| Hook | Operation interception | `HookHandler` trait |
| Streaming Iterator | Progressive results | `StreamEvent` enum |
| Capability Flags | Feature detection | `ProviderCapabilities` struct |

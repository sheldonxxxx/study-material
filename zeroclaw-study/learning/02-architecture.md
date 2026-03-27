# ZeroClaw Architecture

**Repository:** `/Users/sheldon/Documents/claw/reference/zeroclaw/`
**Generated:** 2026-03-26

## Architectural Overview

ZeroClaw is a **modular monolithic agent framework** with plugin extensibility. The architecture follows a **layered + plugin-based** pattern where core orchestration (Agent) sits at the center, surrounded by pluggable abstractions for LLM providers, messaging channels, memory storage, and runtime environments.

```
┌─────────────────────────────────────────────────────────────┐
│                      Channels (45+)                        │
│  Discord, Slack, Telegram, WhatsApp, Matrix, etc.         │
├─────────────────────────────────────────────────────────────┤
│                      Agent Loop                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Provider   │  │   Tools     │  │      Memory         │ │
│  │  (LLM)      │  │  (96 impl)  │  │   (SQLite-backed)   │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│  Hooks │ Observability │ Security │ Runtime (native/Docker)│
└─────────────────────────────────────────────────────────────┘
```

## Core Architectural Principles

### 1. Trait-Based Pluggability

All major components implement traits that enable runtime substitution:

| Boundary | Trait | File | Purpose |
|----------|-------|------|---------|
| LLM Backend | `Provider` | `src/providers/traits.rs` | Abstract over OpenAI, Anthropic, Ollama, etc. |
| Messaging | `Channel` | `src/channels/traits.rs` | Abstract over Discord, Slack, Telegram, etc. |
| Storage | `Memory` | `src/memory/traits.rs` | Abstract over SQLite, in-memory, etc. |
| Execution | `RuntimeAdapter` | `src/runtime/traits.rs` | Abstract over native vs Docker |
| Isolation | `Sandbox` | `src/security/traits.rs` | OS-level process isolation |
| Tools | `Tool` | `src/tools/traits.rs` | Tool execution interface |
| Observability | `Observer` | `src/observability/traits.rs` | Event emission |

### 2. Dependency Injection via Builder

The `AgentBuilder` provides fluent configuration:

```rust
// src/agent/agent.rs
let agent = AgentBuilder::new()
    .provider(Box::new(provider))
    .tools(tools)
    .memory(Arc::new(memory))
    .observer(Arc::new(observer))
    .build()?;
```

### 3. Trait Object Storage

Components hold trait objects for runtime polymorphism:

```rust
pub struct Agent {
    provider: Box<dyn Provider>,
    tools: Vec<Box<dyn Tool>>,
    memory: Arc<dyn Memory>,
    observer: Arc<dyn Observer>,
    tool_dispatcher: Box<dyn ToolDispatcher>,
}
```

## Module Responsibilities

### Core Modules (`src/`)

| Module | Files | Responsibility |
|--------|-------|----------------|
| `agent/` | 17 | Orchestration loop, context management, turn events, tool dispatch |
| `providers/` | 21 | LLM backend implementations (OpenAI, Anthropic, Bedrock, Ollama, LM Studio) |
| `channels/` | 45+ | Messaging platform integrations and session management |
| `tools/` | 96 | Tool implementations (file ops, web fetch, git, shell, etc.) |
| `memory/` | 26 | Memory storage, retrieval, importance scoring, hygiene |
| `security/` | 25 | Auth, sandboxing, trust management, encryption |
| `hooks/` | - | Event interception hooks (before/after LLM calls, tool calls) |
| `observability/` | - | Prometheus metrics, tracing, event emission |
| `runtime/` | - | Native and Docker execution environments |
| `gateway/` | 8 | HTTP/WebSocket gateway server (Axum-based) |
| `hardware/` | 14+ | Hardware discovery, USB VID/PID lookup, peripheral management |
| `config/` | 6 | YAML configuration schema and loading |
| `skills/` | - | Skill system for capability discovery |
| `sop/` | - | Standard operating procedures |
| `identity/` | - | Agent persona and personality |
| `tunnel/` | - | Network tunneling (Tailscale, Pinggy) |
| `rag/` | - | Retrieval-augmented generation support |

### Workspace Crates

| Crate | Path | Purpose |
|-------|------|---------|
| `zeroclawlabs` | `./` | Main CLI + library root |
| `robot-kit` | `crates/robot-kit/` | Robot hardware abstraction |
| `aardvark-sys` | `crates/aardvark-sys/` | Total Phase Aardvark I2C/SPI/GPIO bindings |
| `tauri` | `apps/tauri/` | Desktop GUI application |

### External Targets

| Target | Path | Purpose |
|--------|------|---------|
| Web Frontend | `web/` | Vite + React TypeScript frontend |
| ESP32 Firmware | `firmware/esp32/` | ESP32 main firmware |
| ESP32-UI | `firmware/esp32-ui/` | ESP32 display/UI firmware |
| STM32 Nucleo | `firmware/nucleo/` | STM32 Nucleo firmware |
| Raspberry Pi Pico | `firmware/pico/` | Pi Pico firmware |
| Python Bindings | `python/zeroclaw_tools/` | Python package |

## Communication Patterns

### 1. Direct Trait Objects (Most Common)

Components communicate via `Box<dyn Trait>` or `Arc<dyn Trait>`:

```rust
// Agent holds trait objects
provider: Box<dyn Provider>,
tools: Vec<Box<dyn Tool>>,

// Shared access via Arc
memory: Arc<dyn Memory>,
observer: Arc<dyn Observer>,
```

### 2. tokio::sync::mpsc for Message Passing

Channels use async message passing for streaming:

```rust
// Channel listener pattern
async fn listen(&self, tx: tokio::sync::mpsc::Sender<ChannelMessage>) -> anyhow::Result<()>;

// Turn events for streaming responses
pub enum TurnEvent {
    Chunk { delta: String },
    Thinking { delta: String },
    ToolCall { name: String, args: serde_json::Value },
    ToolResult { name: String, output: String },
}
```

### 3. Hook System (Event-Driven Interception)

```rust
pub trait HookHandler: Send + Sync {
    // Void hooks (fire-and-forget)
    async fn on_llm_input(&self, messages: &[ChatMessage], model: &str) {}
    async fn on_after_tool_call(&self, tool: &str, result: &ToolResult, duration: Duration) {}

    // Modifying hooks (can intercept and transform)
    async fn before_tool_call(&self, name: String, args: Value) -> HookResult<(String, Value)>;
    async fn before_llm_call(&self, messages: Vec<ChatMessage>, model: String) -> HookResult<(Vec<ChatMessage>, String)>;
}
```

### 4. tokio::task_local for Request-Scoped Context

```rust
tokio::task_local! {
    pub(crate) static TOOL_LOOP_COST_TRACKING_CONTEXT: Option<ToolLoopCostTrackingContext>;
}
```

### 5. CancellationToken for Interruptible Operations

```rust
pub struct SendMessage {
    pub cancellation_token: Option<CancellationToken>,
}

// Usage: propagation through streaming and long-running tool calls
```

## Agent Loop Data Flow

```
1. Channel receives message
   └── ChannelMessage via mpsc

2. Orchestrator (channels/mod.rs)
   └── Validates/authenticates
   └── Creates/retrieves session
   └── Calls run_agent_session()

3. Agent Loop (agent/loop_.rs)
   ├── MemoryLoader.load() → retrieves relevant memories
   ├── SystemPromptBuilder.build() → constructs system prompt
   ├── Provider.chat() → LLM request
   ├── If streaming: processes TurnEvent stream
   ├── Parses response for tool calls
   └── ToolDispatcher.dispatch() → routes to tools

4. Tool Execution
   └── tool.execute(args) → ToolResult

5. Memory
   └── Save conversation, update importance scores, prune if needed

6. Response Delivery
   └── Channel.send() → delivers to user
   └── Observer emits events
```

## Configuration Architecture

Layered configuration with environment variable support:

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

## Notable Design Decisions

| Decision | Rationale | Evidence |
|----------|-----------|----------|
| Trait objects over generics | Runtime flexibility for plugin systems | `Box<dyn Provider>` everywhere |
| Arc vs Box for shared access | Multiple owners need shared memory | `Arc<dyn Memory>`, `Arc<dyn Observer>` |
| task_local for request context | Thread-safe request-scoped data | Cost tracking context |
| CancellationToken propagation | Interruptible streaming/operations | Present in SendMessage, streaming calls |
| Draft message pattern | Progressive disclosure for long responses | `draft()` method on SendMessage |
| Importance-weighted memory | Prioritized retrieval | `MemoryEntry` has importance field |
| Builder pattern for Agent | Fluent configuration with validation | `AgentBuilder::new()` |
| Default trait methods | Optional capabilities without boilerplate | Channel trait health_check() default |

## Architectural Concerns

1. **Large monolithic files** - `channels/mod.rs` is 7000+ lines
2. **Feature-gated complexity** - Optional features make architecture harder to trace
3. **Inconsistent Arc/Box usage** - Some traits use `Arc<dyn>` while others use `Box<dyn>`
4. **Deep feature堆叠** - Some modules have complex nested conditional logic

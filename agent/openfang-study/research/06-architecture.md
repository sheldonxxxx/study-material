# OpenFang Architecture Analysis

## Architectural Pattern: Layered + Event-Driven Agent OS

OpenFang follows a **layered architecture** with **event-driven inter-component communication**. The system is structured as an "Agent Operating System" -- a runtime environment for deploying and orchestrating AI agents with multiple communication channels.

The architecture separates concerns into:
1. **Kernel Layer** -- orchestration, scheduling, event bus, agent lifecycle
2. **Runtime Layer** -- agent execution loop, LLM interaction, tool execution
3. **Abstraction Layer** -- shared types and traits
4. **Integration Layer** -- channel adapters, skills, extensions
5. **Interface Layer** -- HTTP API, CLI

---

## Dependency Graph

```
openfang-types          (shared types, no internal deps)
       |
       v
openfang-wire           (OFP protocol, serialization)
openfang-memory         (SQLite persistence)
openfang-skills         (skill registry)
       ^
       |
openfang-runtime -----> (runtime execution, LLM drivers)
       |                         ^
       |                         |
       v                         |
openfang-kernel ----------------+  (kernel orchestrates runtime)
       |
       v
openfang-channels        (external integrations)
openfang-hands           (tool execution)
openfang-extensions      (extension framework)
       |
       +-------> openfang-api (HTTP server)
                 openfang-cli  (CLI entry point)
```

**Key Dependency Rules:**
- `openfang-types` has zero internal dependencies (pure type definitions)
- `openfang-runtime` depends only on `types`, `memory`, `skills` -- no kernel dependency
- `openfang-kernel` depends on `runtime` but passes a `KernelHandle` trait to avoid circular deps
- `openfang-api` depends only on `openfang-kernel` (receives `Arc<OpenFangKernel>`)

---

## Kernel Responsibilities (`openfang-kernel`)

The kernel is the **central coordinator** -- it does not execute agent logic but orchestrates all subsystems.

### Core Components

| Component | Responsibility |
|-----------|----------------|
| `AgentRegistry` | Tracks all running agents, their state, manifests |
| `AgentScheduler` | Cron/schedule-based agent invocation |
| `EventBus` | Pub/sub event distribution with history |
| `CapabilityManager` | RBAC permissions for agent capabilities |
| `Supervisor` | Monitors agent health, restarts crashed agents |
| `MeteringEngine` | Cost tracking per agent, per model |
| `WorkflowEngine` | Multi-step workflow orchestration |
| `TriggerEngine` | Event-driven triggers for proactive agents |
| `AuthManager` | Session management, API key auth |
| `DeliveryTracker` | Message delivery receipts (bounded LRU) |
| `CronScheduler` | Time-based job scheduling |
| `ApprovalManager` | Human-in-the-loop approval gating |
| `AutoReplyEngine` | Rule-based auto-response |
| `HookRegistry` | Plugin lifecycle hooks |
| `PairingManager` | Device pairing for local agents |

### Kernel Handle Trait (Runtime -> Kernel Bridge)

The `KernelHandle` trait (`openfang-runtime/src/kernel_handle.rs`) allows the runtime to call kernel operations without a direct dependency:

```rust
#[async_trait]
pub trait KernelHandle: Send + Sync {
    async fn spawn_agent(&self, manifest_toml: &str, parent_id: Option<&str>) -> Result<(String, String), String>;
    async fn send_to_agent(&self, agent_id: &str, message: &str) -> Result<String, String>;
    fn list_agents(&self) -> Vec<AgentInfo>;
    fn kill_agent(&self, agent_id: &str) -> Result<(), String>;
    fn memory_store(&self, key: &str, value: serde_json::Value) -> Result<(), String>;
    fn memory_recall(&self, key: &str) -> Result<Option<serde_json::Value>, String>;
    fn find_agents(&self, query: &str) -> Vec<AgentInfo>;
    async fn task_post(&self, title: &str, description: &str, assigned_to: Option<&str>, created_by: Option<&str>) -> Result<String, String>;
    async fn task_claim(&self, agent_id: &str) -> Result<Option<serde_json::Value>, String>;
    async fn task_complete(&self, task_id: &str, result: &str) -> Result<(), String>;
    async fn task_list(&self, status: Option<&str>) -> Result<Vec<serde_json::Value>, String>;
    async fn publish_event(&self, event_type: &str, payload: serde_json::Value) -> Result<(), String>;
    async fn knowledge_add_entity(&self, entity: Entity) -> Result<String, String>;
    async fn knowledge_add_relation(&self, relation: Relation) -> Result<String, String>;
    async fn knowledge_query(&self, pattern: GraphPattern) -> Result<Vec<GraphMatch>, String>;
    async fn cron_create(&self, ...) -> Result<String, String>;
    // ... and more
}
```

This trait inversion breaks the circular dependency: kernel depends on runtime, runtime uses trait to call kernel.

---

## Runtime Responsibilities (`openfang-runtime`)

The runtime is the **agent execution engine** -- it runs the agent loop, manages LLM interactions, and executes tools.

### Core Execution Loop (`agent_loop.rs`)

The `run_agent_loop` function is the heart of OpenFang:

```
User Message
    |
    v
Memory Recall (vector or text search)
    |
    v
Build Prompt (system + memories + history)
    |
    v
LLM Call ---> [Tool Calls?] ---> Execute Tool ---> Loop
    |
    v
Save to Session (conversation history)
    |
    v
Response
```

**Key loop characteristics:**
- Max 50 iterations before giving up (prevents infinite loops)
- Max 3 retries with exponential backoff for rate-limited APIs
- Max 5 continuations for max-token truncation
- Max 20 message history before auto-trim
- Phantom action detection (prevents hallucinated completions)
- Context budget enforcement (token counting and truncation)

### LLM Driver Abstraction

Multiple LLM providers supported via the `LlmDriver` trait:

```rust
pub trait LlmDriver: Send + Sync {
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError>;
    async fn stream(&self, request: CompletionRequest) -> Result<StreamEvent, LlmError>;
}
```

Implementations:
- `openai.rs` -- OpenAI-compatible API
- `anthropic.rs` -- Anthropic Claude
- `gemini.rs` -- Google Gemini
- `copilot.rs` -- GitHub Copilot
- `claude_code.rs` -- Claude Code mode
- `qwen_code.rs` -- Qwen Code
- `fallback.rs` -- Fallback driver

### Tool Execution

Tools are defined via `ToolDefinition` and executed through:
- `ToolRunner` -- orchestration
- `Sandbox` -- WASM sandbox for untrusted code
- `DockerSandbox` -- Docker container isolation
- `PythonRuntime` -- Python script execution

### Memory Integration

Runtime uses `MemorySubstrate` from `openfang-memory` for:
- `recall()` -- text-based memory search
- `recall_with_embedding_async()` -- vector similarity search
- `remember()` -- storing memories
- Session management (conversation history)

---

## Event Bus Architecture

The `EventBus` provides pub/sub inter-component communication:

```rust
pub struct EventBus {
    sender: broadcast::Sender<Event>,           // Global broadcast
    agent_channels: DashMap<AgentId, broadcast::Sender<Event>>,  // Per-agent
    history: Arc<RwLock<VecDeque<Event>>>,     // Ring buffer (1000 events)
}
```

**Event routing:**
- `EventTarget::Agent(id)` -- delivered to specific agent channel
- `EventTarget::Broadcast` -- sent to all agents
- `EventTarget::System` -- system-wide events
- `EventTarget::Pattern` -- pattern-matched delivery

Events are stored in a history ring buffer for replay and debugging.

---

## API Layer (`openfang-api`)

The API server is built on **Axum** with a layered middleware stack:

```
Request
   |
   v
Rate Limiter (GCRA)
   |
   v
Auth Middleware (API key or session)
   |
   v
Security Headers
   |
   v
Request Logging
   |
   v
Compression
   |
   v
Tracing
   |
   v
CORS
   |
   v
Routes
```

### Key Design: AppState Bridge

```rust
pub struct AppState {
    kernel: Arc<OpenFangKernel>,
    started_at: Instant,
    peer_registry: Option<Arc<PeerRegistry>>,
    bridge_manager: tokio::sync::Mutex<BridgeManager>,
    channels_config: tokio::sync::RwLock<ChannelsConfig>,
    shutdown_notify: Arc<tokio::sync::Notify>,
    clawhub_cache: dashmap::DashMap<String, ClawhubEntry>,
    provider_probe_cache: ProbeCache,
}
```

The API routes access kernel functionality exclusively through `AppState`, which holds `Arc<OpenFangKernel>`. This maintains the kernel as the single source of truth.

---

## Channel Integration (`openfang-channels`)

Channels are pluggable adapters following the `ChannelAdapter` trait:

| Channel | File | Protocol |
|---------|------|----------|
| Discord | `discord.rs` | Discord Bot API |
| Slack | `slack.rs` | Slack API |
| Telegram | `telegram.rs` | Telegram Bot API |
| WhatsApp | `whatsapp.rs` | WhatsApp Web |
| Email | `email.rs` | SMTP/IMAP |
| Matrix | `matrix.rs` | Matrix Protocol |
| and 30+ more | various | various |

The `BridgeManager` (`bridge.rs`, 71KB) orchestrates all channel connections.

---

## Data Flow Examples

### Agent Spawn Flow
```
CLI: openfang agents spawn agent.toml
   |
   v
API: POST /api/agents
   |
   v
Routes: spawn_agent()
   |
   v
Kernel: AgentRegistry::spawn()
   |
   v
AgentManifest: parse and validate
   |
   v
Workspace: create directory structure
   |
   v
Agent Loop: run_agent_loop() begins
   |
   v
EventBus: AgentStarted event published
```

### Message Handling Flow
```
External Channel (e.g., Discord message)
   |
   v
ChannelBridge: receives and normalizes
   |
   v
EventBus: publishes AgentMessage event
   |
   v
Agent: receives via subscription
   |
   v
Runtime: run_agent_loop() executes
   |
   v
LLM: generates response
   |
   v
ChannelBridge: sends response back
   |
   v
DeliveryTracker: records receipt
```

### Inter-Agent Communication
```
Agent A: sends message to Agent B via tool
   |
   v
Runtime: calls KernelHandle::send_to_agent()
   |
   v
Kernel: looks up Agent B in registry
   |
   v
Kernel: creates Task for Agent B
   |
   v
Scheduler: queues task
   |
   v
Agent B: receives via EventBus subscription
   |
   v
Agent B: processes in agent_loop()
```

---

## Design Patterns in Use

### 1. Trait-Based Abstraction
Used extensively for:
- `LlmDriver` -- multiple LLM providers
- `KernelHandle` -- kernel/runtime decoupling
- `ChannelAdapter` -- pluggable integrations
- `EmbeddingDriver` -- vector search providers

### 2. Event Bus (Observer Pattern)
`EventBus` provides pub/sub for loose coupling between components. Agents subscribe to relevant events without knowing the publisher.

### 3. Actor-like Agent Model
Each agent has:
- Independent lifecycle (spawn, run, stop, kill)
- Per-agent message serialization via `agent_msg_locks: DashMap<AgentId, Arc<Mutex<()>>>`
- State machine: Idle -> Running -> Waiting -> Stopped

### 4. Strategy Pattern
LLM driver selection at runtime based on agent configuration:
```rust
let driver: Arc<dyn LlmDriver> = match provider {
    "openai" => Arc::new(OpenAiDriver::new()),
    "anthropic" => Arc::new(AnthropicDriver::new()),
    // ...
};
```

### 5. Registry Pattern
Multiple registries track system state:
- `AgentRegistry` -- running agents
- `SkillRegistry` -- installed skills
- `HandRegistry` -- tool packages
- `PeerRegistry` -- OFP network peers

### 6. Builder Pattern
Used for complex object construction:
- `AgentManifest::builder()` for agent configuration
- `CompletionRequest::builder()` for LLM requests
- `Router::builder()` for Axum routes

### 7. Plugin Architecture
- `openfang-skills` -- extensibility via skill packages
- `openfang-extensions` -- integration framework
- `openfang-hands` -- curated autonomous capability packages
- WASM sandboxing for untrusted skill code

### 8. Circuit Breaker
`auth_cooldown` module implements provider cooldown tracking to prevent hammering rate-limited APIs.

---

## Communication Patterns

| Pattern | Mechanism | Use Case |
|---------|-----------|----------|
| Direct calls | `Arc<Kernel>` method invocation | API -> Kernel, Kernel -> Runtime |
| Trait callbacks | `KernelHandle` trait | Runtime -> Kernel |
| Pub/sub | `EventBus::publish()` / `subscribe()` | Agent events, system notifications |
| Channel messaging | `broadcast::channel` | Per-agent event delivery |
| Request/Response | `tokio::sync::oneshot` | Tool execution results |
| Shared state | `Arc<T>`, `RwLock`, `Mutex`, `DashMap` | Concurrent access to registries |

---

## Session and State Management

### Session Model
Each agent has a `Session` containing:
- Conversation history (`Vec<Message>`)
- Context window (token budget)
- Working memory
- Agent-specific configuration

Sessions are persisted to SQLite via `MemorySubstrate`.

### Concurrency Control
```rust
// Per-agent message serialization
agent_msg_locks: DashMap<AgentId, Arc<tokio::sync::Mutex<()>>>

// Global registries (concurrent reads, exclusive writes)
registry: RwLock<AgentRegistry>
model_catalog: RwLock<ModelCatalog>
```

---

## Key Architectural Decisions

### 1. Kernel/Runtime Split
**Decision:** Separate orchestration (kernel) from execution (runtime).

**Rationale:** Avoids circular dependencies, allows testing runtime independently, enables different execution strategies.

### 2. Trait-Based Dependency Injection
**Decision:** Use `KernelHandle` trait instead of direct kernel dependency in runtime.

**Rationale:** The runtime needs kernel capabilities but shouldn't depend on the concrete kernel implementation. Trait allows mocking and different kernel implementations.

### 3. Event Bus for Inter-Agent Communication
**Decision:** Agents communicate via events, not direct calls.

**Rationale:** Loose coupling, natural support for pub/sub patterns, event history for debugging.

### 4. SQLite for Persistence
**Decision:** Use SQLite via `rusqlite` for memory and session storage.

**Rationale:** Zero-configuration, ACID compliant, sufficient for single-machine agent OS, simple deployment.

### 5. MessagePack for Wire Protocol
**Decision:** Use MessagePack (`rmp-serde`) for OFP (OpenFang Protocol) serialization.

**Rationale:** Compact binary format, faster than JSON, good Rust support.

### 6. No Agent Dependency on Channels
**Decision:** Agents are channel-agnostic; channels bridge to events.

**Rationale:** Same agent can work across multiple channels; channel logic is isolated.

---

## Security Architecture

| Mechanism | Component |
|-----------|-----------|
| API Key Auth | `middleware::auth` |
| Session Auth | `session_auth` |
| RBAC Permissions | `CapabilityManager` |
| Credential Storage | `CredentialResolver` (vault -> dotenv -> env) |
| TLS/HTTPS | `rustls` / `native-tls` |
| File Permissions | `restrict_permissions()` (0600 on Unix) |
| WASM Sandboxing | `WasmSandbox` for untrusted code |
| Docker Sandboxing | `DockerSandbox` for tool isolation |
| Audit Trail | `AuditLog` (Merkle hash chain) |
| Delivery Verification | `DeliveryTracker` receipts |
| Input Sanitization | Channel formatters, HTML escaping |

---

## Startup Sequence

```
1. CLI: parse command, load config
2. Kernel: load config, initialize all subsystems
   - MemorySubstrate: open SQLite
   - AgentRegistry: load agent manifests
   - EventBus: initialize
   - CapabilityManager: load permissions
   - SkillRegistry: scan skills/
   - HandRegistry: scan hands/
   - ExtensionRegistry: scan extensions/
   - CronScheduler: load cron jobs
   - WebToolsContext: initialize search/fetch
   - BrowserManager: initialize Playwright
   - PeerNode: start OFP networking
3. API: build router with AppState
4. ChannelBridge: start all configured channels
5. BackgroundAgents: start autonomous agents
6. Daemon: begin listening on HTTP port
7. ConfigHotReload: spawn watcher task
```

---

## Summary

OpenFang's architecture reflects its nature as an **Agent Operating System**:

- **Layered design** with clear separation: types -> memory/wire -> runtime -> kernel -> channels/api
- **Event-driven** inter-component communication via `EventBus`
- **Trait-based abstractions** for extensibility (LLM drivers, channels, skills)
- **Actor-like agents** with independent lifecycles and message-driven behavior
- **Kernel as orchestrator** -- it coordinates subsystems but doesn't execute agent logic
- **Runtime as execution engine** -- runs the agent loop, manages LLM interactions, executes tools
- **Rich plugin ecosystem** via skills, hands, extensions, and channel adapters

The architecture prioritizes **extensibility** (pluggable everything), **observability** (event history, metering, audit), **security** (sandboxing, RBAC, credential management), and **resilience** (circuit breakers, delivery receipts, session repair).

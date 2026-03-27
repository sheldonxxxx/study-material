# OpenFang Architecture

## Overview

OpenFang is a Rust-based **Agent Operating System** — a runtime environment for deploying and orchestrating AI agents across multiple communication channels. The architecture follows a **layered design** with **event-driven inter-component communication**.

## Architectural Decisions

### 1. Kernel/Runtime Split

**Decision:** Separate orchestration (kernel) from execution (runtime).

| Aspect | Kernel (`openfang-kernel`) | Runtime (`openfang-runtime`) |
|--------|---------------------------|----------------------------|
| Role | Coordinator, registry, scheduler | Execution engine |
| Responsibility | Agent lifecycle, routing, events | Agent loop, LLM calls, tools |
| Dependencies | Depends on runtime | Depends only on types, memory, skills |

**Rationale:** Avoids circular dependencies, enables testing runtime independently, supports different execution strategies.

**Evidence:** `openfang-runtime/src/kernel_handle.rs` — runtime uses `KernelHandle` trait to call kernel operations without direct dependency.

```rust
#[async_trait]
pub trait KernelHandle: Send + Sync {
    async fn spawn_agent(&self, manifest_toml: &str, parent_id: Option<&str>) -> Result<(String, String), String>;
    async fn send_to_agent(&self, agent_id: &str, message: &str) -> Result<String, String>;
    fn list_agents(&self) -> Vec<AgentInfo>;
    // ... 20+ methods
}
```

### 2. Trait-Based Dependency Injection

**Decision:** Use `KernelHandle` trait instead of concrete kernel dependency in runtime.

**Rationale:** Runtime needs kernel capabilities but should not depend on concrete implementation. Trait enables mocking and alternate kernel implementations.

**Evidence:** `openfang-runtime/src/kernel_handle.rs` defines the bridge. Kernel implements the trait; runtime receives `Arc<dyn KernelHandle>`.

### 3. Event Bus for Inter-Agent Communication

**Decision:** Agents communicate via events, not direct calls.

**Rationale:** Loose coupling, natural support for pub/sub patterns, event history for debugging and replay.

**Evidence:** `crates/openfang-runtime/src/event_bus.rs` (referenced in architecture docs):

```rust
pub struct EventBus {
    sender: broadcast::Sender<Event>,                      // Global broadcast
    agent_channels: DashMap<AgentId, broadcast::Sender<Event>>,  // Per-agent
    history: Arc<RwLock<VecDeque<Event>>>,                // Ring buffer (1000 events)
}
```

### 4. SQLite for Persistence

**Decision:** Use SQLite via `rusqlite` for memory and session storage.

**Rationale:** Zero-configuration, ACID compliant, sufficient for single-machine Agent OS, simple deployment.

**Evidence:** `crates/openfang-memory/src/substrate.rs` — `MemorySubstrate` wraps SQLite with WAL mode:

```rust
conn.execute_batch("PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000;");
```

### 5. MessagePack for Wire Protocol

**Decision:** Use MessagePack (`rmp-serde`) for OFP (OpenFang Protocol) serialization.

**Rationale:** Compact binary format, faster than JSON, good Rust support.

**Evidence:** `crates/openfang-wire/src/lib.rs`

### 6. No Agent Dependency on Channels

**Decision:** Agents are channel-agnostic; channels bridge to events.

**Rationale:** Same agent works across multiple channels; channel logic isolated for maintainability.

**Evidence:** `crates/openfang-channels/src/bridge.rs` — `BridgeManager` dispatches to kernel events, not agent logic.

---

## Module Responsibilities and Boundaries

### Crate Dependency Graph

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

### Module Responsibilities

| Crate | Responsibility | Key Files |
|-------|---------------|-----------|
| `openfang-types` | Shared type definitions, no internal deps | `Message`, `Agent`, `Task`, `ToolDefinition` |
| `openfang-kernel` | Orchestration, scheduling, event bus, agent lifecycle | `AgentRegistry`, `AgentScheduler`, `EventBus`, `CapabilityManager` |
| `openfang-runtime` | Agent execution loop, LLM interaction, tool execution | `agent_loop.rs`, `tool_runner.rs`, `drivers/` |
| `openfang-memory` | SQLite persistence, vector search, knowledge graph | `substrate.rs`, `session.rs`, `knowledge.rs` |
| `openfang-wire` | MessagePack serialization, OFP protocol | `lib.rs`, `message.rs`, `peer.rs` |
| `openfang-channels` | 40+ channel adapters (Discord, Slack, Telegram, etc.) | `bridge.rs`, `discord.rs`, `slack.rs` |
| `openfang-skills` | Skill loading and execution | `loader.rs`, `registry.rs`, `clawhub.rs` |
| `openfang-api` | HTTP API server (Axum), 140+ endpoints | `server.rs`, `routes.rs` |
| `openfang-extensions` | Extension framework, credential vault, OAuth2 | `vault.rs`, `oauth.rs` |
| `openfang-hands` | Autonomous agent configurations | `registry.rs`, `bundled.rs` |
| `openfang-desktop` | Tauri desktop app, system tray | `lib.rs`, `tray.rs` |
| `openfang-migrate` | OpenClaw migration engine | `openclaw.rs`, `report.rs` |

---

## Component Communication

### Communication Patterns

| Pattern | Mechanism | Use Case |
|---------|-----------|----------|
| Direct calls | `Arc<Kernel>` method invocation | API -> Kernel, Kernel -> Runtime |
| Trait callbacks | `KernelHandle` trait | Runtime -> Kernel |
| Pub/sub | `EventBus::publish()` / `subscribe()` | Agent events, system notifications |
| Channel messaging | `broadcast::channel` | Per-agent event delivery |
| Request/Response | `tokio::sync::oneshot` | Tool execution results |
| Shared state | `Arc<T>`, `RwLock`, `Mutex`, `DashMap` | Concurrent access to registries |

### Event Bus Architecture

The `EventBus` provides pub/sub inter-component communication with event routing:

- `EventTarget::Agent(id)` — delivered to specific agent channel
- `EventTarget::Broadcast` — sent to all agents
- `EventTarget::System` — system-wide events
- `EventTarget::Pattern` — pattern-matched delivery

Events stored in history ring buffer (1000 events) for replay and debugging.

### Data Flow: Agent Spawn

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

### Data Flow: Message Handling

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

---

## Kernel vs Runtime Separation Pattern

### The Core Split

The **kernel** is the **central coordinator** — it does NOT execute agent logic but orchestrates all subsystems. The **runtime** is the **execution engine** — it runs the agent loop, manages LLM interactions, and executes tools.

### Why This Split Exists

1. **Circular dependency avoidance** — kernel depends on runtime; runtime uses `KernelHandle` trait to call back to kernel
2. **Testability** — runtime can be tested with a mock `KernelHandle`
3. **Flexibility** — different execution strategies possible without changing orchestration

### Kernel Components

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

### Runtime Components

| Component | Responsibility |
|-----------|----------------|
| `agent_loop.rs` | Main execution loop, 50-iteration limit, LLM orchestration |
| `tool_runner.rs` | Tool execution with capability enforcement |
| `drivers/` | LLM provider implementations (OpenAI, Anthropic, Gemini, etc.) |
| `event_bus.rs` | Local event pub/sub |
| `sandbox.rs` | WASM execution sandbox with dual metering |
| `loop_guard.rs` | Infinite loop prevention |

### The Execution Loop (`agent_loop.rs`)

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

### Trait Inversion: KernelHandle

```rust
// Runtime defines what it needs from kernel
#[async_trait]
pub trait KernelHandle: Send + Sync {
    async fn spawn_agent(&self, manifest_toml: &str, parent_id: Option<&str>) -> Result<(String, String), String>;
    async fn send_to_agent(&self, agent_id: &str, message: &str) -> Result<String, String>;
    fn list_agents(&self) -> Vec<AgentInfo>;
    fn kill_agent(&self, agent_id: &str) -> Result<(), String>;
    async fn memory_store(&self, key: &str, value: serde_json::Value) -> Result<(), String>;
    async fn memory_recall(&self, key: &str) -> Result<Option<serde_json::Value>, String>;
    // ... more methods
}
```

Kernel implements this trait; runtime receives `Arc<dyn KernelHandle>`.

---

## Layered Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                    Interface Layer                       │
│         HTTP API (Axum)  │  CLI  │  Desktop (Tauri)    │
├─────────────────────────────────────────────────────────┤
│                  Integration Layer                      │
│    Channels (40+)  │  Skills  │  Extensions  │  Hands  │
├─────────────────────────────────────────────────────────┤
│                   Kernel Layer                          │
│  AgentRegistry │ EventBus │ Scheduler │ Auth │ Metering │
├─────────────────────────────────────────────────────────┤
│                  Runtime Layer                          │
│    Agent Loop  │  LLM Drivers  │  Tool Runner  │  WASM  │
├─────────────────────────────────────────────────────────┤
│                  Abstraction Layer                      │
│              Types  │  Wire  │  Memory (SQLite)        │
└─────────────────────────────────────────────────────────┘
```

---

## Security Architecture

| Layer | Mechanism | Component |
|-------|-----------|-----------|
| API Key Auth | `middleware::auth` | `openfang-api` |
| Session Auth | `session_auth` | `openfang-api` |
| RBAC Permissions | `CapabilityManager` | `openfang-kernel` |
| Credential Storage | `CredentialVault` (AES-256-GCM) | `openfang-extensions` |
| TLS/HTTPS | `rustls` / `native-tls` | System |
| File Permissions | `restrict_permissions()` (0600) | Runtime |
| WASM Sandboxing | `WasmSandbox` + dual metering | `openfang-runtime` |
| Docker Sandboxing | `DockerSandbox` | `openfang-runtime` |
| Audit Trail | `AuditLog` (Merkle hash chain) | `openfang-runtime` |
| Taint Tracking | Lattice-based flow analysis | `openfang-types` |
| Manifest Signing | Ed25519 signatures | `openfang-types` |

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

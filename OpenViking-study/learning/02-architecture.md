# OpenViking Architecture

**Repository:** `/Users/sheldon/Documents/claw/reference/OpenViking/`
**Analysis Date:** 2026-03-27
**Source:** Research analysis from `01-topology.md` and `06-architecture.md`

---

## 1. Architectural Decisions and Rationale

### 1.1 Python-Primary with Rust Optimization

**Decision:** Python handles business logic; Rust provides performant CLI/TUI operations.

**Rationale:**
- Python's ecosystem enables rapid development of complex business logic (FastAPI, Pydantic, httpx, LiteLLM)
- Rust provides low-latency CLI operations and terminal UI rendering
- Communication via subprocess execution (`os.execv`), not FFI complexity

**Evidence:**
- `openviking/` - Python core package (service layer, storage, retrieval)
- `crates/ov_cli/` - Rust CLI with TUI (`ratatui`) and HTTP client (`reqwest`)
- `openviking_cli/rust_cli.py` - Thin Python wrapper that locates and execs Rust binary

### 1.2 Virtual Filesystem as Universal Abstraction

**Decision:** `viking://` URI scheme provides unified interface for all context.

**Rationale:**
- Filesystem metaphor is intuitive for developers
- Enables standard tools (ls, cat, tree) to work with heterogeneous storage
- Separates content semantics from storage implementation

**Evidence:**
- `openviking/storage/viking_fs.py` - VikingFS implements URI resolution and content management
- URI namespaces: `viking://resources/`, `viking://user/`, `viking://agent/`

### 1.3 Dual Client Strategy

**Decision:** Same API supports embedded mode (direct access) and HTTP mode (client-server).

**Rationale:**
- Embedded mode: Local development without server overhead
- HTTP mode: Distributed deployment, shared state across clients
- Transparent failover via common `BaseClient` interface

**Evidence:**
- `openviking_cli/client/base.py` - `BaseClient` ABC defines interface
- `openviking/async_client.py` - `AsyncOpenViking` singleton with `LocalClient`
- `openviking_cli/client/http.py` - `AsyncHTTPClient` for server communication

### 1.4 Service Composition over Inheritance

**Decision:** `OpenVikingService` composes sub-services rather than inheriting from a monolithic base.

**Rationale:**
- Independent scaling of subsystems (FS, Search, Session, Resource)
- Loose coupling enables testing and replacement
- Lifecycle management decoupled from usage

**Evidence:**
- `openviking/service/core.py` lines 42-92: `OpenVikingService` facade with `_fs_service`, `_relation_service`, `_resource_service`, `_search_service`, `_session_service`

---

## 2. Module Responsibilities and Boundaries

### 2.1 Core Python Package (`openviking/`)

| Module | Responsibility | Key Classes/Files |
|--------|---------------|-------------------|
| `service/core.py` | Service facade | `OpenVikingService` |
| `storage/viking_fs.py` | Virtual filesystem | `VikingFS` |
| `storage/vikingdb_manager.py` | Vector storage | `VikingDBManager` |
| `session/session.py` | Session state | `Session` |
| `retrieve/hierarchical_retriever.py` | Context retrieval | `HierarchicalRetriever` |
| `parse/registry.py` | Code parsing | Tree-sitter bindings |
| `server/app.py` | HTTP API | FastAPI application |

**Boundaries:**
- Service layer exposes typed property accessors (`fs`, `resources`, `sessions`, etc.)
- Storage layer abstracted via `VikingFS` and `VikingDBManager`
- Client layer has no direct access to storage internals

### 2.2 VikingBot AI Agent (`bot/vikingbot/`)

| Module | Responsibility | Key Classes |
|--------|---------------|-------------|
| `agent/loop.py` | Agent processing engine | `AgentLoop` |
| `bus/queue.py` | Event/message handling | `MessageBus` |
| `providers/base.py` | LLM abstraction | `LLMProvider` |
| `channels/` | Multi-platform integration | `ChannelAdapter` |
| `hooks/` | Extension system | `HookManager` |
| `agent/tools/registry.py` | Tool management | `ToolRegistry` |

**Boundaries:**
- Agent loop communicates via typed events (`OutboundEventType`)
- Channels are adapters converting platform-specific messages to/from `InboundMessage`
- Tools are isolated and registered dynamically

### 2.3 Rust CLI (`crates/ov_cli/`)

| Module | Responsibility |
|--------|---------------|
| `src/main.rs` | CLI entry, argument parsing (`clap`) |
| `src/commands/` | Subcommands (chat, session, admin, search) |
| `src/tui/` | Terminal UI (`ratatui`) |
| `src/client.rs` | HTTP client for server communication |

**Boundaries:**
- CLI is a thin wrapper; all business logic in Python
- Binary resolution: `./target/release/ov` (dev) or bundled wheel location

---

## 3. Communication Patterns Between Components

### 3.1 Python to Rust CLI: Subprocess Execution

```
openviking_cli/rust_cli.py → os.execv(binary, argv)
                           → Rust main.rs
```

**Binary resolution priority:**
1. `./target/release/ov` (development)
2. Wheel bundled: `{package_dir}/openviking/bin/ov`
3. PATH lookup (skips Python scripts)

**Evidence:** `openviking_cli/rust_cli.py` lines 27-39

### 3.2 CLI/Client to Server: HTTP + JSON

```
AsyncHTTPClient → OpenViking HTTP Server (FastAPI)
              ← JSON responses
```

**Error mapping:**
- HTTP status codes mapped to typed exceptions
- `ERROR_CODE_TO_EXCEPTION` dictionary in `openviking_cli/client/http.py` lines 42-57

### 3.3 VikingBot to OpenViking: Hook System + HTTP

```
VikingBot AgentLoop
    → HookManager.execute_hooks()
    → OpenVikingHook (registered)
    → HTTP calls to OpenViking Server
```

**Hook execution:** Async and sync hooks executed separately via `asyncio.gather`

**Evidence:** `bot/vikingbot/hooks/manager.py` lines 46-64

### 3.4 VikingBot Channels: Adapter Pattern

```
Telegram/Slack/Discord message
    → ChannelAdapter.inbound()
    → InboundMessage
    → MessageBus
    → AgentLoop
```

**Evidence:** `bot/vikingbot/channels/base.py` - `ChannelAdapter` ABC

---

## 4. Key Design Choices

### 4.1 Tiered Context (L0/L1/L2)

**Choice:** Automatic summarization at multiple levels enables cost-effective context loading.

```
L0 (ABSTRACT): ~100 tokens - Quick relevance check
L1 (OVERVIEW): ~2k tokens - Understand structure
L2 (DETAIL): Full content - Load on demand
```

**Evidence:** `openviking/core/context.py` lines 32-37 (`ContextLevel` enum)

### 4.2 Async-First Design

**Choice:** Heavy use of `asyncio` throughout for I/O-bound concurrency.

**Evidence:**
- `openviking/service/core.py` - `async def initialize()`
- `openviking/storage/viking_fs.py` - Async file operations
- `bot/vikingbot/bus/queue.py` - `async def consume_inbound()`

### 4.3 Singleton Initialization

**Choice:** Module-level singleton for global state (VikingFS, AsyncOpenViking).

**Evidence:**
- `openviking/storage/viking_fs.py` lines 66-100: `_instance` global + `init_viking_fs()`
- `openviking/async_client.py` lines 36-44: Double-checked locking singleton

### 4.4 Factory Pattern for Versioned Implementations

**Choice:** Factory function selects implementation version.

**Evidence:** `openviking/session/__init__.py` lines 35-70 - `create_session_compressor()` returns `SessionCompressor` or `SessionCompressorV2` based on config

### 4.5 Event Bus for Agent Communication

**Choice:** Typed event system decouples components.

**Evidence:**
- `bot/vikingbot/bus/events.py` - `OutboundEventType` enum (RESPONSE, TOOL_CALL, TOOL_RESULT, REASONING, ITERATION)
- `bot/vikingbot/bus/queue.py` - `MessageBus` async message queue

---

## 5. Dependency Injection

### 5.1 FastAPI Lifespan Injection

**Pattern:** Service created once per application lifespan.

```python
# openviking/server/app.py lines 63-73
@asynccontextmanager
async def lifespan(app: FastAPI):
    service = OpenVikingService()
    await service.initialize()
    set_service(service)
    yield
```

### 5.2 Service Wiring

**Pattern:** Sub-services receive dependencies via setter injection.

```python
# openviking/service/core.py lines 324-344
async def initialize(self):
    self._fs_service.set_viking_fs(self._viking_fs)
    self._relation_service.set_viking_fs(self._viking_fs)
    self._resource_service.set_dependencies(
        vikingdb=self._vikingdb_manager,
        viking_fs=self._viking_fs,
        ...
    )
```

---

## 6. Cross-Cutting Concerns

### 6.1 Telemetry

- **Tracing:** Langfuse integration for LLM calls, tool execution, file operations
- **Metrics:** Prometheus observer at `/metrics` endpoint
- **Evidence:** `openviking/telemetry/`, `openviking/server/app.py` lines 112-117

### 6.2 Error Handling

- **HTTP Client:** Status code to exception mapping (`ERROR_CODE_TO_EXCEPTION`)
- **Server:** Typed exceptions (`NotFoundError`, `InvalidArgumentError`, etc.)

### 6.3 Observability

- Health observers via `observer_router`
- Prometheus metrics endpoint
- Debug endpoints in `debug_router`

---

## 7. Architectural Summary

| Aspect | Choice | Pattern |
|--------|--------|---------|
| Language split | Python + Rust | Hybrid |
| Context abstraction | Virtual filesystem | Adapter |
| Client modes | Embedded + HTTP | Strategy |
| Service organization | Facade + composition | Facade |
| Agent communication | Event bus | Observer |
| Extensibility | Hook system | Template Method |
| Version management | Factory | Factory |
| State management | Singleton | Singleton |
| Async model | Asyncio-first | Async |
| Context tiers | L0/L1/L2 | Lazy loading |

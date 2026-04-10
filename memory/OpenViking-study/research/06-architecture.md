# OpenViking Architecture Analysis

**Repository:** `/Users/sheldon/Documents/claw/reference/OpenViking/`
**Analysis Date:** 2026-03-27
**Output:** `/Users/sheldon/Documents/claw/OpenViking-study/research/06-architecture.md`

---

## Executive Summary

OpenViking is a hybrid Python + Rust system designed as an **Agent-native context database**. It implements a virtual filesystem paradigm (`viking://` URI scheme) for unified context management, combining vector storage, session management, and hierarchical retrieval. The architecture follows a **layered service composition** pattern with clear separation between the core database, HTTP API server, Rust CLI, and VikingBot AI agent framework.

---

## 1. Architectural Overview

### 1.1 Component Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                        VikingBot (AI Agent)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Channels │  │ Providers│  │  Tools   │  │  Agent Loop     │  │
│  │ (Telegram│  │ (LiteLLM │  │ Registry │  │  + Context       │  │
│  │  Slack..)│  │  OpenAI) │  │          │  │  + Memory       │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬────────┘  │
│       └───────────────┴──────────────┴─────────────────┘         │
│                              │                                    │
│                    ┌─────────▼─────────┐                          │
│                    │   Hook System    │ (OpenViking integration)  │
│                    └─────────┬─────────┘                          │
└──────────────────────────────┼───────────────────────────────────┘
                               │ HTTP / Hooks
┌──────────────────────────────▼───────────────────────────────────┐
│                    OpenViking HTTP Server                          │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    FastAPI Application                       │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────┐   │  │
│  │  │Routers: │ │Resources│ │Sessions │ │ Search/Relations│   │  │
│  │  │System   │ │         │ │         │ │                 │   │  │
│  │  │Admin    │ │         │ │         │ │                 │   │  │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────────┬────────┘   │  │
│  └───────┼───────────┼───────────┼──────────────┼─────────────┘  │
│          └───────────┴───────────┴──────────────┘                 │
│                         │                                         │
│         ┌───────────────▼────────────────┐                        │
│         │    OpenVikingService (Core)    │                        │
│         │  ┌─────────────────────────┐  │                        │
│         │  │ Sub-services:           │  │                        │
│         │  │ FS, Resource, Search,   │  │                        │
│         │  │ Session, Relation, Pack │  │                        │
│         │  └─────────────────────────┘  │                        │
│         └───────────────┬────────────────┘                        │
│                         │                                         │
│  ┌─────────────────────▼────────────────────────┐                │
│  │              Storage Layer                    │                │
│  │  ┌─────────────┐  ┌─────────────────────┐   │                │
│  │  │ VikingFS    │  │ VikingDBManager     │   │                │
│  │  │ (AGFS)      │  │ (Vector Store)     │   │                │
│  │  └─────────────┘  └─────────────────────┘   │                │
│  └─────────────────────────────────────────────┘                │
└───────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────┐
│                    Rust CLI (ov binary)                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  Commands:   │  │    TUI      │  │   HTTP Client        │   │
│  │  chat, add,  │  │  (ratatui)  │  │   (reqwest)          │   │
│  │  ls, search │  │             │  │                      │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Python Primary** | Python 3.10+, Pydantic, FastAPI, httpx | Core business logic, HTTP server |
| **Rust CLI** | Rust, Clap, Ratatui, Reqwest | Performance-critical CLI operations |
| **Storage** | AGFS (Abstract File System), Vector DB | Virtual filesystem + embeddings |
| **AI Integration** | LiteLLM, OpenAI SDK | Multi-provider LLM access |
| **Parsing** | Tree-sitter | Multi-language code parsing |
| **Agent Framework** | Asyncio, custom event bus | VikingBot agent execution |

---

## 2. Core Architectural Patterns

### 2.1 Pattern: Service Composition (Facade)

The `OpenVikingService` class in `openviking/service/core.py` acts as a **facade** that composes multiple sub-services:

```python
# From openviking/service/core.py (lines 42-92)
class OpenVikingService:
    def __init__(self, path: Optional[str] = None, user: Optional[UserIdentifier] = None):
        # Infrastructure components
        self._vikingdb_manager: Optional[VikingDBManager] = None
        self._viking_fs: Optional[VikingFS] = None
        self._queue_manager: Optional[QueueManager] = None
        self._lock_manager: Optional[LockManager] = None

        # Sub-services (facade pattern)
        self._fs_service = FSService()
        self._relation_service = RelationService()
        self._resource_service = ResourceService()
        self._search_service = SearchService()
        self._session_service = SessionService()
```

**Key insight:** The service exposes typed property accessors (`fs`, `resources`, `sessions`, etc.) that delegate to initialized infrastructure. This decouples lifecycle management from service usage.

### 2.2 Pattern: Dual Client Modes (Strategy)

The system supports two client modes via a common `BaseClient` interface:

```
openviking_cli/client/base.py (BaseClient ABC)
         │
         ├── LocalClient ──► Direct VikingFS/VikingDB access (embedded mode)
         │
         └── AsyncHTTPClient ──► HTTP calls to OpenViking Server (HTTP mode)
```

From `openviking/client.py`:
```python
# Lines 8-13: Export both client types
from openviking.async_client import AsyncOpenViking
from openviking.sync_client import SyncOpenViking
from openviking_cli.client.http import AsyncHTTPClient
from openviking_cli.client.sync_http import SyncHTTPClient
```

The `AsyncOpenViking` class uses singleton pattern with `LocalClient`:
```python
# From openviking/async_client.py (lines 36-44)
class AsyncOpenViking:
    _instance: Optional["AsyncOpenViking"] = None
    _lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = object.__new__(cls)
        return cls._instance
```

### 2.3 Pattern: Virtual Filesystem (Adapter)

`VikingFS` (`openviking/storage/viking_fs.py`) implements a **virtual filesystem** abstraction over heterogeneous storage:

```python
# From openviking/storage/viking_fs.py (lines 4-13)
"""
VikingFS: OpenViking file system abstraction layer

Responsibilities:
- URI conversion (viking:// <-> /local/)
- L0/L1 reading (.abstract.md, .overview.md)
- Relation management (.relations.json)
- Semantic search (vector retrieval + rerank)
- Vector sync (sync vector store on rm/mv)
"""
```

URI scheme:
- `viking://resources/` - User resources
- `viking://user/` - User memories/preferences
- `viking://agent/` - Agent skills/instructions

### 2.4 Pattern: Singleton Initialization

VikingFS uses module-level singleton initialization:

```python
# From openviking/storage/viking_fs.py (lines 66-100)
_instance: Optional["VikingFS"] = None

def init_viking_fs(
    agfs: Any,
    query_embedder: Optional[Any] = None,
    rerank_config: Optional["RerankConfig"] = None,
    vector_store: Optional["VikingVectorIndexBackend"] = None,
    ...
) -> "VikingFS":
    global _instance
    _instance = VikingFS(...)
    return _instance
```

---

## 3. Component Communication Patterns

### 3.1 Python ↔ Rust: Subprocess Execution

The Rust CLI is a **thin wrapper pattern**. `openviking_cli/rust_cli.py` is a Python entry point that locates and execs the Rust binary:

```python
# From openviking_cli/rust_cli.py (lines 27-39)
def _exec_binary(binary: str, argv: list[str]) -> None:
    """Execute a binary, replacing the current process on Unix."""
    if sys.platform == "win32":
        sys.exit(subprocess.call([binary] + argv))
    else:
        os.execv(binary, [binary] + argv)
```

Binary resolution priority:
1. `./target/release/ov` (development)
2. Wheel bundled: `{package_dir}/openviking/bin/ov`
3. PATH lookup (skips Python scripts to avoid infinite loop)

### 3.2 Python ↔ Server: HTTP + JSON

The Rust CLI and Python CLI client communicate with the HTTP server via `AsyncHTTPClient`:

```python
# From openviking_cli/client/http.py (lines 1-10)
"""Async HTTP Client for OpenViking.

Implements BaseClient interface using HTTP calls to OpenViking Server.
"""
```

Error mapping from HTTP status to typed exceptions:
```python
# From openviking_cli/client/http.py (lines 42-57)
ERROR_CODE_TO_EXCEPTION = {
    "INVALID_ARGUMENT": InvalidArgumentError,
    "NOT_FOUND": NotFoundError,
    "ALREADY_EXISTS": AlreadyExistsError,
    "UNAUTHENTICATED": UnauthenticatedError,
    ...
}
```

### 3.3 VikingBot ↔ OpenViking: Hook System + HTTP

VikingBot integrates with OpenViking through a **hook-based event system**:

```python
# From bot/vikingbot/hooks/base.py (lines 33-39)
class Hook(ABC):
    name: str
    is_sync: bool = False

    @abstractmethod
    async def execute(self, context: HookContext, **kwargs) -> Any:
        pass
```

Hook manager executes async/sync hooks separately:
```python
# From bot/vikingbot/hooks/manager.py (lines 46-64)
async def execute_hooks(self, context: HookContext, **kwargs) -> List[Any]:
    async_hooks = [hook for hook in self._hooks[context.event_type] if not hook.is_sync]
    sync_hooks = [hook for hook in self._hooks[context.event_type] if hook.is_sync]

    if async_hooks:
        async_results = await asyncio.gather(
            *[hook.execute(context, **kwargs) for hook in async_hooks],
            return_exceptions=True
        )
```

---

## 4. VikingBot Agent Architecture

### 4.1 Agent Loop Pattern

The `AgentLoop` (`bot/vikingbot/agent/loop.py`) implements the core agent processing:

```python
# From bot/vikingbot/agent/loop.py (lines 35-45)
class AgentLoop:
    """
    The agent loop is the core processing engine.

    It:
    1. Receives messages from the bus
    2. Builds context with history, memory, skills
    3. Calls the LLM
    4. Executes tool calls
    5. Sends responses back
    """
```

**Message flow:**
```
InboundMessage → AgentLoop._process_message()
    → ContextBuilder.build_messages()
    → LLM (provider.chat)
    → Tool execution (if tool_calls)
    → OutboundMessage
```

### 4.2 Event Bus Pattern

The `MessageBus` (`bot/vikingbot/bus/queue.py`) provides async message queue:

```python
# From bot/vikingbot/bus/queue.py
class MessageBus:
    async def consume_inbound(self) -> InboundMessage: ...
    async def publish_outbound(self, message: OutboundMessage) -> None: ...
```

Event types for outbound messages:
```python
# From bot/vikingbot/bus/events.py (lines 11-18)
class OutboundEventType(str, Enum):
    RESPONSE = "response"
    TOOL_CALL = "tool_call"
    TOOL_RESULT = "tool_result"
    REASONING = "reasoning"
    ITERATION = "iteration"
```

### 4.3 Tool Registry Pattern

Tools are registered dynamically in a registry:

```python
# From bot/vikingbot/agent/tools/registry.py (lines 17-26)
class ToolRegistry:
    """
    Registry for agent tools.

    Allows dynamic registration and execution of tools.
    """

    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None: ...
    async def execute(self, name: str, params: dict, ...) -> str: ...
```

Built-in tools in `bot/vikingbot/agent/tools/`:
- `filesystem.py` - File operations
- `shell.py` - Shell commands
- `ov_file.py` - OpenViking operations
- `web.py` / `websearch/` - Web access
- `image.py` - Image generation
- `cron.py` - Scheduled tasks

### 4.4 LLM Provider Pattern

Abstract base for multi-provider support:

```python
# From bot/vikingbot/providers/base.py (lines 34-75)
class LLMProvider(ABC):
    """Abstract base class for LLM providers."""

    @abstractmethod
    async def chat(
        self,
        messages: list[dict[str, Any]],
        tools: list[dict[str, Any]] | None = None,
        model: str | None = None,
        ...
    ) -> LLMResponse:
        pass

@dataclass
class LLMResponse:
    content: str | None
    tool_calls: list[ToolCallRequest] = field(default_factory=list)
    finish_reason: str = "stop"
    reasoning_content: str | None = None  # For reasoning models
```

Provider implementations:
- `litellm_provider.py` - LiteLLM wrapper (supports 100+ models)
- `openai_compatible_provider.py` - OpenAI-compatible endpoints

### 4.5 Channel Integration Pattern

Channels follow adapter pattern for multi-platform support:

```
bot/vikingbot/channels/
├── base.py           # Abstract ChannelAdapter
├── telegram.py
├── slack.py
├── discord.py
├── feishu.py         # Lark/Feishu integration
├── dingtalk.py
├── email.py
└── ...
```

---

## 5. Storage Architecture

### 5.1 Tiered Storage Model

```
┌─────────────────────────────────────────┐
│           AGFS (Abstract FS)            │
│  - Virtual URI resolution               │
│  - L0/L1/L2 content management           │
│  - Relation tracking                     │
└────────────────┬────────────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
    ▼            ▼            ▼
┌────────┐  ┌─────────┐  ┌──────────────┐
│Local FS│  │ S3/OSS  │  │ QueueFS      │
│(native)│  │ (remote)│  │ (async ops)  │
└────────┘  └─────────┘  └──────────────┘
                               │
                               ▼
                    ┌──────────────────┐
                    │ VikingDBManager  │
                    │ (Vector Store)    │
                    │ - Embeddings      │
                    │ - Semantic search│
                    └──────────────────┘
```

### 5.2 Transaction & Locking

Distributed locking via AGFS:

```python
# From openviking/storage/transaction/lock_manager.py
class LockManager:
    """Manages distributed locks via AGFS."""
    async def acquire(self, path: str, ...) -> LockHandle: ...
    async def release(self, handle: LockHandle) -> None: ...
```

Lock types:
- **Path locks** - Protect individual file operations
- **Process locks** - Prevent multi-process contention
- **Transaction locks** - ACID semantics for compound operations

### 5.3 Queue System

Async processing via `QueueManager`:

```python
# From openviking/storage/queuefs/queue_manager.py
class QueueManager:
    """Manages async operation queues."""
    def setup_standard_queues(self, vikingdb, start=True): ...
    def start(self) -> None: ...
    def stop(self) -> None: ...
```

Queue types:
- **Embedding queue** - Vectorization tasks
- **Semantic queue** - LLM processing tasks
- **Index queue** - Search index updates

---

## 6. API Server Architecture

### 6.1 FastAPI Router Organization

```python
# From openviking/server/app.py (lines 204-219)
app.include_router(system_router)      # Health, version
app.include_router(admin_router)       # Admin operations
app.include_router(resources_router)   # Resource CRUD
app.include_router(filesystem_router)  # FS operations (ls, tree, etc.)
app.include_router(content_router)     # Content reading
app.include_router(search_router)      # Semantic search
app.include_router(relations_router)   # Relation management
app.include_router(sessions_router)    # Session management
app.include_router(stats_router)       # Statistics
app.include_router(pack_router)        # Import/export
app.include_router(debug_router)       # Debug endpoints
app.include_router(observer_router)    # Health observers
app.include_router(metrics_router)     # Prometheus metrics
app.include_router(tasks_router)       # Background tasks
app.include_router(bot_router)         # VikingBot integration
```

### 6.2 Service Dependency Injection

Services are injected via FastAPI's lifespan mechanism:

```python
# From openviking/server/app.py (lines 63-73)
@asynccontextmanager
async def lifespan(app: FastAPI):
    nonlocal service
    owns_service = service is None
    if owns_service:
        service = OpenVikingService()
        await service.initialize()

    set_service(service)  # Dependency injection
    yield
```

Request context access via dependency injection:
```python
# From openviking/server/dependencies.py
def set_service(service: OpenVikingService) -> None: ...
def get_service() -> OpenVikingService: ...
```

---

## 7. Session Management Architecture

### 7.1 Session Lifecycle

```python
# From openviking/session/session.py
class Session:
    """Session with message history and metadata."""

    def add_message(self, role: str, content: str, ...) -> None: ...
    def get_history(self) -> list[dict]: ...
    def clone(self) -> "Session": ...
    def clear(self) -> None: ...
```

### 7.2 Memory Consolidation

Two compression versions via factory:

```python
# From openviking/session/__init__.py (lines 35-70)
def create_session_compressor(
    vikingdb: VikingDBManager,
    memory_version: Optional[str] = None,
) -> SessionCompressor:
    """Create SessionCompressor based on configuration."""
    if memory_version == "v2":
        from openviking.session.compressor_v2 import SessionCompressorV2
        return SessionCompressorV2(vikingdb=vikingdb)
    return SessionCompressor(vikingdb=vikingdb)
```

Memory consolidation flow:
1. Session messages → MemoryExtractor → CandidateMemory
2. MemoryDeduplicator → Remove duplicates
3. MemoryArchiver → Write to viking://user/memories/

---

## 8. Retrieval Architecture

### 8.1 Hierarchical Retrieval

```python
# From openviking/retrieve/hierarchical_retriever.py
class HierarchicalRetriever:
    """
    Implements directory recursive retrieval strategy:

    1. Intent Analysis - Generate multiple retrieval conditions
    2. Initial Positioning - Vector retrieval to locate high-score directory
    3. Refined Exploration - Secondary retrieval within directory
    4. Recursive Drill-down - Repeat for subdirectories
    5. Result Aggregation - Return most relevant context
    """
```

### 8.2 Context Levels (L0/L1/L2)

```python
# From openviking/core/context.py (lines 32-37)
class ContextLevel(int, Enum):
    """Context level (L0/L1/L2) for vector indexing"""

    ABSTRACT = 0  # L0: abstract (~100 tokens) - Quick relevance check
    OVERVIEW = 1  # L1: overview (~2k tokens) - Understand structure
    DETAIL = 2    # L2: detail/content - Load on demand
```

---

## 9. VikingBot Tools for OpenViking

### 9.1 OpenVikingTool (Tool)

Located in `bot/vikingbot/agent/tools/ov_file.py`, provides OpenViking operations to the agent:

```python
class OpenVikingTool(Tool):
    """Tools for interacting with OpenViking context database."""

    @property
    def name(self) -> str: return "openviking"

    @property
    def description(self) -> str:
        return "..."  # File operations, search, memory access
```

### 9.2 OpenVikingMount

FUSE-based local filesystem mounting of viking:// URIs:

```python
# From bot/vikingbot/openviking_mount/__init__.py
from .mount import OpenVikingMount, MountScope, MountConfig
from .manager import OpenVikingMountManager, get_mount_manager

class OpenVikingMount:
    """Mount OpenViking viking:// paths to local filesystem."""
```

---

## 10. Key Design Patterns Summary

| Pattern | Location | Purpose |
|---------|----------|---------|
| **Facade** | `OpenVikingService` | Unified access to subsystems |
| **Strategy** | `BaseClient` | Embedded vs HTTP mode |
| **Singleton** | `AsyncOpenViking`, `VikingFS` | Single instance management |
| **Adapter** | `VikingFS`, `ChannelAdapter` | Uniform interfaces over heterogeneous systems |
| **Registry** | `ToolRegistry`, `HookManager` | Dynamic tool/hook registration |
| **Event Bus** | `MessageBus` | Async message passing |
| **Factory** | `create_session_compressor` | Versioned implementation selection |
| **Template Method** | `Hook.execute` | Extension points |
| **Observer** | `PrometheusObserver` | Health monitoring |

---

## 11. Dependency Injection Points

### 11.1 OpenVikingService Wiring

```python
# From openviking/service/core.py (lines 324-344)
async def initialize(self) -> None:
    # Wire up sub-services
    self._fs_service.set_viking_fs(self._viking_fs)
    self._relation_service.set_viking_fs(self._viking_fs)
    self._resource_service.set_dependencies(
        vikingdb=self._vikingdb_manager,
        viking_fs=self._viking_fs,
        resource_processor=self._resource_processor,
        skill_processor=self._skill_processor,
        watch_scheduler=self._watch_scheduler,
    )
    self._session_service.set_dependencies(
        vikingdb=self._vikingdb_manager,
        viking_fs=self._viking_fs,
        session_compressor=self._session_compressor,
    )
```

### 11.2 AgentLoop Dependencies

```python
# From bot/vikingbot/agent/loop.py (lines 47-131)
def __init__(
    self,
    bus: MessageBus,
    provider: LLMProvider,
    workspace: Path,
    ...
):
    self.bus = bus
    self.provider = provider
    self.context = ContextBuilder(workspace, sandbox_manager=sandbox_manager)
    self.sessions = session_manager or SessionManager(...)
    self.tools = ToolRegistry()
    self.subagents = SubagentManager(provider=provider, ...)
```

---

## 12. Cross-Cutting Concerns

### 12.1 Telemetry

```python
# From openviking/telemetry/
class TelemetryRequest:
    """Attaches telemetry data to operations."""
```

Traced operations:
- Tool execution (via Langfuse)
- LLM calls
- File operations
- Search queries

### 12.2 Error Handling

Centralized error mapping in HTTP client:
```python
# From openviking_cli/client/http.py (lines 42-57)
ERROR_CODE_TO_EXCEPTION = {
    "NOT_FOUND": NotFoundError,
    "INVALID_ARGUMENT": InvalidArgumentError,
    ...
}
```

### 12.3 Observability

Prometheus metrics endpoint:
```python
# From openviking/server/app.py (lines 112-117)
if config.telemetry.prometheus.enabled:
    observer = PrometheusObserver()
    app.state.prometheus_observer = observer
    set_prometheus_observer(observer)
    logger.info("Prometheus metrics enabled at /metrics")
```

---

## 13. File Structure Map

```
OpenViking/
|-- openviking/                    # Core Python package
|   |-- client.py                  # Client exports
|   |-- async_client.py            # Async client (singleton)
|   |-- sync_client.py             # Sync wrapper
|   |-- core/                      # Core abstractions
|   |   |-- context.py             # Context dataclasses (L0/L1/L2)
|   |   |-- directories.py         # Directory initialization
|   |   |-- building_tree.py       # Tree building
|   |-- service/                   # Service layer
|   |   |-- core.py                # OpenVikingService (facade)
|   |   |-- fs_service.py          # Filesystem operations
|   |   |-- resource_service.py    # Resource management
|   |   |-- search_service.py      # Search operations
|   |   |-- session_service.py     # Session management
|   |   |-- debug_service.py       # Debug/health
|   |-- storage/                   # Storage layer
|   |   |-- viking_fs.py           # Virtual filesystem
|   |   |-- vikingdb_manager.py   # Vector DB manager
|   |   |-- transaction/           # Locking/transaction
|   |   |-- queuefs/               # Async queue
|   |   |-- vectordb/              # Vector DB adapters
|   |-- session/                   # Session management
|   |   |-- session.py             # Session class
|   |   |-- compressor.py          # Memory compression v1
|   |   |-- compressor_v2.py       # Memory compression v2
|   |   |-- memory_extractor.py    # Memory extraction
|   |   |-- memory_archiver.py     # Memory archival
|   |-- retrieve/                  # Retrieval logic
|   |   |-- hierarchical_retriever.py
|   |   |-- intent_analyzer.py
|   |-- parse/                    # Parsing (tree-sitter)
|   |   |-- registry.py
|   |   |-- tree_builder.py
|   |   |-- parsers/              # Language-specific
|   |-- server/                   # HTTP server
|   |   |-- app.py                # FastAPI app
|   |   |-- routers/              # API routes
|   |-- models/                   # Data models
|   |-- prompts/                  # Prompt templates
|
|-- bot/vikingbot/                # VikingBot AI Agent
|   |-- cli/commands.py           # CLI entry (Typer)
|   |-- agent/
|   |   |-- loop.py              # AgentLoop (core engine)
|   |   |-- context.py           # ContextBuilder
|   |   |-- memory.py            # MemoryStore
|   |   |-- tools/               # Tool implementations
|   |   |   |-- registry.py      # ToolRegistry
|   |   |   |-- base.py          # Tool ABC
|   |   |   |-- ov_file.py       # OpenViking tools
|   |   |-- subagent.py          # Subagent management
|   |-- bus/
|   |   |-- queue.py             # MessageBus
|   |   |-- events.py            # Event types
|   |-- providers/
|   |   |-- base.py              # LLMProvider ABC
|   |   |-- litellm_provider.py  # LiteLLM impl
|   |-- channels/                # Channel adapters
|   |-- hooks/                   # Hook system
|   |-- sandbox/                 # Sandboxing
|   |-- openviking_mount/        # OpenViking mounting
|
|-- crates/ov_cli/              # Rust CLI
|   |-- src/
|   |   |-- main.rs             # Entry point
|   |   |-- client.rs           # HTTP client
|   |   |-- commands/           # Subcommands
|   |   |-- tui/                # Terminal UI
|
|-- openviking_cli/             # Python CLI wrapper
|   |-- rust_cli.py             # Binary exec wrapper
|   |-- server_bootstrap.py     # Server launcher
|   |-- client/
|       |-- http.py             # HTTP client
|       |-- base.py             # BaseClient interface
```

---

## 14. Key Insights

1. **Python-primary with Rust optimization**: Python handles business logic; Rust provides performant CLI/TUI. Communication via exec/subprocess, not FFI.

2. **Virtual filesystem as universal abstraction**: `viking://` URIs provide a unified interface for memories, resources, and skills, enabling filesystem-like context management.

3. **Service composition over inheritance**: `OpenVikingService` composes sub-services rather than inheriting from a monolithic base, allowing independent scaling.

4. **Dual client strategy**: Same API works in embedded mode (direct access) or HTTP mode (client-server), enabling both local development and distributed deployment.

5. **Event-driven agent architecture**: VikingBot uses an event bus pattern with typed events, enabling loose coupling between channels, agent logic, and tool execution.

6. **Hook-based extensibility**: The hook system allows OpenViking integration with VikingBot without tight coupling, following the plugin pattern.

7. **Tiered context (L0/L1/L2)**: Automatic summarization at multiple levels enables cost-effective context loading, a key innovation for long-running agents.

8. **Async-first design**: Heavy use of asyncio throughout enables high concurrency for I/O-bound operations (embedding, search, file ops).

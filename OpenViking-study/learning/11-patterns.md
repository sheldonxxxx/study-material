# OpenViking Design Patterns

**Repository:** `/Users/sheldon/Documents/claw/reference/OpenViking/`
**Analysis Date:** 2026-03-27
**Source:** Research analysis from `06-architecture.md`

---

## 1. Facade Pattern

**Problem:** Complex subsystem with multiple components needs simplified unified interface.

**Evidence:** `openviking/service/core.py` lines 42-92

```python
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

**Solution:** `OpenVikingService` provides typed property accessors (`fs`, `resources`, `sessions`, etc.) delegating to initialized infrastructure. Clients interact with one interface rather than managing multiple subsystem instances.

---

## 2. Strategy Pattern

**Problem:** Same API needs different execution strategies (embedded vs distributed).

**Evidence:** `openviking_cli/client/base.py` - `BaseClient` ABC

```
BaseClient ABC
    │
    ├── LocalClient ──► Direct VikingFS/VikingDB access (embedded mode)
    │
    └── AsyncHTTPClient ──► HTTP calls to OpenViking Server (HTTP mode)
```

**Evidence:** `openviking/async_client.py` lines 36-44 and `openviking_cli/client/http.py`

```python
# From async_client.py - Local/singleton mode
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

**Solution:** `BaseClient` interface allows runtime selection between direct local access and HTTP client-server communication without changing consumer code.

---

## 3. Singleton Pattern

**Problem:** Global state需要一个唯一的全局实例，避免重复初始化。

**Evidence:** `openviking/storage/viking_fs.py` lines 66-100

```python
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

**Evidence:** `openviking/async_client.py` lines 36-44 - Double-checked locking singleton

**Solution:** Module-level singleton with `init_viking_fs()` factory function ensures one VikingFS instance per process. AsyncOpenViking uses threading lock for thread-safe singleton access.

---

## 4. Adapter Pattern

**Problem:** Heterogeneous storage systems need uniform interface.

**Evidence:** `openviking/storage/viking_fs.py` lines 4-13

```python
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

**Evidence:** `bot/vikingbot/channels/base.py` - `ChannelAdapter` ABC

```python
class ChannelAdapter(ABC):
    """Abstract base class for channel adapters."""
    @abstractmethod
    async def inbound(self, message: Any) -> InboundMessage: ...
    @abstractmethod
    async def outbound(self, message: OutboundMessage) -> None: ...
```

**Solution:** VikingFS adapts AGFS (Abstract File System) to filesystem-like API. ChannelAdapter adapts platform-specific APIs (Telegram, Slack, etc.) to unified InboundMessage/OutboundMessage format.

---

## 5. Registry Pattern

**Problem:** Dynamic registration and lookup of tools or hooks without hardcoded dependencies.

**Evidence:** `bot/vikingbot/agent/tools/registry.py` lines 17-26

```python
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

**Evidence:** `bot/vikingbot/hooks/manager.py` - `HookManager` with hook registration

**Solution:** Registry pattern enables runtime tool/hook registration. Components register themselves without the registry knowing about them upfront, achieving loose coupling.

---

## 6. Event Bus / Observer Pattern

**Problem:** Loose coupling between message producers and consumers in agent system.

**Evidence:** `bot/vikingbot/bus/queue.py`

```python
class MessageBus:
    async def consume_inbound(self) -> InboundMessage: ...
    async def publish_outbound(self, message: OutboundMessage) -> None: ...
```

**Evidence:** `bot/vikingbot/bus/events.py` lines 11-18

```python
class OutboundEventType(str, Enum):
    RESPONSE = "response"
    TOOL_CALL = "tool_call"
    TOOL_RESULT = "tool_result"
    REASONING = "reasoning"
    ITERATION = "iteration"
```

**Solution:** MessageBus provides async queue for decoupled communication. Producers publish messages without knowing consumers. Event types define the contract.

---

## 7. Factory Pattern

**Problem:** Versioned implementations need selection at runtime based on configuration.

**Evidence:** `openviking/session/__init__.py` lines 35-70

```python
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

**Solution:** Factory function `create_session_compressor()` returns appropriate implementation based on `memory_version` config. Consumer code is decoupled from concrete class selection.

---

## 8. Template Method Pattern

**Problem:** Extension points needed with fixed algorithm structure.

**Evidence:** `bot/vikingbot/hooks/base.py` lines 33-39

```python
class Hook(ABC):
    name: str
    is_sync: bool = False

    @abstractmethod
    async def execute(self, context: HookContext, **kwargs) -> Any:
        pass
```

**Evidence:** `bot/vikingbot/hooks/manager.py` lines 46-64 - Hook execution order is fixed

```python
async def execute_hooks(self, context: HookContext, **kwargs) -> List[Any]:
    async_hooks = [hook for hook in self._hooks[context.event_type] if not hook.is_sync]
    sync_hooks = [hook for hook in self._hooks[context.event_type] if hook.is_sync]

    if async_hooks:
        async_results = await asyncio.gather(...)
```

**Solution:** Abstract `Hook.execute()` defines extension point. `HookManager.execute_hooks()` defines fixed algorithm (async/sync separation, parallel execution). Subclasses implement concrete behavior.

---

## 9. Observer Pattern (Health Monitoring)

**Problem:** Health and metrics collection without tight coupling to monitored components.

**Evidence:** `openviking/server/app.py` lines 112-117

```python
if config.telemetry.prometheus.enabled:
    observer = PrometheusObserver()
    app.state.prometheus_observer = observer
    set_prometheus_observer(observer)
    logger.info("Prometheus metrics enabled at /metrics")
```

**Solution:** `PrometheusObserver` attaches to application state. Components emit metrics without knowing about the observer. Observer pattern enables decoupled observability.

---

## 10. Dependency Injection (Lifespan-based)

**Problem:** Service instances need controlled lifecycle shared across request handlers.

**Evidence:** `openviking/server/app.py` lines 63-73

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    nonlocal service
    owns_service = service is None
    if owns_service:
        service = OpenVikingService()
        await service.initialize()

    set_service(service)
    yield
```

**Evidence:** `openviking/server/dependencies.py`

```python
def set_service(service: OpenVikingService) -> None: ...
def get_service() -> OpenVikingService: ...
```

**Solution:** FastAPI lifespan context manager creates `OpenVikingService` once and injects via module-level `set_service()`. Request handlers retrieve via `get_service()`.

---

## 11. Wrapper/Bridge Pattern (Python-Rust Interop)

**Problem:** Thin Python wrapper needed for Rust binary without complex FFI.

**Evidence:** `openviking_cli/rust_cli.py` lines 27-39

```python
def _exec_binary(binary: str, argv: list[str]) -> None:
    """Execute a binary, replacing the current process on Unix."""
    if sys.platform == "win32":
        sys.exit(subprocess.call([binary] + argv))
    else:
        os.execv(binary, [binary] + argv)
```

**Binary resolution priority:**
1. `./target/release/ov` (development)
2. Wheel bundled: `{package_dir}/openviking/bin/ov`
3. PATH lookup

**Solution:** `os.execv` replaces Python process with Rust binary. Python entry point is a thin shim that locates the binary. No shared memory or FFI complexity.

---

## 12. Hierarchical Retrieval Pattern

**Problem:** Large context needs efficient selective loading without full scan.

**Evidence:** `openviking/retrieve/hierarchical_retriever.py`

```python
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

**Evidence:** `openviking/core/context.py` lines 32-37

```python
class ContextLevel(int, Enum):
    """Context level (L0/L1/L2) for vector indexing"""
    ABSTRACT = 0  # L0: abstract (~100 tokens) - Quick relevance check
    OVERVIEW = 1  # L1: overview (~2k tokens) - Understand structure
    DETAIL = 2    # L2: detail/content - Load on demand
```

**Solution:** HierarchicalRetriever implements progressive drill-down with L0/L1/L2 context levels. Abstract (L0) checked first; only relevant paths load Overview (L1), then Detail (L2) on demand.

---

## 13. Async Iterator Pattern

**Problem:** Streaming responses and background tasks need async consumption.

**Evidence:** `bot/vikingbot/bus/queue.py`

```python
class MessageBus:
    async def consume_inbound(self) -> InboundMessage: ...
    async def publish_outbound(self, message: OutboundMessage) -> None: ...
```

**Evidence:** `openviking/storage/queuefs/queue_manager.py`

```python
class QueueManager:
    """Manages async operation queues."""
    def setup_standard_queues(self, vikingdb, start=True): ...
    async def start(self) -> None: ...
    async def stop(self) -> None: ...
```

**Solution:** Async methods enable non-blocking I/O. Queues for embedding, semantic, and index operations decouple production from consumption.

---

## 14. Error Mapping Pattern

**Problem:** HTTP error codes need conversion to typed exceptions.

**Evidence:** `openviking_cli/client/http.py` lines 42-57

```python
ERROR_CODE_TO_EXCEPTION = {
    "INVALID_ARGUMENT": InvalidArgumentError,
    "NOT_FOUND": NotFoundError,
    "ALREADY_EXISTS": AlreadyExistsError,
    "UNAUTHENTICATED": UnauthenticatedError,
    ...
}
```

**Solution:** Dictionary mapping from error codes to exception types enables centralized error translation. Client throws typed exceptions regardless of transport mechanism.

---

## Pattern Summary Table

| Pattern | Location | Purpose |
|---------|----------|---------|
| Facade | `OpenVikingService` | Unified subsystem access |
| Strategy | `BaseClient` | Embedded vs HTTP mode |
| Singleton | `VikingFS`, `AsyncOpenViking` | Single instance management |
| Adapter | `VikingFS`, `ChannelAdapter` | Uniform interfaces over heterogeneous systems |
| Registry | `ToolRegistry`, `HookManager` | Dynamic tool/hook registration |
| Event Bus | `MessageBus` | Async message passing |
| Factory | `create_session_compressor` | Versioned implementation selection |
| Template Method | `Hook.execute` | Extension points |
| Observer | `PrometheusObserver` | Health monitoring |
| Dependency Injection | FastAPI lifespan | Service lifecycle |
| Wrapper | `rust_cli.py` | Python-Rust interop |
| Hierarchical Retrieval | `HierarchicalRetriever` | Tiered context loading |
| Async Iterator | `MessageBus`, `QueueManager` | Streaming operations |
| Error Mapping | `ERROR_CODE_TO_EXCEPTION` | Typed error translation |

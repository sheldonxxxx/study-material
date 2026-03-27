# OpenViking Features Deep Dive - Batch 2 (Features 4-6)

**Research Date:** 2026-03-27
**Repository:** `/Users/sheldon/Documents/claw/reference/OpenViking/`
**Previous Research:** `01-topology.md`, `04-features-index.md`

---

## Feature 4: Vector Database Integration

### Overview

OpenViking's vector database integration provides a pluggable backend system for semantic search, supporting multiple embedding providers (Volcengine, OpenAI, Jina, Voyage, MiniMax, VikingDB) with a unified adapter interface.

### Architecture

The integration follows an **Adapter Pattern** with three layers:

1. **CollectionAdapter** (`openviking/storage/vectordb_adapters/base.py`) - Abstract base defining the contract
2. **Backend-specific adapters** - Volcengine, VikingDB Private, HTTP, Local
3. **VikingVectorIndexBackend** (`openviking/storage/viking_vector_index_backend.py`) - Facade managing per-account backends

### Core Implementation Details

#### CollectionAdapter Base Class

The `CollectionAdapter` abstract base (lines 56-520 in `base.py`) defines the unified interface:

```python
class CollectionAdapter(ABC):
    mode: str
    _URI_FIELD_NAMES = {"uri", "parent_uri"}

    def __init__(self, collection_name: str, index_name: str = DEFAULT_INDEX_NAME):
        self._collection_name = collection_name
        self._index_name = index_name
        self._collection: Optional[Collection] = None
```

**Key public methods:**
- `create_collection()` - Creates with HNSW/hybrid index
- `upsert()` / `get()` / `delete()` - CRUD operations
- `query()` - Vector search with filters
- `count()` / `clear()` - Collection management

**Filter expression compilation** (lines 267-343 in `base.py`) converts OpenViking's domain-specific expressions to backend-native filters:

```python
def _compile_filter(self, expr: FilterExpr | Dict[str, Any] | None) -> Dict[str, Any]:
    if isinstance(expr, Eq):
        payload = {"op": "must", "field": expr.field, "conds": [value]}
        if expr.field in self._URI_FIELD_NAMES:
            payload["para"] = "-d=0"  # Path depth parameter
        return payload
    if isinstance(expr, PathScope):
        return {
            "op": "must",
            "field": expr.field,
            "conds": [path],
            "para": f"-d={expr.depth}",  # Recursive depth for directory search
        }
```

#### URI Field Encoding

A notable pattern is URI normalization for storage (lines 218-238 in `base.py`):

```python
@staticmethod
def _encode_uri_field_value(value: Any) -> Any:
    if not isinstance(value, str):
        return value
    suffix = stripped[len("viking://") :].strip("/")
    return f"/{suffix}" if suffix else "/"

@staticmethod
def _decode_uri_field_value(value: Any) -> Any:
    if stripped.startswith("viking://"):
        return stripped
    if not stripped.startswith("/"):
        return value
    return f"viking://{suffix}"
```

**Design insight:** The `viking://` protocol URIs are stored internally as `/path/segments` for backend compatibility, then decoded back to `viking://` format on read. This is transparent to callers.

#### Volcengine Adapter

The Volcengine adapter (`volcengine_adapter.py`) implements the backend for ByteDance's VikingDB cloud service:

```python
class VolcengineCollectionAdapter(CollectionAdapter):
    def __init__(self, *, ak: str, sk: str, region: str,
                 project_name: str, collection_name: str, index_name: str):
        super().__init__(collection_name=collection_name, index_name=index_name)
        self.mode = "volcengine"
        self._ak = ak
        self._sk = sk
        self._region = region
        self._project_name = project_name
```

**Index configuration** (lines 103-125) supports hybrid dense+sparse retrieval:

```python
def _build_default_index_meta(self, *, index_name: str, distance: str,
                              use_sparse: bool, sparse_weight: float,
                              scalar_index_fields: list[str]) -> Dict[str, Any]:
    index_type = "hnsw_hybrid" if use_sparse else "hnsw"
    index_meta: Dict[str, Any] = {
        "IndexName": index_name,
        "VectorIndex": {
            "IndexType": index_type,
            "Distance": distance,
            "Quant": "int8",  # Vector quantization
        },
        "ScalarIndex": scalar_index_fields,
    }
    if use_sparse:
        index_meta["VectorIndex"]["EnableSparse"] = True
        index_meta["VectorIndex"]["SearchWithSparseLogitAlpha"] = sparse_weight
    return index_meta
```

#### VikingVectorIndexBackend Facade

The facade class (`viking_vector_index_backend.py`) manages multi-tenancy with per-account backends:

```python
class VikingVectorIndexBackend:
    """Singleton facade managing per-account backend instances"""

    ALLOWED_CONTEXT_TYPES = {"resource", "skill", "memory"}

    def __init__(self, config: Optional[VectorDBBackendConfig]):
        self._account_backends: Dict[str, _SingleAccountBackend] = {}
        self._root_backend: Optional[_SingleAccountBackend] = None
        # Share single adapter across backends to avoid RocksDB LOCK contention
        self._shared_adapter = create_collection_adapter(config)
```

**Shared adapter pattern** (lines 443-445): Multiple account backends reuse a single adapter with its underlying PersistStore/RocksDB instance to avoid file locking contention.

#### Tenant Isolation

The backend enforces tenant isolation through context-aware filtering (lines 931-995):

```python
@staticmethod
def _tenant_filter(ctx: RequestContext,
                   context_type: Optional[str] = None) -> Optional[FilterExpr]:
    if ctx.role == Role.ROOT:
        return None  # Root bypasses filtering

    user_spaces = [ctx.user.user_space_name(), ctx.user.agent_space_name()]
    resource_spaces = [*user_spaces, ""]
    account_filter = Eq("account_id", ctx.account_id)

    if context_type == "resource":
        return And([account_filter, In("owner_space", resource_spaces)])
    if context_type in {"memory", "skill"}:
        return And([account_filter, In("owner_space", user_spaces)])
```

### Key Files

| File | Purpose |
|------|---------|
| `openviking/storage/vectordb_adapters/base.py` | Abstract adapter, filter compilation, URI encoding |
| `openviking/storage/vectordb_adapters/volcengine_adapter.py` | Volcengine/VikingDB cloud implementation |
| `openviking/storage/vectordb_adapters/vikingdb_private_adapter.py` | Private VikingDB deployment |
| `openviking/storage/vectordb_adapters/http_adapter.py` | HTTP-based remote backend |
| `openviking/storage/vectordb_adapters/local_adapter.py` | Local/in-process backend |
| `openviking/storage/viking_vector_index_backend.py` | Multi-tenant facade |

### Notable Patterns & Technical Debt

1. **Shared adapter for RocksDB**: The comment on lines 40-44 explicitly mentions avoiding LOCK contention - a common distributed systems issue

2. **TODO comment** (line 741): `In("level", [0, 1, 2])  # TODO: smj fix this` - suggests the level filtering was a known workaround

3. **Context type validation** (lines 154-161): Only `resource`, `skill`, `memory` are allowed, enforced at upsert time

4. **Backward-compatible aliases** (lines 345-377): Underscore-prefixed internal methods have non-underscored aliases for backward compatibility

---

## Feature 5: Session Management & Memory Self-Iteration

### Overview

Session management integrates with the tiered context system (L0/L1/L2), automatically compressing conversations, extracting long-term memories, and iteratively improving agent context. The system extracts 6+ categories of memories and uses a ReAct-style loop for memory updates.

### Architecture Components

1. **Session** (`openviking/session/session.py`) - Message container with auto-commit
2. **MemoryExtractor** (`openviking/session/memory_extractor.py`) - Extracts memories from conversations
3. **MemoryArchiver** (`openviking/session/memory_archiver.py`) - Cold storage for stale memories
4. **MemoryReAct** (`openviking/session/memory/memory_react.py`) - ReAct orchestrator for memory updates
5. **MemoryUpdater** (`openviking/session/memory/memory_updater.py`) - Applies LLM operations to storage

### Session Class

The Session class (lines 140-200 in `session.py`) manages conversation state:

```python
class Session:
    def __init__(self, viking_fs, vikingdb_manager=None,
                 session_compressor=None, user=None, ctx=None,
                 session_id=None, auto_commit_threshold=8000):
        self._session_uri = f"viking://session/{self.user.user_space_name()}/{self.session_id}"
        self._messages: List[Message] = []
        self._usage_records: List[Usage] = []
        self._compression: SessionCompression = SessionCompression()
        self._stats: SessionStats = SessionStats()
        self._meta = SessionMeta(session_id=self.session_id, created_at=get_current_timestamp())
```

**SessionMeta structure** tracks extraction counts across memory categories:
- profile, preferences, entities, events (UserMemory)
- cases, patterns (AgentMemory)
- tools, skills (Tool/Skill Memory)

### Memory Extraction Pipeline

The MemoryExtractor (`memory_extractor.py`) processes conversations in stages:

```python
async def extract(self, context: dict, user: UserIdentifier,
                  session_id: str, *, strict: bool = False) -> List[CandidateMemory]:
    # Stage 1: Prepare inputs
    with telemetry.measure("memory.extract.stage.prepare_inputs"):
        formatted_messages = self._format_message_with_parts(messages)
        output_language = self._detect_output_language(messages)

    # Stage 2: Collect tool/skill statistics
    with telemetry.measure("memory.extract.stage.tool_skill_stats"):
        tool_stats_map = collect_tool_stats(tool_parts)
        skill_stats_map = collect_skill_stats(tool_parts)

    # Stage 3: LLM extraction
    with telemetry.measure("memory.extract.stage.llm_extract"):
        response = await vlm.get_completion_async(prompt)

    # Stage 4: Normalize candidates
    with telemetry.measure("memory.extract.stage.normalize_candidates"):
        candidates = self._normalize_candidates(data, tool_stats_map, skill_stats_map)
```

**Memory categories** (lines 40-55):

```python
class MemoryCategory(str, Enum):
    # UserMemory categories
    PROFILE = "profile"
    PREFERENCES = "preferences"
    ENTITIES = "entities"
    EVENTS = "events"
    # AgentMemory categories
    CASES = "cases"
    PATTERNS = "patterns"
    # Tool/Skill Memory
    TOOLS = "tools"
    SKILLS = "skills"
```

**Tool/Skill tracking** extends the base `CandidateMemory` with execution metadata:

```python
@dataclass
class ToolSkillCandidateMemory(CandidateMemory):
    tool_name: str = ""
    skill_name: str = ""
    duration_ms: int = 0
    prompt_tokens: int = 0
    completion_tokens: int = 0
    call_time: int = 0
    success_time: int = 0
    best_for: str = ""
    optimal_params: str = ""
    common_failures: str = ""
```

### Language Detection

The extractor detects output language from user messages only (lines 148-195), avoiding assistant/system text bias:

```python
@staticmethod
def _detect_output_language(messages: List, fallback_language: str = "en") -> str:
    user_text = "\n".join(
        str(getattr(m, "content", "") or "")
        for m in messages
        if getattr(m, "role", "") == "user"
    )
    # CJK disambiguation: Japanese has Kana, Chinese has Han
    kana_count = len(re.findall(r"[\u3040-\u30ff\u31f0-\u31ff\uff66-\uff9f]", user_text))
    han_count = len(re.findall(r"[\u4e00-\u9fff]", user_text))
    if kana_count > 0:
        return "ja"
    if han_count > 0:
        return "zh-CN"
```

### Memory Archiver

The MemoryArchiver (`memory_archiver.py`) moves cold memories to `_archive/` directories:

```python
class MemoryArchiver:
    DEFAULT_THRESHOLD: float = 0.1
    DEFAULT_MIN_AGE_DAYS: int = 7

    async def scan(self, scope_uri: str, ctx=None, now=None) -> List[ArchivalCandidate]:
        # Only scan L2 content -- never archive L0 abstracts or L1 overviews
        filter_expr = And(conds=[Eq("level", 2)])
        # ...
        score = hotness_score(active_count=active_count, updated_at=updated_at, now=now)
        if score < self.threshold:
            candidates.append(ArchivalCandidate(...))
```

**Archive URI transformation:**

```python
def _build_archive_uri(uri: str) -> str:
    # viking://memories/facts/greeting.md
    # -> viking://memories/facts/_archive/greeting.md
    last_slash = uri.rfind("/")
    parent = uri[:last_slash]
    filename = uri[last_slash + 1:]
    return f"{parent}/{ARCHIVE_DIR}/{filename}"
```

### Memory Self-Iteration with ReAct

The `MemoryReAct` class (`memory/memory_react.py`) implements a ReAct loop for memory updates:

```python
class MemoryReAct:
    """
    Simplified ReAct orchestrator for memory updates.

    Workflow:
    0. Pre-fetch: System performs ls + read .overview.md + search
    1. LLM call with tools: Model decides to either use tools OR output final operations
    2. If tools used: Execute and continue loop
    3. If operations output: Return and finish
    """
```

**Pre-fetch optimization** (lines 102-150): Distinguishes between multi-file schemas (directories to `ls`) and single-file schemas (direct read):

```python
for schema in self.registry.list_all(include_disabled=False):
    has_variables = "{" in schema.filename_template and "}" in schema.filename_template
    if has_variables or not schema.filename_template:
        ls_dirs.add(dir_path)  # Multi-file: ls directory
    else:
        file_uri = f"{dir_path}/{schema.filename_template}"
        read_files.add(file_uri)  # Single-file: direct read
```

### Memory Updater

The `MemoryUpdater` (`memory/memory_updater.py`) applies LLM outputs directly without function calls:

```python
class MemoryUpdater:
    """
    Applies MemoryOperations directly using the flat model format.

    This is the system executor - no LLM involved at this stage.
    """

    async def apply_operations(self, operations: Any, ctx: RequestContext,
                               registry=None) -> MemoryUpdateResult:
        # Resolve all URIs first
        resolved_ops = resolve_all_operations(operations, registry, ...)

        # Apply write/edit/delete operations directly
        for op, uri in resolved_ops.write_operations:
            await self._apply_write(op, uri, ctx)
            result.add_written(uri)

        for op, uri in resolved_ops.edit_operations:
            await self._apply_edit(op, uri, ctx)
            result.add_edited(uri)
```

### Key Files

| File | Purpose |
|------|---------|
| `openviking/session/session.py` | Session container with auto-commit |
| `openviking/session/memory_extractor.py` | LLM-based memory extraction from conversations |
| `openviking/session/memory_archiver.py` | Cold storage management via hotness score |
| `openviking/session/memory/memory_react.py` | ReAct orchestrator for memory updates |
| `openviking/session/memory/memory_updater.py` | Direct executor for memory operations |
| `openviking/session/memory/tools.py` | Memory-specific tools for ReAct |

### Notable Patterns

1. **Hotness scoring**: Archives memories with low access frequency/recency to reduce token consumption

2. **Tool/Skill statistics**: Extraction collects `call_time`, `success_time`, `duration_ms`, `prompt_tokens`, `completion_tokens` for optimization insights

3. **Language-specific output**: Detects conversation language and generates memories in the user's language

4. **Direct execution model**: MemoryUpdater applies LLM outputs directly, not via function calls - a simplification from the agent's tool-calling approach

---

## Feature 6: VikingBot AI Agent Framework

### Overview

VikingBot is an AI agent framework built on OpenViking, providing channels (CLI, Telegram, Slack, Lark), agent implementations, tool system, and OpenViking mount integration for context access.

### Architecture

The framework follows an event-driven architecture:

1. **AgentLoop** (`bot/vikingbot/agent/loop.py`) - Core processing engine
2. **ContextBuilder** (`bot/vikingbot/agent/context.py`) - Message context assembly
3. **ToolRegistry** - Dynamic tool registration and execution
4. **MessageBus** - Event publishing/subscribing
5. **Providers** - Multi-provider LLM abstraction
6. **OpenVikingMount** - Filesystem integration

### Agent Loop

The `AgentLoop` class (lines 35-130 in `loop.py`) is the core engine:

```python
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

    def __init__(self, bus: MessageBus, provider: LLMProvider, workspace: Path,
                 model: str | None = None, max_iterations: int = 50,
                 memory_window: int = 50, ...):
        self.bus = bus
        self.provider = provider
        self.context = ContextBuilder(workspace, sandbox_manager=sandbox_manager)
        self.sessions = session_manager or SessionManager(...)
        self.tools = ToolRegistry()
        self.subagents = SubagentManager(...)
```

**Message processing flow** (lines 214-366 in `loop.py`):

```python
async def _run_agent_loop(self, messages: list[dict], session_key: SessionKey,
                         publish_events: bool = True) -> tuple:
    iteration = 0
    while iteration < self.max_iterations:
        iteration += 1

        # Call LLM with tools
        response = await self.provider.chat(
            messages=messages,
            tools=self.tools.get_definitions(),
            model=self.model,
            session_id=session_key.safe_name(),
        )

        if response.has_tool_calls:
            # Execute all tools in parallel
            tool_tasks = [execute_single_tool(idx, tc) for tc in response.tool_calls]
            results = await asyncio.gather(*tool_tasks)

            # Process results sequentially
            for idx, tool_call, result, duration in results:
                messages = self.context.add_tool_result(messages, ...)
        else:
            final_content = response.content
            break
```

**Parallel tool execution** (lines 308-313):

```python
# Run all tool executions in parallel
tool_tasks = [
    execute_single_tool(idx, tool_call)
    for idx, tool_call in enumerate(response.tool_calls)
]
results = await asyncio.gather(*tool_tasks)
```

### OpenViking Mount Integration

The `OpenVikingMount` class (`openviking_mount/mount.py`) provides FUSE-like filesystem access to OpenViking:

```python
class OpenVikingMount:
    """
    将OpenViking的虚拟文件系统映射到本地文件系统操作
    """

    def _uri_to_path(self, uri: str) -> Path:
        # viking://resources/path/to/file -> mount_point/resources/path/to/file
        if uri.startswith("viking://"):
            uri = uri[len("viking://") :]
        parts = uri.split("/", 1)
        return self.config.mount_point / scope / rest
```

**Mount scopes** (lines 20-27):

```python
class MountScope(Enum):
    RESOURCES = "resources"
    SESSION = "session"
    USER = "user"
    AGENT = "agent"
    ALL = "all"
```

**OpenVikingMountManager** (`manager.py`) manages multiple mount points:

```python
class OpenVikingMountManager:
    """管理多个挂载点的创建、访问和销毁"""

    def create_resources_mount(self, mount_id: str = "resources",
                              openviking_data_path: Optional[Path] = None,
                              read_only: bool = False) -> OpenVikingMount:

    def create_session_mount(self, session_id: str,
                            openviking_data_path: Path,
                            read_only: bool = False) -> OpenVikingMount:
```

### Multi-Provider Support

The `LiteLLMProvider` (`providers/litellm_provider.py`) provides unified access to multiple LLM providers:

```python
class LiteLLMProvider(LLMProvider):
    """
    LLM provider using LiteLLM for multi-provider support.

    Supports OpenRouter, Anthropic, OpenAI, Gemini, MiniMax, and many other
    providers through a unified interface.
    """

    def _resolve_model(self, model: str) -> str:
        if self._gateway:
            # Gateway mode: apply gateway prefix
            prefix = self._gateway.litellm_prefix
            if self._gateway.strip_model_prefix:
                model = model.split("/")[-1]
            if prefix and not model.startswith(f"{prefix}/"):
                model = f"{prefix}/{model}"
            return model

        # Standard mode: auto-prefix for known providers
        spec = find_by_model(model)
        if spec and spec.litellm_prefix:
            model = f"{spec.litellm_prefix}/{model}"
        return model
```

**MiniMax system message handling** (lines 107-150): Providers that don't support system messages have them merged into user messages:

```python
def _handle_system_message(self, model: str, messages: list[dict]) -> list[dict]:
    if model.startswith("minimax/") or "/minimax/" in model:
        # Combine all system prompts
        full_system_prompt = "\n\n".join([str(c) for c in system_contents])
        # Merge into the first user message
        for msg in cleaned_messages:
            if msg.get("role") == "user":
                msg["content"] = merge_content(msg["content"], full_system_prompt)
```

### Tool System

Tools are registered dynamically and support sandboxed execution:

```python
def _register_default_tools(self) -> None:
    register_default_tools(
        registry=self.tools,
        config=self.config,
        send_callback=self.bus.publish_outbound,
        subagent_manager=self.subagents,
        cron_service=self.cron_service,
    )
```

### Event Bus

The framework uses an event-driven message bus:

```python
class OutboundMessage:
    session_key: SessionKey
    content: str
    event_type: OutboundEventType  # THINKING, REASONING, TOOL_CALL, TOOL_RESULT, ITERATION

class InboundMessage:
    session_key: SessionKey
    content: str
    sender_id: str
    metadata: dict
```

### Key Files

| File | Purpose |
|------|---------|
| `bot/vikingbot/agent/loop.py` | Core agent processing engine |
| `bot/vikingbot/agent/context.py` | Context assembly |
| `bot/vikingbot/agent/tools.py` | Tool registration |
| `bot/vikingbot/openviking_mount/mount.py` | FUSE-like OpenViking access |
| `bot/vikingbot/openviking_mount/manager.py` | Mount lifecycle management |
| `bot/vikingbot/providers/litellm_provider.py` | Multi-provider LLM abstraction |
| `bot/vikingbot/providers/registry.py` | Provider/model registry |
| `bot/vikingbot/bus/events.py` | Event types |
| `bot/vikingbot/session/manager.py` | Session management |

### Notable Patterns

1. **Parallel tool execution with sequential result processing**: Tools run concurrently via `asyncio.gather`, but results are processed in original order to maintain deterministic message flow

2. **Reasoning content publishing**: The loop publishes internal reasoning separately from tool calls (lines 265-272)

3. **Session key routing**: `SessionKey` with `safe_name()` used for provider session IDs and event routing

4. **Long-running operation monitoring**: 40-second tick checks for long-running operations (lines 390-399)

5. **Gateway/provider auto-detection**: `_gateway = find_gateway(provider_name, api_key, api_base)` determines deployment type

---

## Cross-Feature Insights

### Shared Patterns

1. **Context isolation**: All features respect `RequestContext` for tenant isolation with `account_id`, `user_space_name()`, `agent_space_name()`

2. **Tiered storage**: L0 (abstract ~100 tokens), L1 (overview ~2KB), L2 (full content) appears across features

3. **URI-based addressing**: `viking://` protocol unifies filesystem, resources, memories, skills

4. **Event-driven architecture**: VikingBot uses MessageBus; other components use telemetry events

### Technical Debt Indicators

1. **TODO in production code**: `In("level", [0, 1, 2])  # TODO: smj fix this` (vector backend)

2. **NotImplementedError for writes**: OpenVikingMount.write_file() raises `NotImplementedError("Direct file write requires special handling in OpenViking")`

3. **Backward-compatible aliases**: Underscore-prefixed internal methods have non-underscored aliases suggesting API evolution

### Integration Points

```
VikingBot AgentLoop
    |
    +-- ContextBuilder --> OpenVikingMount (openviking:// access)
    |                           |
    |                           +-- SyncOpenViking client
    |                                   |
    |                                   +-- VikingVectorIndexBackend
    |                                           |
    +-- SessionManager                         +-- CollectionAdapter (multi-backend)
    |                                           |
    +-- ToolRegistry                           +-- Vector DB (Volcengine, VikingDB, etc.)
            |
            +-- Memory tools --> MemoryReAct --> MemoryUpdater
                                                     |
                                                     +-- VikingFS
```

---

## Conclusion

Features 4-6 demonstrate a mature system with:

1. **Vector DB Integration**: Well-abstracted adapter pattern supporting multiple backends with tenant isolation and URI normalization

2. **Session Management**: Comprehensive memory lifecycle with extraction, archival, and self-iteration through ReAct-style reasoning

3. **VikingBot Framework**: Event-driven agent architecture with parallel tool execution, multi-provider support, and seamless OpenViking integration

The architecture shows careful consideration for multi-tenancy, language-specific processing, and the filesystem paradigm unifying disparate data types under `viking://` URIs.
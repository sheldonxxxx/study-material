# Feature Batch 4: Features 10-12 Deep Dive

**Repository:** `/Users/sheldon/Documents/claw/reference/OpenViking/`
**Research Date:** 2026-03-27
**Features Covered:** AGFS C++ Backend (10), OpenClaw/OpenCode Plugin Integration (11), Observability & Telemetry (12)

---

## Feature 10: AGFS (Abstract Google File System) C++ Backend

### Overview

AGFS is a plugin-based filesystem abstraction providing a virtual filesystem layer. The C++ server (`agfs-server`) provides high-performance file operations, while the Python SDK provides client bindings for the OpenViking Python package.

### Architecture

```
OpenViking Python Package
    |
    +-- opnviking/agfs_manager.py (process manager)
    +-- openviking/pyagfs/ (Python SDK)
    |       +-- client.py (HTTP client, 35KB)
    |       +-- binding_client.py (optional native binding)
    |       +-- helpers.py (cp, upload, download utilities)
    |
    +-- openviking_cli/utils/config/agfs_config.py (configuration)
    |
    v
third_party/agfs/ (C++ Implementation)
    +-- agfs-server/ (server implementation)
    +-- agfs-sdk/ (shared SDK)
    +-- agfs-fuse/ (FUSE integration)
    +-- agfs-shell/ (CLI shell)
```

### Key Implementation Files

#### 1. AGFSManager (`openviking/agfs_manager.py`)

Manages the lifecycle of the AGFS server process.

```python
class AGFSManager:
    def __init__(self, config: "AGFSConfig"):
        self.data_path = Path(config.path).resolve()
        self.port = config.port
        self.backend = config.backend  # 'local', 's3', 'memory'
        self.process: Optional[subprocess.Popen] = None
        atexit.register(self.stop)  # Auto-cleanup on exit

    @property
    def binary_path(self) -> Path:
        package_dir = Path(__file__).parent
        binary_name = "agfs-server"
        if platform.system() == "Windows":
            binary_name += ".exe"
        return package_dir / "bin" / binary_name

    def _generate_config(self) -> dict:
        # Generates plugin configuration for:
        # - serverinfofs (server metadata at /serverinfo)
        # - queuefs (async queue with SQLite backend at /queue)
        # - localfs/s3fs/memfs (storage at /local)
        config = {
            "server": {"address": f":{self.port}", "log_level": self.log_level},
            "plugins": {
                "serverinfofs": {"enabled": True, "path": "/serverinfo"},
                "queuefs": {
                    "enabled": True,
                    "path": "/queue",
                    "config": {"backend": "sqlite", "db_path": ...}
                },
            },
        }
        if self.backend == "local":
            config["plugins"]["localfs"] = {"enabled": True, "path": "/local"}
        elif self.backend == "s3":
            config["plugins"]["s3fs"] = {"enabled": True, "path": "/local", ...}
        return config
```

**Notable Technical Debt (from code comments):**
```python
# TODO(multi-node): SQLite backend is single-node only. Each AGFS instance
# gets its own isolated queue.db under its own data_path, so messages
# enqueued on node A are invisible to node B. For multi-node deployments,
# switch backend to "tidb" or "mysql" so all nodes share the same queue.
#
# Additionally, the TiDB backend currently uses immediate soft-delete on
# Dequeue (no two-phase status='processing' transition), meaning there is
# no at-least-once guarantee: a worker crash loses the in-flight message.
# The TiDB backend's Ack() and RecoverStale() are both no-ops and must be
# implemented before it can be used safely in production.
```

#### 2. AGFSClient (`openviking/pyagfs/client.py`)

Pure HTTP client for AGFS Server API. 35KB with comprehensive coverage:

**Core Operations:**
```python
class AGFSClient:
    def __init__(self, api_base_url="http://localhost:8080", timeout=10):
        if not api_base_url.endswith("/api/v1"):
            api_base_url = api_base_url + "/api/v1"
        self.api_base = api_base_url

    # File operations
    def ls(self, path="/") -> List[Dict[str, Any]]       # List directory
    def cat(self, path, offset=0, size=-1, stream=False) # Read file
    def write(self, path, data, max_retries=3) -> str    # Write with retry
    def mkdir(self, path, mode="755")                     # Create directory
    def rm(self, path, recursive=False, force=True)      # Remove file/dir
    def stat(self, path) -> Dict[str, Any]              # File metadata
    def mv(self, old_path, new_path)                     # Rename
    def chmod(self, path, mode: int)                     # Change permissions
    def touch(self, path)                                # Update timestamp

    # Mount/plugin operations
    def mounts(self) -> List[Dict]                        # List mounted plugins
    def mount(self, fstype, path, config)               # Dynamic mount
    def load_plugin(self, library_path)                  # Load external plugin
    def list_plugins(self) -> List[str]                   # List loaded plugins

    # Search
    def grep(self, path, pattern, recursive=False, case_insensitive=False,
             stream=False, node_limit=None)              # Regex search

    # HandleFS (stateful file operations)
    def open_handle(self, path, flags=0, mode=0o644, lease=60) -> FileHandle
    def handle_read(self, handle_id, size=-1, offset=None) -> bytes
    def handle_write(self, handle_id, data, offset=None) -> int
    def handle_seek(self, handle_id, offset, whence=0) -> int
    def handle_sync(self, handle_id)                      # Flush to storage
```

**HandleFS FileHandle (POSIX-like file handles):**
```python
class FileHandle:
    O_RDONLY = 0
    O_WRONLY = 1
    O_RDWR = 2
    O_APPEND = 8
    O_CREATE = 16
    O_EXCL = 32
    O_TRUNC = 64

    SEEK_SET = 0  # Absolute position
    SEEK_CUR = 1  # Relative to current
    SEEK_END = 2  # Relative to end

    def read(self, size=-1) -> bytes
    def read_at(self, size, offset) -> bytes  # pread
    def write(self, data) -> int
    def write_at(self, data, offset) -> int   # pwrite
    def seek(self, offset, whence=0) -> int
    def tell(self) -> int
    def sync(self) -> None
    def renew(self, lease=60) -> dict          # Extend lease
```

**Error Handling Pattern:**
```python
def _handle_request_error(self, e: Exception, operation="request"):
    if isinstance(e, ConnectionError):
        raise AGFSClientError(f"Connection refused - server not running")
    elif isinstance(e, Timeout):
        raise AGFSClientError(f"Request timeout after {self.timeout}s")
    elif isinstance(e, requests.exceptions.HTTPError):
        if status_code == 501:
            raise AGFSNotSupportedError(error_msg)
        # Maps HTTP errors to filesystem-like errors
        elif status_code == 404:
            raise AGFSHTTPError("No such file or directory", status_code)
        elif status_code == 403:
            raise AGFSHTTPError("Permission denied", status_code)
        elif status_code == 409:
            raise AGFSHTTPError("Resource already exists", status_code)
```

**Retry Logic for Writes:**
```python
def write(self, path, data, max_retries=3):
    # Exponential backoff: 1s, 2s, 4s
    for attempt in range(max_retries + 1):
        try:
            response = self.session.put(f"{self.api_base}/files", ...)
        except (ConnectionError, Timeout):
            if attempt < max_retries:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            continue
        except requests.exceptions.HTTPError as e:
            # Only retry 502, 503, 504 (not 500 or 4xx)
            if status_code in [502, 503, 504] and attempt < max_retries:
                time.sleep(2 ** attempt)
                continue
            else:
                self._handle_request_error(e)  # Fail immediately
```

#### 3. AGFSConfig (`openviking_cli/utils/config/agfs_config.py`)

```python
class AGFSConfig(BaseModel):
    port: int = 1833
    backend: str = "local"  # 'local' | 's3' | 'memory'
    mode: str = "binding-client"  # 'http-client' | 'binding-client'
    log_level: str = "warn"
    timeout: int = 10
    retry_times: int = 3
    s3: S3Config  # S3 backend configuration

class S3Config(BaseModel):
    bucket: Optional[str]
    region: Optional[str]
    access_key: Optional[str]
    secret_key: Optional[str]
    endpoint: Optional[str]  # Custom endpoint for MinIO/LocalStack
    prefix: str = ""  # Namespace isolation
    use_ssl: bool = True
    use_path_style: bool = True  # True=PathStyle (MinIO), False=VirtualHostStyle (TOS)
    directory_marker_mode: DirectoryMarkerMode = DirectoryMarkerMode.EMPTY
```

#### 4. C++ Backend Structure (`third_party/agfs/`)

```
third_party/agfs/
|-- agfs-server/    # Main server implementation
|-- agfs-sdk/       # Shared SDK for plugins
|-- agfs-fuse/      # FUSE (Filesystem in Userspace) integration
|-- agfs-shell/     # Interactive shell
|-- agfs-mcp/       # MCP (Model Context Protocol) integration
```

### Key Insights

1. **Plugin Architecture**: AGFS uses a plugin system allowing different filesystem backends (local, S3, memory) to be mounted at paths like `/local`, `/queue`, `/serverinfo`.

2. **HandleFS**: Provides POSIX-like file handle operations with lease management for stateful access patterns.

3. **Optional Native Binding**: The `AGFSBindingClient` is optional - falls back to pure HTTP client if native library unavailable.

4. **Technical Debt**: SQLite queue backend is single-node only; TiDB backend's at-least-once delivery not yet implemented.

---

## Feature 11: OpenClaw/OpenCode Plugin Integration

### Overview

Two plugin implementations demonstrate OpenViking integration with external agent frameworks:
- **OpenClaw Plugin 2.0**: Context-engine architecture with auto-recall/capture
- **OpenCode Memory Plugin**: Tool-based memory access (memsearch, memread, membrowse, memcommit)

### OpenClaw Plugin (`examples/openclaw-plugin/`)

#### Architecture

TypeScript-based plugin registering as OpenClaw's `contextEngine`. Key files:

```
examples/openclaw-plugin/
|-- index.ts           # Main plugin (32KB)
|-- client.ts          # OpenViking client wrapper
|-- config.ts          # Configuration schema
|-- context-engine.ts  # Context engine factory
|-- process-manager.ts # Local server spawning
|-- text-utils.ts      # Text extraction utilities
|-- memory-ranking.ts  # Memory selection/ranking
|-- openclaw.plugin.json
|-- install.sh
```

#### Plugin Registration Pattern

```typescript
const contextEnginePlugin = {
  id: "openviking",
  name: "Context Engine (OpenViking)",
  description: "OpenViking-backed context-engine memory with auto-recall/capture",
  kind: "context-engine" as const,
  configSchema: memoryOpenVikingConfigSchema,

  register(api: OpenClawPluginApi) {
    // Register tools
    api.registerTool({
      name: "memory_recall",
      label: "Memory Recall (OpenViking)",
      execute: async (_toolCallId, params) => {
        const { query, limit, scoreThreshold, targetUri } = params;
        const result = await client.find(query, { targetUri, limit, scoreThreshold });
        return formatMemoryLines(result.memories);
      }
    }, { name: "memory_recall" });

    // Register service (lifecycle management)
    api.registerService({
      id: "openviking",
      start: async () => { /* spawn local server */ },
      stop: () => { /* cleanup */ }
    });

    // Register context engine
    if (typeof api.registerContextEngine === "function") {
      api.registerContextEngine("openviking", () => createMemoryOpenVikingContextEngine(...));
    }
  }
};
```

#### Two Deployment Modes

**Local Mode**: Plugin spawns OpenViking subprocess alongside OpenClaw
```typescript
// In registerService.start():
const pythonCmd = resolvePythonCommand();
const child = spawn(pythonCmd, ["-c", runpyCode], { env, ... });
await waitForHealth(baseUrl);  // Poll health endpoint
```

**Remote Mode**: Connect to existing OpenViking server
```typescript
clientPromise = Promise.resolve(new OpenVikingClient(cfg.baseUrl, cfg.apiKey, ...));
```

#### Hook System Integration

```typescript
api.on("session_start", async (_event, ctx) => { /* map session */ });
api.on("session_end", async (_event, ctx) => { /* cleanup */ });
api.on("before_prompt_build", async (event, ctx) => {
  // Auto-recall: inject relevant memories before prompt
  const memories = await client.find(queryText, {...});
  return { prependContext: formatMemoryLines(memories) };
});
api.on("before_reset", async (_event, ctx) => {
  // Commit session on reset
  await contextEngineRef.commitOVSession(sessionKey);
});
```

### OpenCode Memory Plugin (`examples/opencode-memory-plugin/`)

#### Architecture

TypeScript plugin exposing memory as explicit tools for OpenCode agents.

```
examples/opencode-memory-plugin/
|-- openviking-memory.ts  # Main plugin (63KB)
|-- openviking-config.example.json
|-- README.md
```

#### Tool-Based Memory Access

```typescript
// Four explicit memory tools
export const memsearch = tool({
  name: "memsearch",
  description: "Unified search across memories, resources, and skills",
  parameters: z.object({
    query: z.string(),
    target_uri: z.string().optional(),  // viking://user/memories/
    mode: z.enum(["auto", "fast", "deep"]).optional(),
    limit: z.number().optional(),
    score_threshold: z.number().optional()
  }),
  execute: async (params) => {
    const results = await openVikingClient.find(params.query, {
      targetUri: params.target_uri,
      limit: params.limit,
      scoreThreshold: params.score_threshold
    });
    return formatMemoryLines(results.memories);
  }
});

export const memread = tool({ ... });   // Read from viking:// URI
export const membrowse = tool({ ... });  // Browse filesystem layout
export const memcommit = tool({ ... }); // Trigger memory extraction
```

#### Session Mapping

Maps OpenCode sessions to OpenViking sessions for persistent context:

```typescript
interface SessionMapping {
  ovSessionId: string
  createdAt: number
  capturedMessages: Set<string>  // Deduplication
  messageRoles: Map<string, "user" | "assistant">
  pendingMessages: Map<string, string>  // Buffer before completion
  lastCommitTime?: number
  commitInFlight?: boolean
  commitTaskId?: string
}
```

#### Key Implementation Details

**Message Buffering with TTL:**
```typescript
const MAX_BUFFERED_MESSAGES_PER_SESSION = 100;
const BUFFERED_MESSAGE_TTL_MS = 15 * 60 * 1000;

function flushMessageBuffer(sessionId: string) {
  const buffered = sessionMessageBuffer.get(sessionId) || [];
  const valid = buffered.filter(m => Date.now() - m.timestamp < BUFFERED_MESSAGE_TTL_MS);
  // Send to OpenViking, then clear
}
```

**Token Budget Enforcement:**
```typescript
export async function buildMemoryLinesWithBudget(
  memories: FindResultItem[],
  readFn: (uri: string) => Promise<string>,
  options: BuildMemoryLinesWithBudgetOptions
): Promise<{ lines: string[]; estimatedTokens: number }> {
  let budgetRemaining = options.recallTokenBudget;
  for (const item of memories) {
    if (budgetRemaining <= 0) break;
    const content = await resolveMemoryContent(item, readFn, options);
    const line = `- [${item.category}] ${content}`;
    const lineTokens = estimateTokenCount(line);

    // First line always included even if over budget (spec §6.2)
    if (lineTokens > budgetRemaining && lines.length > 0) break;

    lines.push(line);
    budgetRemaining -= lineTokens;
  }
  return { lines, estimatedTokens: totalTokens };
}
```

**Session Persistence:**
```typescript
interface SessionMapFile {
  version: 1
  sessions: Record<string, SessionMappingPersisted>
  lastSaved: number
}
// Saved to openviking-session-map.json next to plugin file
```

### Comparison: OpenClaw vs OpenCode Plugins

| Aspect | OpenClaw Plugin 2.0 | OpenCode Memory Plugin |
|--------|---------------------|----------------------|
| Architecture | Context Engine | Tool-based |
| Memory Access | Auto-inject via hooks | Explicit tool calls |
| Session Management | Context engine factory | Session mapping with persistence |
| Auto-recall | `before_prompt_build` hook | Tool-based (agent decides) |
| Auto-capture | `afterTurn` hook | `memcommit` tool |
| Deployment | Local (subprocess) or Remote | Remote only |

---

## Feature 12: Observability & Telemetry

### Overview

OpenViking's observability system provides operation-scoped telemetry collection, retrieval statistics, and evaluation capabilities.

### Module Structure

```
openviking/telemetry/
|-- __init__.py
|-- context.py         # ContextVar binding for request-scoped telemetry
|-- operation.py       # OperationTelemetry collector (15KB)
|-- registry.py        # Global telemetry registry
|-- request.py        # Telemetry request parsing
|-- runtime.py        # TelemetryRuntime with MemoryTelemetryMeter
|-- execution.py      # Async telemetry wrapper
|-- resource_summary.py # Resource-specific metrics
|-- snapshot.py       # TelemetrySnapshot

openviking/eval/
|-- __init__.py       # RAGAS evaluation framework
|-- recorder/         # Recording/playback for evaluation
|-- ragas/            # RAG evaluation metrics

openviking/retrieve/
|-- retrieval_stats.py # RetrievalStatsCollector (8KB)
```

### Core Telemetry Architecture

#### 1. OperationTelemetry (`operation.py`)

Low-overhead collector with disabled mode for production safety:

```python
class OperationTelemetry:
    def __init__(self, operation: str, enabled: bool = False):
        self.operation = operation
        self.enabled = enabled
        self.telemetry_id = f"tm_{uuid4().hex}" if enabled else ""
        self._start_time = time.perf_counter()
        self._counters: Dict[str, float] = defaultdict(float)
        self._gauges: Dict[str, Any] = {}
        self._lock = Lock()

    # Counters (monotonically increasing)
    def count(self, key: str, delta: float = 1):
        with self._lock:
            self._counters[key] += delta

    # Gauges (latest value)
    def set(self, key: str, value: Any):
        with self._lock:
            self._gauges[key] = value

    # Duration (accumulates milliseconds)
    def add_duration(self, key: str, duration_ms: float):
        gauge_key = key if key.endswith(".duration_ms") else f"{key}.duration_ms"
        with self._lock:
            existing = self._gauges.get(gauge_key, 0.0)
            self._gauges[gauge_key] = existing + max(float(duration_ms), 0.0)

    # Context manager for timing
    @contextmanager
    def measure(self, key: str):
        start = time.perf_counter()
        yield
        self.add_duration(key, (time.perf_counter() - start) * 1000)

    # Token accounting
    def add_token_usage_by_source(self, source: str, input_tokens: int, output_tokens: int = 0):
        self.count(f"tokens.{source}.input", input_tokens)
        self.count(f"tokens.{source}.output", output_tokens)
        self.count(f"tokens.{source}.total", input_tokens + output_tokens)

    # Error recording
    def set_error(self, stage: str, code: str, message: str):
        with self._lock:
            self._error_stage = stage
            self._error_code = code
            self._error_message = message

    def finish(self, status: str = "ok") -> Optional[TelemetrySnapshot]:
        duration_ms = (time.perf_counter() - self._start_time) * 1000
        summary = TelemetrySummaryBuilder.build(
            operation=self.operation,
            status=status,
            duration_ms=duration_ms,
            counters=dict(self._counters),
            gauges=dict(self._gauges),
            ...
        )
        return TelemetrySnapshot(telemetry_id=self.telemetry_id, summary=summary)
```

#### 2. TelemetrySummaryBuilder

Normalizes raw metrics into structured summary:

```python
class TelemetrySummaryBuilder:
    # Metric key mappings for memory extraction stages
    _MEMORY_EXTRACT_STAGE_KEYS = {
        "prepare_inputs_ms": "memory.extract.stage.prepare_inputs.duration_ms",
        "llm_extract_ms": "memory.extract.stage.llm_extract.duration_ms",
        "normalize_candidates_ms": "memory.extract.stage.normalize_candidates.duration_ms",
        "tool_skill_stats_ms": "memory.extract.stage.tool_skill_stats.duration_ms",
        "profile_create_ms": "memory.extract.stage.profile_create.duration_ms",
        ...
    }

    @classmethod
    def build(cls, operation, status, duration_ms, counters, gauges, ...) -> Dict[str, Any]:
        summary = {
            "operation": operation,
            "status": status,
            "duration_ms": round(float(duration_ms), 3),
            "tokens": {
                "llm": {"input": ..., "output": ..., "total": ...},
                "embedding": {"total": ...}
            }
        }

        if cls._has_metric_prefix("vector", counters, gauges):
            summary["vector"] = {
                "searches": counters.get("vector.searches", 0),
                "scored": counters.get("vector.scored", 0),
                "passed": counters.get("vector.passed", 0),
                "returned": gauges.get("vector.returned", 0),
                "scanned": counters.get("vector.scanned", 0),
                "scan_reason": gauges.get("vector.scan_reason", "")
            }

        if cls._has_metric_prefix("memory", counters, gauges):
            summary["memory"] = {
                "extracted": gauges.get("memory.extracted", 0),
                "extract": {
                    "duration_ms": gauges.get("memory.extract.total.duration_ms", 0),
                    "candidates": {...},
                    "actions": {"created": ..., "merged": ..., "deleted": ...},
                    "stages": {...}
                }
            }
        return summary
```

#### 3. Context Binding

Uses ContextVar for request-scoped telemetry:

```python
# context.py
_NOOP_TELEMETRY = OperationTelemetry(operation="noop", enabled=False)
_CURRENT_TELEMETRY: contextvars.ContextVar[OperationTelemetry] = contextvars.ContextVar(
    "openviking_operation_telemetry",
    default=_NOOP_TELEMETRY,
)

def get_current_telemetry() -> OperationTelemetry:
    return _CURRENT_TELEMETRY.get()

@contextmanager
def bind_telemetry(handle: OperationTelemetry):
    token = _CURRENT_TELEMETRY.set(handle)
    try:
        yield handle
    finally:
        _CURRENT_TELEMETRY.reset(token)
```

#### 4. TelemetryRuntime

Global runtime with in-memory meter:

```python
class MemoryTelemetryMeter:
    def __init__(self):
        self._counters: Dict[Tuple[str, Tuple], float] = {}
        self._gauges: Dict[Tuple[str, Tuple], Any] = {}
        self._histograms: Dict[Tuple[str, Tuple], list[float]] = {}
        self._lock = Lock()

    def increment(self, metric: str, value: float = 1, attrs: Dict[str, Any] | None = None):
        key = (metric, tuple(sorted((attrs or {}).items())))
        with self._lock:
            self._counters[key] = self._counters.get(key, 0) + value

    def set_gauge(self, metric: str, value: Any, attrs: Dict[str, Any] | None = None):
        key = (metric, tuple(sorted((attrs or {}).items())))
        with self._lock:
            self._gauges[key] = value

@dataclass
class TelemetryRuntime:
    meter_instance: MemoryTelemetryMeter = field(default_factory=MemoryTelemetryMeter)

_RUNTIME = TelemetryRuntime()

def get_telemetry_runtime() -> TelemetryRuntime:
    return _RUNTIME
```

#### 5. Registry

Thread-safe global registry for telemetry handles:

```python
_REGISTERED_TELEMETRY: dict[str, OperationTelemetry] = {}
_REGISTERED_TELEMETRY_LOCK = threading.Lock()

def register_telemetry(handle: OperationTelemetry) -> None:
    if not handle.enabled or not handle.telemetry_id:
        return
    with _REGISTERED_TELEMETRY_LOCK:
        _REGISTERED_TELEMETRY[handle.telemetry_id] = handle

def resolve_telemetry(telemetry_id: str) -> OperationTelemetry | None:
    with _REGISTERED_TELEMETRY_LOCK:
        return _REGISTERED_TELEMETRY.get(telemetry_id)
```

### Async Execution Wrapper

```python
async def run_with_telemetry(
    operation: str,
    telemetry: TelemetryRequest,
    fn: Callable[[], Awaitable[T]],
) -> TelemetryExecutionResult[T]:
    selection = parse_telemetry_selection(telemetry)
    collector = OperationTelemetry(operation=operation, enabled=True)

    try:
        with bind_telemetry(collector):
            result = await fn()
    except Exception as exc:
        collector.set_error(operation, type(exc).__name__, str(exc))
        collector.finish(status="error")
        raise

    telemetry_payload = build_telemetry_payload(collector, selection, status="ok")
    return TelemetryExecutionResult(result=result, telemetry=telemetry_payload, selection=selection)
```

### Retrieval Statistics (`retrieve/retrieval_stats.py`)

Thread-safe singleton collector for retrieval metrics:

```python
@dataclass
class RetrievalStats:
    total_queries: int = 0
    total_results: int = 0
    zero_result_queries: int = 0
    total_score_sum: float = 0.0
    max_score: float = 0.0
    min_score: float = float("inf")
    queries_by_type: Dict[str, int] = field(default_factory=dict)
    rerank_used: int = 0
    rerank_fallback: int = 0
    total_latency_ms: float = 0.0
    max_latency_ms: float = 0.0

    @property
    def avg_results_per_query(self) -> float: ...
    @property
    def zero_result_rate(self) -> float: ...
    @property
    def avg_score(self) -> float: ...
    @property
    def avg_latency_ms(self) -> float: ...

class RetrievalStatsCollector:
    def record_query(self, context_type: str, result_count: int,
                     scores: list[float], latency_ms: float = 0.0,
                     rerank_used: bool = False, rerank_fallback: bool = False):
        with self._lock:
            self._stats.total_queries += 1
            self._stats.total_results += result_count
            ...

        # Notify Prometheus observer (if enabled) outside lock
        try:
            from openviking.storage.observers.prometheus_observer import get_prometheus_observer
            prom = get_prometheus_observer()
            if prom is not None:
                prom.record_retrieval(latency_ms / 1000.0)
        except Exception:
            pass
```

### Evaluation Framework (`eval/`)

RAGAS-based evaluation with recording/playback:

```python
from openviking.eval import (
    RagasEvaluator,      # RAG evaluation
    DatasetGenerator,    # Generate eval datasets
    EvalDataset,         # Dataset container
    IOPlayback,          # Record/playback for offline eval
    analyze_records,     # Analyze recorded data
)
```

### Key Insights

1. **Low-Overhead Disabled Mode**: `OperationTelemetry` is designed to have zero overhead when disabled.

2. **Thread-Safe Collectors**: All collectors use locks for safe concurrent access.

3. **ContextVar Binding**: Enables request-scoped telemetry without explicit passing through call chains.

4. **Metric Normalization**: `TelemetrySummaryBuilder` maps internal metric keys to public-facing names.

5. **Prometheus Integration**: Retrieval stats optionally notify Prometheus observer for metrics export.

6. **Zero-Copy Snapshots**: Uses `copy.deepcopy` for stats snapshot to avoid lock contention.

---

## Cross-Feature Relationships

### Feature 10-12 Integration

```
Feature 10 (AGFS Backend)
    |
    +-- Used by Feature 5 (Session Management) for memory storage
    +-- Used by Feature 12 (Telemetry) for queue operations
    |
    v
Feature 11 (Plugin Integration)
    |
    +-- Connects OpenClaw/OpenCode to OpenViking server
    +-- Uses Feature 12 telemetry for auto-recall metrics
    |
    v
Feature 12 (Observability)
    |
    +-- Tracks retrieval statistics across all features
    +-- Memory extraction metrics for session management
    +-- Telemetry attached to HTTP responses in server
```

---

## Notable Technical Debt & Patterns

### AGFS Backend
1. SQLite queue backend is single-node only (no cross-instance messaging)
2. TiDB backend lacks at-least-once delivery guarantee (Ack/RecoverStale are no-ops)
3. Optional native binding gracefully degrades to HTTP client

### Plugin Integration
1. OpenClaw Plugin 2.0 not backward-compatible with legacy `memory-openviking` plugin
2. Historical compatibility issue with OpenClaw `2026.3.12` causing conversation hangs
3. Auto-recall has 5-second timeout protection to prevent agent hangs

### Observability
1. Event capture intentionally disabled for summary-only telemetry (comment: "event capture is intentionally disabled")
2. Prometheus observer import in try/except to avoid hard dependency
3. Token estimation uses chars/4 heuristic (adequate but not precise)

---

## File Summary

| Feature | Key Files | Size | Purpose |
|---------|-----------|------|---------|
| 10 | `openviking/agfs_manager.py` | 10KB | Process lifecycle |
| 10 | `openviking/pyagfs/client.py` | 35KB | HTTP client |
| 10 | `openviking_cli/utils/config/agfs_config.py` | 5KB | Configuration |
| 10 | `third_party/agfs/` | - | C++ implementation |
| 11 | `examples/openclaw-plugin/index.ts` | 32KB | OpenClaw plugin |
| 11 | `examples/opencode-memory-plugin/openviking-memory.ts` | 63KB | OpenCode plugin |
| 12 | `openviking/telemetry/operation.py` | 15KB | Core collector |
| 12 | `openviking/retrieve/retrieval_stats.py` | 8KB | Retrieval metrics |
| 12 | `openviking/eval/` | - | RAGAS evaluation |

# CoPaw Features Deep-Dive

A comprehensive synthesis of CoPaw's feature architecture, implementation patterns, and cross-cutting concerns.

---

## Core Features

### Feature 1: Multi-Channel Communication

**What it does:** CoPaw connects to multiple communication channels (Slack, Discord, Telegram, Mattermost, etc.) enabling agents to interact through familiar platforms. Each channel has its own adapter handling platform-specific protocols, authentication, and message formats.

**How it's implemented:** The channel system uses an adapter pattern with a base `ChannelAdapter` abstract class. Each platform-specific adapter (`SlackAdapter`, `DiscordAdapter`, etc.) implements:
- Message receiving and parsing
- Message sending with platform-specific formatting
- User mapping between platform users and CoPaw's internal user model
- Webhook handling for real-time events
- Rate limit handling and backoff strategies

**Key architectural decisions:**
- Channels are configured via `channels_cmd.py` (35KB, the largest CLI command)
- Channel configuration is stored centrally and loaded at startup
- Each channel runs in its own async context to prevent blocking
- Message normalization: all platform messages are converted to a common `ChannelMessage` format before processing

**Notable code patterns:**
```python
# Base adapter interface pattern
class ChannelAdapter(ABC):
    @abstractmethod
    async def receive(self) -> ChannelMessage: ...
    @abstractmethod
    async def send(self, message: OutgoingMessage) -> None: ...
    @abstractmethod
    async def start_listening(self) -> None: ...
    @abstractmethod
    async def stop_listening(self) -> None: ...
```

**Dependencies:**
- Requires Memory system for conversation context
- Requires Skills system for message handling
- Requires LLM Integration for generating responses

---

### Feature 2: Web Console

**What it does:** A React-based web interface providing real-time monitoring, agent management, chat interaction, and system administration. Serves as the primary operational dashboard for CoPaw deployments.

**How it's implemented:** The console is a React application served by the FastAPI backend:

```
src/copaw/console/          # React frontend source
src/copaw/server/console.py  # FastAPI console routes
```

The frontend communicates with the backend via WebSocket for real-time updates (agent status, chat messages, system metrics) and REST API for CRUD operations.

**Key architectural decisions:**
- Real-time updates via WebSocket (`ws://host/console/ws`)
- Console serves bundled static files from `dist/` directory
- Server-side rendering not used; fully client-side SPA
- State management using React hooks with a custom `useWebSocket` hook for real-time data

**Notable code patterns:**
```javascript
// WebSocket hook pattern
const useWebSocket = (url) => {
  const [data, setData] = useState(null);
  useEffect(() => {
    const ws = new WebSocket(url);
    ws.onmessage = (event) => setData(JSON.parse(event.data));
    return () => ws.close();
  }, [url]);
  return data;
};
```

**Dependencies:**
- Requires FastAPI server running
- No direct dependency on other features (standalone operational UI)

---

### Feature 3: LLM Integration

**What it does:** Unified interface for Large Language Model providers (OpenAI, Anthropic, Google, Azure, custom OpenAI-compatible APIs). Handles authentication, request/response normalization, streaming, and fallback strategies.

**How it's implemented:** Provider abstraction with a factory pattern:

```
src/copaw/providers/
├── base.py              # Provider abstract base class
├── openai_provider.py   # OpenAI + compatible APIs
├── anthropic_provider.py # Anthropic Claude
├── google_provider.py   # Google AI
├── azure_provider.py    # Microsoft Azure OpenAI
└── ollama_provider.py   # Ollama daemon (OpenAI-compatible)
```

Each provider implements:
- Chat completion (streaming and non-streaming)
- Token counting and context management
- Model listing and selection
- API key management and validation
- Error handling and retry logic

**Key architectural decisions:**
- OpenAI-compatible API format as the common interface (`/v1/chat/completions`)
- Providers are selected by model ID at runtime via factory
- `ChatModelBase` interface normalizes across providers
- Connection pooling per provider to reduce overhead

**Notable code patterns:**
```python
# Provider factory pattern
class LLMProviderFactory:
    @staticmethod
    def get_provider(model_id: str) -> LLMProvider:
        if model_id.startswith("gpt-"):
            return OpenAIProvider()
        elif model_id.startswith("claude-"):
            return AnthropicProvider()
        # ... etc

# Streaming response normalization
async def stream_response(provider: LLMProvider, messages: list[Message]):
    async for chunk in provider.stream(messages):
        yield normalize_chunk(chunk)  # Always OpenAI-compatible format
```

**Technical debt/concerns:**
- Multiple providers may have subtle compatibility issues (function calling format varies)
- No built-in circuit breaker pattern for provider failures
- Token counting is provider-specific and may be inaccurate

**Dependencies:**
- Core feature required by most other features (agents, skills, memory)

---

### Feature 4: Skills System

**What it does:** Extensible skill framework allowing agents to perform specific tasks. Skills are self-contained modules with metadata, code, and documentation that can be enabled/disabled and configured per-agent.

**How it's implemented:** Skill structure:
```
skills/
├── skill_name/
│   ├── __init__.py      # Skill class inheriting from SkillBase
│   ├── config.yaml      # Skill metadata and configuration
│   ├── handlers.py      # Event handlers and logic
│   └── README.md        # Skill documentation
└── registry.py         # Central skill registry
```

**Key architectural decisions:**
- Skills are discovered at startup from the `skills/` directory
- Each skill registers handlers for specific events (message, cron, agent_start, etc.)
- Configuration is schema-validated via `config.yaml`
- Skills can depend on other skills via explicit declaration
- Skills sync from remote git repositories

**Notable code patterns:**
```python
class SkillBase(ABC):
    name: str
    version: str

    @abstractmethod
    async def initialize(self, config: dict) -> None: ...
    @abstractmethod
    async def handle_event(self, event: Event) -> Response | None: ...

# Skill registry with lifecycle hooks
class SkillRegistry:
    def register(self, skill: SkillBase) -> None:
        self._skills[skill.name] = skill
        for event_type in skill.handles_events:
            self._dispatcher.register(event_type, skill)
```

**Dependencies:**
- Requires LLM Integration for skills that use AI
- Requires Memory for context
- Works with Multi-Agent system (skills assigned per-agent)

---

### Feature 5: Multi-Agent System

**What it does:** Coordination layer for multiple AI agents that can work collaboratively, delegate tasks, and share context. Supports different agent personas, capabilities, and specialization.

**How it's implemented:**
```
src/copaw/agents/
├── agent.py           # Base Agent class
├── manager.py        # Agent lifecycle management
├── coordinator.py    # Inter-agent coordination
└── registry.py      # Agent registry and discovery
```

**Key architectural decisions:**
- Agents are identified by unique ID and maintain their own state
- Agent communication via message passing with typed envelopes
- `AgentCoordinator` handles routing and delegation
- Agents can spawn sub-agents for parallel task execution
- Heartbeat system monitors agent health and restarts failed agents

**Notable code patterns:**
```python
# Agent message passing
class AgentMessage:
    sender_id: str
    recipient_id: str | None  # None = broadcast
    content: str
    metadata: dict
    reply_to: str | None

# Coordinator delegation
class AgentCoordinator:
    async def delegate_task(self, task: Task, suitable_agents: list[Agent]) -> Agent:
        # Select best agent based on capability matching, load, availability
        ...
```

**Dependencies:**
- Requires LLM Integration (each agent uses a configured LLM)
- Requires Memory (agents maintain conversation context)
- Requires Skills (agents use skills to perform actions)
- Requires Heartbeat for health monitoring

---

### Feature 6: Memory System

**What it does:** Persistent context management for agents. Stores conversation history, learned facts, user preferences, and cross-session data. Provides retrieval by time, relevance, or explicit key.

**How it's implemented:**
```
src/copaw/memory/
├── store.py         # Base memory store interface
├── vector_store.py  # Vector embedding storage (Chroma, FAISS)
├── graph_store.py   # Knowledge graph storage
├── episodic.py      # Conversation/session memory
└── semantic.py      # Semantic memory (facts, preferences)
```

**Key architectural decisions:**
-分层存储: Episdic (conversations), Semantic (facts), Procedural (skills/preferences)
- Vector embeddings for semantic search using configurable backend
- Knowledge graph for relationship tracking between entities
- Automatic summarization of old conversations to save context window
- Memory consolidation during idle periods

**Notable code patterns:**
```python
# Memory retrieval with relevance scoring
class MemoryStore:
    async def retrieve(self, query: str, limit: int = 10) -> list[MemoryEntry]:
        embedding = await self._embed(query)
        scored = await self._vector_store.search(embedding, top_k=limit * 2)
        return self._rerank_and_filter(scored, limit)
```

**Dependencies:**
- Required by Multi-Agent (shared context)
- Required by Skills (memory of past executions)
- Can use external vector stores (Chroma, FAISS, Qdrant)

---

### Feature 7: Heartbeat and Cron System

**What it does:** Scheduled and periodic task execution for agents. Heartbeats provide liveness detection and automatic recovery. Cron jobs enable time-based task scheduling (daily reports, periodic data fetches, etc.).

**How it's implemented:**

**Heartbeat System:**
```
src/copaw/heartbeat/
├── monitor.py    # Liveness monitoring
├── worker.py      # Heartbeat task execution
└── recovery.py   # Automatic recovery procedures
```

Heartbeats run on a configurable interval (default 60 seconds). If an agent misses N heartbeats, it's considered failed and recovery procedures are triggered.

**Cron System:**
```
src/copaw/cron/
├── scheduler.py   # Cron expression parsing and scheduling
├── runner.py       # Job execution with timeout/cancellation
└── jobs.py         # Job definitions and management
```

**Key architectural decisions:**
- Heartbeat uses in-memory timestamp with persistent fallback
- Cron expressions parsed using `croniter` library
- Jobs can be one-shot or recurring
- `HEARTBEAT.md` file defines the default heartbeat query
- Agent active hours restrict when heartbeats are sent

**Notable code patterns:**
```python
# Heartbeat with automatic recovery
class HeartbeatMonitor:
    async def check_and_recover(self) -> None:
        for agent in self._tracked_agents:
            if await agent.is_overdue:
                await self._initiate_recovery(agent)

# Cron job scheduling
class CronScheduler:
    def __init__(self):
        self._croniter = croniter
        self._jobs: dict[str, CronJob] = {}

    def add_job(self, cron_expr: str, job: Callable, **kwargs) -> str:
        # Validates cron expression, calculates next run
        ...
```

**Dependencies:**
- Requires Multi-Agent (heartbeats monitor agents)
- Cron jobs can trigger any skill or agent action
- No external cron daemon required (in-process scheduler)

---

## Secondary Features

### Feature 8: MCP (Model Context Protocol)

**What it does:** Standardized protocol for connecting CoPaw to external tools and data sources. MCP clients can invoke CoPaw skills; CoPaw can use MCP servers as tools.

**How it's implemented:**
```
src/copaw/mcp/
├── client.py      # MCP client implementation
├── server.py       # MCP server for exposing CoPaw skills
└── protocol.py     # Message types and protocol handling
```

**Key architectural decisions:**
- MCP is symmetrical: CoPaw can act as client or server
- Server mode exposes registered skills as MCP tools
- Client mode imports external MCP tools for agent use
- Protocol uses JSON-RPC 2.0 over stdio or HTTP+SSE

**Dependencies:**
- Requires Skills system (skills become MCP tools)
- Works with Multi-Agent (agents use MCP tools)

---

### Feature 9: Security

**What it does:** Authentication, authorization, input sanitization, and secure communication. Enforces channel authentication, API access control, and protects against prompt injection.

**How it's implemented:**
```
src/copaw/security/
├── auth.py         # Authentication (API keys, OAuth, channel auth)
├── permissions.py  # RBAC permission system
├── sanitization.py # Input/output sanitization
└── encryption.py   # Data encryption at rest
```

**Key architectural decisions:**
- API keys for programmatic access
- Per-channel authentication tokens
- RBAC: Owner > Admin > User > Guest roles
- Input sanitization strips potentially dangerous content from user messages
- Sensitive data (API keys, tokens) encrypted at rest

**Notable code patterns:**
```python
# Permission checking decorator
def require_permission(permission: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(user: User, *args, **kwargs):
            if not user.has_permission(permission):
                raise PermissionDenied(permission)
            return await func(user, *args, **kwargs)
        return wrapper
    return decorator
```

**Dependencies:**
- Affects all features (security is cross-cutting)
- Required for multi-user deployments

---

### Feature 10: Local Models

**What it does:** Running LLMs locally via llama.cpp or MLX (Apple Silicon), eliminating API dependency and providing privacy. Supports GGUF format (llama.cpp) and MLX format (Apple Silicon).

**How it's implemented:**
```
src/copaw/local/
├── factory.py              # Singleton backend factory
├── backends/
│   ├── base.py             # Abstract LocalBackend interface
│   ├── llamacpp_backend.py # llama-cpp-python implementation
│   └── mlx_backend.py      # mlx-lm implementation
├── providers/
│   ├── ollama_provider.py  # Ollama daemon integration
│   └── ollama_manager.py   # Ollama model lifecycle
└── manager.py              # Model download and management
```

**Key architectural decisions:**

**Singleton Backend Pattern:**
```python
# factory.py - Reuses loaded backend for same model_id
_lock = threading.Lock()
_active_backend: Optional[LocalBackend] = None
_active_model_id: Optional[str] = None

def get_active_local_model() -> Optional[tuple[Optional[str], LocalBackend]]:
    with _lock:
        if _active_backend is not None and _active_backend.is_loaded:
            return _active_model_id, _active_backend
        return None
```

**Backend Abstraction:**
```python
class LocalBackend(ABC):
    @abstractmethod
    def chat_completion(...) -> dict: ...
    @abstractmethod
    def chat_completion_stream(...) -> Iterator[dict]: ...
    @abstractmethod
    def unload() -> None: ...
    @property
    @abstractmethod
    def is_loaded(self) -> bool: ...
```

**Message Normalization for llama.cpp:**
```python
def _normalize_messages(messages: list[dict]) -> list[dict]:
    """Handle multi-modal content blocks and None tool_calls that crash Jinja."""
    for msg in messages:
        if isinstance(msg.get("content"), list):
            msg["content"] = "\n".join(
                b.get("text", "") if isinstance(b, dict) else b
                for b in msg["content"]
            )
        if msg.get("tool_calls") is None:
            del msg["tool_calls"]  # Jinja templates crash on None
```

**Tag Parser for Local Model Outputs:**
Local models like Qwen3 embed structured data in raw text. The `tag_parser.py` handles `<think>...</think>` blocks (reasoning) and `<tool_call>...</tool_call>` blocks (function calling) with `has_open_tag` flags for incomplete tags during streaming.

**Dependencies:**
- Independent feature (provides LLM capability without external APIs)
- Model downloads from HuggingFace or ModelScope

---

### Feature 11: Desktop Application

**What it does:** Native desktop application using pywebview to embed the React console in a native window. Spawns FastAPI backend as subprocess for a self-contained experience.

**How it's implemented:**
```
src/copaw/cli/desktop_cmd.py    # Main desktop command
scripts/pack/
├── build_common.py             # Shared conda-pack logic
├── build_macos.sh              # macOS .app bundle
├── build_win.ps1               # Windows NSIS installer
└── copaw_desktop.nsi           # Installer template
```

**Key architectural decisions:**

**pywebview Integration:**
```python
# desktop_cmd.py - Subprocess management + webview
proc = subprocess.Popen([
    sys.executable, "-m", "copaw", "app",
    "--host", host, "--port", str(port)
])
if _wait_for_http(host, port):
    webview.create_window("CoPaw Desktop", url, width=1280, height=800)
    webview.start(private_mode=False)
```

**Subprocess Cleanup:**
```python
# Robust process termination with race condition handling
if proc and proc.poll() is None:
    proc.terminate()
    try:
        proc.wait(timeout=5.0)
    except subprocess.TimeoutExpired:
        proc.kill()
```

**macOS Bundle Structure:**
```
CoPaw.app/
└── Contents/
    ├── MacOS/CoPaw          # Launcher script
    ├── Resources/
    │   ├── env/             # Conda-pack environment
    │   └── icon.icns
    └── Info.plist
```

**Dependencies:**
- Requires Web Console (renders console UI)
- Requires CLI (backend is `copaw app` command)
- Platform-specific build scripts

---

### Feature 12: CLI (Command-Line Interface)

**What it does:** Comprehensive CLI for all CoPaw operations. Uses Click with lazy loading for fast startup. Manages agents, channels, skills, providers, models, and system configuration.

**How it's implemented:**
```
src/copaw/cli/
├── main.py           # Entry point with LazyGroup
├── init_cmd.py       # Interactive setup wizard
├── agents_cmd.py     # Agent management
├── channels_cmd.py   # Channel configuration (35KB)
├── providers_cmd.py  # LLM provider + model (889 lines)
├── skills_cmd.py     # Skills management
├── cron_cmd.py       # Cron job management
├── desktop_cmd.py    # Desktop app launcher
└── utils.py          # Shared utilities
```

**Key architectural decisions:**

**Lazy Loading Pattern:**
```python
class LazyGroup(click.Group):
    def __init__(self, lazy_subcommands=None, **kwargs):
        self.lazy_subcommands = lazy_subcommands or {}

    def get_command(self, ctx, cmd_name):
        # Import module only when command is invoked
        if cmd_name in self.lazy_subcommands:
            module_path, attr_name, label = self.lazy_subcommands[cmd_name]
            module = __import__(module_path, fromlist=[attr_name])
            cmd = getattr(module, attr_name)
            self.add_command(cmd, cmd_name)  # Cache for next time
            return cmd
```

**19 Subcommands Registered:**
```
app, channels, agents, chats, cron, desktop, env,
models, providers, skills, update, shutdown, uninstall,
auth, clean, daemon, init, completion, help
```

**Init Command:** 12-step interactive wizard covering security, telemetry, workspace, heartbeat, audio, channels, LLM, skills, and environment variables.

**Technical debt/concerns:**
- `channels_cmd.py` is 35KB - should be split per-channel
- `models` group conflates API providers, local files, and Ollama models
- No validation on `add-provider` for URL format

**Dependencies:**
- Core feature used for all configuration
- Invokes daemon, desktop, and app modes

---

### Feature 13: Docker Support

**What it does:** Containerized deployment via Docker and Docker Compose. Provides reproducible environments, easy scaling, and simplified dependency management.

**How it's implemented:**
```
Dockerfile              # Multi-stage build
docker-compose.yml      # Service orchestration
docker/
├── Dockerfile
└── entrypoint.sh
```

**Key architectural decisions:**
- Multi-stage build: Python deps installed, then copied to minimal runtime image
- Volume mounts for persistent data (`~/.copaw`)
- Environment variable configuration via `.env`
- Health check endpoints for container orchestration
- GPU passthrough support for local models (NVIDIA Docker)

**Notable patterns:**
```dockerfile
# Multi-stage build
FROM python:3.11-slim AS builder
RUN pip install --user -r requirements.txt

FROM python:3.11-slim
COPY --from=builder /root/.local /root/.local
COPY . /app
WORKDIR /app
```

**Dependencies:**
- Independent deployment feature
- Can use Local Models with GPU passthrough
- Can use all other features in containerized form

---

### Feature 14: Voice and Audio

**What it does:** Voice input/output capabilities including speech-to-text transcription and text-to-speech synthesis. Supports multiple backends (auto-detect, native platform APIs).

**How it's implemented:**
```
src/copaw/audio/
├── transcriber.py    # Speech-to-text
├── synthesizer.py    # Text-to-speech
└── vad.py            # Voice activity detection
```

**Key architectural decisions:**
- Backend abstraction similar to LLM providers
- Supported backends: Whisper (local/API), macOS native, Windows native
- Voice activity detection to identify speech segments
- Audio format normalization across platforms
- Configurable language for transcription

**Notable code patterns:**
```python
class AudioBackend(ABC):
    @abstractmethod
    async def transcribe(self, audio_data: bytes) -> str: ...
    @abstractmethod
    async def synthesize(self, text: str) -> bytes: ...
```

**Dependencies:**
- Works with Multi-Agent (voice conversation)
- Works with Skills (voice-activated skills)
- Independent audio pipeline

---

## Cross-Cutting Concerns and Dependencies

### Feature Dependency Map

```
                    ┌─────────────┐
                    │   Security  │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│   Console     │  │   CLI         │  │   Docker      │
└───────┬───────┘  └───────────────┘  └───────────────┘
        │                                      │
        └──────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│                   LLM Integration                    │
│         (OpenAI, Anthropic, Google, Local)          │
└─────────────────────────┬───────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ Multi-Agent   │  │ Memory System │  │ Skills System │
└───────┬───────┘  └───────────────┘  └───────────────┘
        │                │                │
        │                │                │
        └────────────────┼────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │  Multi-Channel     │
              │  Communication     │
              └─────────┬─────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   ┌─────────┐     ┌─────────┐     ┌─────────┐
   │  Slack  │     │ Discord │     │Telegram │
   └─────────┘     └─────────┘     └─────────┘

              ┌─────────────────────┐
              │  Heartbeat/Cron     │
              │  (monitoring)       │
              └─────────┬───────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │   Desktop App       │
              │   (pywebview)       │
              └─────────────────────┘

              ┌─────────────────────┐
              │   Voice/Audio       │
              └─────────────────────┘
```

### Shared Patterns Across Features

| Pattern | Used By |
|---------|---------|
| Abstract Base Class (ABC) | LocalBackend, ChannelAdapter, SkillBase, LLMProvider |
| Factory Pattern | LLMProviderFactory, LocalModelFactory |
| Singleton | Local backend instance, Agent registry |
| Lazy Loading | CLI commands, Plugin loading |
| Message Envelope | Agent communication, Channel messages |
| Event Dispatcher | Skills, Heartbeat, Cron |

### Common Technical Debt Themes

1. **Provider normalization**: LLM providers return slightly different formats for function calling, requiring normalization adapters
2. **Configuration sprawl**: Channel, skill, agent, and provider configs stored separately with no unified schema
3. **Error handling inconsistency**: Some features have robust error handling; others rely on exceptions propagating
4. **Testing gaps**: Integration tests require running services (Ollama, Discord, etc.)

---

## Summary

CoPaw is a multi-channel AI agent platform with sophisticated capabilities for multi-agent orchestration, local LLM execution, and extensible skill systems. The architecture follows established patterns:

**Strengths:**
- Clean abstraction layers (providers, backends, adapters)
- Lazy loading for CLI performance
- Unified OpenAI-compatible API format across providers
- Comprehensive CLI for all operations
- Desktop app for non-technical users

**Key architectural choices:**
- Async/await throughout for concurrency
- Abstract base classes for extensibility
- Singleton pattern for expensive resources (loaded models)
- Message passing for agent communication
- In-process cron/heartbeat (no external dependencies)

**Notable technical concerns:**
- Two separate Ollama systems (provider vs manager)
- MLX structured output via prompt engineering (not native)
- CLI conflating different model types
- No circuit breakers for provider failures
- conda-unpack bug workaround on Windows

**Dependencies flow:**
The system builds from LLM Integration upward: Local Models and Providers form the foundation, then Memory and Skills provide capability layers, Multi-Agent coordinates them, Multi-Channel exposes them to users, and Heartbeat/Cron maintains system health. Desktop, CLI, and Console provide operational interfaces.

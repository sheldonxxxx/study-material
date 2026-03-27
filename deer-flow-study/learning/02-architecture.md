# deer-flow Architecture

## Overview

deer-flow is a **layered agent architecture** with **event-driven channel integrations** and a **strict harness/app separation**. The system implements an AI super-agent using LangGraph for orchestration, with multiple messaging platform integrations bridged through an async pub/sub message bus.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend (Next.js)                       │
│              TanStack Query + React Hooks + Streaming            │
└────────────────────────┬────────────────────────────────────────┘
                         │ LangGraph SDK (HTTP)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LangGraph Server (port 2024)                  │
│              Lead Agent + Middleware Chain + Tools               │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Harness Layer (deerflow.*)               │ │
│  │  agents/  sandbox/  subagents/  tools/  models/  skills/   │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                   Gateway API (port 8001)                         │
│              FastAPI - REST endpoints for config,                │
│              memory, uploads, artifacts, MCP, skills              │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                Channels Layer (IM Integrations)                  │
│        MessageBus pub/sub + ChannelManager dispatcher            │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                   │
│  │  Feishu   │  │   Slack   │  │ Telegram  │                   │
│  └───────────┘  └───────────┘  └───────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

## Architectural Decisions

### AD-01: Harness/App Split

**Decision**: Separate the codebase into two layers with strict dependency direction.

- **Harness** (`packages/harness/deerflow/`): Publishable agent framework package (`deerflow-harness`). Import prefix: `deerflow.*`. Contains agent orchestration, tools, sandbox, models, MCP, skills, config.
- **App** (`app/`): Unpublished application code. Import prefix: `app.*`. Contains FastAPI Gateway API and IM channel integrations.

**Dependency Rule**: App imports deerflow, but deerflow never imports app.

```python
# Harness internal
from deerflow.agents import make_lead_agent
from deerflow.models import create_chat_model

# App -> Harness (allowed)
from deerflow.config import get_app_config

# Harness -> App (FORBIDDEN)
# from app.gateway.routers.uploads import ...  # CI fails
```

**Evidence**: `tests/test_harness_boundary.py` scans all Python files in harness and fails if any `from app.` or `import app.` statement is found.

### AD-02: Two-Layer Backend

**Decision**: Backend split into LangGraph Server (agent runtime) and Gateway API (auxiliary REST endpoints).

| Service | Port | Responsibility |
|---------|------|----------------|
| LangGraph Server | 2024 | Agent runtime, thread management, streaming runs |
| Gateway API | 8001 | Models list, MCP config, skills, memory, uploads, artifacts |

Frontend communicates via **LangGraph SDK** for thread operations and **Gateway REST API** for auxiliary operations.

### AD-03: Virtual Path System

**Decision**: Agent operates in virtualized filesystem space, mapped to physical paths.

- Agent sees: `/mnt/user-data/{workspace,uploads,outputs}`, `/mnt/skills`
- Physical: `backend/.deer-flow/threads/{thread_id}/user-data/...`
- Translation via `replace_virtual_path()` in `packages/harness/deerflow/sandbox/tools.py`

### AD-04: Config-Driven Architecture

**Decision**: All models, tools, and providers instantiated via configuration with reflection.

- Config file: `config.yaml` at project root
- Config values starting with `$` resolved as environment variables
- Class instantiation via `resolve_class(path, base_class)` in `packages/harness/deerflow/reflection/resolvers.py`

---

## Module Map and Responsibilities

### Backend - Harness Layer (`packages/harness/deerflow/`)

| Module | Files | Responsibility |
|--------|-------|----------------|
| `agents/lead_agent/` | `agent.py`, `prompt.py` | Main agent factory (`make_lead_agent`), system prompt template |
| `agents/middlewares/` | 12 middleware files | 12-stage middleware chain components |
| `agents/memory/` | `updater.py`, `queue.py`, `prompt.py` | LLM-based memory extraction, debounced update queue, fact storage |
| `agents/thread_state.py` | Single file | `ThreadState` extension with `sandbox`, `artifacts`, `todos`, `uploaded_files` |
| `sandbox/` | `sandbox.py`, `sandbox_provider.py`, `tools.py`, `local/` | Abstract `Sandbox` interface, `SandboxProvider` lifecycle, virtual path translation |
| `subagents/` | `executor.py`, `registry.py`, `builtins/` | Background task delegation, `SubagentExecutor` with thread pools |
| `tools/builtins/` | 7 tool files | `present_files`, `ask_clarification`, `view_image`, `task`, etc. |
| `mcp/` | `client.py`, `cache.py`, `tools.py` | MCP client (`MultiServerMCPClient`), lazy tool loading, mtime-based cache invalidation |
| `models/factory.py` | Single file | `create_chat_model()` with thinking/vision support, config-driven instantiation |
| `skills/` | `loader.py`, `parser.py`, `installer.py` | Skills discovery, metadata parsing, installation |
| `config/` | 16 config files | YAML config loader, env var resolution, mtime-based cache invalidation |
| `community/` | `tavily/`, `jina_ai/`, `firecrawl/`, `image_search/`, `aio_sandbox/` | External integrations |
| `client.py` | Single file | `DeerFlowClient` - embedded mode without HTTP services |

### Backend - App Layer (`app/`)

#### Gateway (`app/gateway/`)

FastAPI application on port 8001.

| Router | Prefix | Key Endpoints |
|--------|--------|---------------|
| `agents.py` | `/api/agents` | `GET /` list, `POST /` create, `PUT /{name}` update |
| `artifacts.py` | `/api/threads/{id}/artifacts` | `GET /{path}` serve artifacts |
| `channels.py` | `/api/channels` | `GET /` list configured channels |
| `mcp.py` | `/api/mcp` | `GET/PUT /config` MCP server configuration |
| `memory.py` | `/api/memory` | `GET /`, `GET /config`, `GET /status`, `POST /reload` |
| `models.py` | `/api/models` | `GET /` list models, `GET /{name}` details |
| `skills.py` | `/api/skills` | `GET /`, `GET /{name}`, `PUT /{name}`, `POST /install` |
| `suggestions.py` | `/api/threads/{id}/suggestions` | `POST /` generate follow-up questions |
| `threads.py` | `/api/threads/{id}` | `DELETE /` cleanup local thread data |
| `uploads.py` | `/api/threads/{id}/uploads` | `POST /`, `GET /list`, `DELETE /{filename}` |

#### Channels (`app/channels/`)

| File | Responsibility |
|------|----------------|
| `base.py` | Abstract `Channel` base class with `start()`, `stop()`, `send()` lifecycle |
| `message_bus.py` | `MessageBus` - async pub/sub hub connecting channels to dispatcher |
| `manager.py` | `ChannelManager` - consumes inbound, dispatches to LangGraph, handles streaming/non-streaming |
| `store.py` | JSON-file persistence mapping `channel:chat[:topic]` to LangGraph `thread_id` |
| `service.py` | Manages lifecycle of all configured channels from `config.yaml` |
| `feishu.py` | Feishu (Lark) platform integration with streaming card updates |
| `slack.py` | Slack integration (non-streaming) |
| `telegram.py` | Telegram integration (non-streaming) |

### Frontend (`frontend/src/`)

Next.js 16 application with App Router.

| Directory | Files | Responsibility |
|-----------|-------|----------------|
| `core/threads/` | `hooks.ts`, `types.ts` | Thread creation, streaming (`useThreadStream`), state hooks |
| `core/api/` | `api-client.ts` | `getAPIClient()` - LangGraph SDK singleton |
| `core/artifacts/` | Multiple files | Artifact loading and caching |
| `core/mcp/` | Multiple files | MCP configuration UI integration |
| `core/memory/` | Multiple files | User memory display |
| `core/models/` | Multiple files | Model selection types and hooks |
| `core/skills/` | Multiple files | Skills management UI |
| `components/workspace/` | Multiple files | Chat UI (messages, artifacts, settings panels) |
| `hooks/` | Multiple files | Shared React hooks (global shortcuts, mobile detection) |

---

## Communication Patterns

### Frontend to Backend

1. **LangGraph SDK** (`@langchain/langgraph-sdk`) - direct HTTP to LangGraph Server
   - Thread management, streaming runs
   - Singleton via `getAPIClient()` in `core/api/api-client.ts`

2. **Gateway REST API** - FastAPI on port 8001
   - Models list, MCP config, skills, memory, uploads, artifacts
   - Used for auxiliary operations not in LangGraph SDK

### Backend Internal - MessageBus Pub/Sub

```
Channel implementations
    │ publish_inbound()
    ▼
MessageBus._inbound_queue
    │ get_inbound()
    ▼
ChannelManager._dispatch_loop()
    │ runs.wait() / runs.stream()
    ▼
LangGraph Server
    │
    │ (response)
    ▼
ChannelManager
    │ publish_outbound()
    ▼
MessageBus._outbound_listeners
    │ callbacks
    ▼
Channel implementations (send)
```

### LangGraph Server Internal - Middleware Chain

12-stage middleware executes in strict order:

1. `ThreadDataMiddleware` - Creates per-thread directories
2. `UploadsMiddleware` - Tracks and injects uploaded files
3. `SandboxMiddleware` - Acquires sandbox, stores `sandbox_id`
4. `DanglingToolCallMiddleware` - Injects placeholder tool responses
5. `GuardrailMiddleware` - Pre-tool-call authorization (optional)
6. `SummarizationMiddleware` - Context reduction (optional)
7. `TodoListMiddleware` - Task tracking via `write_todos` (optional)
8. `TitleMiddleware` - Auto-generates thread title
9. `MemoryMiddleware` - Queues conversations for memory update
10. `ViewImageMiddleware` - Injects base64 image data (vision models)
11. `SubagentLimitMiddleware` - Enforces max concurrent subagents
12. `ClarificationMiddleware` - Intercepts `ask_clarification` tool calls

---

## Data Flow Examples

### Thread Creation and Streaming (Frontend)

1. User types message in `frontend/src/components/workspace/`
2. `useThreadStream` hook captures via `thread.submit()`
3. LangGraph SDK sends `POST /threads/{id}/runs/stream` to LangGraph Server
4. SSE stream delivers `messages-tuple` and `values` events
5. Hook updates React state; components re-render

### IM Channel Message Flow (Feishu)

1. Feishu webhook delivers message to `feishu.py` endpoint
2. `FeishuChannel` wraps as `InboundMessage`, calls `message_bus.publish_inbound()`
3. `ChannelManager._dispatch_loop()` consumes via `bus.get_inbound()`
4. Looks up/creates LangGraph thread via `client.threads.create()`
5. Uses `runs.stream()` for Feishu (supports streaming)
6. Accumulates AI text via `_accumulate_stream_text()`
7. Publishes incremental `OutboundMessage(is_final=False)` to bus
8. Feishu callback patches running card message
9. Final `OutboundMessage(is_final=True)` confirms completion
10. FeishuChannel sends final text via `send()` -> Feishu API

### File Upload Flow

1. User attaches file in frontend UI
2. `uploadFiles()` sends `POST /api/threads/{id}/uploads` to Gateway
3. Gateway's `uploads.py` router handles via `markitdown` conversion
4. File stored in `.deer-flow/threads/{thread_id}/user-data/uploads/`
5. Metadata returned: `{ filename, size, virtual_path }`
6. `UploadsMiddleware` in LangGraph injects file list into conversation state
7. Agent can read files via sandbox tools

---

## Key Abstractions

### Channel Base Class

```python
# backend/app/channels/base.py
class Channel(ABC):
    def __init__(self, name: str, bus: MessageBus, config: dict[str, Any]) -> None:
        self.name = name
        self.bus = bus
        self.config = config

    @abstractmethod
    async def start(self) -> None: ...

    @abstractmethod
    async def stop(self) -> None: ...

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None: ...
```

### Sandbox Interface

```python
# backend/packages/harness/deerflow/sandbox/sandbox.py
class Sandbox(ABC):
    @abstractmethod
    def execute_command(self, command: str) -> str: ...

    @abstractmethod
    def read_file(self, path: str) -> str: ...

    @abstractmethod
    def list_dir(self, path: str, max_depth=2) -> list[str]: ...

    @abstractmethod
    def write_file(self, path: str, content: str, append: bool = False) -> None: ...
```

### SandboxProvider

```python
# backend/packages/harness/deerflow/sandbox/sandbox_provider.py
class SandboxProvider(ABC):
    @abstractmethod
    def acquire(self, thread_id: str | None = None) -> str: ...

    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox | None: ...

    @abstractmethod
    def release(self, sandbox_id: str) -> None: ...
```

### MessageBus

```python
# backend/app/channels/message_bus.py
class MessageBus:
    def __init__(self) -> None:
        self._inbound_queue: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self._outbound_listeners: list[OutboundCallback] = []

    async def publish_inbound(self, msg: InboundMessage) -> None: ...
    async def get_inbound(self) -> InboundMessage: ...
    def subscribe_outbound(self, callback: OutboundCallback) -> None: ...
    async def publish_outbound(self, msg: OutboundMessage) -> None: ...
```

### ThreadState

```python
# backend/packages/harness/deerflow/agents/thread_state.py
class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]
    todos: NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

---

## Configuration Architecture

### Config Files

| File | Purpose | Location |
|------|---------|----------|
| `config.yaml` | Main config: models, tools, sandbox, memory, title, summarization | Project root |
| `extensions_config.json` | MCP servers, skills enabled state | Project root |

### Config Priority

1. Explicit `config_path` argument
2. `DEER_FLOW_CONFIG_PATH` env var
3. `config.yaml` in current directory
4. `config.yaml` in parent directory (recommended)

---

## Dependency Rules

### Harness/App Boundary

```
Harness (deerflow.*) ──NO DEPENDENCY──► App (app.*)
                  ◄─── CAN IMPORT ─────
```

This allows:
- `app.channels.manager` imports `langgraph_sdk` and `deerflow.config.paths`
- `deerflow.agents.lead_agent` is pure agent logic with no FastAPI dependency

### Frontend/Backend Boundary

```
Frontend ──HTTP/SSE──► LangGraph SDK
Frontend ──REST──────► Gateway API
```

No shared code; all communication is protocol-based.

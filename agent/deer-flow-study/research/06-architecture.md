# deer-flow Architecture Analysis

## Architectural Pattern

**deer-flow** is a **layered agent architecture** with **event-driven channel integrations** and a **strict harness/app separation**.

The system implements an AI super-agent using LangGraph for orchestration, with multiple messaging platform integrations bridged through an async pub/sub message bus.

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
│  │                    Harness Layer (deerflow.*)              │ │
│  │  agents/  sandbox/  subagents/  tools/  models/  skills/  │ │
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

---

## Module Map and Responsibilities

### Backend - Harness Layer (`packages/harness/deerflow/`)

The harness is a **publishable Python package** (`deerflow-harness`) that contains all agent logic. It must never import from `app.*`.

| Module | Responsibility |
|--------|----------------|
| `agents/lead_agent/` | Main agent factory (`make_lead_agent`), system prompt template, 12 middleware components |
| `agents/thread_state.py` | `AgentState` extension with `sandbox`, `artifacts`, `todos`, `uploaded_files` |
| `agents/memory/` | LLM-based memory extraction, debounced update queue, fact storage |
| `sandbox/` | Abstract `Sandbox` interface, `SandboxProvider` lifecycle (acquire/release), virtual path translation |
| `sandbox/local/` | `LocalSandboxProvider` - singleton filesystem execution |
| `subagents/` | Background task delegation, `SubagentExecutor` with thread pools, built-in agents (general-purpose, bash) |
| `tools/builtins/` | `present_files`, `ask_clarification`, `view_image` built-in tools |
| `tools/` | Tool assembly (`get_available_tools`) combining sandbox, built-in, MCP, community, subagent |
| `mcp/` | MCP client (`MultiServerMCPClient`), lazy tool loading, mtime-based cache invalidation |
| `models/factory.py` | `create_chat_model()` with thinking/vision support, config-driven instantiation |
| `skills/` | Skills discovery from `skills/{public,custom}/`, metadata parsing, installation |
| `config/` | YAML config loader with env var resolution, mtime-based cache invalidation |
| `community/` | External integrations: Tavily search, Jina AI fetch, Firecrawl scrape, DuckDuckGo image search |
| `client.py` | `DeerFlowClient` - embedded mode without HTTP services |

### Backend - App Layer (`app/`)

The app layer contains **unpublished application code** that imports from harness.

#### Gateway (`app/gateway/`)

FastAPI application on port 8001. Health check at `GET /health`.

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

Bridges external messaging platforms to the LangGraph agent.

| File | Responsibility |
|------|----------------|
| `base.py` | Abstract `Channel` base class with `start()`, `stop()`, `send()` lifecycle |
| `message_bus.py` | `MessageBus` - async pub/sub hub connecting channels to dispatcher |
| `manager.py` | `ChannelManager` - consumes inbound, dispatches to LangGraph, handles streaming/non-streaming channels |
| `store.py` | JSON-file persistence mapping `channel:chat[:topic]` to LangGraph `thread_id` |
| `service.py` | Manages lifecycle of all configured channels from `config.yaml` |
| `feishu.py` | Feishu (Lark) platform integration with streaming card updates |
| `slack.py` | Slack integration (non-streaming) |
| `telegram.py` | Telegram integration (non-streaming) |

### Frontend (`frontend/src/`)

Next.js 16 application with App Router.

| Directory | Responsibility |
|-----------|----------------|
| `core/` | Business logic - threads, API client, artifacts, memory, skills, MCP, models, i18n |
| `core/threads/` | Thread creation, streaming (`useThreadStream`), state hooks |
| `core/api/` | `getAPIClient()` - LangGraph SDK singleton |
| `core/artifacts/` | Artifact loading and caching |
| `core/mcp/` | MCP configuration UI integration |
| `core/memory/` | User memory display |
| `core/models/` | Model selection types and hooks |
| `core/skills/` | Skills management UI |
| `components/workspace/` | Chat UI (messages, artifacts, settings panels) |
| `hooks/` | Shared React hooks (global shortcuts, mobile detection) |

---

## Key Abstractions and Interfaces

### Backend Abstractions

**1. Channel Base Class** (`app/channels/base.py`)
```python
class Channel(ABC):
    @abstractmethod
    async def start(self) -> None: ...
    @abstractmethod
    async def stop(self) -> None: ...
    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None: ...
```

**2. Sandbox Interface** (`packages/harness/deerflow/sandbox/sandbox.py`)
```python
class Sandbox(ABC):
    async def execute_command(self, command: str, cwd: str | None = None) -> CommandResult: ...
    async def read_file(self, path: str) -> str: ...
    async def write_file(self, path: str, content: str) -> None: ...
    async def list_dir(self, path: str) -> list[str]: ...
```

**3. SandboxProvider** (`packages/harness/deerflow/sandbox/sandbox.py`)
```python
class SandboxProvider(ABC):
    async def acquire(self, thread_id: str) -> Sandbox: ...
    async def get(self, sandbox_id: str) -> Sandbox | None: ...
    async def release(self, sandbox_id: str) -> None: ...
```

**4. MessageBus** (`app/channels/message_bus.py`)
```python
class MessageBus:
    async def publish_inbound(self, msg: InboundMessage) -> None: ...
    async def get_inbound(self) -> InboundMessage: ...
    def subscribe_outbound(self, callback: OutboundCallback) -> None: ...
    async def publish_outbound(self, msg: OutboundMessage) -> None: ...
```

**5. Model Factory** (`packages/harness/deerflow/models/factory.py`)
```python
def create_chat_model(name: str, thinking_enabled: bool = False) -> BaseChatModel: ...
```

### Frontend Interfaces

**AgentThreadState** (`core/threads/types.ts`)
```typescript
interface AgentThreadState extends Record<string, unknown> {
  title: string;
  messages: Message[];
  artifacts: string[];
  todos?: Todo[];
}
```

**AgentThreadContext** (`core/threads/types.ts`)
```typescript
interface AgentThreadContext {
  thread_id: string;
  model_name: string | undefined;
  thinking_enabled: boolean;
  is_plan_mode: boolean;
  subagent_enabled: boolean;
  reasoning_effort?: "minimal" | "low" | "medium" | "high";
}
```

**Agent** (`core/agents/types.ts`)
```typescript
interface Agent {
  name: string;
  description: string;
  model: string | null;
  tool_groups: string[] | null;
  soul?: string | null;
}
```

---

## Component Communication

### Frontend to Backend

The frontend communicates via **two channels**:

1. **LangGraph SDK** (`@langchain/langgraph-sdk`) - direct HTTP to LangGraph Server
   - Thread management, streaming runs
   - Singleton via `getAPIClient()` in `core/api/api-client.ts`

2. **Gateway REST API** - FastAPI on port 8001
   - Models list, MCP config, skills, memory, uploads, artifacts
   - Used for auxiliary operations not in LangGraph SDK

### Backend Internal

**MessageBus pub/sub** decouples channels from the agent dispatcher:

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

### LangGraph Server Internal

The lead agent processes messages through a **12-stage middleware chain**:

1. ThreadDataMiddleware - creates per-thread directories
2. UploadsMiddleware - tracks and injects uploaded files
3. SandboxMiddleware - acquires sandbox, stores `sandbox_id`
4. DanglingToolCallMiddleware - injects placeholder tool responses
5. GuardrailMiddleware - pre-tool-call authorization (optional)
6. SummarizationMiddleware - context reduction (optional)
7. TodoListMiddleware - task tracking via `write_todos` (optional)
8. TitleMiddleware - auto-generates thread title
9. MemoryMiddleware - queues conversations for memory update
10. ViewImageMiddleware - injects base64 image data (vision models)
11. SubagentLimitMiddleware - enforces max concurrent subagents
12. ClarificationMiddleware - intercepts `ask_clarification` tool calls

---

## Design Patterns in Use

### 1. Pub/Sub (MessageBus)
**File:** `app/channels/message_bus.py`
Decouples inbound message sources from the dispatcher. Channels publish without knowing who consumes; dispatcher subscribes without knowing message sources.

### 2. Abstract Base Class (Channel)
**File:** `app/channels/base.py`
Defines the contract for platform integrations. All IM platforms implement `start()`, `stop()`, `send()`.

### 3. Manager / Dispatcher (ChannelManager)
**File:** `app/channels/manager.py`
Central coordinator that reads from MessageBus, creates/reuses threads, routes commands, handles streaming vs blocking responses.

### 4. Middleware Chain (Lead Agent)
**File:** `packages/harness/deerflow/agents/lead_agent/agent.py`
Sequential processing where each middleware wraps the next, modifying state or behavior at each stage.

### 5. Factory (Model Factory)
**File:** `packages/harness/deerflow/models/factory.py`
Config-driven object creation with reflection (`resolve_variable()`).

### 6. Provider (Sandbox)
**File:** `packages/harness/deerflow/sandbox/sandbox.py`
Abstract resource acquisition lifecycle (`acquire`/`get`/`release`) enabling pluggable backends (local, Docker).

### 7. Singleton (APIClient)
**File:** `frontend/src/core/api/api-client.ts`
Module-level singleton ensures single HTTP client connection pool.

### 8. Hook Pattern (Frontend)
**Files:** `core/threads/hooks.ts`, `core/agents/hooks.ts`, etc.
Custom React hooks encapsulate stateful logic, providing reactive data interface to components.

### 9. Promise-based Async Queue (MessageBus)
**File:** `app/channels/message_bus.py`
Uses `asyncio.Queue` for producer/consumer decoupling with backpressure.

### 10. Strategy (Streaming vs Blocking)
**File:** `app/channels/manager.py`
`_handle_chat()` and `_handle_streaming_chat()` implement different strategies based on channel capability (`CHANNEL_CAPABILITIES`).

---

## Dependency Rules

**Harness/App Boundary** (enforced by `tests/test_harness_boundary.py`):

```
Harness (deerflow.*) ──NO DEPENDENCY──► App (app.*)
                  ◄─── CAN IMPORT ─────
```

This allows:
- `app.channels.manager` imports `langgraph_sdk` and `deerflow.config.paths`
- `deerflow.agents.lead_agent` is pure agent logic with no FastAPI dependency

**Frontend/Backend Boundary:**

```
Frontend ──HTTP/SSE──► LangGraph SDK
Frontend ──REST──────► Gateway API
```

No shared code; all communication is protocol-based.

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
10. FeishuChannel sends final text via `send()` → Feishu API

### File Upload Flow

1. User attaches file in frontend UI
2. `uploadFiles()` sends `POST /api/threads/{id}/uploads` to Gateway
3. Gateway's `uploads.py` router handles via `markitdown` conversion
4. File stored in `.deer-flow/threads/{thread_id}/user-data/uploads/`
5. Metadata returned: `{ filename, size, virtual_path }`
6. `UploadsMiddleware` in LangGraph injects file list into conversation state
7. Agent can read files via sandbox tools

---

## Configuration Architecture

**Config Files:**

| File | Purpose | Location |
|------|---------|----------|
| `config.yaml` | Main config: models, tools, sandbox, memory, title, summarization | Project root |
| `extensions_config.json` | MCP servers, skills enabled state | Project root |

**Config Priority:**
1. Explicit `config_path` argument
2. `DEER_FLOW_CONFIG_PATH` env var
3. `config.yaml` in current directory
4. `config.yaml` in parent directory (recommended)

**Virtual Path System:**
- Agent sees: `/mnt/user-data/{workspace,uploads,outputs}`, `/mnt/skills`
- Physical: `backend/.deer-flow/threads/{thread_id}/user-data/...`
- Translation via `replace_virtual_path()` in `packages/harness/deerflow/sandbox/tools.py`

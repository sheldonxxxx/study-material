# Batch 4 Feature Analysis

**Repository:** deer-flow
**Batch Features:** Guardrails/Safety System, Gateway HTTP API, Artifact System
**Analysis Date:** 2026-03-26

---

## Feature 1: Guardrails/Safety System

### Overview

Guardrails is a middleware-based pre-tool-call authorization system that evaluates every tool call against a configurable policy before execution. It provides deterministic, policy-driven authorization without requiring human intervention.

### Core Implementation

**Location:** `backend/packages/harness/deerflow/guardrails/`

**Files:**
- `provider.py` - Protocol and data structures
- `middleware.py` - GuardrailMiddleware implementation
- `builtin.py` - AllowlistProvider implementation
- `__init__.py` - Public exports

### Architecture

The system follows a middleware chain pattern integrated into LangGraph's agent execution:

```
GuardrailMiddleware (position 5 in middleware chain)
         │
         ▼
┌─────────────────────────┐
│   GuardrailProvider     │  ← Pluggable via config
│   (Protocol)            │
└─────────┬───────────────┘
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
Built-in AllowlistProvider   OAP Passport Provider   Custom Provider
(zero deps)                  (open standard)         (user code)
```

### Data Structures (`provider.py`)

```python
@dataclass
class GuardrailRequest:
    tool_name: str
    tool_input: dict[str, Any]
    agent_id: str | None = None
    thread_id: str | None = None
    is_subagent: bool = False
    timestamp: str = ""

@dataclass
class GuardrailReason:
    code: str           # OAP-compliant reason code
    message: str = ""

@dataclass
class GuardrailDecision:
    allow: bool
    reasons: list[GuardrailReason] = field(default_factory=list)
    policy_id: str | None = None
    metadata: dict[str, Any] = field(default_factory=dict)
```

The system uses OAP (Open Agent Passport) reason codes for standardization:
- `oap.allowed` - Tool call authorized
- `oap.tool_not_allowed` - Tool not in allowlist
- `oap.command_not_allowed` - Command not in allowed list
- `oap.blocked_pattern` - Command matches blocked pattern
- `oap.evaluator_error` - Provider error (triggers fail-closed)

### GuardrailMiddleware (`middleware.py`)

Key design decisions:

1. **Implements `AgentMiddleware` pattern** - Same pattern as `ToolErrorHandlingMiddleware`, using `wrap_tool_call` and `awrap_tool_call` methods

2. **Fail-closed by default** - If provider raises exception, tool call is blocked (`fail_closed=True`). Can be set to `fail_open=False` to allow through with warning

3. **GraphBubbleUp propagation** - LangGraph control signals (interrupt/pause/resume) always propagate through, never caught

4. **Denied calls return error ToolMessage** - Agent sees denial reason and can adapt with alternative approach

5. **Passport forwarding** - Optional `passport` field passed to provider for OAP-based evaluations

```python
class GuardrailMiddleware(AgentMiddleware[AgentState]):
    def __init__(self, provider: GuardrailProvider, *, fail_closed: bool = True, passport: str | None = None):
        self.provider = provider
        self.fail_closed = fail_closed
        self.passport = passport
```

### Built-in AllowlistProvider (`builtin.py`)

Simple allowlist/denylist provider with zero external dependencies:

```python
class AllowlistProvider:
    def __init__(self, *, allowed_tools: list[str] | None = None, denied_tools: list[str] | None = None):
        self._allowed = set(allowed_tools) if allowed_tools else None
        self._denied = set(denied_tools) if denied_tools else set()
```

**Decision logic:**
- If `_allowed` is set and tool not in allowlist → DENY
- If tool in `_denied` → DENY
- Otherwise → ALLOW

### Configuration

Configured via `config.yaml`:

```yaml
guardrails:
  enabled: true
  fail_closed: true          # Default: block on provider error
  passport: null              # OAP passport reference
  provider:
    use: deerflow.guardrails.builtin:AllowlistProvider
    config:
      denied_tools: ["bash", "write_file"]
```

### Clever Solutions / Design Patterns

1. **Protocol-based provider interface** - Uses `@runtime_checkable Protocol` instead of abstract base class, allowing any class with required methods to work without inheritance

2. **Framework injection** - Provider `__init__` receives `framework="deerflow"` kwarg, allowing provider to know which system it's running in

3. **Config caching** - GuardrailConfig singleton with reset capability for clean reloading

4. **Async-first design** - Both sync `evaluate` and async `aevaluate` methods

### Technical Debt / Concerns

1. **Provider loading via string path** - Uses `resolve_variable()` which imports by class path string - standard pattern but magic strings can be fragile

2. **No built-in rate limiting** - Only authorization, no throttling of repeated tool calls

3. **Limited audit trail** - Only logs warnings on denial, no persistent audit log

4. **Passport format OAP-specific** - While flexible, the provider interface is heavily shaped by OAP concepts

---

## Feature 2: Gateway HTTP API

### Overview

The Gateway API is a FastAPI application (port 8001) providing REST endpoints for models, MCP, skills, memory, artifacts, uploads, threads, agents, suggestions, and IM channels. It complements the LangGraph API by handling management operations separate from agent runtime.

### Core Implementation

**Location:** `backend/app/gateway/`

**Files:**
- `app.py` - FastAPI application factory with lifespan management
- `config.py` - GatewayConfig (host, port, CORS origins)
- `path_utils.py` - Virtual path resolution helper
- `routers/` - API route handlers

### Router Architecture

| Router | Prefix | Purpose |
|--------|--------|---------|
| `models.py` | `/api/models` | List/query available LLM models |
| `mcp.py` | `/api/mcp` | MCP server configuration |
| `skills.py` | `/api/skills` | Skill listing, enable/disable, install |
| `memory.py` | `/api/memory` | Global memory data |
| `uploads.py` | `/api/threads/{id}/uploads` | File upload management |
| `artifacts.py` | `/api/threads/{id}/artifacts` | Artifact file serving |
| `threads.py` | `/api/threads/{id}` | Thread cleanup |
| `agents.py` | `/api/agents` | Custom agent CRUD |
| `suggestions.py` | `/api/threads/{id}/suggestions` | Follow-up question generation |
| `channels.py` | `/api/channels` | IM channel status |

### Application Lifecycle (`app.py`)

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup:
    1. Load config via get_app_config()
    2. Start IM channel service (if any configured)
    # Shutdown:
    1. Stop channel service
```

**Key design:** MCP tools NOT initialized in Gateway - they're lazily initialized in LangGraph Server (separate process).

### Models Router (`models.py`)

Simple configuration-driven model listing:

```python
GET /api/models                    # List all configured models
GET /api/models/{model_name}       # Get specific model details
```

Response includes: `name`, `model` (provider identifier), `display_name`, `description`, `supports_thinking`, `supports_reasoning_effort`

### Skills Router (`skills.py`)

```python
GET  /api/skills                    # List all skills
GET  /api/skills/{skill_name}       # Get skill details
PUT  /api/skills/{skill_name}       # Update enabled state
POST /api/skills/install            # Install from .skill archive
```

**Notable:** Skill installation extracts `.skill` ZIP files from thread virtual paths.

### Memory Router (`memory.py`)

```python
GET  /api/memory                    # Get memory data
POST /api/memory/reload             # Force reload from disk
GET  /api/memory/config             # Get memory config
GET  /api/memory/status             # Get config + data combined
```

### Uploads Router (`uploads.py`)

```python
POST /api/threads/{thread_id}/uploads       # Upload files (multipart)
GET  /api/threads/{thread_id}/uploads/list  # List uploaded files
DELETE /api/threads/{thread_id}/uploads/{filename}
```

**Features:**
- Automatic document conversion (PDF, PPT, Excel, Word) to Markdown
- Sandbox-aware file synchronization
- Virtual path management

### Artifacts Router (`artifacts.py`)

```python
GET /api/threads/{thread_id}/artifacts/{path}
```

**Security measures:**
1. **Active content always downloaded** - HTML/XHTML/SVG forced as attachment to prevent XSS
2. **Path traversal protection** - Via `resolve_thread_virtual_path()`
3. **ZIP archive extraction** - For `.skill` files, extracts internal files (e.g., `SKILL.md`)
4. **Content type detection** - MIME type sniffing for text vs binary
5. **Cache headers** - 5-minute cache for ZIP extraction results

**Active content types (always downloaded):**
```python
ACTIVE_CONTENT_MIME_TYPES = {
    "text/html",
    "application/xhtml+xml",
    "image/svg+xml",
}
```

### Agents Router (`agents.py`)

Full CRUD for custom agents:

```python
GET    /api/agents                        # List all custom agents
GET    /api/agents/check?name=x           # Check name availability
GET    /api/agents/{name}                # Get agent (with SOUL.md)
POST   /api/agents                        # Create agent
PUT    /api/agents/{name}                 # Update agent
DELETE /api/agents/{name}                 # Delete agent
GET    /api/user-profile                  # Get USER.md
PUT    /api/user-profile                  # Update USER.md
```

**Agent name validation:** `^[A-Za-z0-9-]+$` (hyphen-case)

### Suggestions Router (`suggestions.py`)

```python
POST /api/threads/{thread_id}/suggestions
```

**Implementation approach:**
- Takes recent conversation messages
- Generates follow-up questions via LLM
- Parses JSON array response with robust fallback
- Handles rich content types (list/block model content normalized)

**Prompt engineering:**
- Exactly N questions requested
- Same language as user
- Concise (<=20 words / <=40 Chinese chars)
- JSON array only, no markdown or numbering

### Path Resolution (`path_utils.py`)

```python
def resolve_thread_virtual_path(thread_id: str, virtual_path: str) -> Path:
    return get_paths().resolve_virtual_path(thread_id, virtual_path)
```

- Translates sandbox virtual paths (e.g., `/mnt/user-data/outputs/file.txt`) to actual filesystem paths
- Raises `ValueError` on path traversal attempts
- HTTP status: 403 for traversal, 400 for other errors

### Clever Solutions / Design Patterns

1. **Routed middleware inclusion** - Clean separation via `include_router()` calls with tags

2. **Virtual path abstraction** - Sandbox paths abstracted from actual filesystem layout

3. **Thread-isolated uploads** - Each thread has isolated uploads directory

4. **Skill archive handling** - `.skill` files treated as ZIP with internal path resolution

5. **Agent name normalization** - Lowercase storage, validated pattern

6. **Lifespan context manager** - Clean startup/shutdown for channel services

### Technical Debt / Concerns

1. **No authentication** - All endpoints publicly accessible (documented limitation)

2. **CORS handled by nginx** - FastAPI CORS config not actually used

3. **MCP tools split responsibility** - Gateway manages MCP config, LangGraph manages MCP tools - can cause confusion about which process owns what

4. **Error handling inconsistency** - Some endpoints return `500` with generic message + detailed server logs, others return detailed errors

5. **No request validation middleware** - Pydantic models used per-endpoint but no global validation

---

## Feature 3: Artifact System

### Overview

The Artifact System provides rich content rendering for code blocks, web previews, markdown, HTML, and visualizations. It spans both backend (artifact file serving) and frontend (rendering components).

### Backend Implementation

**Location:** `backend/app/gateway/routers/artifacts.py`

**Endpoint:** `GET /api/threads/{thread_id}/artifacts/{path}`

**Features:**
1. Virtual path resolution with traversal protection
2. Active content (HTML/XHTML/SVG) always served as download
3. ZIP extraction for `.skill` files
4. Text vs binary content type detection
5. Cache headers for ZIP extraction (5 minutes)

### Frontend Implementation

**Locations:**
- `frontend/src/components/workspace/artifacts/` - UI components
- `frontend/src/components/ai-elements/code-block.tsx` - Code highlighting
- `frontend/src/components/ai-elements/web-preview.tsx` - Web preview iframe
- `frontend/src/components/workspace/code-editor.tsx` - CodeMirror editor
- `frontend/src/core/artifacts/` - Business logic

### Component Architecture

```
ArtifactFileList (artifact trigger)
         │
         ▼
ArtifactFileDetail (main viewer)
    │
    ├── CodeEditor (CodeMirror)
    │     - Syntax highlighting via @uiw/react-codemirror
    │     - Multiple language support (Python, JS, HTML, CSS, JSON, Markdown)
    │
    ├── ArtifactFilePreview (content renderer)
    │     │
    │     ├── Markdown → Streamdown (markdown renderer)
    │     └── HTML → sandboxed iframe (sandbox="allow-scripts allow-forms")
    │
    └── ArtifactActions (toolbar)
          - Copy to clipboard
          - Download
          - Open in new window
          - Install skill (.skill files only)
          - Close
```

### Core Components

**ArtifactsProvider (`context.tsx`)**
```typescript
interface ArtifactsContextType {
  artifacts: string[];
  selectedArtifact: string | null;
  autoSelect: boolean;
  select: (artifact: string, autoSelect?: boolean) => void;
  open: boolean;
  setOpen: (open: boolean) => void;
}
```

State management for artifact sidebar, handles auto-selection and sidebar visibility.

**ArtifactFileList (`artifact-file-list.tsx`)**
- Card-based file listing
- Per-file actions: Install skill (for `.skill`), Download
- Click to select and view

**ArtifactFileDetail (`artifact-file-detail.tsx`)**
- File selector dropdown (when multiple artifacts)
- View mode toggle: Code vs Preview (for HTML/Markdown)
- Toolbar with actions

**CodeBlock (`code-block.tsx`)**
```typescript
highlightCode(code, language, showLineNumbers)
// Uses Shiki for syntax highlighting
// Dual theme support: light (one-light) + dark (one-dark-pro)
```

Features:
- Dual-theme (light/dark) with CSS toggle
- Line numbers via custom Shiki transformer
- Copy button with success feedback

**WebPreview (`web-preview.tsx`)**
- URL input with navigation
- iframe with comprehensive sandbox permissions
- Collapsible console for log viewing

**CodeEditor (`code-editor.tsx`)**
- CodeMirror 6 based
- Language extensions: CSS, HTML, JavaScript, JSON, Markdown, Python
- Theme switching (light/dark)
- Lazy fallback to textarea during loading

### Content Loading (`loader.ts`)

```typescript
loadArtifactContent({ filepath, threadId, isMock })
// 1. Appends "/SKILL.md" for .skill files
// 2. Fetches via urlOfArtifact()
// 3. Returns text content

loadArtifactContentFromToolCall({ url, thread })
// Extracts content from tool call arguments
// Used for "write-file:" virtual URLs
```

### URL Generation (`utils.ts`)

```typescript
urlOfArtifact({ filepath, threadId, download, isMock })
// Mock mode: /mock/api/threads/{id}/artifacts{filepath}
// Production: /api/threads/{id}/artifacts{filepath}
// download param adds ?download=true
```

### Artifact Type Detection

From `artifact-file-detail.tsx`:
```typescript
const isCodeFile = checkCodeFile(filepath)  // From core/utils/files
const isSkillFile = filepath.endsWith(".skill")
const isSupportPreview = language === "html" || language === "markdown"
```

### Security Measures

1. **HTML sandboxing** - `sandbox="allow-scripts allow-forms"` prevents script execution while allowing forms

2. **Active content download** - HTML/XHTML/SVG never rendered inline, always forced as attachment

3. **Path traversal protection** - Backend validates virtual paths

4. **Skill ZIP extraction** - Only specific internal paths accessible

### Clever Solutions / Design Patterns

1. **Dual-theme code highlighting** - Pre-renders both light/dark HTML, CSS controls which is visible (no flash)

2. **Streamdown for Markdown** - Custom markdown renderer with `ArtifactLink` component for internal links

3. **Write-file virtual URL handling** - `write-file:{path}?tool_call_id=...&message_id=...` pattern extracts content directly from tool call

4. **Artifact caching** - 5-minute staleTime in React Query for ZIP extraction results

5. **Auto view mode** - Preview mode automatically selected for HTML/Markdown files

6. **Lazy CodeMirror** - Falls back to textarea during stream loading to avoid rendering glitches

### Technical Debt / Concerns

1. **Limited preview types** - Only HTML and Markdown have preview; other files show code view

2. **No syntax highlighting for all languages** - CodeMirror supports many but not all possible file types

3. **Artifact list in thread state** - `thread.values.artifacts` is array of strings, no metadata (type, size, etc.)

4. **No versioning** - Artifacts are static once created, no revision history

5. **Memory consideration** - Pre-rendered dual-theme HTML could be memory-intensive for large files

6. **No search within artifacts** - Cannot search inside artifact content

---

## Cross-Feature Observations

### Shared Patterns

1. **Thread-isolated resources** - Uploads, artifacts, and sandbox all use thread ID for isolation

2. **Virtual path system** - `/mnt/user-data/{workspace,uploads,outputs}` pattern used consistently

3. **Provider/plugin architecture** - Guardrails uses Provider protocol, MCP uses similar pattern

4. **Configuration-driven** - All features heavily configuration-based with YAML/JSON

### Integration Points

1. **Guardrails evaluates tool calls** - Including `write_file` which creates artifacts
2. **Artifacts served via Gateway** - Same API surface as uploads and other thread resources
3. **Skills can be artifacts** - `.skill` files are ZIPs that can be installed from artifact list

### Potential Improvements

1. **Unified resource management** - Uploads, artifacts, and thread data all have separate APIs
2. **Consistent error responses** - Some endpoints generic 500, others detailed errors
3. **Artifact metadata** - Current artifact list is just strings, could include type/size/created
4. **Guardrails audit trail** - No persistent logging of authorization decisions

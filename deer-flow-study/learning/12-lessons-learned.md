# Lessons Learned from deer-flow

An honest assessment of what worked, what did not, and what surprised us in the deer-flow codebase.

---

## What to Emulate

### 1. Middleware Chain Architecture (12-stage pipeline)

The lead agent uses a 12-stage middleware chain that processes requests sequentially. Each middleware wraps the next, modifying state or behavior at each stage:

```
ThreadDataMiddleware → UploadsMiddleware → SandboxMiddleware →
DanglingToolCallMiddleware → GuardrailMiddleware → SummarizationMiddleware →
TodoListMiddleware → TitleMiddleware → MemoryMiddleware →
ViewImageMiddleware → SubagentLimitMiddleware → ClarificationMiddleware
```

**Why it works:** Clear separation of concerns, each middleware is independently testable and configurable. Adding a new middleware is a single-file change.

**Key insight:** Middleware executes in a defined order. Context injection (UploadsMiddleware) happens BEFORE the agent runs, so the agent sees complete context including uploaded files.

---

### 2. Harness/App Split

The codebase enforces a strict architectural boundary:

- **`packages/harness/deerflow/`** — Publishable Python package (`deerflow-harness`) containing all agent logic
- **`app/`** — Unpublished FastAPI application that imports from harness

```
Harness (deerflow.*) ──NO DEPENDENCY──► App (app.*)
                  ◄─── CAN IMPORT ─────
```

**Enforced by:** `tests/test_harness_boundary.py` — automated test that fails CI if harness imports app.

**Why it matters:** The harness can be published to PyPI and used standalone (as `DeerFlowClient` demonstrates). The app layer is the integration layer, not the framework.

---

### 3. Virtual Path Abstraction

Agent code sees only virtual paths:

```
/mnt/user-data/workspace   →   backend/.deer-flow/threads/{thread_id}/user-data/workspace
/mnt/user-data/uploads     →   backend/.deer-flow/threads/{thread_id}/user-data/uploads
/mnt/user-data/outputs     →   backend/.deer-flow/threads/{thread_id}/user-data/outputs
/mnt/skills                →   /path/to/skills
```

**Why it works:** Same agent code runs in local mode, Docker containers, or Kubernetes pods. Path translation is a single module (`sandbox/tools.py`).

---

### 4. Pub/Sub MessageBus Pattern

Channels (Telegram, Slack, Feishu) publish to a MessageBus without knowing who consumes:

```python
class MessageBus:
    async def publish_inbound(self, msg: InboundMessage) -> None
    async def get_inbound(self) -> InboundMessage
    def subscribe_outbound(self, callback: OutboundCallback) -> None
    async def publish_outbound(self, msg: OutboundMessage) -> None
```

**Why it works:** Channels are decoupled from dispatcher. Adding a new platform means implementing `Channel` interface and registering in config. The dispatcher does not change.

---

### 5. Configuration-Driven Architecture

Everything configured via `config.yaml`:

```yaml
models:
  - name: claude-sonnet-4.6
    use: deerflow.models.claude_provider:ClaudeChatModel
    model: claude-sonnet-4-6
    max_tokens: 16384
    enable_prompt_caching: true

tools:
  - name: web_search
    group: web
    use: deerflow.community.infoquest.tools:web_search_tool
    search_time_range: 10
```

**Why it works:** New tools, models, and channels require no code changes — only config. The factory pattern (`resolve_class()`) dynamically loads any class from its import path.

---

### 6. Comprehensive Backend Test Suite

67 test files covering:
- Middleware (dangling tool calls, guardrails, loop detection, subagent limits, todos, titles)
- Config (ACP, app config reload, model config, version)
- Client (77 unit tests including Gateway conformance)
- Memory (updater, prompt injection, upload filtering)
- Skills (parser, loader, installer, archive root)
- Channels (Feishu parser)
- Sandbox (Docker mode detection, local encoding, tools security)
- MCP (OAuth, sync wrapper)
- Boundary enforcement

**Why it works:** TDD policy in CONTRIBUTING.md: "Every new feature or bug fix MUST be accompanied by unit tests. No exceptions."

---

### 7. Skills System

Skills are directories with `SKILL.md` containing YAML frontmatter:

```yaml
---
name: deep-research
description: Conduct comprehensive web research on a topic
license: MIT
---
# Deep Research Skill

[Detailed guidance for the agent]
```

**Why it works:** Skills are data, not code. Users can create custom skills without touching the framework. Built-in skills ship in `skills/public/`, custom skills in `skills/custom/`.

---

### 8. Atomic File Operations

Throughout the codebase, file writes use temp file + rename:

```python
temp_path = file_path.with_suffix(".tmp")
with open(temp_path, "w", encoding="utf-8") as f:
    json.dump(memory_data, f, indent=2, ensure_ascii=False)
temp_path.replace(file_path)  # Atomic on POSIX
```

**Used in:** Memory updater, ChannelStore, DeerFlowClient, config writes.

---

### 9. Model Agnostic Design

`create_chat_model()` accepts any LangChain-compatible model:

```python
def create_chat_model(name: str | None = None, thinking_enabled: bool = False, **kwargs) -> BaseChatModel
```

Works with OpenAI, Anthropic, DeepSeek, Google GenAI, Claude Code OAuth, Codex CLI — or any provider implementing OpenAI-compatible API.

---

### 10. Lazy Initialization

- MCP tools: loaded on first use, cache invalidated by config file mtime
- Model clients: created on first `stream()` call
- Config: cached with mtime-based reload
- LangGraph client: `_get_client()` creates on first use

**Why it matters:** Startup time is fast. Expensive operations (MCP initialization, model loading) happen only when needed.

---

## What to Avoid

### 1. No Python Static Type Checking

Type hints exist but mypy/pyright is not in CI:

```python
# Type hints are documentation only
def create_chat_model(name: str | None = None, thinking_enabled: bool = False, **kwargs) -> BaseChatModel
```

**Impact:** Type errors only caught at runtime. Refactoring is riskier.

**Fix:** Add `uvx mypy deerflow/ app/` to CI.

---

### 2. Minimal Frontend Testing

Only 1 test file exists (`frontend/src/core/api/stream-mode.test.ts`). Core logic — thread hooks, API client, artifact loading — has zero test coverage.

**Impact:** Frontend refactoring is high-risk. Regression detection relies on manual testing.

**Fix:** Add Vitest + React Testing Library for component and hook tests.

---

### 3. Global Singletons

Module-level singletons make testing harder:

```python
# queue.py
def get_memory_queue() -> MemoryUpdateQueue:
    global _memory_queue  # Must call reset_memory_queue() in tests

# service.py
def get_channel_service() -> ChannelService:  # Same pattern
```

**Impact:** Tests must carefully reset state between runs. Parallel test execution is fragile.

**Fix:** Use dependency injection or context managers instead.

---

### 4. print() for Logging

In production code:

```python
# queue.py lines 64, 82, 103, 110, 117, 121
print(f"[MemoryUpdateQueue] Processing {len(items)} items")
```

**Impact:** No log levels, no structured output, no file rotation. Observability is limited.

**Fix:** Use `logging.getLogger(__name__)` throughout.

---

### 5. No Pagination

Thread messages and file listings return all data:

```python
# No offset/limit on thread messages
# No pagination on file listings via list_files_in_dir()
```

**Impact:** Long-running threads with many messages will be slow to load. Threads with hundreds of uploads timeout.

**Fix:** Add cursor-based pagination. Consider virtual scrolling in frontend.

---

### 6. Wildcard CORS in Production

```nginx
add_header 'Access-Control-Allow-Origin' '*' always;
```

**Impact:** Any website can make API requests to DeerFlow backend. Appropriate for development, dangerous in production.

**Fix:** Restrict to known origins via environment variable.

---

### 7. Hard-coded Provider Awareness

Factory has explicit awareness of specific providers:

```python
# factory.py
if issubclass(model_class, CodexChatModel):
    # Handle reasoning_effort mapping
```

**Impact:** Adding a new CLI provider requires modifying the factory. Violates open/closed principle.

**Fix:** Use a provider registry pattern instead.

---

### 8. No Auth Rate Limiting

Authentication endpoints have no rate limiting:

```python
# No slowapi or similar on auth endpoints
```

**Impact:** Brute-force attacks on login are possible. DoS via repeated auth requests.

**Fix:** Add `slowapi` with per-IP limits.

---

### 9. Singleton Local Sandbox

Local sandbox is a true singleton:

```python
# local_sandbox_provider.py
_singleton: LocalSandbox | None = None

def acquire(self, thread_id: str | None = None) -> str:
    global _singleton
    if _singleton is None:
        _singleton = LocalSandbox(...)
    return _singleton.id
```

**Impact:** All threads share the same filesystem. No isolation between concurrent conversations.

**Fix:** Per-thread sandbox instances in local mode.

---

### 10. Hard-coded Timeouts

```python
# Subagent timeout: 900 seconds (15 minutes), not configurable per-task
# Command timeout: 600 seconds (10 minutes), hard-coded in execute_command()
```

**Impact:** Long-running tasks cannot request more time. Quick tasks cannot opt out of 10-minute wait.

**Fix:** Make timeouts configurable per-invocation.

---

## What Surprised Us

### 1. Shell Scripts for CLI Integration

The `claude-to-deerflow` skill uses pure `curl` + Python SSE parsing:

```bash
curl -N -X POST "${DEERFLOW_LANGGRAPH_URL}/threads/${thread_id}/runs/stream" \
  -H "Content-Type: application/json" \
  -d "{...}"
```

**Why surprising:** No SDK dependency. Just HTTP + shell. Works from any terminal.

---

### 2. OAuth Credential Auto-Loading

Claude Code OAuth tokens are automatically loaded from:

```python
# credential_loader.py checks in order:
1. $CLAUDE_CODE_OAUTH_TOKEN
2. $CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR
3. $CLAUDE_CODE_CREDENTIALS_PATH
4. ~/.claude/.credentials.json
```

**Why surprising:** Zero-configuration auth when Claude Code is already configured on the machine.

---

### 3. Emoji Logging in Production

InfoQuest uses emoji prefixes for debug logs:

```python
logger.info("\n============================================\n🚀 BytePlus InfoQuest Client Initialization 🚀\n============================================")
```

**Why surprising:** Unconventional style choice in a professional codebase. However, effective for development debugging.

---

### 4. No Formal Releases

No GitHub Releases exist. SECURITY.md states: "As deer-flow doesn't provide an official release yet, please use the latest version."

**Why surprising:** 47K stars with no versioned releases. Users must track `main` branch or use Docker image tags.

---

### 5. Per-Thread ACP Workspace Isolation

Each thread gets its own ACP workspace:

```
{base_dir}/threads/{thread_id}/acp-workspace/
```

**Why surprising:** Subagents share the same sandbox but get isolated ACP directories. Clever compromise between isolation and resource sharing.

---

### 6. Running Reply Pattern in Channels

IM channels send "Working on it..." before processing:

```python
# telegram.py
await self._send_running_reply(msg.chat_id, msg.message_id)
await self.bus.publish_inbound(inbound)
```

**Why surprising:** Users get immediate feedback even for slow operations. Reduces perceived latency.

---

### 7. Dual-Theme Code Highlighting

Code blocks pre-render both light and dark themes:

```typescript
// code-block.tsx
// Shiki outputs both themes
// CSS controls which is visible
```

**Why surprising:** Zero flash on theme switch. Memory cost is 2x HTML, but no JS re-render.

---

### 8. Temp File Pattern Without Cleanup

Many uses of temp files do not clean up on failure:

```python
# Expected: try/finally with Path(fd.name).unlink()
# Actual: relies on tempfile delete=True behavior
# But: not all code follows this pattern consistently
```

**Why surprising:** Some code is careful, some is not. Inconsistent error handling.

---

### 9. Feishu WebSocket Workaround

lark-oapi captures the event loop at import time:

```python
# feishu.py
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)
import lark_oapi.ws.client as _ws_client_mod
_ws_client_mod.loop = loop  # Patch SDK's module-level reference
```

**Why surprising:** SDK quirk requires patching module internals. Feishu must run in its own thread with a plain asyncio loop (not uvloop).

---

### 10. Path Traversal Defense Layers

Three layers of protection:

```python
# 1. PurePosixPath normalization
PurePosixPath(filename).name  # Strips .., /, etc.

# 2. Regex validation
_SAFE_THREAD_ID = re.compile(r"^[a-zA-Z0-9._-]+$")

# 3. is_relative_to() check
path.resolve().relative_to(base.resolve())
```

**Why surprising:** Belt-and-suspenders approach. Any single layer would likely be sufficient, but defense in depth is reassuring.

---

## Summary

| Category | Count |
|----------|-------|
| Emulation-worthy patterns | 10 |
| Avoid patterns | 10 |
| Surprises | 10 |

**Net assessment:** DeerFlow is a well-architected system with clear separation of concerns, comprehensive testing (backend), and thoughtful patterns (middleware chain, pub/sub, virtual paths). Main weaknesses are in production hardening (CORS, rate limiting, pagination) and Python type safety. The surprise items are mostly implementation quirks rather than architectural flaws.

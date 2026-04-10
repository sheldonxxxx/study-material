# My Action Items for Building a Similar Project

Prioritized concrete next steps derived from deer-flow analysis.

---

## Phase 1: Architecture Foundations

### 1.1 Implement Middleware Chain Pattern

**Why:** DeerFlow's 12-stage middleware chain is the backbone of agent processing. It enables modular, testable, configurable processing stages.

**How:**
1. Define `AgentMiddleware` protocol with `before_agent()`, `after_agent()`, `wrap_tool_call()`, `awrap_tool_call()` methods
2. Create `MiddlewareChain` class that composes middleware in order
3. Implement first 3 middleware: `ThreadDataMiddleware`, `UploadsMiddleware`, `SandboxMiddleware`
4. Add middleware to agent factory

**Verify:** Unit tests for each middleware in isolation.

---

### 1.2 Build Virtual Path System

**Why:** Essential for portability across execution environments (local, Docker, K8s).

**How:**
1. Define path mappings in config:
   ```yaml
   sandbox:
     path_mappings:
       /mnt/user-data/workspace: ./data/{thread_id}/workspace
       /mnt/user-data/uploads: ./data/{thread_id}/uploads
       /mnt/user-data/outputs: ./data/{thread_id}/outputs
   ```
2. Create `PathTranslator` class with bidirectional translation
3. Use `PurePosixPath` for normalization, `is_relative_to()` for traversal checks
4. Expose via single `resolve_virtual_path(thread_id, virtual_path)` function

**Verify:** Test path translation round-trips and traversal rejection.

---

### 1.3 Create Harness/App Split

**Why:** Enables publishing framework to PyPI. Application code imports framework, never vice versa.

**How:**
1. Create `packages/harness/` directory with `pyproject.toml`
2. Move agent logic, sandbox, tools, models, skills, memory, subagents to harness
3. Keep FastAPI app, channels, gateway in `app/`
4. Write `tests/test_harness_boundary.py` that imports harness and verifies no `app.` imports
5. Add to CI gate

**Verify:** CI fails if harness imports app.

---

### 1.4 Implement Pub/Sub MessageBus

**Why:** Decouples inbound message sources (channels) from dispatcher. Adding a new platform is a config change, not a code change.

**How:**
1. Define `InboundMessage`, `OutboundMessage`, `OutboundCallback` types
2. Implement `MessageBus` with `asyncio.Queue` for inbound, list of callbacks for outbound
3. Create abstract `Channel` base class with `start()`, `stop()`, `send()` methods
4. Implement `ChannelManager` that consumes from bus and dispatches to LangGraph

**Verify:** Unit tests with mock channels.

---

## Phase 2: Core Features

### 2.1 Skills System

**Why:** Extensible workflow definitions without code changes.

**How:**
1. Define `Skill` dataclass with `name`, `description`, `license`, `skill_dir`, `enabled`
2. Create `load_skills()` that recursively scans `skills/{public,custom}/` for `SKILL.md`
3. Parse frontmatter with regex (like DeerFlow) or proper YAML
4. Validate frontmatter: allowed properties, name format, description length
5. Implement skill installation from `.skill` ZIP archives with:
   - Zip bomb defense (512MB max)
   - Path traversal prevention
   - Symlink handling
6. Inject skills into agent prompt via `get_skills_prompt_section()`

**Verify:** Test skill loading, parsing, validation, and installation.

---

### 2.2 Sandbox Provider Pattern

**Why:** Pluggable execution environments (local, Docker, K8s) with same interface.

**How:**
1. Define abstract `Sandbox` interface: `execute_command()`, `read_file()`, `write_file()`, `list_dir()`
2. Define abstract `SandboxProvider`: `acquire()`, `get()`, `release()`
3. Implement `LocalSandboxProvider` (singleton, path translation)
4. Implement `DockerSandboxProvider` (container lifecycle)
5. Implement `KubernetesSandboxProvider` (pod provisioning with idle timeout and LRU eviction)
6. Add sandbox middleware that acquires/releases per thread

**Verify:** All three providers pass same integration tests.

---

### 2.3 Configuration-Driven Factory

**Why:** New models, tools, channels via config only — no code changes.

**How:**
1. Implement `resolve_class(use: str)` that dynamically imports class from `"module:ClassName"` string
2. Create `create_chat_model()` factory with config-driven instantiation
3. Create `get_available_tools()` that assembles from config-defined tools, MCP tools, built-in tools
4. Config schema validation on startup

**Verify:** Add new model/tool via config, verify it loads without code changes.

---

### 2.4 Memory System

**Why:** Persistent cross-session memory improves agent performance over time.

**How:**
1. Define memory data structure:
   ```python
   {
       "user": {
           "workContext": {"summary": "", "updatedAt": ""},
           "personalContext": {"summary": "", "updatedAt": ""},
           "topOfMind": {"summary": "", "updatedAt": ""},
       },
       "history": {
           "recentMonths": {"summary": "", "updatedAt": ""},
           "earlierContext": {"summary": "", "updatedAt": ""},
           "longTermBackground": {"summary": "", "updatedAt": ""},
       },
       "facts": [{"id", "content", "category", "confidence", "createdAt", "source"}],
   }
   ```
2. Implement `MemoryUpdateQueue` with debouncing and per-thread deduplication
3. Implement `MemoryUpdater` with LLM-based extraction and atomic file I/O
4. Add `MemoryMiddleware` to agent chain (position 9)
5. Implement token-aware injection with confidence-sorted facts

**Verify:** Memory persists across restarts, facts accumulate and are injected correctly.

---

### 2.5 Subagent Orchestration

**Why:** Complex tasks require decomposition and parallel execution.

**How:**
1. Define `SubagentConfig` with name, tools, disallowed_tools, model, max_turns, timeout
2. Implement `SubagentExecutor` with dual thread pools (scheduler + execution)
3. Create `task_tool` that delegates to subagent asynchronously
4. Backend polls for completion (not LLM polling), streams SSE events
5. Implement `SubagentLimitMiddleware` to enforce max concurrent (default 3)
6. Built-in subagent types: `general-purpose` (full tools), `bash` (sandbox only)

**Verify:** Subagent runs complete, results returned to lead agent, limits enforced.

---

## Phase 3: Integrations

### 3.1 Claude Code CLI OAuth

**Why:** Zero-config authentication for users with Claude Code installed.

**How:**
1. Implement `CredentialLoader` checking multiple sources:
   - `$CLAUDE_CODE_OAUTH_TOKEN` env var
   - `$CLAUDE_CODE_CREDENTIALS_PATH`
   - `~/.claude/.credentials.json`
2. Create `ClaudeChatModel` extending `ChatAnthropic` with OAuth Bearer auth
3. Handle token refresh (DeerFlow does not implement refresh — add it)
4. Auto-thinking budget: 80% of max_tokens

**Verify:** Works with existing Claude Code credentials without explicit API key.

---

### 3.2 Model Agnostic with CLI Providers

**Why:** Users should bring their own models via CLI backends.

**How:**
1. Implement `CodexChatModel` for Codex CLI (uses `chatgpt.com/backend-api/codex/responses`)
2. Support any OpenAI-compatible API via `langchain_openai:ChatOpenAI`
3. Implement `create_chat_model()` factory with config-driven provider selection
4. Handle thinking/reasoning effort mapping per provider

**Verify:** Switch between Claude OAuth, Codex CLI, and standard API with only config change.

---

### 3.3 IM Channel Integrations

**Why:** Receive tasks from Telegram, Slack, Feishu without web UI.

**How:**
1. Implement Telegram channel (long-polling via `python-telegram-bot`)
2. Implement Slack channel (Socket Mode WebSocket)
3. Implement Feishu channel (WebSocket with card updates)
4. Add "Working on it..." running reply pattern
5. Implement slash commands: `/new`, `/status`, `/help`, `/models`, `/memory`
6. Add per-channel capability flags (streaming vs blocking)

**Verify:** Create thread via Telegram message, interact fully, receive response via Telegram.

---

## Phase 4: Production Hardening

### 4.1 Add Python Type Checking

**Why:** Catch type errors at CI time, not runtime.

**How:**
1. Add `mypy` to dev dependencies
2. Configure `mypy.ini` with reasonable ignores (external libraries)
3. Fix type errors incrementally — start with harness module
4. Add to CI: `uvx mypy packages/harness/ app/`

**Priority:** After core features, before production launch.

---

### 4.2 Pagination

**Why:** Prevent timeouts on large thread histories.

**How:**
1. Add `cursor` and `limit` parameters to thread message retrieval
2. Add pagination to file listings in uploads
3. Frontend: implement virtual scrolling for long message lists
4. Backend: return `has_more` and `next_cursor` in list responses

**Priority:** When thread length exceeds 100 messages.

---

### 4.3 Auth Rate Limiting

**Why:** Prevent brute-force attacks on authentication.

**How:**
1. Add `slowapi` to dependencies
2. Configure per-IP rate limits on auth endpoints:
   ```python
   from slowapi import Limiter
   limiter = Limiter(key_func=get_remote_address)
   # 5 requests per minute on login
   ```
3. Add to CI (test rate limiting is enforced)

**Priority:** Before public deployment.

---

### 4.4 Restrict CORS

**Why:** Wildcard CORS is a security risk in production.

**How:**
1. Use environment variable for allowed origins:
   ```nginx
   set $allowed_origin "${CORS_ORIGIN}";
   if ($allowed_origin = "") {
       set $allowed_origin "*";
   }
   add_header 'Access-Control-Allow-Origin' "$allowed_origin" always;
   ```
2. Default to restrictive in production, permissive in development
3. Document required `CORS_ORIGIN` env var for production

**Priority:** Before public deployment.

---

### 4.5 Frontend Test Coverage

**Why:** Core logic (hooks, API client) has zero test coverage.

**How:**
1. Add Vitest + React Testing Library
2. Write tests for:
   - `useThreadStream` hook (mock LangGraph SDK)
   - `useThreads` hook with pagination
   - Artifact loading and caching
   - Message parsing and serialization
3. Add to CI: `pnpm test`

**Priority:** After Phase 2, when frontend stabilizes.

---

## Phase 5: Polish

### 5.1 Structured Logging

**Why:** `print()` statements in production code limit observability.

**How:**
1. Standardize on `logging.getLogger(__name__)` pattern
2. Configure log levels via environment variable
3. Add contextual fields (thread_id, user_id) where relevant
4. Consider `structlog` for JSON output in production

**Priority:** Low — works without it, but harder to debug.

---

### 5.2 Configurable Timeouts

**Why:** Hard-coded 15-minute subagent timeout and 10-minute command timeout are inflexible.

**How:**
1. Make timeouts configurable per invocation:
   ```yaml
   subagents:
     general-purpose:
       timeout_seconds: 900  # 15 min default
     bash:
       timeout_seconds: 300   # 5 min for bash
   ```
2. Add per-call timeout override in `task_tool`
3. Document the configuration

**Priority:** Medium — workaround is to not use the feature that needs different timeout.

---

### 5.3 Gateway Conformance Tests

**Why:** DeerFlow's `TestGatewayConformance` catches API drift between client and server.

**How:**
1. Define Pydantic models for all API request/response shapes
2. Write tests that parse client output through server schemas
3. Fail CI if validation raises `ValidationError`
4. Document the expected API contract

**Priority:** Medium — ensures client/server stay in sync.

---

## Summary Checklist

| Phase | Item | Effort | Impact |
|-------|------|--------|--------|
| 1.1 | Middleware chain | Medium | Core architecture |
| 1.2 | Virtual paths | Medium | Portability |
| 1.3 | Harness/app split | Medium | Publishability |
| 1.4 | MessageBus | Medium | Extensibility |
| 2.1 | Skills system | Medium | User extensibility |
| 2.2 | Sandbox providers | High | Security/isolation |
| 2.3 | Config factories | Medium | Configurability |
| 2.4 | Memory system | High | Long-term value |
| 2.5 | Subagents | High | Complex tasks |
| 3.1 | Claude OAuth | Medium | Zero-config auth |
| 3.2 | CLI providers | Medium | Flexibility |
| 3.3 | IM channels | High | Reach |
| 4.1 | mypy | Low | Code quality |
| 4.2 | Pagination | Medium | Scalability |
| 4.3 | Rate limiting | Low | Security |
| 4.4 | CORS | Low | Security |
| 4.5 | Frontend tests | Medium | Code quality |
| 5.1 | Structured logging | Low | Observability |
| 5.2 | Configurable timeouts | Low | Flexibility |
| 5.3 | Gateway conformance | Low | API stability |

**Start with:** Phase 1 (architecture foundations) — these enable everything else.
**Next:** Phase 2 core features — skills, sandbox, config, memory, subagents.
**Then:** Phase 3 integrations — Claude OAuth, CLI providers, IM channels.
**Finally:** Phase 4 production hardening and Phase 5 polish.

# Lessons Learned

## What to Emulate (Good Patterns)

### 1. Event-Driven Architecture with Message Bus

Decoupling channels from the agent via an async message bus is a clean pattern that enables:
- Multiple chat platforms operating concurrently
- Hot-swapping LLM providers without affecting other components
- Clean separation of concerns

**Evidence:** `nanobot/bus/queue.py` with `asyncio.Queue`

---

### 2. Registry/Plugin Architecture

Using centralized registries for channels, providers, tools, and skills enables runtime discovery without modifying core code.

**Evidence:** `providers/registry.py`, `channels/registry.py`, `agent/tools/registry.py`

---

### 3. Token-Based Memory Consolidation

Automatic memory management based on token budgets prevents context overflow while maintaining conversation history.

**Evidence:** `nanobot/agent/memory.py` with `maybe_consolidate_by_tokens()`

---

### 4. Defense in Depth Security

Multiple security layers: allowlist auth, path traversal protection, SSRF blocking, command pattern denial.

**Evidence:** `nanobot/security/network.py`, `nanobot/agent/tools/shell.py`

---

### 5. Concurrent Tool Execution

Using `asyncio.gather` with `return_exceptions=True` for parallel tool execution is clean and efficient.

**Evidence:** `nanobot/agent/loop.py` lines 84-87

---

### 6. Graceful Degradation

Falling back to degraded mode (raw archive) after repeated failures rather than failing completely.

**Evidence:** Memory consolidation with `_MAX_FAILURES_BEFORE_RAW_ARCHIVE = 3`

---

### 7. Exponential Backoff Retry

Retry delays of (1, 2, 4) seconds for transient failures is a proven pattern.

**Evidence:** `nanobot/providers/base.py` `_CHAT_RETRY_DELAYS`

---

### 8. Streaming with Think-Block Filtering

Incrementally stripping `<think>` blocks during streaming ensures downstream consumers never see internal reasoning.

**Evidence:** `nanobot/agent/loop.py` `_filtered_stream()`

---

### 9. Pydantic for Configuration

Using Pydantic Settings with type validation, environment variable support, and schema evolution is excellent.

**Evidence:** `nanobot/config/schema.py`

---

### 10. Markdown-Based Skills

Skills as markdown prompts with frontmatter is elegant - simple, readable, and extensible.

**Evidence:** `nanobot/skills/*/SKILL.md`

---

## What to Avoid (Bad Patterns/Gaps)

### 1. Global Mutable State for Config Path

```python
_current_config_path: Path | None = None  # Not thread-safe!
```

Multi-instance support uses global state that breaks if instances run in same process.

**Location:** `nanobot/config/loader.py` lines 11-25

---

### 2. In-Memory Session Locks

```python
self._session_locks.setdefault(msg.session_key, asyncio.Lock())
```

Session locks are in-memory only - won't work across multiple nanobot instances.

**Location:** `nanobot/agent/loop.py`

---

### 3. No Rate Limiting

SECURITY.md explicitly states: "No application-level rate limiting" - relies entirely on provider-side limits.

**Impact:** DoS possible by flooding messages.

---

### 4. No Delete in Memory System

Facts persist indefinitely in `MEMORY.md`. No mechanism to remove or redact.

**Impact:** Privacy issues if sensitive data stored.

---

### 5. Shell Execution Not Truly Sandboxed

`asyncio.create_subprocess_shell` with regex deny patterns is not a true sandbox.

**Impact:** If patterns bypassed, shell injection possible.

---

### 6. OAuth Providers Incomplete

OpenAI Codex and GitHub Copilot have `is_oauth=True` but OAuth flow implementation not visible.

**Impact:** These providers may not actually work.

---

### 7. WhatsApp Requires Separate Bridge

WhatsApp needs TypeScript/Baileys because WhatsApp Web requires a browser environment. Adds complexity.

**Impact:** Larger Docker image, separate build process, additional maintenance burden.

---

### 8. No Pre-Commit Hooks

No automated code formatting or linting before commits.

**Impact:** Inconsistent formatting, style issues enter codebase.

---

### 9. Large Files

`agent/loop.py` is 645 lines. Some files could be split for maintainability.

---

### 10. SSRF Gap in MCP HTTP Transport

streamableHttp transport does NOT inherit SSRF validation - potential security gap.

**Location:** `nanobot/agent/tools/mcp.py`

---

## Discrepancies Between Docs and Code

### 1. "Auto-discovers and registers tools on startup" (MCP)

**Documentation:** Claims MCP servers are auto-discovered.

**Reality:** MCP servers are configured in config file, not auto-discovered.

---

### 2. "Just send one message and your nanobot joins automatically" (Social Network)

**Documentation:** Implies seamless automatic joining.

**Reality:** Agent must successfully fetch URL, parse instructions, execute installation commands, and start new session. Many failure points.

---

### 3. "Docker deployment via docker-compose or standalone Dockerfile, plus systemd" (Docker)

**Documentation:** Lists systemd as a deployment option.

**Reality:** Systemd service files are NOT in repository. Users must manually create from README instructions.

---

### 4. "Skills are extensible plugins"

**Documentation:** Implies code-based plugin system.

**Reality:** Skills are markdown files loaded into context, not compiled code plugins.

---

### 5. "Built-in Agent Tools: message sending"

**Feature Index:** Lists "message sending" as a built-in tool.

**Reality:** Message functionality is through bus system, not a standalone tool.

---

## Key Insights

1. **Simplicity wins**: Markdown-based skills are more maintainable than code plugins
2. **Defense in depth**: Multiple security layers compensate for imperfect individual guards
3. **Graceful degradation**: Failing degraded is better than failing hard
4. **Async all the way**: Consistent use of asyncio throughout enables concurrency
5. **Registry patterns**: Runtime discovery enables clean extensibility without modification

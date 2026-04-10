# Features Deep Dive

## Priority 1: Core Features

### 1. Agent Core Loop

**Purpose:** Central LLM-driven processing engine that orchestrates tool execution, memory management, and response generation.

**Key files:**
- `nanobot/agent/loop.py` - AgentLoop (645 lines)
- `nanobot/agent/context.py` - ContextBuilder (280 lines)
- `nanobot/agent/memory.py` - MemoryConsolidator (450 lines)

**Processing flow:**
```
consume_inbound() → _dispatch()
  ├── Priority commands (/stop, /restart) → bypass lock
  └── Per-session lock → _process_message()
        ├── Slash command dispatch
        ├── memory_consolidation
        ├── context.build_messages()
        └── _run_agent_loop()
              ├── provider.chat_stream_with_retry() or chat_with_retry()
              ├── If tool_calls → execute all concurrently via asyncio.gather()
              └── If no tool_calls → return final content
```

**Key characteristics:**
- Concurrent tool execution with `asyncio.gather`
- Tool context isolation via `_set_tool_context()`
- Max 25 iterations default (prevents infinite loops)
- Streaming think-block filtering

---

### 2. Multi-Channel Integration

**Purpose:** Connect to 12 chat platforms simultaneously via plugin architecture.

**Supported channels:** Telegram, Discord, Slack, DingTalk, Feishu, WeCom, WeChat, QQ, Matrix, Email, MoChat, WhatsApp

**Plugin discovery pattern:**
```python
# Zero-import scanning via pkgutil
for _, name, ispkg in pkgutil.iter_modules(pkg.__path__):
    if name not in _INTERNAL and not ispkg:
        channel_names.append(name)
```

**Permission model:**
```python
def is_allowed(self, sender_id: str) -> bool:
    allow_list = getattr(self.config, "allow_from", [])
    if not allow_list:
        return False  # Empty = deny all (v0.1.4.post4+)
    return "*" in allow_list or str(sender_id) in allow_list
```

**Outbound retry:**
```python
_SEND_RETRY_DELAYS = (1, 2, 4)  # Exponential backoff
```

---

### 3. Multi-Provider LLM Support

**Purpose:** Unified interface to 20+ LLM providers.

**Registry pattern (ProviderSpec):**
```python
@dataclass(frozen=True)
class ProviderSpec:
    name: str
    keywords: tuple[str, ...]
    env_key: str
    backend: str  # "openai_compat" | "anthropic" | "azure_openai"
    is_gateway: bool
    detect_by_key_prefix: str
    default_api_base: str
    strip_model_prefix: bool
    supports_prompt_caching: bool
```

**Notable implementations:**
- **OpenAICompatProvider**: Handles OpenAI + all OpenAI-compatible APIs
- **AnthropicProvider**: Native SDK with message format conversion
- **Tool call ID normalization**: 9-char alphanumeric compatible with all providers

---

### 4. Built-in Agent Tools

**Tool registry:** `nanobot/agent/tools/registry.py`

| Tool | File | Purpose |
|------|------|---------|
| ExecTool | `shell.py` | Shell command execution |
| ReadFileTool | `filesystem.py` | File reading with pagination |
| WriteFileTool | `filesystem.py` | File writing |
| EditFileTool | `filesystem.py` | File editing with fuzzy match |
| ListDirTool | `filesystem.py` | Directory listing |
| WebSearchTool | `web.py` | Multi-provider search |
| WebFetchTool | `web.py` | URL content extraction |
| CronTool | `cron.py` | Cron scheduling |
| SpawnTool | `spawn.py` | Subagent spawning |
| MCPToolWrapper | `mcp.py` | MCP protocol tools |

**Shell tool security:**
```python
deny_patterns = [
    r"\brm\s+-[rf]{1,2}\b",    # rm -r, rm -rf
    r"\bdd\s+if=",              # dd operations
    r"\b(shutdown|reboot)\b",   # system power
    r":\(\)\s*\{.*\};\s*:",     # fork bomb
    # ... more patterns
]
```

---

### 5. Persistent Memory System

**Architecture:** Two-layer system

| Layer | File | Purpose |
|-------|------|---------|
| `MEMORY.md` | `memory/` | LLM-summarized long-term facts |
| `HISTORY.md` | `memory/` | Append-only searchable log |

**Token-based consolidation:**
```python
budget = context_window_tokens - max_completion_tokens - 1024  # Safety buffer
target = budget // 2  # 50% of budget to prevent thrashing
```

**Consolidation boundary:** Only `user` message boundaries are considered natural conversation turns.

**Error handling:** Falls back to raw-archive after 3 consecutive LLM failures.

**Gap:** No delete/redact capability. Facts persist indefinitely.

---

### 6. Cron and Heartbeat

**CronService:** Timer-based scheduler for recurring jobs.

```python
# Schedule types
kind: Literal["at", "every", "cron"]  # One-time, interval, cron expr
every_ms: int | None     # For "every": interval ms
expr: str | None         # For "cron": cron expression
tz: str | None           # Timezone for cron expressions
```

**HeartbeatService:** Periodic wake-up with LLM-driven decision.

```
Phase 1 (decision): Read HEARTBEAT.md, ask LLM via tool call
Phase 2 (execution): Only if Phase 1 returns "run"
```

**Notification gate:** Both use `evaluate_response()` to decide if user should be notified, preventing spam.

---

## Priority 2: Secondary Features

### 7. MCP (Model Context Protocol) Support

**Purpose:** Connect external MCP tool servers as native agent tools.

**Transport types:** stdio, SSE, streamable HTTP

**Schema normalization:** Converts MCP JSON Schema nullable unions to OpenAI format.

**Tool naming:** `mcp_{server_name}_{original_tool_name}`

---

### 8. Web Search

**Provider chain (with fallback):**
1. Brave (API key required)
2. Tavily (API key required)
3. SearXNG (self-hosted)
4. Jina (API key required)
5. DuckDuckGo (no key, fallback)

```python
async def _search_brave(self, query: str, n: int) -> str:
    if not api_key:
        logger.warning("BRAVE_API_KEY not set, falling back to DuckDuckGo")
        return await self._search_duckduckgo(query, n)
```

---

### 9. Skills System

**Architecture:** Markdown-based prompts, not code plugins.

```yaml
---
name: github
description: "Interact with GitHub using the `gh` CLI..."
metadata: {"nanobot":{
  "emoji": "🐙",
  "requires": {"bins": ["gh"]},
  "install": [{"id": "brew", "kind": "brew", "formula": "gh"}]
}}
---
```

**Skill sources (priority order):**
1. Workspace skills (`~/.nanobot/workspace/skills/`) - User-installed
2. Built-in skills (`nanobot/skills/`) - Bundled

**Bundled skills:** GitHub, Weather, Tmux, Cron, Summarize, Memory, ClawHub, Skill-Creator

**ClawHub marketplace:** Installed via `npx clawhub@latest install <slug>`

---

### 10. Agent Social Network

**Implementation:** Not a native protocol. Works via:
1. Agent reads `skill.md` from URL via `web_fetch`
2. Follows instructions to install skill via ClawHub
3. New skill teaches agent how to join network

**Example:** `Read https://moltbook.com/skill.md and follow the instructions to join Moltbook`

---

### 11. Multi-Instance Support

**Mechanism:** Separate config file + workspace per instance.

```bash
nanobot gateway --config /path/to/instance-a/config.json --workspace /path/to/instance-a/workspace
```

**Isolation:**
| Component | Method |
|-----------|--------|
| Config file | `--config` flag via `set_config_path()` |
| Workspace | `--workspace` flag |
| Sessions | Stored in workspace |
| Memory | Stored in workspace |
| Port | `--port` flag (default 18790) |

**Gap:** Global `_current_config_path` is not thread-safe.

---

### 12. Docker/Service

**Dockerfile:** Uses `uv` for fast package management + Node.js 20 for WhatsApp bridge.

**docker-compose:** Gateway + optional CLI profile, resource limits (1 CPU, 1GB RAM).

**Systemd:** Documented but NOT in repository. Users must manually create service files.

---

## Feature Priority Summary

| Priority | Features |
|----------|----------|
| **CORE** | Agent Core Loop, Multi-Channel (12), Multi-Provider (20+), Built-in Tools, Memory System, Cron/Heartbeat, Skills |
| **SECONDARY** | MCP Support, Web Search, Agent Social Network, Multi-Instance, Docker/Service |

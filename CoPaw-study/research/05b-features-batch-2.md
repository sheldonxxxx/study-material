# Feature Deep Dive: Batch 2

## Skills System

### Implementation Overview

The Skills System is an extensible plugin architecture built around a directory-based convention with YAML frontmatter.

**Key Files:**
- `src/copaw/agents/skills_manager.py` (1233 lines) - Core skill management
- `src/copaw/agents/skills_hub.py` (1619 lines) - Hub client for importing skills
- `src/copaw/app/routers/skills.py` - API routes for skill management

**Architecture:**

```
builtin_skills/     (codeship deliver)
        |
        v
customized_skills/  (user workspace)
        |
        v
active_skills/      (what agent actually uses)
```

**Three-tier Directory System:**

```python
# skills_manager.py lines 63-75
def get_builtin_skills_dir() -> Path:
    return Path(__file__).parent / "skills"

def get_customized_skills_dir(workspace_dir: Path) -> Path:
    return workspace_dir / "customized_skills"

def get_active_skills_dir(workspace_dir: Path) -> Path:
    return workspace_dir / "active_skills"
```

**Skill Structure Convention:**

Each skill directory contains:
- `SKILL.md` - YAML frontmatter with `name`, `description`, and metadata
- `references/` - Documentation files accessible to the agent
- `scripts/` - Executable scripts the agent can run

**SkillInfo Model (lines 28-61):**
```python
class SkillInfo(BaseModel):
    name: str
    description: str = ""
    content: str
    source: str  # "builtin", "customized", or "active"
    path: str
    references: dict[str, Any] = {}
    scripts: dict[str, Any] = {}
```

**Notable Code Patterns:**

1. **Security Scanning on Enable:**
```python
# skills_manager.py lines 935-958
def enable_skill(self, name: str, force: bool = False) -> bool:
    # Security scan runs BEFORE activation
    from ..security.skill_scanner import scan_skill_directory
    scan_skill_directory(source_dir, skill_name=name)
    # ...
```

2. **Version-based Sync Logic:**
```python
# skills_manager.py lines 325-345
# Builtin skill: check version upgrade, skip back-sync
if active_ver is not None and builtin_ver is not None and builtin_ver > active_ver:
    _replace_skill_dir(builtin_dir, skill_dir)
```

3. **Zip Import with Security Validation:**
```python
# skills_manager.py lines 556-576
def _extract_and_validate_zip(data: bytes, tmp_dir: Path) -> None:
    # Uncompressed size limit check
    total = sum(i.file_size for i in zf.infolist())
    if total > _MAX_ZIP_BYTES:
        raise ValueError(f"Uncompressed size exceeds {mb}MB limit")

    # Path traversal protection
    for info in zf.infolist():
        target = (tmp_dir / info.filename).resolve()
        if not target.is_relative_to(root_path):
            raise ValueError(f"Unsafe path: {info.filename}")
```

**Skills Hub Multi-Source Support (skills_hub.py):**

The hub client can import from multiple sources:
- ClawHub (primary): `https://clawhub.ai`
- GitHub: Direct repo or tree URLs
- LobeHub: `https://market.lobehub.com`
- ModelScope: `https://modelscope.cn`
- Skills.sh: `https://skills.sh`
- SkillsMP: `https://skillsmp.com`

**URL Parsing Patterns:**
```python
# skills_hub.py lines 723-734
def _extract_skills_sh_spec(url: str) -> tuple[str, str, str] | None:
    # Parse: https://skills.sh/owner/repo/skill

def _extract_github_spec(url: str) -> tuple[str, str, str, str] | None:
    # Parse: https://github.com/owner/repo/tree/branch/path
```

**Environment Variables for Hub:**
```python
COPAW_SKILLS_HUB_HTTP_TIMEOUT=15
COPAW_SKILLS_HUB_HTTP_RETRIES=3
COPAW_SKILLS_HUB_BACKOFF_BASE=0.8
COPAW_SKILLS_HUB_BACKOFF_CAP=6
```

**Technical Debt/Concerns:**

1. **Blocking Security Scan on Create:** The `create_skill()` method runs security scanning post-write (lines 857-872), but catches and warns on scan failures rather than failing the operation.

2. **Sync Race Condition Potential:** The sync between builtin/customized/active has subtle timing dependencies where customized can override builtin but the comparison only checks SKILL.md content.

3. **Hub Retry Logic:** Exponential backoff with hardcoded base (0.8s) and cap (6s) may be too aggressive for some network conditions.

---

## Multi-Agent Support

### Implementation Overview

Multi-agent support is implemented through workspace isolation plus a CLI-based collaboration skill. Each agent runs in its own workspace directory.

**Key Files:**
- `src/copaw/app/multi_agent_manager.py` (462 lines) - Workspace lifecycle management
- `src/copaw/agents/react_agent.py` (930 lines) - ReAct-based agent implementation
- `src/copaw/agents/skills/multi_agent_collaboration/SKILL.md` - Collaboration skill

**Architecture:**

```
MultiAgentManager
    |
    +-- Workspace (agent_id: str, workspace_dir: Path)
    |       |
    |       +-- CoPawAgent (ReActAgent + ToolGuardMixin)
    |       +-- ServiceManager (shared reusable components)
    |       +-- TaskTracker (active task monitoring)
    |
    +-- Workspace
    +-- ...
```

**Lazy Loading with Lock Protection (multi_agent_manager.py lines 34-81):**
```python
async def get_agent(self, agent_id: str) -> Workspace:
    async with self._lock:
        if agent_id in self.agents:
            return self.agents[agent_id]  # Return cached

        # Load from config and create new workspace
        config = load_config()
        agent_ref = config.agents.profiles[agent_id]
        instance = Workspace(agent_id=agent_id, workspace_dir=agent_ref.workspace_dir)
        await instance.start()
        self.agents[agent_id] = instance
        return instance
```

**Zero-Downtime Reload (lines 200-311):**

The reload mechanism ensures continuous availability:
```python
async def reload_agent(self, agent_id: str) -> bool:
    # 1. Create new workspace instance (outside lock)
    new_instance = Workspace(agent_id=agent_id, workspace_dir=agent_ref.workspace_dir)
    await new_instance.start()

    # 2. Transfer reusable services from old instance
    reusable = old_instance._service_manager.get_reusable_services()
    await new_instance.set_reusable_components(reusable)

    # 3. Atomic swap (minimal lock time)
    async with self._lock:
        old_instance = self.agents[agent_id]
        self.agents[agent_id] = new_instance

    # 4. Graceful stop old instance
    await self._graceful_stop_old_instance(old_instance, agent_id)
```

**Task-Aware Cleanup (lines 83-178):**
```python
async def _graceful_stop_old_instance(self, old_instance: Workspace, agent_id: str):
    has_active = await old_instance.task_tracker.has_active_tasks()

    if has_active:
        # Schedule delayed cleanup (wait up to 1 minute)
        async def delayed_cleanup():
            completed = await old_instance.task_tracker.wait_all_done(timeout=60.0)
            await old_instance.stop(final=False)

        cleanup_task = asyncio.create_task(delayed_cleanup())
        self._cleanup_tasks.add(cleanup_task)
    else:
        # Stop immediately
        await old_instance.stop(final=False)
```

**CoPawAgent Initialization (react_agent.py lines 87-178):**
```python
def __init__(self, agent_config: "AgentProfileConfig", ...):
    # 1. Create toolkit with built-in tools
    toolkit = self._create_toolkit(namesake_strategy=namesake_strategy)

    # 2. Register skills from workspace
    self._register_skills(toolkit)

    # 3. Build system prompt
    sys_prompt = self._build_sys_prompt()

    # 4. Create model via factory
    model, formatter = create_model_and_formatter(agent_id=agent_config.id)

    # 5. Initialize parent ReActAgent
    super().__init__(name="Friday", model=model, sys_prompt=sys_prompt,
                     toolkit=toolkit, memory=InMemoryMemory(), ...)

    # 6. Setup memory manager
    self._setup_memory_manager(...)

    # 7. Register hooks
    self._register_hooks()
```

**Multi-Agent Collaboration Skill (SKILL.md):**

Not implemented in code but as documentation instructing the agent to use CLI commands:

```bash
# List agents
copaw agents list

# Start conversation
copaw agents chat \
  --from-agent <your_agent> \
  --to-agent <target_agent> \
  --text "[Agent requesting] ..."

# Continue with session
copaw agents chat \
  --from-agent <your_agent> \
  --to-agent <target_agent> \
  --session-id "<session_id>" \
  --text "[Agent requesting] ..."
```

**Notable Code Patterns:**

1. **Reusable Components Transfer:**
```python
# Preserve expensive-to-create services across reloads
reusable = old_instance._service_manager.get_reusable_services()
await new_instance.set_reusable_components(reusable)
```

2. **Hook-based Extension:**
```python
# react_agent.py lines 352-380
def _register_hooks(self) -> None:
    # Bootstrap hook for first-time setup guidance
    bootstrap_hook = BootstrapHook(working_dir=working_dir, language=self._language)
    self.register_instance_hook(hook_type="pre_reasoning", hook_name="bootstrap_hook", ...)

    # Memory compaction hook for context management
    memory_compact_hook = MemoryCompactionHook(memory_manager=self.memory_manager)
    self.register_instance_hook(hook_type="pre_reasoning", hook_name="memory_compact_hook", ...)
```

3. **MCP Client Recovery (react_agent.py lines 463-578):**
```python
async def _recover_mcp_client(self, client: Any) -> Any | None:
    if await self._reconnect_mcp_client(client):
        return client

    # Rebuild from stored metadata
    rebuilt_client = self._rebuild_mcp_client(client)
    if await self._reconnect_mcp_client(rebuilt_client):
        return self._reuse_shared_client_reference(original_client, rebuilt_client)
    return None
```

**Technical Debt/Concerns:**

1. **Collaboration is Skill-Based, Not API-Based:** The multi-agent collaboration relies entirely on a SKILL.md with instructions rather than a programmatic API. This means agents must use shell commands to communicate.

2. **Session ID Management:** The skill documents that session IDs must be passed to continue conversations, but there's no API to list or manage these sessions from within an agent.

3. **No Built-in Inter-Agent Communication API:** If two agents need to collaborate programmatically, they must use the CLI-based `copaw agents chat` which parses text output.

4. **Agent Name Hardcoded:** `super().__init__(name="Friday", ...)` - The agent name is hardcoded rather than configured per-agent.

---

## Memory System

### Implementation Overview

The Memory System provides long-term storage and context retrieval using the ReMeLight library for memory management with file-based backup.

**Key Files:**
- `src/copaw/agents/memory/memory_manager.py` (360 lines) - ReMeLight wrapper
- `src/copaw/agents/memory/agent_md_manager.py` (124 lines) - File-based markdown storage
- `src/copaw/agents/hooks/memory_compaction.py` (211 lines) - Automatic compaction hook
- `src/copaw/agents/tools/memory_search.py` (69 lines) - Semantic search tool

**Architecture:**

```
MemoryManager (extends ReMeLight)
    |
    +-- Vector Search (embedding-based)
    |       +-- Backend: Chroma (Linux/Mac) or Local (Windows)
    |       +-- Configurable embedding model
    |
    +-- Full-Text Search
    |       +-- FTS_ENABLED=true/false
    |
    +-- File-based Backup
            +-- working_dir/memory/*.md
            +-- working_dir/*.md
```

**MemoryManager Initialization (memory_manager.py lines 57-134):**
```python
def __init__(self, working_dir: str, agent_id: str):
    # Get embedding config from agent config or environment
    emb_config = self.get_embedding_config()

    # Vector search requires base_url and model_name
    vector_enabled = bool(emb_config["base_url"]) and bool(emb_config["model_name"])

    # FTS toggle via environment
    fts_enabled = os.environ.get("FTS_ENABLED", "true").lower() == "true"

    # Auto-select backend by platform
    memory_backend = "local" if platform.system() == "Windows" else "chroma"

    super().__init__(
        working_dir=working_dir,
        default_embedding_model_config=emb_config,
        default_file_store_config={
            "backend": memory_backend,
            "store_name": "copaw",
            "vector_enabled": vector_enabled,
            "fts_enabled": fts_enabled,
        },
    )
```

**Embedding Configuration Priority (lines 153-171):**
```python
def get_embedding_config(self) -> dict:
    cfg = load_agent_config(self.agent_id).running.embedding_config

    return {
        "backend": cfg.backend,
        "api_key": cfg.api_key or os.getenv("EMBEDDING_API_KEY", ""),
        "base_url": cfg.base_url or os.getenv("EMBEDDING_BASE_URL", ""),
        "model_name": cfg.model_name or os.getenv("EMBEDDING_MODEL_NAME", ""),
        # ... other params from config only
    }
```

**Memory Search Tool (memory_search.py lines 17-67):**
```python
async def memory_search(
    query: str,
    max_results: int = 5,
    min_score: float = 0.1,
) -> ToolResponse:
    """
    Search MEMORY.md and memory/*.md files semantically.
    Returns top relevant snippets with file paths and line numbers.
    """
    return await memory_manager.memory_search(
        query=query,
        max_results=max_results,
        min_score=min_score,
    )
```

**Memory Compaction Hook (memory_compaction.py lines 62-210):**

The hook runs before each reasoning cycle:
```python
async def __call__(self, agent: ReActAgent, kwargs: dict[str, Any]) -> None:
    # 1. Calculate available token budget
    token_counter = get_copaw_token_counter(agent_config)
    left_threshold = running_config.memory_compact_threshold - str_token_count

    # 2. Compact old tool results
    await self.memory_manager.compact_tool_result(messages, ...)

    # 3. Check if compaction needed
    messages_to_compact, _, is_valid = await self.memory_manager.check_context(...)

    if messages_to_compact:
        # 4. Async summary task
        self.memory_manager.add_async_summary_task(messages_to_compact)

        # 5. Compact memory
        compact_content = await self.memory_manager.compact_memory(
            messages_to_compact,
            previous_summary=memory.get_compressed_summary(),
        )

        # 6. Mark messages as compacted
        updated_count = await memory.mark_messages_compressed(messages_to_compact)
```

**AgentMdManager - Simple File Storage (agent_md_manager.py):**

```python
class AgentMdManager:
    def __init__(self, working_dir: str | Path):
        self.working_dir = Path(working_dir)
        self.memory_dir = self.working_dir / "memory"

    # working_dir/*.md files
    def list_working_mds(self) -> list[dict]
    def read_working_md(self, md_name: str) -> str
    def write_working_md(self, md_name: str, content: str)

    # working_dir/memory/*.md files
    def list_memory_mds(self) -> list[dict]
    def read_memory_md(self, md_name: str) -> str
    def write_memory_md(self, md_name: str, content: str)
```

**Memory Invalid Result Handling (memory_manager.py lines 239-274):**
```python
# When compaction produces invalid result, save to file for debugging
if isinstance(result, str):
    logger.error(f"compact_memory returned str instead of dict...")
    return result  # or ""

is_valid: bool = result.get("is_valid", True)
if not is_valid:
    unique_id = uuid.uuid4().hex[:8]
    filename = f"compact_invalid_{unique_id}.json"
    filepath = os.path.join(workspace_dir, filename)

    with open(filepath, "w", encoding="utf-8") as f:
        json.dump(result, f, ensure_ascii=False, indent=2)

    logger.error(f"Invalid compact result saved to {filepath}")
```

**Notable Code Patterns:**

1. **ReMeLight Extension:** MemoryManager extends ReMeLight (from `reme` package) and overrides `compact_memory` and `summary_memory` with CoPaw-specific implementations using file tools.

2. **Toolkit for Summary Operations:**
```python
# memory_manager.py lines 136-139
self.summary_toolkit = Toolkit()
self.summary_toolkit.register_tool_function(read_file)
self.summary_toolkit.register_tool_function(write_file)
self.summary_toolkit.register_tool_function(edit_file)
```

3. **Model/Formatter Lazy Initialization:**
```python
def prepare_model_formatter(self) -> None:
    if self.chat_model is None or self.formatter is None:
        chat_model, formatter = create_model_and_formatter(self.agent_id)
        # ... assign if still None
```

**Technical Debt/Concerns:**

1. **ReMeLight Dependency:** MemoryManager heavily depends on the external `reme` package. If `reme` is not installed, only limited functionality is available:
```python
# memory_manager.py lines 30-45
try:
    from reme.reme_light import ReMeLight
    _REME_AVAILABLE = True
except ImportError as e:
    _REME_AVAILABLE = False
    logger.warning(f"reme package not installed. {e}")

    class ReMeLight:  # Placeholder
        async def start(self) -> None: pass
```

2. **Invalid Compact Results:** There's a code path that returns raw string instead of dict from `compact_memory`, suggesting potential version mismatch issues with ReMe.

3. **No Transaction for File+Vector Storage:** When writing to markdown files, there's no guarantee the vector index stays in sync.

4. **Embedding Config Not Hot-Reloadable:** While agent config is hot-reloaded, the embedding model (once initialized) is not restarted when config changes.

---

## Summary Table

| Aspect | Skills System | Multi-Agent | Memory System |
|--------|---------------|-------------|---------------|
| **Core File** | skills_manager.py | multi_agent_manager.py | memory_manager.py |
| **Size** | 1233 lines | 462 lines | 360 lines |
| **Storage** | Directory-based (filesystem) | Workspace directories | ReMeLight + file-based |
| **External Deps** | None | None | `reme` package |
| **Extension** | SKILL.md + scripts | SKILL.md (CLI-based) | ReMeLight plugins |
| **Security** | skill_scanner integration | Workspace isolation | Tool guard for file access |
| **Search** | N/A | N/A | Vector + Full-text search |

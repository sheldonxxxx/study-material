# CrewAI Features Batch 3: Agent Memory System, Built-in Tools & MCP Integration, and CLI & Project Scaffolding

## Executive Summary

This document covers three core CrewAI features:
1. **Agent Memory System** - Unified memory with intelligent recall and pluggable storage
2. **Built-in Tools & MCP Integration** - 75+ pre-built tools and Model Context Protocol support
3. **CLI & Project Scaffolding** - Command-line interface and project generation

---

## 1. Agent Memory System

### Overview

The Agent Memory System (`lib/crewai/src/crewai/memory/`) provides persistent memory capabilities for agents with intelligent recall, LLM-powered analysis, and hierarchical scoping.

### Key Components

#### Memory Record Structure (`memory/types.py`)

```python
class MemoryRecord(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid4()))
    content: str  # The textual content
    scope: str = Field(default="/")  # Hierarchical path (e.g., /company/team/user)
    categories: list[str]  # Tags for the memory
    metadata: dict[str, Any]  # Arbitrary metadata
    importance: float = Field(default=0.5, ge=0.0, le=1.0)  # Affects retrieval ranking
    created_at: datetime
    last_accessed: datetime
    embedding: list[float] | None  # Vector embedding for semantic search
    source: str | None  # Provenance tracking (user ID, session ID)
    private: bool = Field(default=False)  # Privacy flag
```

#### Composite Scoring (`memory/types.py`)

The recall composite score combines three factors:

```python
# composite = w_semantic * similarity + w_recency * decay + w_importance * importance
# where decay = 0.5^(age_days / half_life_days)

def compute_composite_score(
    record: MemoryRecord,
    semantic_score: float,
    config: MemoryConfig,
) -> tuple[float, list[str]]:
```

**Weights (configurable):**
- `semantic_weight`: 0.5 (vector similarity)
- `recency_weight`: 0.3 (time-based decay)
- `importance_weight`: 0.2 (explicit importance score)
- `recency_half_life_days`: 30 (days for recency score to halve)

### Memory Architecture

#### 1. UnifiedMemory Class (`memory/unified_memory.py`)

The main `Memory` class provides:
- **LLM-powered analysis**: Infers scope, categories, importance on save
- **Adaptive-depth recall**: `RecallFlow` for intelligent retrieval
- **Background writes**: Non-blocking saves via `ThreadPoolExecutor`
- **Scoped views**: `MemoryScope` and `MemorySlice` for hierarchical access

```python
class Memory(BaseModel):
    llm: Annotated[BaseLLM | str] = Field(default="gpt-4o-mini")
    storage: Annotated[StorageBackend | str] = Field(default="lancedb")
    embedder: Any = Field(default=None)  # OpenAI text-embedding-3-small default

    # Scoring weights
    recency_weight: float = 0.3
    semantic_weight: float = 0.5
    importance_weight: float = 0.2
    recency_half_life_days: int = 30

    # Consolidation settings
    consolidation_threshold: float = 0.85  # Trigger consolidation above this
    consolidation_limit: int = 5  # Max records to compare during consolidation

    # Recall depth control
    confidence_threshold_high: float = 0.8  # Return directly
    confidence_threshold_low: float = 0.5  # Trigger deeper exploration
    exploration_budget: int = 1  # LLM-driven exploration rounds

    root_scope: str | None = None  # Hierarchical prefix
    read_only: bool = False
```

#### 2. Encoding Flow (`memory/encoding_flow.py`)

Batch-native encoding pipeline with 5 steps:

```
Step 1: Batch embed (ONE embedder call for all items)
    ↓
Step 2: Intra-batch dedup (cosine matrix, drop near-exact duplicates)
    ↓
Step 3: Parallel find similar (concurrent storage searches)
    ↓
Step 4: Parallel analyze (N concurrent LLM calls)
    ↓
Step 5: Execute plans (batch re-embed updates + bulk insert)
```

**Key insight**: Single embedder call for entire batch, then N concurrent LLM calls for analysis.

#### 3. Recall Flow (`memory/recall_flow.py`)

Adaptive-depth retrieval inspired by RLM (Retrieval as Language Model):

```python
@start()
def analyze_query_step(self) -> QueryAnalysis:
    # Short queries (< query_analysis_threshold chars) skip LLM
    # Long queries use LLM to distill into targeted sub-queries
    # Batch-embed all sub-queries in ONE call

@router(search_chunks)
def decide_depth(self) -> str:
    # confidence >= threshold_high -> "synthesize"
    # confidence < threshold_low and budget > 0 -> "explore_deeper"
    # complex query + low confidence + budget > 0 -> "explore_deeper"
```

**Clever optimization**: Short queries (below 200 characters) skip LLM analysis and embed directly, saving ~1-3 seconds per recall.

#### 4. Storage Backends (`memory/storage/`)

**LanceDB Storage** (`storage/lancedb_storage.py`):
- Default storage backend
- Path: `$CREWAI_STORAGE_DIR/memory` or `db_storage_path() / memory`
- Auto-compaction every 100 saves
- Retry logic for commit conflicts (5 retries with exponential backoff)
- Auto-detection of vector dimension from first embedding

```python
class LanceDBStorage:
    DEFAULT_VECTOR_DIM = 1536  # OpenAI text-embedding-3-small
    _MAX_RETRIES = 5
    _RETRY_BASE_DELAY = 0.2  # doubles on each retry

    def _do_write(self, op: str, *args, **kwargs):
        # Retry on "Commit conflict" OSError
        # Exponential backoff: 0.2 + 0.4 + 0.8 + 1.6 + 3.2 = 6.2s max
```

**Qdrant Edge Storage** (`storage/qdrant_edge_storage.py`): Alternative vector storage backend.

### Agent Memory Integration

Agent executors automatically save task results to memory (`agents/agent_builder/base_agent_executor_mixin.py`):

```python
def _save_to_memory(self, output: AgentFinish) -> None:
    memory = getattr(self.agent, "memory", None) or (
        getattr(self.crew, "_memory", None) if self.crew else None
    )
    if memory is None or not self.task or memory.read_only:
        return

    raw = (
        f"Task: {self.task.description}\n"
        f"Agent: {self.agent.role}\n"
        f"Expected result: {self.task.expected_output}\n"
        f"Result: {output.text}"
    )
    extracted = memory.extract_memories(raw)  # LLM extracts discrete memories
    if extracted:
        # Hierarchical scoping under crew/agent
        base_root = getattr(memory, "root_scope", None)
        if base_root:
            agent_root = f"{base_root}/agent/{sanitize_scope_name(self.agent.role)}"
            memory.remember_many(extracted, agent_role=self.agent.role, root_scope=agent_root)
        else:
            memory.remember_many(extracted, agent_role=self.agent.role)
```

**Smart delegation handling**: Skips saving memories for "Delegate work to coworker" actions to avoid noise.

### Memory Scoping (`memory/memory_scope.py`)

Hierarchical memory views:

```python
class MemoryScope(BaseModel):
    """View of Memory restricted to a root path."""
    root_path: str = Field(default="/")
    _memory: Memory = PrivateAttr()
    _root: str = PrivateAttr()

# Usage:
crew_memory = Memory(llm="gpt-4o-mini")
researcher_memory = crew_memory.scope("/crew/my-crew/agent/researcher")
researcher_memory.remember("Found key insight about X", importance=0.9)
```

### Event System Integration

Memory operations emit events (`events/types/memory_events.py`):
- `MemorySaveStartedEvent`, `MemorySaveCompletedEvent`, `MemorySaveFailedEvent`
- `MemoryQueryStartedEvent`, `MemoryQueryCompletedEvent`, `MemoryQueryFailedEvent`

---

## 2. Built-in Tools & MCP Integration

### Tool Structure

#### BaseTool Class (`lib/crewai-tools/src/crewai_tools/base_tool.py`)

All tools inherit from `BaseTool` (Pydantic + ABC):

```python
class BaseTool(BaseModel, ABC):
    name: str  # Unique name
    description: str  # Used to tell the model when/how to use
    args_schema: type[PydanticBaseModel]  # Pydantic model for arguments
    env_vars: list[EnvVar]  # Required environment variables

    cache_function: Callable[..., bool] = lambda _args, _result: True
    result_as_answer: bool = False  # If True, becomes final agent answer
    max_usage_count: int | None = None  # Usage limit
    current_usage_count: int = 0

    @abstractmethod
    def _run(self, *args, **kwargs) -> Any: ...

    async def _arun(self, *args, **kwargs) -> Any:
        raise NotImplementedError("Override _arun for async support")
```

**Key design pattern**: Auto-generated args_schema from `_run` signature:

```python
@field_validator("args_schema", mode="before")
@classmethod
def _default_args_schema(cls, v) -> type[PydanticBaseModel]:
    run_sig = signature(cls._run)
    fields = {}
    for param_name, param in run_sig.parameters.items():
        if param_name in ("self", "return"):
            continue
        annotation = param.annotation if param.annotation != param.empty else Any
        if param.default is param.empty:
            fields[param_name] = (annotation, ...)
        else:
            fields[param_name] = (annotation, param.default)
    return create_model(f"{cls.__name__}Schema", **fields)
```

#### Tool Decorator (`base_tool.py`)

Functional tool creation via decorator:

```python
@tool
def greet(name: str) -> str:
    '''Greet someone.'''
    return f"Hello, {name}!"

@tool("custom_name", result_as_answer=True, max_usage_count=10)
def search(query: str) -> str:
    '''Search the web.'''
    return f"Results for: {query}"
```

#### Structured Tool Conversion

Tools convert to `CrewStructuredTool` for agent use:

```python
def to_structured_tool(self) -> CrewStructuredTool:
    structured_tool = CrewStructuredTool(
        name=self.name,
        description=self.description,
        args_schema=self.args_schema,
        func=self._run,
        result_as_answer=self.result_as_answer,
        max_usage_count=self.max_usage_count,
        cache_function=self.cache_function,
    )
    structured_tool._original_tool = self
    return structured_tool
```

### Tool Discovery and Loading

Tools are discovered via `crewai_tools/__init__.py` which exports 75+ tools:

```python
from crewai_tools import (
    SerperDevTool, BraveSearchTool, DallETool,
    FileReadTool, FileWriterTool, PDFSearchTool,
    # ... 75+ more
)
```

### MCP Integration

#### MCPServerAdapter (`crewai_tools/adapters/mcp_adapter.py`)

Manages MCP server lifecycle and tool discovery:

```python
class MCPServerAdapter:
    """Manages the lifecycle of an MCP server."""

    def __init__(
        self,
        serverparams: StdioServerParameters | dict[str, Any],
        *tool_names: str,  # Optional filter
        connect_timeout: int = 30,
    ):
        self._adapter = MCPAdapt(
            self._serverparams, CrewAIToolAdapter(), connect_timeout
        )
        self.start()

    def start(self) -> None:
        self._tools = self._adapter.__enter__()

    def stop(self) -> None:
        self._adapter.__exit__(None, None, None)

    @property
    def tools(self) -> ToolCollection[BaseTool]:
        if self._tool_names:
            return tools_collection.filter_by_names(self._tool_names)
        return tools_collection
```

**Usage patterns**:

```python
# STDIO
with MCPServerAdapter({"command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]}) as tools:
    agent = Agent(tools=tools)

# SSE (HTTP)
with MCPServerAdapter({"url": "http://localhost:8000/sse"}) as tools:
    agent = Agent(tools=tools)

# Filtered tools
with MCPServerAdapter(params, "read_file", "write_file") as filtered_tools:
    agent = Agent(tools=filtered_tools)
```

#### CrewAIToolAdapter

Custom adapter that creates CrewAI tools with properly normalized JSON schemas:

```python
class CrewAIToolAdapter(ToolAdapter):
    def adapt(
        self,
        func: Callable[[dict | None], CallToolResult],
        mcp_tool: Tool,
    ) -> BaseTool:
        tool_name = sanitize_tool_name(mcp_tool.name)
        args_model = create_model_from_schema(mcp_tool.inputSchema)

        class CrewAIMCPTool(BaseTool):
            name: str = tool_name
            description: str = tool_description
            args_schema: type[BaseModel] = args_model

            def _run(self, **kwargs: Any) -> Any:
                result = func(kwargs)
                # Handle TextContent responses
                return content_item.text if isinstance(first_content, TextContent) else str(first_content)
```

### MCP Tool Wrapper (`lib/crewai/src/crewai/tools/mcp_tool_wrapper.py`)

On-demand MCP server connection wrapper:

```python
class MCPToolWrapper(BaseTool):
    """Lightweight wrapper for MCP tools that connects on-demand."""

    MCP_CONNECTION_TIMEOUT = 15
    MCP_TOOL_EXECUTION_TIMEOUT = 60
    MCP_DISCOVERY_TIMEOUT = 15
    MCP_MAX_RETRIES = 3

    async def _retry_with_exponential_backoff(self, operation_func, **kwargs) -> str:
        for attempt in range(MCP_MAX_RETRIES):
            result, error, should_retry = await self._execute_single_attempt(
                operation_func, **kwargs
            )
            if result is not None:
                return result
            if not should_retry:
                return error
            if attempt < MCP_MAX_RETRIES - 1:
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
```

**Error classification**:
- Non-retryable: Authentication failures, "not found", JSON parsing errors
- Retryable: Network/connection issues, timeouts

### MCP Native Tool (`tools/mcp_native_tool.py`)

Fresh client per invocation for safe parallel execution:

```python
class MCPNativeTool(BaseTool):
    """Native MCP tool that creates a fresh client per invocation."""

    def _run(self, **kwargs: Any) -> str:
        try:
            asyncio.get_running_loop()
            # Already in async context - use thread pool
            with concurrent.futures.ThreadPoolExecutor() as executor:
                coro = self._run_async(**kwargs)
                future = executor.submit(ctx.run, asyncio.run, coro)
                return future.result()
        except RuntimeError:
            # Not in async context - run directly
            return asyncio.run(self._run_async(**kwargs))

    async def _run_async(self, **kwargs: Any) -> str:
        client = self._client_factory()
        await client.connect()
        try:
            result = await client.call_tool(self.original_tool_name, kwargs)
        finally:
            await client.disconnect()
        return content_item.text if hasattr(content_item, 'text') else str(result)
```

**Key insight**: Fresh client factory ensures concurrent invocations never share mutable connection state (avoids anyio cancel-scope errors).

### Example Tool: SerperDevTool

```python
class SerperDevTool(BaseTool):
    name: str = "Search the internet with Serper"
    description: str = "A tool that can be used to search the internet..."
    args_schema: type[BaseModel] = SerperDevToolSchema
    base_url: str = "https://google.serper.dev"
    n_results: int = 10
    save_file: bool = False
    search_type: str = "search"
    country: str | None = ""
    location: str | None = ""
    locale: str | None = ""
    env_vars: list[EnvVar] = Field(default_factory=lambda: [
        EnvVar(name="SERPER_API_KEY", description="API key for Serper", required=True)
    ])

    def _run(self, **kwargs: Any) -> FormattedResults:
        search_query = kwargs.get("search_query")
        search_type = kwargs.get("search_type", self.search_type)
        results = self._make_api_request(search_query, search_type)
        return self._process_search_results(results, search_type)

    def _get_search_url(self, search_type: str) -> str:
        return f"{self.base_url}/{search_type}"
```

---

## 3. CLI & Project Scaffolding

### CLI Architecture

#### Main Entry (`cli/cli.py`)

Click-based CLI with subcommands:

```python
@click.group()
@click.version_option(get_version("crewai"))
def crewai() -> None:
    """Top-level command group for crewai."""
```

**Commands**:
- `crewai create crew <name>` - Create new crew project
- `crewai create flow <name>` - Create new flow project
- `crewai run` - Run a crew
- `crewai train` - Train a crew
- `crewai test` - Test a crew
- `crewai replay` - Replay execution from task
- `crewai chat` - Interactive chat mode
- `crewai reset-memories` - Reset crew memories
- `crewai uv` - UV wrapper with tool auth
- And more (auth, deploy, enterprise, settings, triggers, tools)

#### Crew Creation (`cli/create_crew.py`)

```python
def create_crew(
    name: str,
    provider: str | None = None,
    skip_provider: bool = False,
    parent_folder: str | None = None,
) -> None:
    folder_path, folder_name, class_name = create_folder_structure(name, parent_folder)
    # ... provider selection, API key prompts
    copy_template_files(folder_path, name, class_name, parent_folder)
```

**Folder structure validation** (`create_folder_structure`):

```python
def create_folder_structure(name: str, parent_folder: str | None = None):
    folder_name = name.replace(" ", "_").replace("-", "_").lower()
    folder_name = re.sub(r"[^a-zA-Z0-9_]", "", folder_name)

    # Validations:
    # - Cannot start with digit
    # - Cannot be Python keyword
    # - Cannot conflict with reserved script names
    # - Must be valid Python identifier

    class_name = name.replace("_", " ").replace("-", " ").title().replace(" ", "")
    class_name = re.sub(r"[^a-zA-Z0-9_]", "", class_name)

    return folder_path, folder_name, class_name
```

**Reserved script names check**:

```python
def get_reserved_script_names() -> set[str]:
    """Get reserved script names from pyproject.toml template."""
    template_path = package_dir / "templates" / "crew" / "pyproject.toml"
    template_data = tomli.loads(template_content)
    script_names = set(template_data.get("project", {}).get("scripts", {}).keys())
    return script_names
```

### Project Templates

#### Crew Template (`cli/templates/crew/`)

```
crew/
├── pyproject.toml          # Project config with crewai dependency
├── README.md
├── .gitignore
├── main.py                # Entry point (run/train/test/replay)
├── src/
│   └── {folder_name}/
│       ├── __init__.py
│       ├── crew.py        # @CrewBase class
│       ├── config/
│       │   ├── agents.yaml
│       │   └── tasks.yaml
│       └── tools/
│           ├── __init__.py
│           └── custom_tool.py
├── tests/
└── knowledge/
    └── user_preference.txt
```

#### Crew Class Template (`templates/crew/crew.py`)

```python
@CrewBase
class {{crew_name}}():
    """{{crew_name}} crew"""

    agents: list[BaseAgent]
    tasks: list[Task]

    @agent
    def researcher(self) -> Agent:
        return Agent(
            config=self.agents_config['researcher'],
            verbose=True
        )

    @task
    def research_task(self) -> Task:
        return Task(
            config=self.tasks_config['research_task'],
        )

    @crew
    def crew(self) -> Crew:
        return Crew(
            agents=self.agents,
            tasks=self.tasks,
            process=Process.sequential,
            verbose=True,
        )
```

#### Main Entry Template (`templates/crew/main.py`)

```python
def run():
    """Run the crew."""
    inputs = {'topic': 'AI LLMs', 'current_year': str(datetime.now().year)}
    {{crew_name}}().crew().kickoff(inputs=inputs)

def train():
    """Train the crew for a given number of iterations."""
    {{crew_name}}().crew().train(
        n_iterations=int(sys.argv[1]),
        filename=sys.argv[2],
        inputs=inputs
    )

def replay():
    """Replay the crew execution from a specific task."""
    {{crew_name}}().crew().replay(task_id=sys.argv[1])

def test():
    """Test the crew execution."""
    {{crew_name}}().crew().test(
        n_iterations=int(sys.argv[1]),
        eval_llm=sys.argv[2],
        inputs=inputs
    )
```

#### Flow Template (`cli/templates/flow/`)

```
flow/
├── pyproject.toml
├── README.md
├── main.py                # Entry point (kickoff/plot/run_with_trigger)
└── crews/
    └── {crew_name}/
        └── poem_crew.py    # Crew definition
```

**Flow main.py**:

```python
class PoemFlow(Flow[PoemState]):
    @start()
    def generate_sentence_count(self, crewai_trigger_payload: dict = None):
        if crewai_trigger_payload:
            self.state.sentence_count = crewai_trigger_payload.get('sentence_count', randint(1, 5))
        else:
            self.state.sentence_count = randint(1, 5)

    @listen(generate_sentence_count)
    def generate_poem(self):
        result = PoemCrew().crew().kickoff(
            inputs={"sentence_count": self.state.sentence_count}
        )
        self.state.poem = result.raw

def kickoff():
    PoemFlow().kickoff()

def plot():
    PoemFlow().plot()
```

### Template Variable Substitution

Template files use `{{folder_name}}`, `{{name}}`, `{{crew_name}}` placeholders:

```python
def copy_template(src_file, dst_file, name, class_name, folder_name):
    with open(src_file, "r") as f:
        content = f.read()

    content = content.replace("{{folder_name}}", folder_name)
    content = content.replace("{{name}}", name.lower().replace(" ", "_"))
    content = content.replace("{{crew_name}}", class_name)

    with open(dst_file, "w") as f:
        f.write(content)
```

### Provider Selection

Interactive provider/model selection (`cli/provider.py`):

```python
def select_model(selected_provider: str, provider_models: dict) -> str | None:
    # Display numbered list of available models
    # User selects by number or 'q' to quit
    selected_model = click.prompt(...)
    return selected_model
```

### Agent Configuration Template (`templates/crew/config/agents.yaml`)

```yaml
researcher:
  role: "Researcher"
  goal: "Find and synthesize information about {topic}"
  backstory: "You are a thorough researcher..."
  verbose: true

reporting_analyst:
  role: "Reporting Analyst"
  goal: "Create comprehensive reports"
  backstory: "You are a skilled analyst..."
  verbose: true
```

### Memory TUI

Interactive memory visualization (`cli/memory_tui.py`): 14KB+ module for inspecting memory state with tree views and search.

---

## Key Technical Patterns & Insights

### 1. Lazy Initialization

Memory uses lazy LLM/embedder initialization to avoid heavy imports:

```python
@property
def _llm(self) -> BaseLLM:
    if self._llm_instance is None:
        from crewai.llm import LLM
        self._llm_instance = LLM(model=self.llm if isinstance(self.llm, str) else str(self.llm))
    return self._llm_instance
```

### 2. Background Write Queue

Non-blocking saves with thread pool:

```python
_save_pool: ThreadPoolExecutor = PrivateAttr(
    default_factory=lambda: ThreadPoolExecutor(max_workers=1, thread_name_prefix="memory-save")
)
_pending_saves: list[Future[Any]] = PrivateAttr(default_factory=list)

def _submit_save(self, fn: Any, *args, **kwargs) -> Future[Any]:
    ctx = contextvars.copy_context()
    future = self._save_pool.submit(ctx.run, fn, *args, **kwargs)
    future.add_done_callback(self._on_save_done)
    return future
```

### 3. Pluggable Storage Pattern

Storage backend abstraction via `StorageBackend` protocol:

```python
class LanceDBStorage:
    def search(self, embedding, scope_prefix=None, categories=None, limit=10, min_score=0.0): ...
    def save(self, record: MemoryRecord): ...
    def update(self, record: MemoryRecord): ...
    def delete(self, scope_prefix=None, categories=None, ...): ...
```

### 4. Batch Operations

Embedding and storage operations batch for efficiency:

```python
# One embedder call for N texts
embeddings = embed_texts(self._embedder, texts)

# N concurrent storage searches
with ThreadPoolExecutor(max_workers=min(len(tasks), 4)) as pool:
    futures = {pool.submit(_search_one, emb, sc) for emb, sc in tasks}
```

### 5. Error Recovery Patterns

- LanceDB: Exponential backoff for commit conflicts
- MCP: Retry with exponential backoff, non-retryable error classification
- Memory: Silent failures with event emission, never breaks recall on save failure

### 6. Schema Validation

Pydantic-based schema validation with helpful error messages:

```python
def _validate_kwargs(self, kwargs: dict[str, Any]) -> dict[str, Any]:
    if self.args_schema is not None and self.args_schema.model_fields:
        validated = self.args_schema.model_validate(kwargs)
        return validated.model_dump()
    return kwargs
```

### 7. Hierarchical Scoping

Memory uses path-based hierarchical scoping for organization:

```python
# /crew/{crew_name}/agent/{agent_role}/task/{task_id}
# /company/engineering/memory
# /company/sales/memory
```

### 8. Tool Name Sanitization

Tools sanitize names to avoid conflicts:

```python
def sanitize_tool_name(name: str) -> str:
    # Replace spaces/special chars, ensure valid Python identifier
    return re.sub(r"[^a-zA-Z0-9_]", "_", name.lower())
```

### 9. Fresh MCP Client Pattern

MCP native tools create fresh client per invocation to avoid concurrent execution issues:

```python
client = self._client_factory()  # New client each time
await client.connect()
try:
    result = await client.call_tool(...)
finally:
    await client.disconnect()
```

### 10. Template Pattern for Scaffolding

Jinja2-style variable substitution for project generation with comprehensive validation of names before template rendering.

---

## File Locations Summary

| Feature | Key Files |
|---------|-----------|
| Memory | `lib/crewai/src/crewai/memory/unified_memory.py`, `memory/types.py`, `memory/recall_flow.py`, `memory/encoding_flow.py`, `memory/memory_scope.py` |
| Storage | `lib/crewai/src/crewai/memory/storage/lancedb_storage.py`, `storage/qdrant_edge_storage.py` |
| Tools | `lib/crewai-tools/src/crewai_tools/base_tool.py`, `lib/crewai-tools/src/crewai_tools/__init__.py` |
| MCP | `lib/crewai-tools/src/crewai_tools/adapters/mcp_adapter.py`, `lib/crewai/src/crewai/tools/mcp_tool_wrapper.py`, `tools/mcp_native_tool.py` |
| CLI | `lib/crewai/src/crewai/cli/cli.py`, `cli/create_crew.py`, `cli/templates/crew/`, `cli/templates/flow/` |

---

## Version

Current: `1.13.0rc1`

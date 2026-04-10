# Design Patterns in crewAI

This document catalogs the design patterns used throughout the crewAI codebase with specific file references and code examples.

## Table of Contents

1. [Singleton Pattern](#singleton-pattern)
2. [Factory Pattern](#factory-pattern)
3. [Observer Pattern](#observer-pattern)
4. [Strategy Pattern](#strategy-pattern)
5. [Plugin/Extension Pattern](#pluginextension-pattern)
6. [Decorator Pattern](#decorator-pattern)
7. [Builder Pattern](#builder-pattern)
8. [Repository Pattern](#repository-pattern)
9. [Template Method Pattern](#template-method-pattern)
10. [Command Pattern](#command-pattern)

---

## Singleton Pattern

**Purpose:** Ensure a class has only one instance and provide a global point of access.

### Event Bus Singleton

**File:** `lib/crewai/src/crewai/events/event_bus.py` (line 54)

```python
class CrewAIEventsBus:
    """Singleton event bus for handling events in CrewAI."""

    _instance: Self | None = None
    _instance_lock: threading.RLock = threading.RLock()

    def __new__(cls) -> Self:
        """Create or return the singleton instance."""
        if cls._instance is None:
            with cls._instance_lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._initialize()
        return cls._instance
```

**Usage:** `crewai_event_bus = CrewAIEventsBus()` (module-level singleton)

**Why:** The event bus must be global to allow any component to emit or subscribe to events without explicit dependency injection.

### EventListener Singleton

**File:** `lib/crewai/src/crewai/events/event_listener.py` (line 119)

```python
class EventListener(BaseEventListener):
    _instance: EventListener | None = None

    def __new__(cls) -> EventListener:
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance
```

---

## Factory Pattern

**Purpose:** Create objects without specifying exact classes, delegating to subclasses or configuration.

### LLM Factory

**File:** `lib/crewai/src/crewai/utilities/llm_utils.py` (line 13)

```python
def create_llm(
    llm_value: str | LLM | Any | None = None,
) -> LLM | BaseLLM | None:
    """Creates or returns an LLM instance based on the given llm_value."""

    if isinstance(llm_value, (LLM, BaseLLM)):
        return llm_value

    if isinstance(llm_value, str):
        try:
            return LLM(model=llm_value)
        except Exception as e:
            logger.error(f"Error instantiating LLM from string: {e}")
            raise e

    if llm_value is None:
        return _llm_via_environment_or_fallback()
```

**Usage:** Throughout crewAI when any LLM parameter can be string, instance, or None.

**Why:** Allows flexible LLM configuration from strings (model names), objects with LLM attributes, or environment defaults.

### Embedder Factory

**File:** `lib/crewai/src/crewai/rag/embeddings/factory.py` (line 267)

```python
def build_embedder(
    spec: AzureProviderSpec,
) -> OpenAIEmbeddingFunction: ...
def build_embedder(
    spec: BedrockProviderSpec,
) -> AmazonBedrockEmbeddingFunction: ...
def build_embedder(
    spec: OpenAIProviderSpec,
) -> OpenAIEmbeddingFunction: ...
# ... many overloads for different providers

def build_embedder(spec: dict[str, Any]) -> EmbeddingFunction[Any]:
    """Build an embedder from a provider specification dictionary."""
    provider_type = spec.get("provider", "openai").lower()
    provider_path = PROVIDER_PATHS.get(provider_type)

    if not provider_path:
        raise ValueError(f"Unknown embedder provider: {provider_type}")

    provider_class = import_and_validate_definition(provider_path, BaseEmbeddingsProvider)
    return build_embedder_from_provider(provider_class(**spec))
```

**Usage:** Memory and Knowledge systems when creating embedding functions.

**Why:** Decouples embedder creation from specific provider implementations, allowing new providers to be added without modifying calling code.

### Tool Creation Factory

**File:** `lib/crewai/src/crewai/tools/structured_tool.py` (line 89)

```python
class CrewStructuredTool:
    @classmethod
    def from_function(
        cls,
        func: Callable[..., Any],
        name: str | None = None,
        description: str | None = None,
        return_direct: bool = False,
        args_schema: type[BaseModel] | None = None,
        infer_schema: bool = True,
        **kwargs: Any,
    ) -> CrewStructuredTool:
        """Create a tool from a function.

        Automatically infers argument schema from function signature using Pydantic.
        """
        if name is None:
            name = func.__name__

        if description is None:
            description = func.__doc__ or f"Tool {name}"

        if args_schema is None and infer_schema:
            args_schema = cls._create_schema_from_function(func)

        return cls(
            name=sanitize_tool_name(name),
            description=description,
            args_schema=args_schema,
            func=func,
            result_as_answer=return_direct,
            **kwargs,
        )
```

**Usage:** Creating tools from plain Python functions.

**Why:** Enables zero-code tool creation from functions with automatic schema inference.

---

## Observer Pattern

**Purpose:** Define a one-to-many dependency between objects so changes notify dependents.

### Event Bus (Centralized Observer)

**File:** `lib/crewai/src/crewai/events/event_bus.py`

**Event Registration:**
```python
def subscribe(
    self,
    event_type: type[BaseEvent],
    handler: Handler,
    dependencies: list[Depends[Any]] | None = None,
) -> None:
    """Register a handler for an event type."""
```

**Event Emission:**
```python
def emit(
    self,
    event: BaseEvent,
    synchronous: bool = False,
) -> Future[Any] | None:
    """Emit an event to all registered handlers."""
```

**Event Types (lib/crewai/src/crewai/events/types/):**
- `llm_events.py`: LLMCallStartedEvent, LLMCallCompletedEvent, LLMStreamChunkEvent
- `tool_usage_events.py`: ToolUsageStartedEvent, ToolUsageFinishedEvent
- `agent_events.py`: AgentExecutionStartedEvent, AgentExecutionCompletedEvent
- `task_events.py`: TaskStartedEvent, TaskCompletedEvent
- `flow_events.py`: FlowStartedEvent, MethodExecutionStartedEvent

**Usage Example:**
```python
# Subscribe
crewai_event_bus.subscribe(
    LLMCallCompletedEvent,
    my_handler,
    dependencies=[Depends(get_current_call_id)]
)

# Emit
crewai_event_bus.emit(LLMCallStartedEvent(call_id=call_id, model=model))
```

**Why:** Decouples event producers from consumers, enabling logging, tracing, and custom handlers without modifying core code.

### EventListener Abstract Base

**File:** `lib/crewai/src/crewai/events/base_event_listener.py` (line 8)

```python
class BaseEventListener(ABC):
    """Abstract base for event listeners."""

    @staticmethod
    def on_llm_call_started(
        event: LLMCallStartedEvent,
        event_handler: LLMCallHookContext,
    ) -> None:
        """Override to handle LLM call start events."""
        pass
```

**Concrete Implementation:** `TraceCollectionListener` (`events/listeners/tracing/trace_listener.py`)

```python
class TraceCollectionListener(BaseEventListener):
    """OpenTelemetry tracing listener for crew events."""

    @staticmethod
    def on_llm_call_started(
        event: LLMCallStartedEvent,
        event_handler: LLMCallHookContext,
    ) -> None:
        with self._tracer.start_as_current_span(f"llm:{event.model}") as span:
            span.set_attribute("llm.call_id", event.call_id)
            span.set_attribute("llm.model", event.model)
```

---

## Strategy Pattern

**Purpose:** Define a family of algorithms, encapsulate each, and make them interchangeable.

### LLM Strategy

**File:** `lib/crewai/src/crewai/llms/base_llm.py` (line 85)

```python
class BaseLLM(ABC):
    """Abstract base class for LLM implementations."""

    is_litellm: bool = False

    @abstractmethod
    def call(
        self,
        messages: list[LLMMessage],
        callbacks: list[Any] | None = None,
    ) -> LLMResponse:
        """Make an LLM call with messages."""

    def stream(
        self,
        messages: list[LLMMessage],
        callbacks: list[Any] | None = None,
    ) -> Generator[LLMResponse, None, None]:
        """Stream LLM response."""
```

**Provider Implementations:**
- `llms/providers/openai/` - OpenAI models
- `llms/providers/anthropic/` - Anthropic Claude
- `llms/providers/azure/` - Azure OpenAI
- `llms/providers/bedrock/` - AWS Bedrock
- `llms/providers/gemini/` - Google Gemini

**Usage:**
```python
llm = LLM(model="gpt-4")  # Returns OpenAI provider
llm = LLM(model="claude-3-sonnet")  # Returns Anthropic provider
llm = LLM(model="gpt-4", provider="azure")  # Returns Azure provider
```

**Why:** Users can switch LLM providers without changing agent code. Each provider implements the same interface with provider-specific logic.

### Cache Strategy

**File:** `lib/crewai/src/crewai/tools/base_tool.py` (line 83)

```python
class BaseTool(BaseModel, ABC):
    cache_function: Callable[..., bool] = Field(
        default=lambda _args=None, _result=None: True,
        description="Function that will be used to determine if the tool should be cached.",
    )
```

**Usage:** Custom cache logic per tool:
```python
def my_cache_strategy(args, result):
    return len(str(result)) < 1000  # Only cache small results

tool = MyTool(cache_function=my_cache_strategy)
```

**Why:** Different tools may have different caching strategies based on result size, computation cost, or staleness requirements.

---

## Plugin/Extension Pattern

**Purpose:** Allow new functionality to be added without modifying existing code.

### Tool Plugin Architecture

**File:** `lib/crewai/src/crewai/tools/base_tool.py` (line 57)

```python
class BaseTool(BaseModel, ABC):
    """Abstract base class for all tools in CrewAI."""

    name: str
    description: str
    args_schema: type[PydanticBaseModel]
    cache_function: Callable[..., bool]
    max_usage_count: int | None

    @abstractmethod
    def _run(self, *args, **kwargs) -> Any:
        """Implement tool functionality."""

    async def _arun(self, *args, **kwargs) -> Any:
        """Async version - default implentation calls _run in executor."""
        return self._run(*args, **kwargs)
```

**Plugin Registration:** Tools are discovered through the `crewai-tools` package `__init__.py` which exports 75+ tool classes.

**Usage:**
```python
from crewai_tools import TavilySearchTool, SeleniumScrapingTool

tool = TavilySearchTool(api_key=...)
result = tool.run(query="...")
```

**Why:** Users can add new tools by implementing `BaseTool`. The framework doesn't need to know about specific tools at compile time.

### Knowledge Source Plugin

**File:** `lib/crewai/src/crewai/knowledge/source/base_knowledge_source.py`

```python
class BaseKnowledgeSource(ABC):
    """Abstract base for knowledge sources."""

    @abstractmethod
    def load(self) -> list[Document]:
        """Load content and return documents."""

    @abstractmethod
    def add(self) -> None:
        """Add content to vector storage."""
```

**Implementations:** PDFSource, TextSource (extensible by users)

**Why:** Users can create custom knowledge sources for any data format by implementing the interface.

---

## Decorator Pattern

**Purpose:** Attach additional responsibilities to objects dynamically.

### Flow Decorators

**File:** `lib/crewai/src/crewai/flow/flow.py` (line 145)

```python
def start(
    condition: str | FlowCondition | Callable[..., Any] | None = None,
) -> Callable[[Callable[P, R]], StartMethod[P, R]]:
    """Marks a method as a flow's starting point."""
    def decorator(func: Callable[P, R]) -> StartMethod[P, R]:
        wrapper = StartMethod(func)
        wrapper.__trigger_methods__ = None
        wrapper.__condition_type__ = None
        wrapper.__trigger_condition__ = None
        return wrapper
    return decorator

def listen(
    trigger: FlowMethodName | Callable[..., Any],
    condition: str | FlowCondition | None = None,
) -> Callable[[Callable[P, R]], ListenMethod[P, R]]:
    """Marks a method as listening to another method's output."""
    ...

def router(
    condition: FlowCondition | Callable[..., Any],
) -> Callable[[Callable[P, R]], RouterMethod[P, R]]:
    """Marks a method as a router for conditional execution."""
    ...
```

**Usage:**
```python
class PoemFlow(Flow[PoemState]):
    @start()
    def generate_sentence_count(self) -> int:
        return 5

    @listen(generate_sentence_count)
    def generate_poem(self, count: int) -> str:
        # This waits for generate_sentence_count to complete
        return f"Here is a poem with {count} sentences"
```

**Why:** Declaratively defines flow topology without explicit event subscription code.

### FlowMethod Wrapper (Descriptor Pattern)

**File:** `lib/crewai/src/crewai/flow/flow_wrappers.py` (line 41)

```python
class FlowMethod(Generic[P, R]):
    """Wrapper for flow methods with decorator metadata."""

    def __init__(self, meth: Callable[P, R], instance: Any = None) -> None:
        self._meth = meth
        self._instance = instance
        functools.update_wrapper(self, meth, updated=[])

    def __call__(self, *args: P.args, **kwargs: P.kwargs) -> R:
        if self._instance is not None:
            return self._meth(self._instance, *args, **kwargs)
        return self._meth(*args, **kwargs)

    def __get__(self, instance: Any, owner: type | None = None) -> Self:
        """Support the descriptor protocol for method binding."""
        if instance is None:
            return self
        bound = type(self)(self._meth, instance)
        # Copy metadata...
        return bound
```

**Why:** Preserves method metadata while adding decorator attributes. The descriptor protocol allows proper binding when accessed as instance methods.

### Project Decorators (@CrewBase, @agent, @task)

**File:** `lib/crewai/src/crewai/project/crew_base.py`

```python
class CrewBase:
    """Metaclass-like decorator for crew classes."""

    def __init__(self, crew_class: type) -> None:
        self._agents: dict[str, Agent] = {}
        self._tasks: dict[str, Task] = {}
        # Scan for @agent and @task decorated methods
        # Load YAML configs
        # Wire together agents and tasks

    def __call__(self) -> Crew:
        return self._create_crew()
```

**Usage:**
```python
@CrewBase
class ResearchCrew:
    @agent
    def researcher(self) -> Agent:
        return Agent(config=self.agents_config['researcher'])

    @task
    def research_task(self) -> Task:
        return Task(config=self.tasks_config['research_task'])

    @crew
    def crew(self) -> Crew:
        return Crew(agents=self.agents, tasks=self.tasks)
```

**Why:** Reduces boilerplate by auto-wiring decorated methods to YAML configs and assembling the crew automatically.

---

## Builder Pattern

**Purpose:** Construct complex objects step by step.

### Agent Executor Builder

**File:** `lib/crewai/src/crewai/agent/core.py` (line 277)

```python
@classmethod
def create_agent_executor(
    cls,
    tools: list[BaseTool],
    **kwargs: Any,
) -> CrewAgentExecutor:
    """Factory method to create agent executor with proper configuration."""
    # Step-by-step construction of executor with all dependencies
    llm = kwargs.get('llm')
    task = kwargs.get('task')
    crew = kwargs.get('crew')
    # ...
    return CrewAgentExecutor(
        llm=llm,
        task=task,
        crew=crew,
        # ... many more parameters built up
    )
```

**Why:** Complex executor has many dependencies; builder method ensures all are provided in correct order.

### Crew Creation Builder

**File:** `lib/crewai/src/crewai/crew.py` (line 366)

```python
def create_crew_memory(self) -> Crew:
    """Create crew with memory system."""
    if self.memory:
        self._cache_handler = CacheHandler()
        self._memory = Memory(
            llm=self.agents[0].llm if self.agents else None,
            storage=self.memory_storage,
        )
    return self

def create_crew_knowledge(self) -> Crew:
    """Create crew with knowledge system."""
    if self.knowledge:
        # Configure knowledge sources
        pass
    return self
```

**Why:** Fluent interface for optional subsystems that can be added progressively.

### Task Builder

**File:** `lib/crewai/src/crewai/crew.py` (line 610)

```python
def _create_task(self, task_config: dict[str, Any]) -> Task:
    """Build task from configuration dictionary."""
    task = Task(
        description=task_config.get('description'),
        expected_output=task_config.get('expected_output'),
        agent=self._find_agent(task_config.get('agent')),
    )

    # Add context tasks
    for context_name in task_config.get('context', []):
        task.context.append(self._tasks_dict[context_name])

    # Configure output
    if task_config.get('output_file'):
        task.output_file = task_config['output_file']

    return task
```

**Why:** Complex task objects with many optional parameters built from dict configs.

---

## Repository Pattern

**Purpose:** Abstract data layer, providing collection-like interface for domain objects.

### Memory Storage Repository

**File:** `lib/crewai/src/crewai/memory/storage/backend.py`

```python
class StorageBackend(Protocol):
    """Repository interface for memory storage."""

    def save(self, key: str, value: MemoryRecord) -> None:
        """Save a memory record."""

    def search(
        self,
        query: str,
        limit: int = 5,
    ) -> list[tuple[MemoryRecord, float]]:
        """Search memories by query string."""

    def delete(self, key: str) -> None:
        """Delete a memory record."""

    def reset(self) -> None:
        """Clear all memories."""
```

**Implementations:**
- `lancedb_storage.py` - LanceDB backend
- `qdrant_edge_storage.py` - Qdrant backend
- `kickoff_task_outputs_storage.py` - Task output storage

**Why:** Decouples memory operations from storage implementation. Users can swap backends without changing memory logic.

### Knowledge Storage Repository

**File:** `lib/crewai/src/crewai/knowledge/storage/knowledge_storage.py`

```python
class KnowledgeStorage:
    """Repository for knowledge documents with vector search."""

    def add_documents(
        self,
        documents: list[Document],
        metadata: list[dict] | None = None,
    ) -> None:
        """Add documents to vector store."""

    def search(
        self,
        query: list[str],
        limit: int = 5,
        score_threshold: float = 0.6,
    ) -> list[SearchResult]:
        """Semantic search across documents."""
```

**Why:** Abstracts vector database operations behind a repository interface.

---

## Template Method Pattern

**Purpose:** Define algorithm skeleton in base class, let subclasses override specific steps.

### Embedding Function Hook

**File:** `lib/crewai/src/crewai/rag/core/base_embeddings_callable.py` (line 129)

```python
@runtime_checkable
class EmbeddingFunction(Protocol[D]):
    """Protocol for embedding functions."""

    def __call__(self, input: D) -> Embeddings:
        """Convert input data to embeddings."""
        ...

    def __init_subclass__(cls) -> None:
        """Wrap __call__ method to normalize and validate embeddings."""
        super().__init_subclass__()
        original_call = cls.__call__

        def wrapped_call(self: EmbeddingFunction[D], input: D) -> Embeddings:
            result = original_call(self, input)
            # Always validate and normalize
            normalized = normalize_embeddings(result)
            return validate_embeddings(normalized)

        cls.__call__ = wrapped_call
```

**Usage:** Every subclass automatically gets validation without explicit code.

**Why:** All embedding functions must normalize/validate output. The hook ensures this happens automatically.

### Base Tool Async Fallback

**File:** `lib/crewai/src/crewai/tools/base_tool.py`

```python
class BaseTool(BaseModel, ABC):
    @abstractmethod
    def _run(self, *args, **kwargs) -> Any:
        """Synchronous tool implementation."""

    async def _arun(self, *args, **kwargs) -> Any:
        """Default async implementation - runs sync version in executor."""
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(None, self._run, *args, **kwargs)
```

**Why:** Tools only need to implement `_run`. Async version is provided automatically. Subclasses can override `_arun` for true async.

---

## Command Pattern

**Purpose:** Encapsulate requests as objects, allowing parameterization and queuing.

### Tool Execution as Commands

**File:** `lib/crewai/src/crewai/tools/tool_usage.py`

```python
@dataclass
class ToolExecution:
    """Command object representing a tool execution request."""
    tool: BaseTool
    args: dict[str, Any]
    result: Any = None
    error: Exception | None = None
    from_cache: bool = False
```

**Usage:**
```python
execution = ToolExecution(tool=my_tool, args={"query": "search"})
crewai_event_bus.emit(ToolUsageStartedEvent(tool_name=my_tool.name))
result = execution.tool.run(**execution.args)
```

**Why:** Execution details (tool, args, result, timing) are encapsulated, enabling logging, retry, and caching.

### Callback Commands

**File:** `lib/crewai/src/crewai/types/callback.py`

```python
@dataclass
class ToolCallHookContext:
    """Context passed to tool hooks."""
    tool_name: str
    arguments: dict[str, Any]
    agent_id: str
    task_id: str
```

**Usage:** Hooks receive context objects that can be inspected or modified before tool execution.

**Why:** Encapsulates all context needed for a callback, decoupled from actual execution.

---

## Pattern Summary Table

| Pattern | Location | Purpose |
|---------|----------|---------|
| Singleton | `event_bus.py` | Global event coordination |
| Factory | `llm_utils.py`, `factory.py` | Flexible object creation |
| Observer | `event_bus.py` | Decoupled event handling |
| Strategy | `base_llm.py` | Interchangeable LLM providers |
| Plugin | `base_tool.py` | Extensible tool system |
| Decorator | `flow.py`, `crew_base.py` | Method metadata + flow control |
| Builder | `crew.py`, `agent/core.py` | Step-by-step object construction |
| Repository | `memory/storage/`, `knowledge/` | Abstract data layer |
| Template Method | `base_embeddings_callable.py` | Hook into inheritance |
| Command | `tool_usage.py` | Encapsulate execution requests |

---

## Anti-Patterns to Avoid

### Magic String Routing
Some event handlers use string-based event type names rather than direct type references:
```python
# Avoid
handler_registry["LLMCallStartedEvent"]

# Prefer
crewai_event_bus.subscribe(LLMCallStartedEvent, handler)
```

### Deep Parameter Chains
Some executor constructors pass 15+ parameters. Consider using configuration objects or context classes instead.

### Mixed Synchronous/Async
Some components have both sync and async versions with unclear when to use which. The framework is transitioning to async-first.

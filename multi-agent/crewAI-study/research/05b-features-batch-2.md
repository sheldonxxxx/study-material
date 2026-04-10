# CrewAI Features Batch 2: Task Management & Delegation, Flexible LLM Integration, RAG & Knowledge Management

## Table of Contents
1. [Task Management & Delegation](#1-task-management--delegation)
2. [Flexible LLM Integration](#2-flexible-llm-integration)
3. [RAG & Knowledge Management](#3-rag--knowledge-management)

---

## 1. Task Management & Delegation

**Feature Index Section:** `4. Task Management & Delegation` - Sophisticated task definition, coordination, and delegation between agents with output handling.

### 1.1 Task Class Architecture

**Primary File:** `lib/crewai/src/crewai/task.py` (1,317 lines)

The `Task` class is a Pydantic `BaseModel` with extensive validation and execution capabilities:

```python
class Task(BaseModel):
    """Core task definition with 30+ attributes"""
    description: str = Field(description="Description of the actual task.")
    expected_output: str = Field(description="Clear definition of expected output for the task.")
    agent: BaseAgent | None = Field(description="Agent responsible for execution", default=None)
    context: list[Task] | None | _NotSpecified = Field(default=NOT_SPECIFIED)
    async_execution: bool | None = Field(default=False)
    output_json: type[BaseModel] | None = Field(default=None)
    output_pydantic: type[BaseModel] | None = Field(default=None)
    response_model: type[BaseModel] | None = Field(default=None)
    output_file: str | None = Field(default=None)
    tools: list[BaseTool] | None = Field(default_factory=list)
    input_files: dict[str, FileInput] = Field(default_factory=dict)
    guardrail: GuardrailType | None = Field(default=None)
    guardrails: GuardrailsType | None = Field(default=None)
    human_input: bool | None = Field(default=False)
    markdown: bool | None = Field(default=False)
    # ... 20+ more fields
```

### 1.2 Task Execution Modes

Tasks support three execution modes (lines 497-560):

```python
def execute_sync(self, agent, context, tools) -> TaskOutput:
    """Synchronous execution"""

def execute_async(self, agent, context, tools) -> Future[TaskOutput]:
    """Async execution using threading.Future"""
    future: Future[TaskOutput] = Future()
    ctx = contextvars.copy_context()
    threading.Thread(
        daemon=True,
        target=ctx.run,
        args=(self._execute_task_async, agent, context, tools, future),
    ).start()
    return future

async def aexecute_sync(self, agent, context, tools) -> TaskOutput:
    """Native async/await execution"""
```

**Clever Pattern:** Uses `contextvars.copy_context()` to preserve context when spawning threads for async execution.

### 1.3 Delegation Mechanism

**File:** `lib/crewai/src/crewai/tasks/llm_guardrail.py` (103 lines)

The `LLMGuardrail` class enables delegation-based validation:

```python
class LLMGuardrail:
    """Validates task output using a dedicated LLM agent"""
    def _validate_output(self, task_output: TaskOutput) -> LiteAgentOutput:
        agent = Agent(
            role="Guardrail Agent",
            goal="Validate the output of the task",
            backstory="You are an expert at validating task output...",
            llm=self.llm,
        )
        query = f"""Ensure the following task result complies with the given guardrail.
        Task result: {task_output.raw}
        Guardrail: {self.description}
        Your task: Confirm if the Task result complies..."""
        kickoff_result = agent.kickoff(query, response_format=LLMGuardrailResult)
        return kickoff_result
```

**Key insight:** Guardrails create a sub-agent to validate outputs - effectively delegating validation to another LLM call.

### 1.4 Output Handling

**File:** `lib/crewai/src/crewai/tasks/task_output.py` (103 lines)

```python
class TaskOutput(BaseModel):
    """Three output formats supported"""
    raw: str = Field(default="")
    pydantic: BaseModel | None = Field(default=None)
    json_dict: dict[str, Any] | None = Field(default=None)
    output_format: OutputFormat = Field(default=OutputFormat.RAW)

    @property
    def json(self) -> str | None:
        """Returns JSON string if output_format is JSON"""
        if self.output_format != OutputFormat.JSON:
            raise ValueError("Invalid output format requested...")
        return json.dumps(self.json_dict)
```

### 1.5 Conditional Task Execution

**File:** `lib/crewai/src/crewai/tasks/conditional_task.py` (68 lines)

```python
class ConditionalTask(Task):
    """Task that executes based on a condition function"""
    condition: Callable[[TaskOutput], bool] | None = Field(default=None)

    def should_execute(self, context: TaskOutput) -> bool:
        if self.condition is None:
            raise ValueError("No condition function set for conditional task")
        return self.condition(context)

    def get_skipped_task_output(self) -> TaskOutput:
        """Generate empty output for skipped tasks"""
        return TaskOutput(
            description=self.description,
            raw="",
            agent=self.agent.role if self.agent else "",
            output_format=OutputFormat.RAW,
        )
```

### 1.6 Guardrail Retry Logic

The `Task` class implements sophisticated retry logic (lines 1121-1316):

```python
def _invoke_guardrail_function(self, task_output, agent, tools, guardrail, guardrail_index=None):
    max_attempts = self.guardrail_max_retries + 1  # Default 4 attempts
    for attempt in range(max_attempts):
        guardrail_result = process_guardrail(output=task_output, guardrail=guardrail, ...)
        if guardrail_result.success:
            # Update task_output with validated result
            return task_output
        # Retry: Re-execute task with error context
        if attempt >= self.guardrail_max_retries:
            raise Exception(f"Task failed guardrail validation after {self.guardrail_max_retries} retries")
        context = self.i18n.errors("validation_error").format(
            guardrail_result_error=guardrail_result.error,
            task_output=task_output.raw,
        )
        result = agent.execute_task(task=self, context=context, tools=tools)
```

**Validation Approach:** Guards both sync and async execution paths with nearly identical logic.

### 1.7 Security & Path Validation

Output file paths are validated (lines 406-456):

```python
@field_validator("output_file")
@classmethod
def output_file_validation(cls, value: str | None) -> str | None:
    """Security checks for path traversal and shell expansion"""
    if value is None:
        return None
    if ".." in value:
        raise ValueError("Path traversal attempts are not allowed")
    if value.startswith(("~", "$")):
        raise ValueError("Shell expansion characters are not allowed")
    if any(char in value for char in ["|", ">", "<", "&", ";"]):
        raise ValueError("Shell special characters are not allowed")
    # Template variable validation
    if "{" in value:
        for var in template_vars:
            if not var.isidentifier():
                raise ValueError(f"Invalid template variable name: {var}")
```

### 1.8 Error Handling Patterns

**Context Management (lines 682-789):**
```python
def _execute_core(self, agent, context, tools) -> TaskOutput:
    task_id_token = set_current_task_id(str(self.id))
    self._store_input_files()
    try:
        # Execute agent task
        result = agent.execute_task(task=self, context=context, tools=tools)
        # Post-processing, guardrails, callbacks
        ...
    except Exception as e:
        self.end_time = datetime.datetime.now()
        crewai_event_bus.emit(self, TaskFailedEvent(error=str(e), task=self))
        raise e
    finally:
        clear_task_files(self.id)
        reset_current_task_id(task_id_token)
```

### 1.9 Input Interpolation

Tasks support template variable interpolation (lines 882-961):

```python
def interpolate_inputs_and_add_conversation_history(self, inputs: dict[str, Any]) -> None:
    """Interpolate inputs into task description and expected_output"""
    self.description = interpolate_only(self._original_description, inputs=inputs)
    self.expected_output = interpolate_only(self._original_expected_output, inputs=inputs)
    # Also handles conversation history from crew_chat_messages
```

### 1.10 Notable Technical Debt

1. **Duplicate guardrail logic:** Async (`_ainvoke_guardrail_function`) and sync (`_invoke_guardrail_function`) are nearly identical 200-line functions.

2. **Output type mutual exclusion:** `output_json` and `output_pydantic` cannot both be set:
   ```python
   @model_validator(mode="after")
   def check_output(self) -> Self:
       output_types = [self.output_json, self.output_pydantic]
       if len([type for type in output_types if type]) > 1:
           raise PydanticCustomError("output_type", "Only one output type can be set...", {})
   ```

3. **`max_retries` deprecation:** Legacy parameter renamed to `guardrail_max_retries` with warning (lines 486-495).

---

## 2. Flexible LLM Integration

**Feature Index Section:** `5. Flexible LLM Integration` - Connect agents to various LLMs (OpenAI, Ollama, local models) with a unified interface.

### 2.1 LLM Factory Pattern

**Primary File:** `lib/crewai/src/crewai/llm.py` (99KB)

The `LLM` class uses `__new__` as a factory to route to native providers or LiteLLM fallback:

```python
class LLM(BaseLLM):
    def __new__(cls, model: str, is_litellm: bool = False, **kwargs) -> LLM:
        """Factory method routing to native SDK or LiteLLM"""
        explicit_provider = kwargs.get("provider")

        if explicit_provider:
            provider = explicit_provider
            use_native = True
        elif "/" in model:
            prefix, _, model_part = model.partition("/")
            # Map prefix to canonical provider
            provider_mapping = {
                "openai": "openai", "anthropic": "anthropic", "claude": "anthropic",
                "azure": "azure", "google": "gemini", "gemini": "gemini",
                "bedrock": "bedrock", "aws": "bedrock", "ollama": "ollama", ...
            }
            if canonical_provider and cls._validate_model_in_constants(model_part):
                provider = canonical_provider
                use_native = True
            else:
                use_native = False
        else:
            provider = cls._infer_provider_from_model(model)
            use_native = True

        native_class = cls._get_native_provider(provider) if use_native else None
        if native_class and not is_litellm and provider in SUPPORTED_NATIVE_PROVIDERS:
            return native_class(model=model_string, provider=provider, **kwargs_copy)

        # FALLBACK to LiteLLM
        if not LITELLM_AVAILABLE:
            raise ImportError(f"Unable to initialize LLM with model '{model}'...")
        instance = object.__new__(cls)
        super(LLM, instance).__init__(model=model, is_litellm=True, **kwargs)
        return instance
```

### 2.2 Provider Inference

Model-to-provider inference with pattern matching (lines 444-509):

```python
@classmethod
def _matches_provider_pattern(cls, model: str, provider: str) -> bool:
    model_lower = model.lower()
    if provider == "openai":
        return any(model_lower.startswith(p) for p in ["gpt-", "o1", "o3", "o4", "whisper-"])
    if provider == "anthropic" or provider == "claude":
        return any(model_lower.startswith(p) for p in ["claude-", "anthropic."])
    if provider == "gemini" or provider == "google":
        return any(model_lower.startswith(p) for p in ["gemini-", "gemma-", "learnlm-"])
    if provider == "bedrock":
        return "." in model_lower  # AWS format: us.amazon.nova-pro-v1:0
    if provider == "ollama" or provider == "ollama_chat":
        return True  # Ollama accepts any local model name
```

### 2.3 Context Window Sizes

Hardcoded context window sizes for 100+ models (lines 173-298):

```python
LLM_CONTEXT_WINDOW_SIZES: Final[dict[str, int]] = {
    "gpt-4": 8192,
    "gpt-4o": 128000,
    "gpt-4o-mini": 200000,
    "gpt-4.1": 1047576,
    "gemini-1.5-pro": 2097152,
    "claude-3-5-sonnet-v2": 200000,
    # ... 100+ entries
}
```

**Technical Debt:** Manual maintenance required for new models. No automatic fetching from provider APIs.

### 2.4 BaseLLM Interface

**File:** `lib/crewai/src/crewai/llms/base_llm.py` (912 lines)

Abstract base defining the interface:

```python
class BaseLLM(ABC):
    is_litellm: bool = False

    @abstractmethod
    def call(self, messages, tools=None, callbacks=None,
             available_functions=None, from_task=None, from_agent=None,
             response_model=None) -> str | Any:
        """Synchronous LLM call"""

    async def acall(self, messages, ...) -> str | Any:
        """Async LLM call - raises NotImplementedError by default"""
        raise NotImplementedError
```

### 2.5 Native Provider Implementations

**Provider locations:**
- `lib/crewai/src/crewai/llms/providers/openai/completion.py` (92KB)
- `lib/crewai/src/crewai/llms/providers/anthropic/completion.py` (70KB)
- `lib/crewai/src/crewai/llms/providers/gemini/completion.py`
- `lib/crewai/src/crewai/llms/providers/bedrock/completion.py`
- `lib/crewai/src/crewai/llms/providers/azure/completion.py`
- `lib/crewai/src/crewai/llms/providers/openai_compatible/completion.py`

### 2.6 OpenAI Provider Architecture

**File:** `lib/crewai/src/crewai/llms/providers/openai/completion.py`

Supports both Chat Completions API and Responses API:

```python
class OpenAICompletion(BaseLLM):
    """Native OpenAI SDK integration"""
    BUILTIN_TOOL_TYPES: ClassVar[dict[str, str]] = {
        "web_search": "web_search_preview",
        "file_search": "file_search",
        "code_interpreter": "code_interpreter",
        "computer_use": "computer_use_preview",
    }

    def __init__(self, api: str = "completions",  # or "responses"
                 builtin_tools: list[str] | None = None,
                 parse_tool_outputs: bool = False,
                 auto_chain: bool = False,
                 auto_chain_reasoning: bool = False,
                 ...):
        # Supports OpenAI's newer Responses API with built-in tools
```

### 2.7 Streaming Response Handling

**File:** `lib/crewai/src/crewai/llm.py` (lines 778-1082)

Sophisticated streaming with chunk accumulation:

```python
def _handle_streaming_response(self, params, callbacks, available_functions,
                               from_task, from_agent, response_model) -> Any:
    full_response = ""
    accumulated_tool_args: defaultdict[int, AccumulatedToolArgs] = defaultdict(
        AccumulatedToolArgs
    )
    params["stream"] = True
    params["stream_options"] = {"include_usage": True}

    for chunk in litellm.completion(**params):
        # Extract content from various chunk formats
        choices = self._safe_get_choices(chunk)
        if choices:
            delta = choices[0].delta
            if delta.content:
                full_response += delta.content
            # Handle streaming tool calls
            if "tool_calls" in delta:
                self._handle_streaming_tool_calls(...)

        # Emit stream chunk event
        crewai_event_bus.emit(self, LLMStreamChunkEvent(chunk=chunk_content, ...))

    # Fallback to non-streaming if no content received
    if not full_response.strip() and chunk_count == 0:
        return self._handle_non_streaming_response(non_streaming_params, ...)
```

**Clever Pattern:** Safely extracts content from multiple chunk formats using hasattr checks and type guards.

### 2.8 Tool Execution with Event Emission

```python
def _handle_tool_execution(self, function_name, function_args, available_functions,
                          from_task, from_agent) -> str | None:
    """Handle tool execution with event emission"""
    crewai_event_bus.emit(self, ToolUsageStartedEvent(
        tool_name=function_name, tool_args=function_args,
        from_agent=from_agent, from_task=from_task,
    ))
    fn = available_functions[function_name]
    result = fn(**function_args)
    crewai_event_bus.emit(self, ToolUsageFinishedEvent(
        output=result, tool_name=function_name, ...
    ))
    return str(result) if not isinstance(result, str) else result
```

### 2.9 Message Formatting

```python
def _format_messages(self, messages: str | list[LLMMessage]) -> list[LLMMessage]:
    """Convert string or list of dicts to standard format"""
    if isinstance(messages, str):
        return [{"role": "user", "content": messages}]
    for i, msg in enumerate(messages):
        if not isinstance(msg, dict):
            raise ValueError(f"Message at index {i} must be a dictionary")
        if "role" not in msg or "content" not in msg:
            raise ValueError(f"Message at index {i} must have 'role' and 'content'")
    return self._process_message_files(messages)

def _process_message_files(self, messages: list[LLMMessage]) -> list[LLMMessage]:
    """Process files attached to messages for multimodal models"""
    if not self.supports_multimodal():
        if any(msg.get("files") for msg in messages):
            raise ValueError(f"Model '{self.model}' does not support multimodal input...")
    # Format files into provider-specific content blocks
```

### 2.10 Stop Words Handling

Base class provides consistent stop word behavior (lines 295-336):

```python
def _apply_stop_words(self, content: str) -> str:
    """Truncate response at first stop word occurrence"""
    if not self.stop or not content:
        return content
    earliest_stop_pos = len(content)
    for stop_word in self.stop:
        stop_pos = content.find(stop_word)
        if stop_pos != -1 and stop_pos < earliest_stop_pos:
            earliest_stop_pos = stop_pos
            found_stop_word = stop_word
    if found_stop_word:
        return content[:earliest_stop_pos].strip()
    return content
```

### 2.11 Token Usage Tracking

```python
def _track_token_usage_internal(self, usage_data: dict[str, Any]) -> None:
    """Provider-agnostic token usage extraction"""
    prompt_tokens = (usage_data.get("prompt_tokens") or
                     usage_data.get("prompt_token_count") or
                     usage_data.get("input_tokens") or 0)
    completion_tokens = (usage_data.get("completion_tokens") or
                         usage_data.get("candidates_token_count") or
                         usage_data.get("output_tokens") or 0)
    self._token_usage["prompt_tokens"] += prompt_tokens
    self._token_usage["completion_tokens"] += completion_tokens
    self._token_usage["total_tokens"] += prompt_tokens + completion_tokens
```

### 2.12 Hook System for LLM Calls

```python
def _invoke_before_llm_call_hooks(self, messages, from_agent) -> bool:
    """Invoke hooks before LLM call"""
    if from_agent is not None:
        return True  # Only invoke for direct calls
    before_hooks = get_before_llm_call_hooks()
    for hook in before_hooks:
        result = hook(hook_context)
        if result is False:
            return False
    return True
```

### 2.13 Supported Native Providers

```python
SUPPORTED_NATIVE_PROVIDERS: Final[list[str]] = [
    "openai", "anthropic", "claude", "azure", "azure_openai",
    "google", "gemini", "bedrock", "aws",
    # OpenAI-compatible providers
    "openrouter", "deepseek", "ollama", "ollama_chat",
    "hosted_vllm", "cerebras", "dashscope",
]
```

### 2.14 LiteLLM Fallback Pattern

```python
# Suppress LiteLLM debug banners
class FilteredStream(io.TextIOBase):
    """Filter noisy LiteLLM log output"""
    def write(self, s: str) -> int:
        lower_s = s.lower()
        if "litellm.info:" in lower_s:
            return 0
        return self._original_stream.write(s)

# Global application
if not isinstance(sys.stdout, FilteredStream):
    sys.stdout = FilteredStream(sys.stdout)
```

### 2.15 Notable Patterns & Technical Debt

1. **Hardcoded model context windows** - Manual maintenance burden
2. **LiteLLM as fallback** - Heavy dependency (~50+ packages)
3. **Duplicate streaming logic** - Shared between main LLM class and native providers
4. **FilteredStream global side effect** - Modifies sys.stdout globally on import

---

## 3. RAG & Knowledge Management

**Feature Index Section:** `6. RAG (Retrieval-Augmented Generation) & Knowledge Management` - Vector-based knowledge retrieval for agents with multiple storage backends.

### 3.1 Knowledge Class Architecture

**File:** `lib/crewai/src/crewai/knowledge/knowledge.py` (119 lines)

```python
class Knowledge(BaseModel):
    """Knowledge is a collection of sources and setup for the vector store"""
    sources: list[BaseKnowledgeSource] = Field(default_factory=list)
    storage: KnowledgeStorage | None = Field(default=None)
    embedder: EmbedderConfig | None = None
    collection_name: str | None = None

    def __init__(self, collection_name: str, sources: list[BaseKnowledgeSource],
                 embedder: EmbedderConfig | None = None,
                 storage: KnowledgeStorage | None = None, **data):
        super().__init__(**data)
        if storage:
            self.storage = storage
        else:
            self.storage = KnowledgeStorage(embedder=embedder, collection_name=collection_name)
        self.sources = sources

    def query(self, query: list[str], results_limit: int = 5,
              score_threshold: float = 0.6) -> list[SearchResult]:
        return self.storage.search(query, limit=results_limit,
                                    score_threshold=score_threshold)
```

### 3.2 Knowledge Sources

**File:** `lib/crewai/src/crewai/knowledge/source/base_knowledge_source.py` (71 lines)

```python
class BaseKnowledgeSource(BaseModel, ABC):
    """Abstract base for knowledge sources"""
    chunk_size: int = 4000
    chunk_overlap: int = 200
    chunks: list[str] = Field(default_factory=list)
    chunk_embeddings: list[np.ndarray] = Field(default_factory=list)
    storage: KnowledgeStorage | None = Field(default=None)
    metadata: dict[str, Any] = Field(default_factory=dict)

    @abstractmethod
    def validate_content(self) -> Any:
        """Load and preprocess content from the source"""

    @abstractmethod
    def add(self) -> None:
        """Process content, chunk it, compute embeddings, and save"""

    def _chunk_text(self, text: str) -> list[str]:
        """Simple fixed-size chunking with overlap"""
        return [text[i:i+self.chunk_size]
                for i in range(0, len(text), self.chunk_size - self.chunk_overlap)]
```

**Available Source Types:**
- `pdf_knowledge_source.py` - PDF document loading
- `text_file_knowledge_source.py` - Plain text files
- `csv_knowledge_source.py` - CSV data
- `excel_knowledge_source.py` - Excel spreadsheets
- `json_knowledge_source.py` - JSON data
- `string_knowledge_source.py` - Raw string content
- `crew_docling_source.py` - CrewAI-specific docling format

### 3.3 RAG Client Factory

**File:** `lib/crewai/src/crewai/rag/factory.py` (48 lines)

```python
def create_client(config: RagConfigType) -> BaseClient:
    """Factory for creating RAG clients"""
    if config.provider == "chromadb":
        chromadb_mod = require("crewai.rag.chromadb.factory",
                               purpose="The 'chromadb' provider")
        return chromadb_mod.create_client(config)
    if config.provider == "qdrant":
        qdrant_mod = require("crewai.rag.qdrant.factory",
                             purpose="The 'qdrant' provider")
        return qdrant_mod.create_client(config)
    raise ValueError(f"Unsupported provider: {config.provider}")
```

### 3.4 Vector Store Protocol

**File:** `lib/crewai/src/crewai/rag/core/base_client.py` (449 lines)

Uses Python Protocol for duck typing:

```python
@runtime_checkable
class BaseClient(Protocol):
    """Protocol for vector store client implementations"""
    client: Any
    embedding_function: EmbeddingFunction

    @abstractmethod
    def create_collection(self, **kwargs: Unpack[BaseCollectionParams]) -> None: ...
    @abstractmethod
    async def acreate_collection(self, **kwargs: Unpack[BaseCollectionParams]) -> None: ...
    @abstractmethod
    def get_or_create_collection(self, **kwargs) -> Any: ...
    @abstractmethod
    def add_documents(self, **kwargs: Unpack[BaseCollectionAddParams]) -> None: ...
    @abstractmethod
    async def aadd_documents(self, **kwargs) -> None: ...
    @abstractmethod
    def search(self, **kwargs: Unpack[BaseCollectionSearchParams]) -> list[SearchResult]: ...
    @abstractmethod
    async def asearch(self, **kwargs) -> list[SearchResult]: ...
    @abstractmethod
    def delete_collection(self, **kwargs) -> None: ...
    @abstractmethod
    async def adelete_collection(self, **kwargs) -> None: ...
    @abstractmethod
    def reset(self) -> None: ...
    @abstractmethod
    async def areset(self) -> None: ...
```

### 3.5 Knowledge Storage

**File:** `lib/crewai/src/crewai/knowledge/storage/knowledge_storage.py`

```python
class KnowledgeStorage(BaseKnowledgeStorage):
    """Storage for knowledge base with embeddings"""
    def __init__(self, embedder=None, collection_name=None):
        self.collection_name = collection_name
        self._client: BaseClient | None = None
        if embedder:
            embedding_function = build_embedder(embedder)
            config = ChromaDBConfig(embedding_function=embedding_function)
            self._client = create_client(config)

    def search(self, query: list[str], limit=5,
               metadata_filter=None, score_threshold=0.6) -> list[SearchResult]:
        query_text = " ".join(query) if len(query) > 1 else query[0]
        collection_name = f"knowledge_{self.collection_name}" if self.collection_name else "knowledge"
        return client.search(collection_name=collection_name, query=query_text,
                            limit=limit, metadata_filter=metadata_filter,
                            score_threshold=score_threshold)

    def save(self, documents: list[str]) -> None:
        """Upsert documents with auto-generated IDs"""
        collection_name = f"knowledge_{self.collection_name}" if self.collection_name else "knowledge"
        client.get_or_create_collection(collection_name=collection_name)
        rag_documents: list[BaseRecord] = [{"content": doc} for doc in documents]
        client.add_documents(collection_name=collection_name, documents=rag_documents)
```

### 3.6 RAG Types

**File:** `lib/crewai/src/crewai/rag/types.py` (50 lines)

```python
class BaseRecord(TypedDict, total=False):
    """Document record for vector storage"""
    doc_id: str  # Optional - auto-generated from SHA256 hash if missing
    content: Required[str]
    metadata: Mapping[str, str | int | float | bool] | list[Mapping[...]]

class SearchResult(TypedDict):
    """Standard search result format"""
    id: str
    content: str
    metadata: dict[str, Any]
    score: float  # Higher is better, typically 0-1

Embeddings: TypeAlias = list[list[float]]
EmbeddingFunction: TypeAlias = Callable[..., Any]
```

### 3.7 ChromaDB Implementation

**File:** `lib/crewai/src/crewai/rag/chromadb/` (9 files)

ChromaDB factory and client implementation with:
- Collection management
- Document upsert with embeddings
- Similarity search with score threshold
- Metadata filtering

### 3.8 Qdrant Implementation

**File:** `lib/crewai/src/crewai/rag/qdrant/` (9 files)

Qdrant vector database adapter with:
- Named collections
- Batch document operations
- Score threshold filtering
- Async support

### 3.9 Embeddings System

**File:** `lib/crewai/src/crewai/rag/embeddings/`

```python
# Factory pattern for embedding providers
def build_embedder(embedder: ProviderSpec | BaseEmbeddingsProvider) -> EmbeddingFunction:
    """Build embedding function from config or provider"""
    # Supports multiple embedding providers with unified interface
```

### 3.10 Search Result Handling

**File:** `lib/crewai/src/crewai/knowledge/knowledge.py`

Both sync and async search:

```python
async def aquery(self, query: list[str], results_limit=5, score_threshold=0.6):
    """Async query across all knowledge sources"""
    if self.storage is None:
        raise ValueError("Storage is not initialized.")
    return await self.storage.asearch(query, limit=results_limit,
                                       score_threshold=score_threshold)

async def aadd_sources(self) -> None:
    """Add all knowledge sources to storage asynchronously"""
    for source in self.sources:
        source.storage = self.storage
        await source.aadd()
```

### 3.11 Error Handling Patterns

**Dimension Mismatch Detection (knowledge_storage.py lines 120-132):**
```python
try:
    client.add_documents(collection_name=collection_name, documents=rag_documents)
except Exception as e:
    if "dimension mismatch" in str(e).lower():
        Logger(verbose=True).log("error",
            "Embedding dimension mismatch. Try resetting the collection using `crewai reset-memories -a`",
            "red")
        raise ValueError("Embedding dimension mismatch...")
    raise
```

### 3.12 Collection Naming Convention

```python
collection_name = f"knowledge_{self.collection_name}" if self.collection_name else "knowledge"
```

All knowledge collections are prefixed with `knowledge_` to avoid collisions with memory collections.

### 3.13 Notable Patterns & Technical Debt

1. **Simple chunking strategy:** Fixed-size chunks with overlap, no semantic splitting
2. **No document ID preservation:** Custom `doc_id` support exists but rarely used
3. **Embedding provider coupling:** ChromaDB embeds tied to specific config
4. **Global RAG client:** `get_rag_client()` returns singleton - potential thread safety issues

---

## Summary of Key Architectural Patterns

### Task Management
- **Pydantic-first validation** with extensive field validators
- **Three execution modes:** sync, thread-async, native async
- **Guardrail delegation:** Sub-agent validation pattern
- **Retry logic:** Configurable max attempts with error context injection

### LLM Integration
- **Factory pattern via `__new__`:** Dynamic provider routing
- **Native SDK priority with LiteLLM fallback:** Performance vs compatibility
- **Streaming with chunk accumulation:** Handles tool calls in-flight
- **Event-driven telemetry:** All LLM calls emit events

### RAG & Knowledge
- **Protocol-based vector stores:** ChromaDB and Qdrant supported
- **Factory pattern for client creation:** Lazy loading of dependencies
- **Simple chunk-embed-search pipeline:** Each source handles its own processing
- **Collection name isolation:** `knowledge_` prefix separates from memory

---

## File Reference Summary

| Feature | Primary Files | Size |
|---------|--------------|------|
| Task Management | `task.py`, `tasks/llm_guardrail.py`, `tasks/task_output.py`, `tasks/conditional_task.py` | 50KB+ |
| LLM Integration | `llm.py`, `llms/base_llm.py`, `llms/providers/*/completion.py` | 99KB+ |
| RAG/Knowledge | `knowledge/knowledge.py`, `rag/factory.py`, `rag/core/base_client.py`, `knowledge/storage/knowledge_storage.py` | 26KB+ |

---

*Research conducted on crewAI version 1.13.0rc1*

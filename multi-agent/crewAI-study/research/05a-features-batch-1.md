# CrewAI Features Deep Dive: Batch 1

**Repository:** `/Users/sheldon/Documents/claw/reference/crewAI`
**Package:** `lib/crewai/src/crewai/`
**Research Date:** 2026-03-27
**Features Covered:** Multi-Agent Crew Orchestration, Flow-Based Event-Driven Workflows, Autonomous Agent System

---

## Table of Contents

1. [Multi-Agent Crew Orchestration](#1-multi-agent-crew-orchestration)
2. [Flow-Based Event-Driven Workflows](#2-flow-based-event-driven-workflows)
3. [Autonomous Agent System](#3-autonomous-agent-system)
4. [Key Insights and Patterns](#4-key-insights-and-patterns)

---

## 1. Multi-Agent Crew Orchestration

### Overview

Crew orchestration is the core abstraction in CrewAI, enabling multiple agents to collaborate on complex tasks through role-based collaboration and task delegation.

**Primary File:** `lib/crewai/src/crewai/crew.py` (2061 lines)

### Architecture

#### Crew Class (`crew.py`)

The `Crew` class is a Pydantic `BaseModel` that orchestrates agents and tasks:

```python
class Crew(FlowTrackable, BaseModel):
    tasks: list[Task] = Field(default_factory=list)
    agents: list[BaseAgent] = Field(default_factory=list)
    process: Process = Field(default=Process.sequential)
    memory: bool | Memory | MemoryScope | MemorySlice | None = Field(default=False)
    manager_llm: str | InstanceOf[BaseLLM] | None = Field(default=None)
    manager_agent: BaseAgent | None = Field(default=None)
```

**Key Attributes:**
- `tasks`: List of tasks assigned to the crew
- `agents`: Agents participating in the crew
- `process`: Execution mode (`sequential` or `hierarchical`)
- `memory`: Optional unified memory for storing execution context
- `manager_llm`/`manager_agent`: Manager for hierarchical process

#### Execution Processes (`process.py`)

```python
class Process(str, Enum):
    sequential = "sequential"
    hierarchical = "hierarchical"
```

**Sequential Process:**
- Tasks execute in order defined in `self.tasks`
- Each task waits for previous tasks to complete
- Context from prior tasks is aggregated automatically

**Hierarchical Process:**
- A manager agent coordinates task delegation
- Manager uses `DelegateWorkTool` to assign tasks to agents
- Manager can be a custom `manager_agent` or auto-created from `manager_llm`

### Task Delegation

Delegation is implemented via `AgentTools` which provides two tools:

**Location:** `lib/crewai/src/crewai/tools/agent_tools/agent_tools.py`

```python
class AgentTools:
    def tools(self) -> list[BaseTool]:
        coworkers = ", ".join([f"{agent.role}" for agent in self.agents])
        delegate_tool = DelegateWorkTool(agents=self.agents, ...)
        ask_tool = AskQuestionTool(agents=self.agents, ...)
        return [delegate_tool, ask_tool]
```

**Delegation Execution (`base_agent_tools.py`):**

```python
def _execute(self, agent_name: str | None, task: str, context: str | None = None) -> str:
    # Case-insensitive, whitespace-tolerant matching
    sanitized_name = self.sanitize_agent_name(agent_name)
    agent = [a for a in self.agents
             if self.sanitize_agent_name(a.role) == sanitized_name]

    task_with_assigned_agent = Task(
        description=task,
        agent=selected_agent,
        expected_output=selected_agent.i18n.slice("manager_request"),
    )
    return selected_agent.execute_task(task_with_assigned_agent, context)
```

**Key Design: Clever Sanitization**
```python
def sanitize_agent_name(self, name: str) -> str:
    # Normalize whitespace, remove quotes, lowercase
    normalized = " ".join(name.split())
    return normalized.replace('"', "").casefold()
```
This handles LLM JSON output issues where less-powerful models produce malformed JSON with truncated quotes.

### Crew Execution Flow

**Sequential Process (`_run_sequential_process`):**
```python
def _run_sequential_process(self) -> CrewOutput:
    return self._execute_tasks(self.tasks)

def _execute_tasks(self, tasks: list[Task], start_index: int | None = 0, ...) -> CrewOutput:
    for task_index, task in enumerate(tasks):
        exec_data, task_outputs, last_sync_output = prepare_task_execution(...)
        if exec_data.should_skip:
            continue
        context = self._get_context(task, task_outputs)
        task_output = task.execute_sync(agent=exec_data.agent, context=context, tools=exec_data.tools)
        task_outputs.append(task_output)
```

**Context Aggregation:**
```python
@staticmethod
def _get_context(task: Task, task_outputs: list[TaskOutput]) -> str:
    if not task.context:
        return ""
    return aggregate_raw_outputs_from_task_outputs(task_outputs) if task.context is NOT_SPECIFIED \
        else aggregate_raw_outputs_from_tasks(task.context)
```

### Validation Rules

The `Crew` class has extensive Pydantic validators ensuring correct configuration:

```python
@model_validator(mode="after")
def validate_end_with_at_most_one_async_task(self) -> Self:
    # Crew must end with at most one async task
    final_async_task_count = 0
    for task in reversed(self.tasks):
        if task.async_execution:
            final_async_task_count += 1
        else:
            break
    if final_async_task_count > 1:
        raise PydanticCustomError("async_task_count", "...")

@model_validator(mode="after")
def validate_must_have_non_conditional_task(self) -> Crew:
    # At least one non-conditional task required
    non_conditional_count = sum(1 for t in self.tasks if not isinstance(t, ConditionalTask))
    if non_conditional_count == 0:
        raise PydanticCustomError("only_conditional_tasks", "...")

@model_validator(mode="after")
def validate_first_task(self) -> Crew:
    # First task cannot be conditional
    if self.tasks and isinstance(self.tasks[0], ConditionalTask):
        raise PydanticCustomError("invalid_first_task", "...")
```

### Memory Integration

```python
@model_validator(mode="after")
def create_crew_memory(self) -> Crew:
    if self.memory is True:
        embedder = build_embedder(cast(dict[str, Any], self.embedder)) if self.embedder else None
        self._memory = Memory(embedder=embedder, root_scope=f"/crew/{crew_name}")
    elif self.memory:
        self._memory = self.memory  # User passed instance
    return self
```

Memory uses a hierarchical namespace pattern (`/crew/{crew_name}`) to organize memories.

### Async Support

The crew supports multiple async patterns:

```python
async def akickoff(self, inputs=None, input_files=None) -> CrewOutput:
    # Native async task execution throughout
    if self.process == Process.sequential:
        result = await self._arun_sequential_process()
    elif self.process == Process.hierarchical:
        result = await self._arun_hierarchical_process()

async def _arun_sequential_process(self) -> CrewOutput:
    return await self._aexecute_tasks(self.tasks)
```

### Tool Preparation

The crew dynamically assembles tools based on agent capabilities:

```python
def _prepare_tools(self, agent: BaseAgent, task: Task, tools: list[BaseTool]) -> list[BaseTool]:
    # Add delegation tools if agent allows delegation
    if getattr(agent, "allow_delegation", False):
        if self.process == Process.hierarchical:
            tools = self._update_manager_tools(task, tools)
        else:
            tools = self._add_delegation_tools(task, tools)

    # Add code execution tools if allowed
    if getattr(agent, "allow_code_execution", False):
        tools = self._add_code_execution_tools(agent, tools)

    # Add memory tools if available
    resolved_memory = getattr(agent, "memory", None) or self._memory
    if resolved_memory is not None:
        tools = self._add_memory_tools(tools, resolved_memory)
```

### Clever Solutions & Technical Decisions

1. **Task Output Replay:** `replay(task_id)` method allows resuming from a specific task by restoring prior outputs
2. **Input Interpolation:** Placeholders like `{variable}` in task descriptions get replaced at runtime
3. **Soft Reference Matching:** Agent delegation uses fuzzy role matching to handle LLM output variations
4. **Background Memory Saves:** `drain_writes()` ensures async memory operations complete before kickoff returns

### Error Handling

```python
def _check_execution_error(self, e: Exception, task: Task) -> None:
    if e.__class__.__module__.startswith("litellm"):
        raise e  # Don't retry litellm errors
    self._times_executed += 1
    if self._times_executed > self.max_retry_limit:
        crewai_event_bus.emit(AgentExecutionErrorEvent(...))
        raise e
```

---

## 2. Flow-Based Event-Driven Workflows

### Overview

Flows provide a production-ready event-driven architecture for complex automation with state management and conditional branching.

**Primary File:** `lib/crewai/src/crewai/flow/flow.py` (3257 lines)

### Architecture

#### Core Decorators

**`@start()`** - Marks entry points:
```python
def start(condition: str | FlowCondition | Callable[..., Any] | None = None):
    """Marks a method as a flow's starting point."""
    def decorator(func: Callable[P, R]) -> StartMethod[P, R]:
        wrapper = StartMethod(func)
        if condition is not None:
            # Handle trigger conditions
            if is_flow_method_name(condition):
                wrapper.__trigger_methods__ = [condition]
                wrapper.__condition_type__ = OR_CONDITION
            elif is_flow_condition_dict(condition):
                wrapper.__trigger_condition__ = condition
                wrapper.__trigger_methods__ = _extract_all_methods(condition)
        return wrapper
    return decorator
```

**`@listen()`** - Sets up listeners:
```python
def listen(condition: str | FlowCondition | Callable[..., Any]):
    def decorator(func: Callable[P, R]) -> ListenMethod[P, R]:
        wrapper = ListenMethod(func)
        # Similar condition handling to @start
        return wrapper
    return decorator
```

**`@router()`** - Dynamic routing:
```python
def router(condition: str | FlowCondition | Callable[..., Any]):
    # Returns router paths based on source code analysis
    possible_returns = get_possible_return_constants(attr_value)
    router_paths[attr_name] = possible_returns
```

**`@or_()` / `@and_()`** - Compound conditions:
```python
def or_(*conditions: str | FlowCondition | Callable[..., Any]) -> FlowCondition:
    return {"type": OR_CONDITION, "conditions": processed_conditions}

def and_(*conditions: str | FlowCondition | Callable[..., Any]) -> FlowCondition:
    return {"type": AND_CONDITION, "conditions": processed_conditions}
```

### State Management

**FlowState Base:**
```python
class FlowState(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid4()), description="Unique identifier")
```

**StateProxy - Thread-Safe Access:**
```python
class StateProxy(Generic[T]):
    """Proxy for thread-safe access to flow state."""
    def __init__(self, state: T, lock: threading.Lock):
        object.__setattr__(self, "_proxy_state", state)
        object.__setattr__(self, "_proxy_lock", lock)

    def __getattr__(self, name: str) -> Any:
        value = getattr(object.__getattribute__(self, "_proxy_state"), name)
        # Wrap collections in thread-safe proxies
        if isinstance(value, list):
            return LockedListProxy(value, lock)
        if isinstance(value, dict):
            return LockedDictProxy(value, lock)
        return value
```

**Locked Collections for Concurrent Access:**
```python
class LockedListProxy(list, Generic[T]):
    """Thread-safe proxy - all mutations go through lock."""
    def append(self, item: T) -> None:
        with self._lock:
            self._list.append(item)
    # All list operations similarly wrapped

class LockedDictProxy(dict, Generic[T]):
    """Thread-safe proxy - all mutations go through lock."""
    def __setitem__(self, key: str, value: T) -> None:
        with self._lock:
            self._dict[key] = value
    # All dict operations similarly wrapped
```

### FlowMeta - Class-Level Graph Construction

The metaclass builds the execution graph at class definition time:

```python
class FlowMeta(type):
    def __new__(mcs, name: str, bases: tuple[type, ...], namespace: dict[str, Any], **kwargs):
        cls = super().__new__(mcs, name, bases, namespace)

        start_methods = []
        listeners = {}
        router_paths = {}
        routers = set()

        for attr_name, attr_value in namespace.items():
            if hasattr(attr_value, "__is_start_method__"):
                start_methods.append(attr_name)

            if hasattr(attr_value, "__trigger_methods__"):
                methods = attr_value.__trigger_methods__
                condition_type = getattr(attr_value, "__condition_type__", OR_CONDITION)
                listeners[attr_name] = (condition_type, methods)

                if hasattr(attr_value, "__is_router__"):
                    routers.add(attr_name)
                    possible_returns = get_possible_return_constants(attr_value)
                    router_paths[attr_name] = possible_returns

        cls._start_methods = start_methods
        cls._listeners = listeners
        cls._routers = routers
        cls._router_paths = router_paths
        return cls
```

### Execution Flow

**Kickoff:**
```python
async def kickoff_async(self, inputs=None, input_files=None) -> Any | FlowStreamingOutput:
    # Restore from persistence if 'id' provided
    if inputs and "id" in inputs and self._persistence is not None:
        stored_state = self._persistence.load_state(inputs["id"])
        if stored_state:
            self._restore_state(stored_state)

    # Determine start methods to execute
    unconditional_starts = [
        s for s in self._start_methods
        if not getattr(self._methods.get(s), "__trigger_methods__", None)
    ]
    starts_to_execute = unconditional_starts if unconditional_starts else self._start_methods

    # Execute all starts concurrently
    tasks = [self._execute_start_method(s) for s in starts_to_execute]
    await asyncio.gather(*tasks)
```

**Listener Execution with OR Racing:**
```python
async def _execute_racing_listeners(self, racing_listeners: frozenset, other_listeners: list, result: Any):
    """OR listeners - first to complete wins, others cancelled."""
    racing_tasks = [asyncio.create_task(
        self._execute_single_listener(name, result), name=str(name)
    ) for name in racing_listeners]

    # Wait for first to complete
    for coro in asyncio.as_completed(racing_tasks):
        try:
            await coro
        except Exception as e:
            continue
        break  # First one done wins

    # Cancel others
    for task in racing_tasks:
        if not task.done():
            task.cancel()
```

### Human Feedback Integration

Flows support pausing for human feedback:

```python
async def resume_async(self, feedback: str = "") -> Any:
    """Resume flow execution after human feedback."""
    context = self._pending_feedback_context
    emit = context.emit

    # Collapse feedback to outcome using LLM if configured
    if emit and feedback and llm is not None:
        collapsed_outcome = self._collapse_to_outcome(
            feedback=feedback, outcomes=emit, llm=llm
        )
    else:
        collapsed_outcome = feedback or default_outcome or emit[0]

    result = HumanFeedbackResult(output=context.method_output, feedback=feedback, ...)
    self.human_feedback_history.append(result)

    # Trigger downstream listeners
    if emit and collapsed_outcome:
        await self._execute_listeners(FlowMethodName(collapsed_outcome), result)
```

### Source Code Analysis for Routers

**Clever: Static Analysis of Return Values**

The `@router` decorator statically analyzes method source code to determine possible routing paths:

```python
def get_possible_return_constants(func: Callable[..., Any]) -> list[FlowMethodName]:
    """Extract return constants from function source code."""
    source = inspect.getsource(func)
    # Find string literals in return statements
    pattern = r'return\s+["\']([^"\']+)["\']'
    matches = re.findall(pattern, source)
    return [FlowMethodName(m) for m in matches]
```

**Example from `human_feedback.py`:**
```python
@router(method_name)
@human_feedback(emit=["approved", "rejected"])
def review_changes(self) -> str:
    """Review proposed changes."""
    return "approved"  # or "rejected"
```

The source analysis finds `"approved"` and `"rejected"` as possible routing targets.

### Persistence

```python
@classmethod
def from_pending(cls, flow_id: str, persistence: FlowPersistence | None = None) -> Flow[Any]:
    """Restore a flow paused for human feedback."""
    if persistence is None:
        from crewai.flow.persistence import SQLiteFlowPersistence
        persistence = SQLiteFlowPersistence()

    loaded = persistence.load_pending_feedback(flow_id)
    state_data, pending_context = loaded
    instance = cls(persistence=persistence, **kwargs)
    instance._initialize_state(state_data)
    instance._pending_feedback_context = pending_context
    return instance
```

### Flow Visualization

```python
@property
def structure(self) -> dict[str, Any]:
    """Returns the flow structure for visualization."""
    return build_flow_structure(self)
```

The `visualization` module can render flow diagrams showing start methods, listeners, routers, and their connections.

---

## 3. Autonomous Agent System

### Overview

Agents are autonomous units with specialized roles, goals, and the ability to use tools, reason, and delegate tasks.

**Primary Files:**
- `lib/crewai/src/crewai/agent/core.py` (1808 lines) - Full Agent class
- `lib/crewai/src/crewai/lite_agent.py` (1012 lines) - Lightweight agent (deprecated)
- `lib/crewai/src/crewai/agents/crew_agent_executor.py` (2000+ lines) - Execution logic

### Agent Class Architecture

**Core Agent (`agent/core.py`):**

```python
class Agent(BaseAgent):
    role: str = Field(description="Role of the agent")
    goal: str = Field(description="Goal of the agent")
    backstory: str = Field(description="Backstory of the agent")
    llm: str | InstanceOf[BaseLLM] | None = Field(default=None)
    tools: list[BaseTool] = Field(default_factory=list)
    max_iter: int = Field(default=5)
    verbose: bool = Field(default=False)
    allow_delegation: bool = Field(default=False)
    memory: bool | Any | None = Field(default=None)
    planning: bool = Field(default=False)
    reasoning: bool = Field(default=False)  # Deprecated
    guardrail: GuardrailType | None = Field(default=None)
```

**Key Attributes:**
- `role`, `goal`, `backstory`: Agent persona configuration
- `llm`: Language model (auto-created from string)
- `tools`: Available tools for task execution
- `allow_delegation`: Can this agent delegate to others?
- `memory`: Optional memory for context retention
- `planning`: Should agent create a plan before execution?
- `guardrail`: Output validation function

### Execution: CrewAgentExecutor

The executor handles the actual agent reasoning loop with two patterns:

**1. Native Function Calling (Preferred):**
```python
def _invoke_loop_native_tools(self) -> AgentFinish:
    """Use LLM's native function calling capability."""
    openai_tools, available_functions, self._tool_name_mapping = \
        convert_tools_to_openai_schema(self.original_tools)

    while True:
        response = self.llm.call(messages, functions=openai_tools)

        # Parse function calls from response
        for tool_call in response.tool_calls:
            function_name = self._tool_name_mapping[tool_call.function.name]
            tool_result = function_name(tool_call.function.arguments)

            messages.append(tool_result)  # Feedback loop

        if response.finish_reason == "stop":
            return AgentFinish(output=response.content)
```

**2. ReAct Pattern (Fallback):**
```python
def _invoke_loop_react(self) -> AgentFinish:
    """Text-based ReAct pattern when native tools unavailable."""
    while not isinstance(formatted_answer, AgentFinish):
        answer = get_llm_response(self.llm, messages, ...)
        formatted_answer = process_llm_response(answer, self.use_stop_words)

        if isinstance(formatted_answer, AgentAction):
            tool_result = execute_tool_and_check_finality(
                agent_action=formatted_answer,
                tools=self.tools,
                ...
            )
            formatted_answer = handle_agent_action_core(formatted_answer, tool_result)

        self._append_message(formatted_answer.text)
```

### Memory Integration

**Memory Retrieval During Execution:**
```python
def _retrieve_memory_context(self, task: Task, task_prompt: str) -> str:
    """Append relevant memories to task prompt."""
    if not self._is_any_available_memory():
        return task_prompt

    unified_memory = getattr(self, "memory", None) or \
                     getattr(self.crew, "_memory", None) if self.crew else None

    if unified_memory is not None:
        query = task.description
        matches = unified_memory.recall(query, limit=5)
        if matches:
            memory = "Relevant memories:\n" + "\n".join(m.format() for m in matches)
            task_prompt += self.i18n.slice("memory").format(memory=memory)

    return task_prompt
```

**Memory Saving:**
```python
def _save_kickoff_to_memory(self, messages: str | list[LLMMessage], output_text: str) -> None:
    if agent_memory is None:
        return
    input_str = extract_user_message(messages)
    raw = f"Input: {input_str}\nAgent: {self.role}\nResult: {output_text}"
    extracted = agent_memory.extract_memories(raw)
    if extracted:
        agent_memory.remember_many(extracted)
```

### Planning/Reasoning

**Deprecated `reasoning` flag replaced by `planning_config`:**

```python
planning: bool = Field(default=False, description="Whether to create a plan before executing")
planning_config: PlanningConfig | None = Field(default=None)

@property
def planning_enabled(self) -> bool:
    return self.planning_config is not None or self.planning

def _prepare_task_execution(self, task: Task, context: str | None) -> str:
    if self.executor_class is not AgentExecutor:
        handle_reasoning(self, task)  # Creates plan via LLM
    # ... then inject date, build prompt, retrieve memory
```

### Guardrails

**Output Validation with Retry:**
```python
def _process_kickoff_guardrail(self, output: LiteAgentOutput, executor, inputs, retry_count=0):
    guardrail_result = process_guardrail(
        output=output,
        guardrail=self.guardrail,
        retry_count=retry_count,
        from_agent=self,
    )

    if not guardrail_result.success:
        if retry_count >= self.guardrail_max_retries:
            raise ValueError(f"Guardrail failed after {self.guardrail_max_retries} retries")
        # Retry with error message
        executor._append_message_to_state(guardrail_result.error, role="user")
        return self._execute_and_build_output(executor, inputs)
```

### Skills System

Agents can activate skills at multiple disclosure levels:

```python
def set_skills(self, resolved_crew_skills: list[SkillModel] | None = None) -> None:
    """Resolve skill paths and activate skills."""
    crew_skills = self.crew.skills if isinstance(self.crew, Crew) else None

    for item in items:
        if isinstance(item, Path):
            discovered = discover_skills(item, source=self)
            for skill in discovered:
                if skill.name not in seen:
                    seen.add(skill.name)
                    resolved.append(activate_skill(skill, source=self))
        elif isinstance(item, SkillModel):
            if item.disclosure_level >= INSTRUCTIONS:
                # Activate and emit event
```

### LiteAgent (Deprecated)

**Note:** `LiteAgent` is deprecated in favor of `Agent().kickoff()`:

```python
class LiteAgent(FlowTrackable, BaseModel):
    """DEPRECATED: Use Agent().kickoff(messages) instead."""
    warnings.warn(
        "LiteAgent is deprecated and will be removed in a future version. "
        "Use Agent().kickoff(messages) instead.",
        DeprecationWarning,
        stacklevel=2,
    )
```

LiteAgent provides a simpler execution path without full Crew orchestration, but the `Agent` class now supports standalone `kickoff()` for the same use case.

### Agent-to-Agent Communication (A2A)

Agents support the Agent-to-Agent protocol for delegation:

```python
a2a: list[A2AConfig | A2AServerConfig | A2AClientConfig] | None = Field(
    default=None,
    description="A2A configuration for delegating tasks to remote agents"
)
```

A2A allows agents to delegate to remote agents via the `DelegateWorkTool`.

### Timeout Handling

```python
def _execute_with_timeout(self, task_prompt: str, task: Task, timeout: int) -> Any:
    ctx = contextvars.copy_context()
    with concurrent.futures.ThreadPoolExecutor() as executor:
        future = executor.submit(ctx.run, self._execute_without_timeout, task_prompt, task)
        try:
            return future.result(timeout=timeout)
        except concurrent.futures.TimeoutError:
            raise TimeoutError(
                f"Task '{task.description}' execution timed out after {timeout} seconds"
            )
```

### Tool Execution

**Tool Calling with Finality Check:**
```python
def execute_tool_and_check_finality(
    agent_action: AgentAction,
    tools: list[CrewStructuredTool],
    ...
) -> AgentAction:
    """Execute tool and check if result is final."""
    for tool in tools:
        if tool.name == agent_action.tool:
            try:
                result = tool.run(agent_action.tool_input)
                # Check if result indicates completion
                if _is_final_result(result):
                    return AgentFinish(output=result)
                return AgentAction(tool=agent_action.tool, tool_input=agent_action.tool_input)
            except Exception as e:
                return AgentAction(tool=agent_action.tool, tool_input=str(e))
```

---

## 4. Key Insights and Patterns

### Thread-Safety Patterns

1. **LockedListProxy/LockedDictProxy**: All collection mutations go through a lock, preventing race conditions in parallel listener execution
2. **StateProxy**: Provides thread-safe access to flow state with automatic wrapping of nested collections
3. **Copy-on-read for dict states**: State modifications use `dict.update()` rather than in-place mutation

### Memory Hierarchies

- **Crew memory**: `/crew/{crew_name}` namespace
- **Flow memory**: `/flow/{flow_name}` namespace
- **Agent memory**: Can be independent or share crew memory

### Error Handling Philosophy

1. **Litellm errors are fatal**: No retry - these indicate API issues
2. **Other errors get retries**: Up to `max_retry_limit` attempts
3. **Guardrails validate output**: Can retry on guardrail failure
4. **Context length handled specially**: May truncate messages and retry

### Delegation Matching

**Clever Solution - Fuzzy Role Matching:**
```python
def sanitize_agent_name(self, name: str) -> str:
    normalized = " ".join(name.split())  # Normalize whitespace
    return normalized.replace('"', "").casefold()  # Remove quotes, lowercase
```
This handles JSON output issues where LLMs produce malformed strings.

### Event-Driven Architecture

The Flow system uses:
1. **Metaclass for graph construction**: At class definition time, not runtime
2. **Source code analysis for routers**: Determines routing paths statically
3. **OR racing with cancellation**: First listener to complete wins, others cancelled
4. **Async-first design**: All execution is async-capable

### Tool Calling Strategy

**Priority Order:**
1. Native function calling (if supported by LLM)
2. ReAct text-based pattern (fallback)
3. Tool result fed back as message

### Pydantic Validation Heavy Usage

The codebase uses extensive Pydantic validators for:
- Configuration validation at initialization
- Type coercion (strings to LLM instances, etc.)
- Ensuring valid state transitions
- Preventing invalid combinations (e.g., async task at end of sync tasks)

---

## Code Reference Summary

| Feature | Key Files | Lines | Key Classes/Functions |
|---------|-----------|-------|----------------------|
| Crew Orchestration | `crew.py` | 2061 | `Crew`, `Process`, `_execute_tasks` |
| Agent Execution | `agent/core.py` | 1808 | `Agent`, `execute_task`, `_prepare_task_execution` |
| Lite Agent | `lite_agent.py` | 1012 | `LiteAgent`, `_invoke_loop` |
| Crew Executor | `agents/crew_agent_executor.py` | 2000+ | `CrewAgentExecutor`, `_invoke_loop_native_tools`, `_invoke_loop_react` |
| Flow Base | `flow/flow.py` | 3257 | `Flow`, `FlowMeta`, `FlowState`, `StateProxy` |
| Flow Decorators | `flow/flow.py` | 145-430 | `@start`, `@listen`, `@router`, `@or_`, `@and_` |
| Delegation | `tools/agent_tools/` | ~300 | `AgentTools`, `DelegateWorkTool`, `BaseAgentTool` |
| State Proxies | `flow/flow.py` | 433-730 | `LockedListProxy`, `LockedDictProxy`, `StateProxy` |

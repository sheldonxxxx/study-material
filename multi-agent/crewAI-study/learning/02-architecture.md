# crewAI Architecture Analysis

## Overview

crewAI is an AI agent orchestration framework written in Python. It enables users to create crews of autonomous agents that collaborate to accomplish tasks through a flexible system of agents, tasks, and large language models (LLMs).

The architecture follows a **modular monorepo pattern** using `uv` workspace management, with clear separation of concerns across four packages. The system is designed around two complementary orchestration models: a **task-based model** (Crew/Agent/Task) and an **event-driven model** (Flow with decorators).

## Architectural Pattern Classification

crewAI is a **hybrid orchestration framework** combining:

1. **Task-Based Orchestration** (Crew/Agent/Task) - Sequential or hierarchical execution of defined tasks
2. **Event-Driven Architecture** (Flow with @start/@listen/@router decorators) - Reactive flow-based execution
3. **Plugin Architecture** (Tools) - Extensible tool system with 75+ integrations

This hybrid approach allows users to choose between structured task workflows (suitable for well-defined processes) and flexible event-driven flows (suitable for dynamic, reactive systems).

## Package Relationships

```
crewAI Repository (uv workspace root)
│
├── crewAI (Main Framework)
│   ├── Core: Agent, Crew, Task, Flow, LLM classes
│   ├── Execution: Agent executors, task handlers
│   ├── Events: Event bus, event types, listeners
│   ├── Memory: Unified memory with storage backends
│   ├── Knowledge: RAG system with sources and storage
│   └── CLI: Command-line interface and templates
│
├── crewAI-Tools (75+ Tool Integrations)
│   ├── Search: Tavily, Brave, Serper, Exa
│   ├── Scraping: Firecrawl, Selenium, Jina
│   ├── Databases: MySQL, Snowflake, MongoDB
│   ├── File Processing: PDF, CSV, DOCX
│   └── Cloud: S3, Bedrock
│
├── crewAI-Files (File Processing Utilities)
│   └── Format detection, multimodal content handling
│
└── Devtools (Development Tools)
```

**Dependency Direction:**
```
crewAI-Tools ──imports──> crewAI (BaseTool interface)
crewAI-Files ──imports──> crewAI (format_multimodal_content)
Devtools ──────imports──> crewAI
crewAI ────────standalone── No dependencies on other packages
```

## Core Class Relationships

### Agent-Crew-Task-LLM Hierarchy

```
Agent (agent/core.py)
├── role: str
├── goal: str
├── backstory: str
├── llm: LLM (wraps BaseLLM)
├── tools: list[BaseTool]
├── max_iter: int
├── allow_delegation: bool
└── execute_task(task, context) → TaskOutput

Task (task.py)
├── description: str
├── expected_output: str
├── agent: Agent (assigned)
├── context: list[Task] (upstream tasks)
├── async_execution: bool
├── output_file: str
├── guardrails: list[Guardrail]
└── execute(agent, context) → TaskOutput

Crew (crew.py)
├── agents: list[Agent]
├── tasks: list[Task]
├── process: Process (sequential | hierarchical)
├── memory: Memory (optional)
├── knowledge: Knowledge (optional)
├── manager_llm: LLM (for hierarchical mode)
└── kickoff(inputs) → CrewOutput

LLM (llm.py)
├── model: str
├── temperature: float
├── api_key, base_url: str
├── supports_function_calling: bool
└── call(messages) → response

BaseLLM (llms/base_llm.py) [Abstract]
├── is_litellm: bool
├── call(), stream(), structured_call()
└── [OpenAI, Anthropic, Azure, Bedrock, Gemini providers]
```

### Execution Flow

```
User Code
    │
    ▼
Crew.kickoff(inputs)
    │
    ├── Sequential Mode:
    │   │
    │   └── For each Task in order:
    │       ├── Task.execute(agent, context)
    │       │   │
    │       │   └── Agent.execute_task(task, llm, tools)
    │       │       │
    │       │       └── CrewAgentExecutor.execute()
    │       │           │
    │       │           ├── LLM.call() ──> [Provider API]
    │       │           │       │
    │       │           │       └── [LLMCallStartedEvent, LLMCallCompletedEvent]
    │       │           │
    │       │           └── Tool.execute() ──> [Tool implementation]
    │       │                   │
    │       │                   └── [ToolUsageStartedEvent, ToolUsageFinishedEvent]
    │       │
    │       └── TaskOutput
    │
    ├── Hierarchical Mode:
    │   │
    │   └── Manager Agent coordinates task distribution
    │       (Manager is a special Agent with delegation tools)
    │
    └── CrewOutput (aggregates all TaskOutputs)
```

## Flow Event-Driven Model

Flow provides an alternative orchestration model based on event-driven programming:

```
Flow (flow/flow.py)
├── State: FlowState (Pydantic BaseModel with unique id)
├── Methods decorated with @start, @listen, @router
└── kickoff() → FlowOutput

@start() ── Entry point decorator
@listen(method_name) ── Wait for another method's output
@router(condition) ── Conditional routing
```

### Flow Execution Model

```
Flow.kickoff()
    │
    ├── _run_start_methods()
    │   │
    │   └── @start decorated methods execute
    │
    ├── _process_listeners()
    │   │
    │   └── Methods decorated with @listen() wait for triggers
    │       │
    │       └── When trigger method completes, listener executes
    │
    └── _handle_routes()
        │
        └── Methods decorated with @router() determine next path
```

### Flow Integration with Event System

Flow emits its own events and integrates with the central event bus:

```python
# Flow events
FlowStartedEvent, FlowFinishedEvent, FlowPausedEvent
MethodExecutionStartedEvent, MethodExecutionFinishedEvent, MethodExecutionFailedEvent
HumanFeedbackRequestedEvent, HumanFeedbackReceivedEvent

# Events emitted during flow execution
crewai_event_bus.emit(FlowStartedEvent(flow_id=flow.id))
```

## Communication Patterns

### 1. Direct Method Calls (Tight Coupling)

Used within packages for immediate results:
- `Agent.execute_task()` calls `CrewAgentExecutor.execute()`
- `Task.execute()` calls agent methods
- `Crew._execute_tasks()` iterates tasks

### 2. Event Bus (Loose Coupling)

Central `CrewAIEventsBus` singleton for cross-cutting concerns:

```python
# Event emission
crewai_event_bus.emit(LLMCallStartedEvent(call_id=call_id, model=model))
crewai_event_bus.emit(ToolUsageStartedEvent(tool_name=name))

# Event registration
crewai_event_bus.subscribe(LLMCallCompletedEvent, handler_fn)
```

**Event Categories:**
- LLM Events: LLMCallStartedEvent, LLMCallCompletedEvent, LLMStreamChunkEvent
- Tool Events: ToolUsageStartedEvent, ToolUsageFinishedEvent, ToolUsageErrorEvent
- Agent Events: AgentExecutionStartedEvent, AgentExecutionCompletedEvent
- Task Events: TaskStartedEvent, TaskCompletedEvent, TaskFailedEvent
- Flow Events: FlowStartedEvent, MethodExecutionStartedEvent

### 3. Context Variables (Thread-Local State)

For request-scoped data that needs to propagate through call stacks:

```python
# LLM call context
_current_call_id: ContextVar[str | None] = ContextVar("_current_call_id", default=None)

# Flow context
current_flow_id: ContextVar[str | None] = ContextVar("current_flow_id", default=None)
current_flow_request_id: ContextVar[str | None] = ContextVar("current_flow_request_id", default=None)
```

### 4. Callback Functions (Hooks)

For user-defined customization points:

```python
# Step callback - called after each agent step
agent.step_callback = my_callback_fn

# Task callbacks - called on task completion
task.callback = my_task_callback
```

## Extensibility Points

### What is Extensible

| Component | Extension Point | Mechanism |
|-----------|----------------|-----------|
| **LLM Providers** | `BaseLLM` abstract class | Subclass and implement `call()`, `stream()` |
| **Tools** | `BaseTool` abstract class | Implement `_run()` or `_arun()` |
| **Memory Storage** | `StorageBackend` protocol | Implement `save()`, `search()`, `delete()` |
| **Knowledge Sources** | `BaseKnowledgeSource` | Implement `load()`, `add()` |
| **Embedding Providers** | `BaseEmbeddingsProvider` | Configure via dict spec |
| **Event Handlers** | `BaseEventListener` | Implement handler methods |
| **Flow Decorators** | `@start`, `@listen`, `@router` | Method decorators |
| **Agent Executors** | `CrewAgentExecutorMixin` | Extend execution behavior |

### What is Fixed

| Component | Constraint | Reason |
|-----------|------------|--------|
| **Core Class Hierarchy** | Agent > Crew > Task | Fundamental abstraction |
| **Execution Flow** | Sequential or Hierarchical | Process enum defines modes |
| **Event Bus Architecture** | Singleton pattern | System-wide event coordination |
| **Pydantic Models** | All configs are Pydantic | Type safety and serialization |
| **Flow Decorator Semantics** | @start/@listen/@router | Core flow semantics |

## Memory and Knowledge Systems

### Memory (Short-Term Context)

```
Memory (unified_memory.py)
├── LLM-based analysis on save
├── RecallFlow for adaptive-depth retrieval
├── Storage: LanceDB (default), SQLite, Qdrant
└── Scope: Agent, Crew, or Task level
```

### Knowledge (Long-Term RAG)

```
Knowledge (knowledge.py)
├── Sources: PDF, Text, (extensible)
├── Storage: Vector embeddings
└── Query: Semantic search with score threshold
```

## Project Structure Pattern

crewAI uses a decorator-based configuration pattern:

```python
@CrewBase
class MyCrew:
    agents: list[BaseAgent]      # Populated by @agent decorator
    tasks: list[Task]            # Populated by @task decorator

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

The `@agent` and `@task` decorators scan methods, load corresponding YAML configs, and auto-populate `agents` and `tasks` lists. The `@crew` decorator assembles the final crew.

## Observability

### Tracing Integration

OpenTelemetry tracing via `TraceCollectionListener`:

```python
# Automatic tracing of:
# - Crew kickoff
# - Task execution
# - LLM calls
# - Tool usage
```

### Streaming Support

Real-time output via streaming callbacks:

```python
crew = Crew(agents=agents, tasks=tasks)
for chunk in crew.kickoff(stream=True):
    print(chunk)
```

## Architectural Insights

1. **Dual Orchestration Models**: Crew provides structured task workflows; Flow provides reactive event-driven programming. Users choose based on their problem domain.

2. **Centralized Event System**: All significant actions emit events, enabling observability, tracing, and hooks without coupling components directly.

3. **Strategy Pattern for LLMs**: `BaseLLM` abstraction allows any LLM provider while maintaining a consistent interface.

4. **Plugin Architecture for Tools**: Both built-in and external tools follow `BaseTool`, enabling 75+ integrations without framework changes.

5. **Decorator-Based Configuration**: The `@CrewBase`, `@agent`, `@task` pattern reduces boilerplate while keeping configuration in version-controlled YAML.

6. **Memory as a First-Class Citizen**: Unlike simple agent frameworks, crewAI provides sophisticated memory with LLM-based analysis and pluggable storage.

7. **Async-First Design**: Flow is built on asyncio; Crew supports async execution via `async_execution` flag on tasks.

## File Structure

```
lib/crewai/src/crewai/
├── __init__.py              # Exports: Agent, Crew, Flow, Task, LLM, Knowledge, Memory
├── agent/
│   └── core.py              # Agent class (66KB)
├── agents/
│   ├── crew_agent_executor.py    # Main executor (65KB)
│   ├── step_executor.py           # Plan-and-Act executor
│   ├── parser.py                  # Action/Finish parsing
│   └── cache/
├── crew.py                  # Crew orchestration (76KB)
├── task.py                  # Task definition (50KB)
├── llm.py                   # LLM wrapper (99KB)
├── flow/
│   ├── flow.py              # Flow base class (129KB)
│   └── flow_wrappers.py     # @start, @listen, @router
├── llms/
│   ├── base_llm.py          # Abstract LLM base
│   └── providers/           # OpenAI, Anthropic, Azure, etc.
├── tools/
│   ├── base_tool.py        # BaseTool abstract
│   ├── structured_tool.py  # CrewStructuredTool
│   └── [agent_tools, cache_tools, rag]
├── memory/
│   ├── unified_memory.py   # Main memory class
│   └── storage/            # LanceDB, SQLite backends
├── knowledge/
│   ├── knowledge.py        # RAG knowledge
│   └── source/             # PDF, Text sources
├── events/
│   ├── event_bus.py       # Singleton event bus
│   └── types/             # Event definitions
└── cli/
    └── templates/          # Project scaffolding
```

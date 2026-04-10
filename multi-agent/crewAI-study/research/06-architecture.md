# crewAI Architecture Analysis

## Overview

crewAI is a Python monorepo for orchestrating autonomous AI agents. It consists of 4 packages managed via `uv` workspace:

- **crewai** (main framework) - Core Agent, Crew, Flow, Task, LLM classes
- **crewai-tools** - 75+ tool integrations (search, scraping, databases, etc.)
- **crewai-files** - File processing utilities
- **devtools** - Development tools

## Core Class Hierarchy

```
BaseModel (Pydantic)
    |
    +-- Agent (agent/core.py) - Individual AI agent
    +-- Task (task.py) - Work unit assigned to agents
    +-- Crew (crew.py) - Orchestrates agents + tasks
    +-- Flow (flow/flow.py) - Event-driven workflow orchestration
    +-- LLM (llm.py) - LLM configuration wrapper
    +-- BaseLLM (llms/base_llm.py) - Abstract LLM provider base
    +-- BaseTool (tools/base_tool.py) - Abstract tool base
```

## Key Architectural Patterns

### 1. Strategy Pattern - LLM Providers

LLM abstraction via `BaseLLM` allows multiple provider implementations:

```
BaseLLM (abstract)
    |
    +-- litellm integration (via __init__.py conditional import)
    +-- Provider-specific implementations:
        +-- OpenAI provider
        +-- Anthropic provider
        +-- Azure provider
        +-- Bedrock provider
        +-- Gemini provider
        +-- OpenAI-compatible provider
```

The `LLM` class (99KB) wraps `BaseLLM` and provides a high-level interface. Users can pass model name, temperature, API keys, etc.

### 2. Observer Pattern - Event System

The event bus (`events/event_bus.py`) implements Observer pattern:

```python
CrewAIEventsBus (singleton)
    |
    +-- Registers handlers for event types
    +-- Emits events: LLMCallStartedEvent, LLMCallCompletedEvent, ToolUsageStartedEvent, etc.
    +-- Supports sync handlers (ThreadPoolExecutor) and async handlers (asyncio loop)
```

Key event types:
- **LLM Events**: `LLMCallStartedEvent`, `LLMCallCompletedEvent`, `LLMCallFailedEvent`, `LLMStreamChunkEvent`
- **Tool Events**: `ToolUsageStartedEvent`, `ToolUsageFinishedEvent`, `ToolUsageErrorEvent`
- **Agent Events**: `AgentExecutionStartedEvent`, `AgentExecutionCompletedEvent`
- **Task Events**: `TaskStartedEvent`, `TaskCompletedEvent`, `TaskFailedEvent`
- **Flow Events**: `FlowStartedEvent`, `FlowFinishedEvent`, `MethodExecutionStartedEvent`

### 3. Strategy/Plugin Pattern - Tools

Tools use a plugin architecture:

```
BaseTool (abstract base class)
    |
    +-- Built-in tools (crewai/tools/):
    |   +-- AgentTools (delegation, task assignment)
    |   +-- CacheTools
    |   +-- RAG tools
    |   +-- MCP tool wrapper
    |
    +-- crewai-tools package (75+ integrations):
        +-- Search tools (Tavily, Brave, Serper, Exa, etc.)
        +-- Scraping tools (Firecrawl, Selenium, Jina, etc.)
        +-- Database tools (MySQL, Snowflake, MongoDB, etc.)
        +-- File tools (PDF, CSV, DOCX, etc.)
        +-- Cloud tools (S3, Bedrock, etc.)
```

Tool schema is auto-generated from the `_run` or `_arun` method signature using Pydantic's `create_model`.

### 4. Factory Pattern - Agent Executor

The `CrewAgentExecutor` is created via factory-like pattern in `BaseAgent`:

```python
class BaseAgent:
    def create_agent_executor(self, tools: list[BaseTool]) -> CrewAgentExecutor:
        # Constructs executor with LLM, task, crew, agent, tools
```

### 5. Template Method Pattern - Flow Decorators

Flow uses decorators for defining workflow:

```python
@start()                    # Mark as entry point
@listen(method_name)        # Listen for another method's output
@router(condition)         # Conditional routing
```

## Class Relationships and Interactions

### Agent Execution Flow

```
User Code
    |
    v
Crew.kickoff()
    |
    v
Crew._execute_tasks() [sequential/hierarchical]
    |
    v
Task.execute()
    |
    v
Agent.execute_task()
    |
    v
CrewAgentExecutor.execute()
    |
    +---> LLM.call() [via BaseLLM]
    |         |
    |         v
    |     [Provider-specific LLM API]
    |
    +---> Tools.execute() [via BaseTool]
    |         |
    |         v
    |     [Tool implementation]
    |
    v
TaskOutput
```

### Flow Execution Model

```
User Code
    |
    v
Flow.kickoff()
    |
    v
Flow._run_start_methods()
    |
    v
@start decorated methods execute
    |
    v
@listen decorated methods wait for triggers
    |
    v
Method outputs become inputs for listeners
```

### Key Interaction Patterns

1. **Crew -> Agent**: Crew orchestrates agents, assigns tasks
2. **Agent -> LLM**: Agent uses LLM for reasoning and tool calls
3. **Agent -> Tools**: Agent has tools at disposal for actions
4. **Task -> Context**: Tasks can use output from other tasks as context
5. **Crew -> Memory**: Optional memory system stores execution history
6. **Crew -> Knowledge**: Optional RAG knowledge base for retrieval

## Process Types

crewAI supports two execution processes (defined in `process.py`):

```python
class Process(str, Enum):
    sequential = "sequential"    # Tasks execute in order
    hierarchical = "hierarchical" # Manager agent coordinates
```

## Memory System

```
Memory (lazy loaded via __getattr__)
    |
    +-- UnifiedMemory (memory/unified_memory.py)
    |     |
    |     +-- Storage backends: SQLite, LanceDB, etc.
    |     +-- Context management
    |     +-- Recall flow
    |     +-- Encoding flow
    |
    +-- MemoryScope / MemorySlice
```

## Knowledge/RAG System

```
Knowledge
    |
    +-- BaseKnowledgeSource (abstract)
    |     |
    |     +-- PDF source
    |     +-- Text source
    |     +-- [Other source types]
    |
    +-- Vector storage (embeddings)
```

## Data Flow Example: Sequential Process

```
1. Crew.kickoff(inputs={"topic": "AI"})
       |
2. Crew iterates through tasks sequentially
       |
3. For each Task:
   a. Task.execute(agent, context)
          |
   b. Agent.execute_task(task, llm, tools)
          |
   c. CrewAgentExecutor (or StepExecutor) runs:
          |
   d. LLM generates response [LLMCallStartedEvent]
          |
   e. If tool_call in response:
          |
   f. Execute tool [ToolUsageStartedEvent]
          |
   g. Tool returns result to LLM
          |
   h. LLM produces final answer [LLMCallCompletedEvent]
          |
   i. TaskOutput created
          |
4. TaskOutput becomes context for next task
       |
5. CrewOutput aggregates all TaskOutputs
```

## File Structure Summary

```
lib/crewai/src/crewai/
    |-- __init__.py              # Exports: Agent, Crew, Flow, Task, LLM, Knowledge, Memory
    |-- crew.py                  # Crew orchestration (76KB)
    |-- task.py                  # Task definition (50KB)
    |-- llm.py                   # LLM wrapper (99KB)
    |-- lite_agent.py            # LiteAgent variant (38KB)
    |-- agent/
    |   |-- core.py              # Agent class (66KB)
    |   |-- planning_config.py
    |   +-- utils.py
    |-- agents/
    |   |-- crew_agent_executor.py  # Main executor (65KB)
    |   |-- step_executor.py         # Plan-and-Act step executor
    |   |-- parser.py                # Action/Finish parsing
    |   |-- tools_handler.py
    |   |-- planner_observer.py
    |   +-- cache/
    |-- crew.py                  # Crew orchestration
    |-- flow/
    |   |-- flow.py              # Flow base class (129KB)
    |   |-- flow_wrappers.py    # @start, @listen, @router decorators
    |   |-- flow_context.py
    |   |-- persistence/
    |   +-- visualization/
    |-- llms/
    |   |-- base_llm.py          # Abstract LLM base
    |   |-- providers/           # Provider implementations
    |   |   |-- openai/
    |   |   |-- anthropic/
    |   |   |-- azure/
    |   |   |-- bedrock/
    |   |   |-- gemini/
    |   |   +-- openai_compatible/
    |   +-- hooks/
    |-- tools/
    |   |-- base_tool.py        # BaseTool abstract class
    |   |-- structured_tool.py  # CrewStructuredTool
    |   |-- tool_usage.py       # Tool orchestration (42KB)
    |   |-- mcp_tool_wrapper.py # MCP protocol support
    |   |-- agent_tools/
    |   |-- cache_tools/
    |   +-- rag/
    |-- memory/
    |   |-- unified_memory.py
    |   |-- memory_scope.py
    |   |-- recall_flow.py
    |   |-- encoding_flow.py
    |   +-- storage/
    |-- knowledge/
    |   |-- knowledge.py
    |   |-- source/
    |   +-- storage/
    |-- events/
    |   |-- event_bus.py        # Event singleton (26KB)
    |   |-- event_listener.py   # EventListener class (29KB)
    |   |-- base_events.py
    |   |-- types/
    |   |   |-- llm_events.py
    |   |   |-- tool_usage_events.py
    |   |   |-- agent_events.py
    |   |   |-- task_events.py
    |   |   +-- flow_events.py
    |   +-- listeners/
    |       +-- tracing/        # OpenTelemetry integration
    |-- tasks/
    |   |-- task_output.py      # TaskOutput model
    |   |-- conditional_task.py
    |   |-- llm_guardrail.py
    |   +-- hallucination_guardrail.py
    |-- cli/
    |   |-- cli.py              # CLI entry point
    |   |-- create_crew.py
    |   |-- run_crew.py
    |   |-- train_crew.py
    |   +-- templates/          # Project scaffolding
    +-- utilities/
        |-- llm_utils.py
        |-- tool_utils.py
        |-- streaming.py
        |-- planning_handler.py
        |-- reasoning_handler.py
        +-- rpm_controller.py

lib/crewai-tools/src/crewai_tools/
    |-- __init__.py             # Exports 75+ tools
    |-- base_tool.py
    |-- adapters/
    |   |-- mcp_adapter.py
    |   |-- zapier_adapter.py
    |   +-- enterprise_adapter.py
    |-- aws/
    |   |-- bedrock/
    |   +-- s3/
    |-- rag/
    |-- tools/
        |-- file_read_tool/
        |-- file_writer_tool/
        |-- pdf_search_tool/
        |-- selenium_scraping_tool/
        |-- tavily_search_tool/
        |-- brave_search_tool/
        |-- [70+ more tools]
```

## Architectural Insights

1. **Event-Driven Core**: The event system is central, with events for LLM calls, tool usage, agent execution, and task completion. This enables observability, tracing, and hooks.

2. **Dual Execution Models**:
   - **Crew/Agent**: Task-based orchestration with sequential or hierarchical processes
   - **Flow**: Event-driven with decorators (@start, @listen, @router)

3. **LLM Abstraction**: BaseLLM provides a common interface while supporting multiple providers. LiteLLM is used as a backend for many providers.

4. **Tool Plugin Architecture**: Both built-in tools (crewai/tools/) and external tools (crewai-tools/) follow the BaseTool interface, enabling 75+ tool integrations.

5. **Memory and Knowledge**: Optional memory system for crew context and RAG-based knowledge retrieval for agents.

6. **Async Support**: Flow uses async/await extensively. Crew supports async execution via `async_execution` flag on tasks.

7. **Observability**: OpenTelemetry tracing integration via event listeners, plus streaming support for real-time output.

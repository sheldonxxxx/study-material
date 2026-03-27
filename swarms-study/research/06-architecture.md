# Swarms Architecture & Design Patterns Analysis

## Architectural Pattern

**Pattern Identified: Multi-Agent Orchestration Framework with Layered Architecture**

The swarms project implements a **multi-agent orchestration framework** combining several architectural patterns:

1. **Agent-Based Architecture** - Core building blocks are autonomous agents that can execute tasks, use tools, and communicate
2. **Workflow Orchestration** - Various workflow patterns (sequential, concurrent, hierarchical, graph-based) for coordinating agents
3. **Hexagonal/Ports-and-Adapters** - Tool system uses a `BaseTool` abstraction with adapters for different tool types
4. **Event-Driven Elements** - Conversation system and task status tracking using event patterns

---

## Module Map and Responsibilities

### Core Module Structure

```
swarms/
├── structs/          # Core data structures and swarm orchestration (65 files)
│   ├── agent.py      # Base Agent class (252KB)
│   ├── base_swarm.py # Abstract base for multi-agent systems
│   ├── base_structure.py  # Persistence and serialization base
│   ├── conversation.py    # Conversation history management
│   ├── graph_workflow.py  # Graph-based workflow orchestration
│   ├── sequential_workflow.py
│   ├── concurrent_workflow.py
│   ├── groupchat.py       # Multi-agent group chat with @mentions
│   ├── hierarchical_swarm.py
│   └── [55+ specialized swarm types]
├── agents/           # Specialized agent implementations (16 files)
│   ├── react_agent.py
│   ├── openai_assistant.py
│   ├── agent_judge.py
│   └── [others]
├── tools/            # Tool system (20 files)
│   ├── base_tool.py  # Base tool with schema conversion, execution
│   ├── mcp_client_tools.py
│   └── [others]
├── prompts/          # Prompt templates (69 files)
├── schemas/          # Pydantic data validation schemas
├── utils/            # Utility functions
├── cli/              # CLI entry point
└── telemetry/        # Monitoring and observability
```

### Key Modules

| Module | Responsibility | Key Classes |
|--------|----------------|-------------|
| `structs/agent.py` | Core agent execution, LLM integration, tool orchestration | `Agent` |
| `structs/base_swarm.py` | Multi-agent orchestration base | `BaseSwarm` |
| `structs/base_structure.py` | Persistence, serialization, async utilities | `BaseStructure` |
| `structs/conversation.py` | Conversation history, context management | `Conversation` |
| `structs/graph_workflow.py` | DAG-based workflow execution | `GraphWorkflow`, `Node`, `Edge` |
| `tools/base_tool.py` | Tool schema conversion, execution, caching | `BaseTool` |

---

## Component Communication

### Communication Patterns

**1. Direct Method Calls**
- Agents call methods on other agents through swarm orchestration
- `BaseSwarm` provides `run()`, `step()`, `broadcast()`, `direct_message()`
- Sequential/Concurrent workflows chain agent executions via `AgentRearrange`

**2. Tool Execution Flow**
```
Agent.run(task)
  -> LiteLLM.execute() [via litellm wrapper]
  -> Tool schema generation [BaseTool]
  -> Function call parsing [parse_and_execute_json]
  -> Tool execution [BaseTool.execute_tool()]
```

**3. Shared State via Conversation**
- `Conversation` class manages shared conversation history
- Agents append to conversation, read previous turns
- Supports `shared_memory_system` in workflows

**4. Graph-Based Communication (GraphWorkflow)**
- Uses `networkx` for DAG representation
- Nodes are agents/tasks, edges define dependencies
- Topological execution order via `nx.topological_sort()`

**5. Async Execution Patterns**
- `asyncio` for concurrent agent execution
- `ThreadPoolExecutor` for blocking I/O
- `multi_agent_exec.py` provides: `run_agents_concurrently_async()`, `run_agent_async()`

---

## Key Abstractions and Interfaces

### Base Classes

**`BaseStructure`** (`structs/base_structure.py`)
- Persistence: `save_to_file()`, `load_from_file()`, `save_metadata()`, `save_artifact()`
- Async wrappers: `run_async()`, `asave_to_file()`, `aload_from_file()`
- Serialization: `to_dict()`, `to_json()`, `to_yaml()`, `to_toml()`
- Threading: `run_in_thread()`, `run_concurrent()`, `run_batched()`
- Resource monitoring: `monitor_resources()`, `run_with_resources()`

**`BaseSwarm`** (`structs/base_swarm.py`)
- Agent management: `add_agent()`, `remove_agent()`, `get_agent_by_name()`, `get_agent_by_id()`
- Execution: `run()`, `step()`, `loop()`, `batch_run()`, `arun()`
- Communication: `broadcast()`, `direct_message()`, `communicate()`
- Scaling: `scale_up()`, `scale_down()`, `autoscaler()`
- State: `save_swarm_state()`, `load_from_json()`

**`Agent`** (`structs/agent.py`)
- LLM execution via `LiteLLM` wrapper
- Tool management with `BaseTool`
- Conversation integration via `Conversation`
- Autonomy loop utilities
- MCP (Model Context Protocol) support

**`BaseTool`** (`tools/base_tool.py`)
- Schema conversion: `func_to_dict()`, `pydantic_to_openai_schema()`
- Tool execution: `execute_tool()`, `execute_tool_call()`
- Caching: `cache_enabled` with TTL
- Error handling: `ToolValidationError`, `ToolExecutionError`, `ToolNotFoundError`

### Interface Patterns

**`SpeakerFunction`** (`structs/groupchat.py`)
```python
SpeakerFunction = Callable[[List[str], "Agent"], bool]
```
Used for custom speaker selection strategies (random, round-robin, expertise-based).

**`GraphBackend`** (`structs/graph_workflow.py`)
Abstract interface for graph libraries, implemented via `NetworkXBackend` and `RustworkXBackend`.

---

## Design Patterns in Use

### 1. **Strategy Pattern**
- Speaker selection in `GroupChat`: `random_speaker`, `round_robin_speaker`, `priority_speaker`, `expertise_based`
- Stopping conditions: `check_done`, `check_complete`, `check_success`, `check_failure`
- Output types: `"final"`, `"dict"`, `"json"`, etc.

### 2. **Template Method**
- `BaseStructure` provides default implementations with async, threading variants
- `BaseSwarm` defines skeleton methods (`run()`, `step()`) that subclasses override
- `AgentRearrange` in `sequential_workflow.py` provides re-usable agent orchestration logic

### 3. **Decorator/AOP**
- `structs/aop.py` - Aspect-Oriented Programming module (102KB)
- Cross-cutting concerns separated into aspects

### 4. **Registry Pattern**
- `AgentRegistry` (`structs/agent_registry.py`)
- `SubagentRegistry` (`structs/async_subagent.py`)
- Dynamic agent lookup by name/id

### 5. **Factory Pattern**
- `AgentLoader` (`structs/agent_loader.py`) - Creates agents from configs
- `AutoSwarmBuilder` (`structs/auto_swarm_builder.py`) - Auto-generates swarm configurations
- `create_agents_from_yaml.py` - YAML-based agent creation

### 6. **Builder Pattern**
- `SpreadSheetSwarm`, `MixtureOfAgents`, `SkillOrchestra` - Fluent interfaces for swarm construction

### 7. **Observer/Event**
- Task callbacks in `BaseSwarm.__init__`: `callbacks: Sequence[callable]`
- Task status updates via `TaskStatus` enum in `async_subagent.py`

### 8. **Promise/Future**
- Async execution: `run_agent_async()`, `run_agents_concurrently_async()`
- ThreadPoolExecutor for non-blocking execution

### 9. **Hexagonal Architecture (Tools)**
- `BaseTool` as the port/adapter abstraction
- Converts Python functions, Pydantic models, dicts to OpenAI function schemas
- `mcp_client_tools.py` provides MCP server adapter

---

## Notable Architectural Decisions

### 1. **LiteLLM as LLM Abstraction**
- Uses `litellm` library (not vendor-specific SDKs)
- Unified interface across 50+ LLM providers
- Enables `model_router.py` for dynamic model selection

### 2. **NetworkX for Graph Workflows**
- Graph-based orchestration via `graph_workflow.py`
- Supports multiple backends: NetworkX (Python), RustworkX (Rust)
- Enables complex DAG execution with conditional branching

### 3. **Conversation as First-Class Citizen**
- `Conversation` class handles context window management
- Token counting via `count_tokens()` from litellm
- Dynamic context window truncation

### 4. **Prompts as Data**
- 69 prompt templates in `prompts/`
- Enables runtime prompt modification
- Prompt marketplace integration via `fetch_prompts_from_marketplace()`

### 5. **Tool Schema Generation**
- Automatic OpenAI function schema generation from Python callables
- Type hint introspection for validation
- Support for Pydantic models as tool inputs/outputs

### 6. **Async-First Design**
- `asyncio` for concurrent operations
- `ThreadPoolExecutor` fallback for sync code
- `uvloop` support for high-performance event loops

### 7. **MCP Integration**
- Model Context Protocol support via `mcp_client_tools.py`
- Multiple concurrent MCP connections
- Tool schema generation from MCP servers

### 8. **Persistence Layers**
- Multiple formats: JSON, YAML, TOML
- Autosave capabilities
- Workspace-based file organization

---

## File References for Key Patterns

| Pattern | Location |
|---------|----------|
| Strategy (speakers) | `structs/groupchat.py:20-100` |
| Strategy (stopping) | `structs/stopping_conditions.py` |
| Template Method | `structs/base_swarm.py:191-214` |
| AOP | `structs/aop.py` |
| Registry | `structs/agent_registry.py` |
| Factory | `structs/auto_swarm_builder.py` |
| Graph Backend | `structs/graph_workflow.py:46-150` |
| Tool System | `tools/base_tool.py:69+` |
| Async Execution | `structs/multi_agent_exec.py` |

---

## Summary

The swarms architecture is a **pragmatic multi-agent orchestration framework** that prioritizes:

1. **Flexibility** - Multiple swarm types and workflow patterns
2. **Extensibility** - Base classes for agents, tools, and swarms
3. **Observability** - Logging, telemetry, conversation history
4. **Performance** - Async execution, parallel agent runs, graph backends
5. **Interoperability** - LiteLLM abstraction, MCP support, multiple LLM providers

The design shows organic growth with 65+ files in `structs/` alone, suggesting rapid iteration on different multi-agent patterns. The separation between `Agent` (single agent), `BaseSwarm` (multi-agent orchestration), and specialized swarm types (hierarchical, graph, sequential, etc.) provides clear boundaries for different use cases.

# Swarms Architecture

## Overview

Swarms is a **multi-agent orchestration framework** that combines agent-based architecture with workflow orchestration patterns. The framework enables multiple autonomous agents to collaborate on complex tasks through various coordination mechanisms.

## Architectural Pattern

The architecture follows a **layered multi-pattern approach**:

```
┌─────────────────────────────────────────────────────────────┐
│                      CLI Entry Point                        │
│                    (swarms.cli.main)                        │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                   Orchestration Layer                       │
│  BaseSwarm, GraphWorkflow, SequentialWorkflow, Concurrent    │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                     Agent Layer                             │
│     Agent, ReActAgent, OpenAIAssistant, AgentJudge         │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    Tool System Layer                        │
│         BaseTool, MCP Client, Function Calling              │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                   LLM Abstraction Layer                    │
│                      LiteLLM                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Module Responsibilities

### Core Modules

| Module | Files | Responsibility |
|--------|-------|----------------|
| `swarms/structs/` | 65 | Core data structures, swarm orchestration, Agent base class |
| `swarms/agents/` | 16 | Specialized agent implementations |
| `swarms/tools/` | 20 | Tool system with schema conversion and execution |
| `swarms/prompts/` | 69 | Prompt templates for various swarm types |
| `swarms/schemas/` | 13 | Pydantic data validation schemas |
| `swarms/utils/` | 29 | Utility functions |
| `swarms/cli/` | 2 | CLI entry point implementation |
| `swarms/telemetry/` | - | Monitoring and observability |

### Key Classes and Their Roles

#### Agent Layer

| Class | File | Role |
|-------|------|------|
| `Agent` | `structs/agent.py` (252KB) | Base agent with LLM integration, tool orchestration, conversation management |
| `ReActAgent` | `agents/react_agent.py` | ReAct pattern implementation (Reasoning + Acting) |
| `OpenAIAssistant` | `agents/openai_assistant.py` | OpenAI Assistant API integration |
| `AgentJudge` | `agents/agent_judge.py` | Evaluation and selection agent |

#### Orchestration Layer

| Class | File | Role |
|-------|------|------|
| `BaseSwarm` | `structs/base_swarm.py` | Abstract base for multi-agent systems |
| `BaseStructure` | `structs/base_structure.py` | Persistence, serialization, async utilities |
| `Conversation` | `structs/conversation.py` | Conversation history and context management |
| `GraphWorkflow` | `structs/graph_workflow.py` | DAG-based workflow orchestration |
| `AgentRearrange` | `structs/sequential_workflow.py` | Sequential agent orchestration |
| `ConcurrentWorkflow` | `structs/concurrent_workflow.py` | Parallel agent execution |
| `GroupChat` | `structs/groupchat.py` | Multi-agent group chat with @mentions |
| `HierarchicalSwarm` | `structs/hierarchical_swarm.py` | Hierarchical communication framework |

#### Tool Layer

| Class | File | Role |
|-------|------|------|
| `BaseTool` | `tools/base_tool.py` (107KB) | Tool schema conversion, execution, caching |
| `MCPTool` | `tools/mcp_client_tools.py` | Model Context Protocol client |

---

## Communication Patterns

### 1. Direct Method Calls

Agents communicate through swarm orchestration methods:

```python
# From BaseSwarm
def run(self, task: str) -> str
def step(self, agent: Agent, task: str) -> str
def broadcast(self, message: str)
def direct_message(self, from_agent: str, to_agent: str, message: str)
```

### 2. Tool Execution Flow

```
Agent.run(task)
    │
    ▼
LiteLLM.execute() ── Unified LLM interface across 50+ providers
    │
    ▼
BaseTool.func_to_dict() ── Schema generation from Python functions
    │
    ▼
parse_and_execute_json() ── Function call parsing
    │
    ▼
BaseTool.execute_tool() ── Tool execution with caching
```

### 3. Shared State via Conversation

```python
class Conversation:
    def add_message(self, role: str, content: str)
    def get_history(self) -> List[Message]
    def count_tokens(self) -> int  # Context window management
    def truncate(self, max_tokens: int)  # Dynamic truncation
```

### 4. Graph-Based Communication

```python
# GraphWorkflow uses networkx for DAG representation
GraphWorkflow.add_node(agent: Agent)
GraphWorkflow.add_edge(from_agent: str, to_agent: str)
execution_order = list(nx.topological_sort(graph))
```

### 5. Async Execution

```python
# From multi_agent_exec.py
async def run_agents_concurrently_async(agents: List[Agent])
async def run_agent_async(agent: Agent, task: str) -> str
```

---

## Architectural Decisions

### 1. LiteLLM for LLM Abstraction

**Decision:** Use `litellm` library instead of vendor-specific SDKs

**Rationale:**
- Unified interface across 50+ LLM providers (OpenAI, Anthropic, Azure, etc.)
- Enables `model_router.py` for dynamic model selection
- Consistent API regardless of underlying provider

**Trade-offs:**
- Additional dependency
- Potential latency overhead from abstraction layer

### 2. NetworkX for Graph Workflows

**Decision:** Graph-based orchestration via NetworkX with RustworkX backend option

**Rationale:**
- DAG representation for complex task dependencies
- Conditional branching support
- Multiple backend options (Python NetworkX, Rust RustworkX)

### 3. Conversation as First-Class Citizen

**Decision:** Explicit Conversation class for context management

**Rationale:**
- Token counting via `count_tokens()` from LiteLLM
- Dynamic context window truncation
- Shared memory across agents

### 4. Prompts as Data

**Decision:** 69 prompt templates stored as code, not strings

**Rationale:**
- Runtime prompt modification
- Marketplace integration via `fetch_prompts_from_marketplace()`
- Structured prompt management

### 5. Hexagonal Architecture for Tools

**Decision:** `BaseTool` as port/adapter abstraction

**Rationale:**
- Consistent interface for any tool type
- Automatic OpenAI function schema generation
- MCP server adapter support

### 6. Async-First Design

**Decision:** `asyncio` as primary concurrency model

**Rationale:**
- High-performance concurrent agent execution
- `uvloop` support for enhanced performance
- `ThreadPoolExecutor` fallback for sync code

---

## File Organization

```
swarms/
├── __init__.py          # Package entry - loads env, telemetry, imports all
├── env.py               # Environment configuration
├── structs/             # Core structures
│   ├── agent.py         # 252KB - Base Agent class
│   ├── base_swarm.py    # Multi-agent orchestration base
│   ├── base_structure.py # Persistence and serialization
│   ├── conversation.py  # Conversation history
│   ├── graph_workflow.py # 111KB - Graph-based workflows
│   ├── aop.py           # 102KB - Aspect-oriented programming
│   └── [60+ more]       # Specialized swarm types
├── agents/              # 16 specialized agent implementations
├── tools/               # 20 tool implementations
│   └── base_tool.py     # 107KB - Tool base class
├── prompts/             # 69 prompt templates
├── schemas/             # 13 Pydantic schemas
├── utils/               # 29 utility functions
├── cli/                 # CLI implementation
│   └── main.py          # 57KB - CLI entry point
└── telemetry/           # Observability
```

---

## Entry Points

### Package Entry Point (`swarms/__init__.py`)

```python
from swarms.env import load_swarms_env
load_swarms_env()

from swarms.telemetry.bootup import bootup
bootup()

from swarms.agents import *
from swarms.artifacts import *
from swarms.prompts import *
from swarms.schemas import *
from swarms.structs import *
from swarms.telemetry import *
from swarms.tools import *
from swarms.utils import *
```

### CLI Entry Point

```toml
# pyproject.toml
[tool.poetry.scripts]
swarms = "swarms.cli.main:main"
```

---

## Key Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `litellm` | 1.76.1 | Unified LLM interface |
| `networkx` | - | Graph-based workflows |
| `pydantic` | - | Data validation |
| `rich` | - | Terminal formatting |
| `asyncio`, `aiohttp`, `httpx` | - | Async operations |
| `mcp` | - | Model Context Protocol |
| `loguru` | - | Logging |

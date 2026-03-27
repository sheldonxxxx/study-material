# Feature Deep Dive - Batch 3 (Features 7-9)

## Repository Path
`/Users/sheldon/Documents/claw/reference/swarms`

## Features Analyzed
- Feature 7: SwarmRouter
- Feature 8: Agent Orchestration Protocol (AOP)
- Feature 9: Prompt Management

---

## Feature 7: SwarmRouter

**Description:** Universal orchestrator providing a single interface to run any swarm type with dynamic selection.

**Key File:** `swarms/structs/swarm_router.py` (1017 lines)

### Architecture Overview

SwarmRouter is a **factory-pattern-based dispatcher** that routes tasks to different swarm implementations based on a `swarm_type` parameter. It provides O(1) lookup performance through a pre-initialized factory dictionary.

### Core Components

#### 1. SwarmFactory Pattern (Lines 456-479)
```python
def _initialize_swarm_factory(self) -> Dict[str, Callable]:
    return {
        "HeavySwarm": self._create_heavy_swarm,
        "AgentRearrange": self._create_agent_rearrange,
        "CouncilAsAJudge": self._create_council_as_judge,
        "HierarchicalSwarm": self._create_hierarchical_swarm,
        "MixtureOfAgents": self._create_mixture_of_agents,
        "MajorityVoting": self._create_majority_voting,
        "GroupChat": self._create_group_chat,
        "MultiAgentRouter": self._create_multi_agent_router,
        "SequentialWorkflow": self._create_sequential_workflow,
        "ConcurrentWorkflow": self._create_concurrent_workflow,
        "BatchedGridWorkflow": self._create_batched_grid_workflow,
        "LLMCouncil": self._create_llm_council,
        "DebateWithJudge": self._create_debate_with_judge,
        "RoundRobin": self._create_round_robin_swarm,
        "PlannerWorkerSwarm": self._create_planner_worker_swarm,
    }
```

**Supported Swarm Types (16 total):**
- `SequentialWorkflow` (default)
- `ConcurrentWorkflow`
- `MixtureOfAgents`
- `AgentRearrange`
- `HierarchicalSwarm`
- `GroupChat`
- `MultiAgentRouter`
- `HeavySwarm`
- `CouncilAsAJudge`
- `DebateWithJudge`
- `MajorityVoting`
- `LLMCouncil`
- `BatchedGridWorkflow`
- `RoundRobin`
- `PlannerWorkerSwarm`
- `AutoSwarmBuilder` (reference only, not in factory)

#### 2. SwarmCache (Lines 276-277, 676-682)
Implements caching for created swarms to avoid recreation:
```python
self._swarm_cache = {}
cache_key = f"{self.swarm_type}_{hash(str(args) + str(kwargs))}"
```

### Configuration Model

**SwarmRouterConfig (Pydantic BaseModel):**
- `name`: Identifier for the SwarmRouter instance
- `description`: Purpose description
- `swarm_type`: Type of swarm (Literal type with 16 options)
- `rearrange_flow`: Flow configuration string (required for AgentRearrange)
- `rules`: Rules to inject into every agent
- `multi_agent_collab_prompt`: Enable multi-agent collaboration prompts
- `task`: The task to be executed

### Key Methods

| Method | Purpose |
|--------|---------|
| `run(task, img, tasks)` | Primary execution method |
| `batch_run(tasks)` | Sequential batch execution |
| `concurrent_run(task)` | ThreadPoolExecutor-based concurrent execution |
| `async_run(task)` | Async execution support |
| `_create_swarm(task)` | Factory-based swarm instantiation with caching |

### Execution Flow

1. User calls `run(task)` with a `swarm_type`
2. `reliability_check()` validates configuration
3. `setup()` applies hooks (APE, shared memory, rules, collaboration prompts)
4. `_create_swarm()` looks up factory, creates swarm, caches it
5. Swarm's `run()` method is invoked

### Setup Hooks (Lines 385-401)

```python
def setup(self):
    if self.auto_generate_prompts is True:
        self.activate_ape()  # Automatic prompt engineering

    if self.shared_memory_system is not None:
        self.activate_shared_memory()

    if self.rules is not None:
        self.handle_rules()  # Injects rules into agent system prompts

    if self.multi_agent_collab_prompt is True:
        self.update_system_prompt_for_agent_in_swarm()
        # Appends MULTI_AGENT_COLLAB_PROMPT_TWO to each agent

    if self.list_all_agents is True:
        self.list_agents_to_eachother()
```

### Error Handling

**Custom Exceptions:**
- `SwarmRouterRunError`: Task execution failures
- `SwarmRouterConfigError`: Configuration validation failures

**Validation (reliability_check, Lines 320-383):**
- Ensures `swarm_type` is not None
- Validates `swarm_type` is a string in valid options
- Requires `rearrange_flow` when using `AgentRearrange`
- Rejects `max_loops == 0`

### Clever Solutions

1. **O(1) Factory Lookup**: Dictionary-based dispatch avoids long if/elif chains
2. **Swarm Caching**: Reuses created swarms to avoid recreation overhead
3. **Autosave Integration**: Optional persistence of configuration and execution metadata
4. **Multi-Agent Collaboration Prompt**: Automatically injects coordination protocols into agent system prompts

### Technical Debt / Issues

1. **Deprecated Parameters**: Many parameters in `__init__` are passed through but unused by certain swarm types (e.g., `speaker_function` for most swarms)
2. **Inconsistent Agent Requirements**: Some swarms require agents (e.g., DebateWithJudge needs exactly 3), but this isn't validated
3. **Missing AutoSwarmBuilder**: Listed in SwarmType but not in factory
4. **Duplicate Parameter Assignment**: `self.autosave` assigned twice (lines 200 and 271)
5. **Circular Reference**: Imports from `swarms.structs.*` modules which may have circular dependencies

### Validation & Edge Cases

- Empty task handling: Delegates to individual swarm implementations
- Queue overflow: Not handled (batch_run loops indefinitely)
- Agent None check: Only in AOP, not in SwarmRouter

---

## Feature 8: Agent Orchestration Protocol (AOP)

**Description:** Framework for deploying and managing agents as distributed services with discovery and management.

**Key File:** `swarms/structs/aop.py` (2949 lines)

### Architecture Overview

AOP transforms swarms agents into **MCP (Model Context Protocol) tools**, deploying them as a cluster of services with built-in task queuing, retry logic, and network resilience.

### Core Components

#### 1. TaskQueue (Lines 103-526)

Thread-safe priority queue for managing agent task execution:

```python
class TaskQueue:
    def __init__(self, agent_name, agent, max_workers=1,
                 max_queue_size=1000, processing_timeout=30,
                 retry_delay=1.0, verbose=False)
```

**Features:**
- Priority-based task scheduling (higher priority = first)
- Worker thread pool for parallel processing
- Automatic retry with configurable delay
- Task status tracking (PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED)
- Statistics collection (average processing time, completion rate)

**Task Dataclass (Lines 45-76):**
```python
@dataclass
class Task:
    task_id: str
    task: str = ""
    img: Optional[str] = None
    imgs: Optional[List[str]] = None
    correct_answer: Optional[str] = None
    priority: int = 0
    status: TaskStatus = TaskStatus.PENDING
    result: Optional[str] = None
    error: Optional[str] = None
    retry_count: int = 0
    max_retries: int = 3
```

#### 2. AOP Main Class (Lines 554-2882)

**Initialization:**
```python
class AOP:
    def __init__(self,
        server_name="AOP Cluster",
        description="A cluster that enables you to deploy multiple agents as tools...",
        agents=None,
        port=8000,
        transport="streamable-http",
        queue_enabled=True,
        persistence=False,
        max_restart_attempts=10,
        network_monitoring=True,
        ...)
```

**Key Attributes:**
- `mcp_server`: FastMCP server instance
- `agents`: Dict[str, Agent] - Registered agents by tool name
- `tool_configs`: Dict[str, AgentToolConfig] - Per-tool configuration
- `task_queues`: Dict[str, TaskQueue] - Per-agent task queues

### Agent Registration Flow

1. `add_agent()` creates AgentToolConfig with input/output schemas
2. If queue_enabled, creates TaskQueue with worker threads
3. `_register_tool()` decorates agent as MCP tool
4. `_register_agent_discovery_tool()` refreshes agent discovery endpoints

**Default Input Schema (Lines 786-808):**
```python
input_schema = {
    "type": "object",
    "properties": {
        "task": {"type": "string", "description": "The task or prompt..."},
        "img": {"type": "string", "description": "Optional image..."},
        "imgs": {"type": "array", "items": {"type": "string"}, ...},
        "correct_answer": {"type": "string", ...}
    },
    "required": ["task"]
}
```

### Queue-Based Execution

When queue_enabled, tool invocation:
1. Adds task to TaskQueue with priority
2. Returns task_id immediately (if wait_for_completion=False)
3. Worker loop processes task asynchronously
4. Polls for completion with timeout

**Execution Methods:**
- `_execute_with_queue()`: Adds task to queue
- `_wait_for_task_completion()`: Polls for task result
- `_execute_agent_with_timeout()`: Direct agent execution with timeout

### Network Resilience

**Persistence Mode (Lines 2416-2547):**
- Automatic restart on shutdown (up to max_restart_attempts)
- Failsafe protection against infinite restart loops
- Network error detection and retry logic

**Network Error Detection (Lines 2549-2595):**
```python
def _is_network_error(self, error: Exception) -> bool:
    network_errors = (ConnectionError, ConnectionRefusedError,
                      ConnectionResetError, TimeoutError, socket.gaierror, ...)
    if isinstance(error, network_errors):
        return True
    # Also checks error message for network keywords
```

**Network Retry Flow:**
1. Detects network error
2. Attempts reconnection up to max_network_retries
3. Tests connectivity before retry
4. Resets retry count on success

### Agent Discovery Tools

AOP registers several MCP tools for agent introspection:

| Tool | Purpose |
|------|---------|
| `discover_agents` | List all agents or get specific agent info |
| `get_agent_details` | Detailed single agent information |
| `get_agents_info` | Batch agent information lookup |
| `list_agents` | Simple agent name list |
| `search_agents` | Keyword search across agents |
| `get_server_info` | Comprehensive server metadata |

**Discovery Info Structure (Lines 2282-2343):**
```python
{
    "tool_name": str,
    "agent_name": str,
    "description": str,
    "short_system_prompt": str,  # Truncated to 200 chars
    "tags": List[str],
    "capabilities": List[str],
    "role": str,
    "model_name": str,
    "max_loops": int,
    "temperature": float,
    "max_tokens": int
}
```

### Queue Management Tools

When queue_enabled, additional tools are registered:

| Tool | Purpose |
|------|---------|
| `get_queue_stats` | Per-agent or aggregate queue statistics |
| `pause_agent_queue` | Pause specific agent queue |
| `resume_agent_queue` | Resume specific agent queue |
| `clear_agent_queue` | Clear pending tasks |
| `get_task_status` | Check individual task status |
| `cancel_task` | Cancel a queued task |
| `pause_all_queues` | Pause all agent queues |
| `resume_all_queues` | Resume all queues |
| `clear_all_queues` | Clear all queues |

### AOPCluster (Lines 2884-2949)

Manages multiple MCP servers for distributed deployment:
```python
class AOPCluster:
    def __init__(self, urls: List[str], transport="streamable-http")
    def get_tools(self, output_type="dict") -> List[Dict[str, Any]]
    def find_tool_by_server_name(self, server_name: str) -> Dict[str, Any]
```

### Clever Solutions

1. **Tool-Agent Duality**: Agents become MCP tools seamlessly
2. **Priority Queue**: Business-critical tasks can jump ahead
3. **Worker Thread Pool**: Per-agent concurrency without blocking
4. **Automatic Retry**: Transient failures don't lose tasks
5. **Discovery Protocol**: Agents can introspect the cluster
6. **Network Resilience**: Automatic reconnection with exponential backoff

### Technical Debt / Issues

1. **Synchronous Threading in Async Context**: Uses `threading.RLock()` in async methods, potential deadlock
2. **Task Polling**: `_wait_for_task_completion` uses sleep polling (line 1276), inefficient
3. **Agent Type Hint**: Uses `AgentType` (from omni_agent_types) but stores as `Agent` (line 678)
4. **FastMCP Import**: From `mcp.server.fastmcp` which may have version compatibility issues
5. **Error Swallowing**: Many catch blocks silently return without full error context
6. **No Graceful Degradation**: If one agent fails, discovery tools may return partial results

### Validation & Edge Cases

- Empty task: Returns `{"success": False, "error": "No task provided"}`
- Queue full: Raises `ValueError`
- Task timeout: Returns error with timeout message
- Agent not found: Returns None for get operations, error for mutations
- Network retry exhaustion: Fails permanently with clear error message

---

## Feature 9: Prompt Management

**Description:** Large library of prompt templates for various agent specializations and tasks.

**Key Directory:** `swarms/prompts/` (69 modules)

### Organization Structure

The prompts directory is organized by **domain/agent type**:

| Category | Files | Purpose |
|----------|-------|---------|
| Agent-specific | `agent_prompts.py`, `agent_judge_prompt.py`, `agent_system_prompts.py` | Base agent prompts |
| Swarm patterns | `moa_prompt.py`, `multi_agent_collab_prompt.py`, `agent_orchestration_prompt.py` | Multi-agent orchestration |
| Domain-specific | `finance_agent_prompt.py`, `legal_agent_prompt.py`, `sales_prompts.py` | Industry vertical prompts |
| Functional | `react.py`, `reasoning_prompt.py`, `code_interpreter.py` | Capability-specific |
| Generator/Builder | `prompt_generator.py`, `autonomous_agent_prompt.py`, `agent_self_builder_prompt.py` | Prompt creation utilities |

### Prompt Class Architecture

**Base Prompt Class (`swarms/prompts/prompt.py`, Lines 21-274):**

```python
class Prompt(BaseModel):
    id: str = Field(default_factory=lambda: uuid.uuid4().hex)
    name: str = Field(default="prompt")
    description: str = Field(default="Simple Prompt")
    content: constr(min_length=1) = Field(...)
    created_at: str = Field(default_factory=lambda: str(time.time()))
    last_modified_at: str
    edit_count: int = 0
    edit_history: List[str] = []
    autosave: bool = False
    autosave_folder: str = "prompts"
    auto_generate_prompt: bool = False
    parent_folder: str = Field(default_factory=get_workspace_dir)
    llm: Any = None
```

**Key Methods:**
- `edit_prompt(new_content)`: Updates content with history tracking
- `rollback(version)`: Revert to previous version
- `get_prompt()`: Returns current content
- `add_tools(tools)`: Appends tool schemas to prompt
- `_autosave()`: Persists to JSON in workspace

### Prompt Template Patterns

#### 1. String Constants (Most Common)
Simple module-level string constants:
```python
# From moa_prompt.py
MOA_RANKER_PROMPT = """
You are a highly efficient assistant who evaluates and selects...
"""
```

#### 2. Function-Generated Prompts
Templates with runtime parameterization:
```python
# From autonomous_agent_prompt.py
def get_autonomous_agent_prompt(task: str, context: str = None) -> str:
    ...
def get_autonomous_agent_prompt_with_context(task, context, ...):
    ...
```

#### 3. Pydantic Model Wrappers
Structured prompts with validation:
```python
# From prompt.py
class Prompt(BaseModel):
    content: str = Field(...)
    edit_history: List[str] = []
```

### Multi-Agent Collaboration Prompts

**File:** `swarms/prompts/multi_agent_collab_prompt.py`

Three versions of collaboration prompts:

1. **MULTI_AGENT_COLLAB_PROMPT** (Full, 154 lines)
   - Comprehensive protocols for multi-agent coordination
   - 5 sections: Task specification, Inter-agent alignment, Verification, Reflective thinking, Behavioral principles

2. **MULTI_AGENT_COLLAB_PROMPT_TWO** (Medium, 157 lines)
   - Condensed version with same sections

3. **MULTI_AGENT_COLLAB_PROMPT_TWO** (Short, 22 lines - overrides previous)
   - Minimal protocol summary:
   ```python
   """
   You are part of a collaborative multi-agent system. Work together...
   ### Core Principles
   1. Clarity, Role Awareness, Communication, Verification, Reflection
   """
   ```

**Note:** Variable shadowing - the same name is redefined. The final version (lines 316-338) is what gets used.

### Mixture of Agents Prompts

**File:** `swarms/prompts/moa_prompt.py`

Two key prompts for the MixtureOfAgents swarm:

1. **MOA_RANKER_PROMPT**: Evaluates model outputs and selects best
2. **MOA_AGGREGATOR_SYSTEM_PROMPT**: Synthesizes multiple responses into single high-quality reply

### Domain-Specific Prompts

Examples from the 69 modules:

| Prompt File | Purpose |
|-------------|---------|
| `finance_agent_prompt.py` | Financial analysis and reporting |
| `legal_agent_prompt.py` | Legal document review |
| `sales_prompts.py` | Sales conversation handling |
| `code_interpreter.py` | Code execution and debugging |
| `react.py` | ReAct reasoning pattern |
| `xray_swarm_prompt.py` | Medical imaging analysis |

### Prompt Generation Utilities

**File:** `swarms/prompts/prompt_generator.py`

Contains utilities for:
- Auto-generating prompts from task descriptions
- Optimizing existing prompts
- Template interpolation

### __init__.py Exports

**File:** `swarms/prompts/__init__.py`

Exposes a curated subset:
```python
from swarms.prompts.code_interpreter import CODE_INTERPRETER
from swarms.prompts.documentation import DOCUMENTATION_WRITER_SOP
from swarms.prompts.finance_agent_prompt import FINANCE_AGENT_PROMPT
# ... (10+ more exports)
from swarms.prompts.autonomous_agent_prompt import (
    AUTONOMOUS_AGENT_SYSTEM_PROMPT,
    get_autonomous_agent_prompt,
    get_autonomous_agent_prompt_with_context,
)
```

### Integration Points

**SwarmRouter Integration (swarm_router.py):**
```python
from swarms.prompts.multi_agent_collab_prompt import (
    MULTI_AGENT_COLLAB_PROMPT_TWO,
)

# In update_system_prompt_for_agent_in_swarm():
for agent in self.agents:
    agent.system_prompt += MULTI_AGENT_COLLAB_PROMPT_TWO
```

**Prompt Class Integration:**
```python
from swarms.prompts.prompt import Prompt
prompt = Prompt(content="...", autosave=True)
prompt.edit_prompt("updated content")
```

### Clever Solutions

1. **Template Versioning**: Edit history tracks all changes
2. **Autosave**: Prompts persist to workspace automatically
3. **Multi-Agent Protocols**: Pre-built collaboration frameworks
4. **Domain Specialization**: Vertical-specific prompts for industry use cases

### Technical Debt / Issues

1. **Inconsistent Organization**: Some files export single constants, others multiple
2. **No Validation**: String constant prompts have no schema validation
3. **Variable Shadowing**: `MULTI_AGENT_COLLAB_PROMPT_TWO` redefined (lines 157 and 316)
4. **Partial Exports**: Only subset of 69 modules exposed in __init__.py
5. **No Prompt Testing**: No visible test infrastructure for prompt quality
6. **Limited Documentation**: Most prompt files lack docstrings explaining intended use

### Validation & Edge Cases

- Empty prompt content: Pydantic constraint `min_length=1` prevents
- Autosave failure: Silently continues without persisting
- Invalid rollback version: Raises IndexError
- Tool addition failure: Schema mismatch between tools and prompt format

---

## Cross-Feature Analysis

### Feature Synergies

| Feature 7 (SwarmRouter) | Feature 8 (AOP) | Feature 9 (Prompts) |
|-------------------------|-----------------|---------------------|
| Uses MULTI_AGENT_COLLAB_PROMPT_TWO for coordination | Transforms agents into discoverable MCP tools | Provides 69 pre-built prompt templates |
| Routes to 16 swarm types | TaskQueue provides reliability layer | Prompt class enables versioning and autosave |
| Factory pattern for O(1) lookup | Network resilience for distributed deployment | Domain-specific prompts for industry verticals |

### Shared Patterns

1. **Configuration Objects**: Both SwarmRouterConfig and AgentToolConfig use Pydantic
2. **Logging**: Both use loguru with initialize_logger
3. **Error Classes**: Custom exceptions with clear naming (SwarmRouterConfigError, etc.)
4. **Autosave**: SwarmRouter and Prompt both implement persistence

### Potential Integration Points

1. **AOP could use SwarmRouter**: Instead of direct agent.run(), AOP could route through SwarmRouter for complex orchestration
2. **Prompt versioning in AOP**: Agent discovery could expose prompt edit history
3. **SwarmRouter with AOP**: Deploy SwarmRouter-managed swarms as MCP tools

---

## Summary Table

| Aspect | SwarmRouter | AOP | Prompt Management |
|--------|-------------|-----|-------------------|
| **File** | swarm_router.py (1017 lines) | aop.py (2949 lines) | prompts/ (69 modules) |
| **Pattern** | Factory + Cache | ThreadPool + FastMCP | Template Constants + Class |
| **Primary Use** | Single interface for 16 swarm types | Distributed agent deployment | Agent specialization |
| **Key Innovation** | O(1) dispatch, autosave | TaskQueue with retry, discovery protocol | Edit history, autosave |
| **Error Handling** | Config validation, typed exceptions | Network retry, timeout polling | Pydantic validation |
| **Technical Debt** | Unused params, missing AutoSwarmBuilder | Polling, type hints | Variable shadowing, partial exports |

---

## Recommendations

### For SwarmRouter
1. Add validation for agent count requirements per swarm type
2. Implement the missing AutoSwarmBuilder in factory
3. Remove duplicate autosave assignment
4. Add circuit-breaker pattern for failing swarms

### For AOP
1. Replace polling with asyncio.Event for task completion
2. Fix AgentType vs Agent type inconsistency
3. Add circuit-breaker per agent tool
4. Implement graceful degradation for discovery tools

### For Prompt Management
1. Fix variable shadowing in multi_agent_collab_prompt.py
2. Add comprehensive __init__.py exports
3. Implement prompt testing framework
4. Add docstrings to all prompt files

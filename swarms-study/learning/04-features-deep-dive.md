# Swarms Framework - Features Deep Dive

**Date:** 2026-03-26
**Source:** Synthesis of 05a-05d feature batch analyses
**Repo:** `/Users/sheldon/Documents/claw/reference/swarms`

---

## Executive Summary

The swarms framework is a vendor-agnostic, multi-agent orchestration platform built around a 252KB monolithic `Agent` class. It provides 65+ structure modules implementing diverse orchestration patterns, from simple sequential workflows to complex hierarchical swarms with autonomous agent generation.

**Core Philosophy:** Simple agent primitive + complex orchestration patterns + vendor-agnostic LLM abstraction

**Key Architectural Choices:**
- LiteLLM as the universal LLM interface (100+ models)
- Pydantic models for all configuration
- ThreadPoolExecutor for concurrency
- MCP (Model Context Protocol) for tool extensibility
- Workspace-based autosave for persistence

---

## Priority 1: Agent Base Class

**File:** `swarms/structs/agent.py` (252KB, 1900+ lines)

### Overview

The Agent is the fundamental building block - an autonomous entity powered by an LLM with tools, memory, and self-contained execution loops. It is the orchestrator that consumes all other components.

### Architecture

The Agent class accepts 50+ initialization parameters organized into:

| Category | Parameters |
|----------|------------|
| Identity | `agent_name`, `agent_description`, `system_prompt` |
| Execution Control | `max_loops`, `stopping_condition`, `retry_attempts`, `dynamic_loops` |
| LLM Configuration | `llm`, `model_name`, `temperature`, `max_tokens`, `fallback_models` |
| Memory | `short_memory` (Conversation), `long_term_memory` (BaseVectorDatabase) |
| Tools | `tools`, `tools_list_dictionary`, `tool_schema` |
| MCP | `mcp_url`, `mcp_urls`, `mcp_config` |
| Advanced | `handoffs`, `skills_dir`, `transforms`, `artifacts_on` |

### Memory System

**Short-term Memory (Conversation):**
- In-memory message list with role/content pairs
- Token counting and message IDs
- Truncation for context window management

**Long-term Memory (RAG):**
- Vector database integration via `BaseVectorDatabase` interface
- `rag_every_loop` flag controls retrieval frequency
- Automatic query formulation

### Tool System

Tools flow through `BaseTool` (`swarms/tools/base_tool.py`, 107KB):

1. **Registration:** User functions converted to OpenAI function schema
2. **Schema Generation:** `convert_multiple_functions_to_openai_function_schema()`
3. **Execution:** `execute_tools()` with retry logic
4. **Result Handling:** Appended to conversation memory

### Autonomous Mode (max_loops="auto")

When `max_loops="auto"`, the agent enters autonomous planning:

1. **Planning Phase:** Subtask breakdown via `create_plan` tool
2. **Execution Phase:** Iterate through subtasks with status tracking
3. **Feedback Loop:** Director-style evaluation of results

Selected autonomous tools:
- Planning: `create_plan`, `think`, `subtask_done`, `complete_task`
- File operations: `create_file`, `update_file`, `read_file`, `list_directory`, `delete_file`, `run_bash`
- Agent management: `create_sub_agent`, `assign_task`

### Handoffs (Agent-to-Agent Delegation)

```python
agent = Agent(handoffs=[researcher_agent, writer_agent])
# LLM can call handoff_task tool to delegate subtasks
```

### LiteLLM Integration

The Agent uses LiteLLM as its primary LLM interface with fallback cascade:

```python
agent = Agent(fallback_models=["gpt-4.1", "gpt-4o-mini", "claude-sonnet"])
# On failure, tries next model in list
```

### Notable Patterns

**Conditional Prompt Augmentation:**
```python
if max_loops == "auto":
    system_prompt += get_autonomous_agent_prompt()
if react_on:
    system_prompt += REACT_SYS_PROMPT
```

**Dynamic Model Selection:**
```python
if random_models_on:
    model_name = set_random_models_for_agents()
```

### Technical Debt

1. **Monolithic 252KB file:** Single class with 1900+ lines
2. **50+ init parameters:** Constructor is unwieldy, suggests need for composition
3. **Inconsistent naming:** `agent_name` vs `name`, `agent_description` vs `description`
4. **Context length hardcoded:** `self.context_length = 16000` despite parameter
5. **Workspace dir from env var:** `self.workspace_dir = get_workspace_dir()` ignores constructor input
6. **Saved state path from UUID:** Not deterministic, complicates replay

---

## Priority 2: Multi-Agent Orchestration Architectures

**Directory:** `swarms/structs/` (65+ modules)

### Overview

The framework provides diverse multi-agent coordination patterns. `AgentRearrange` is the foundational orchestrator that others delegate to.

### Core Patterns

#### SequentialWorkflow (`sequential_workflow.py`)

Linear execution chain where agents process task output sequentially.

```python
workflow = SequentialWorkflow(
    agents=[agent1, agent2, agent3],
    max_loops=1
)
result = workflow.run("task")
```

**Flow:** `agent1 -> agent2 -> agent3`

Delegates to `AgentRearrange` internally. Supports async via `run_async()`, `run_concurrent()`.

#### ConcurrentWorkflow (`concurrent_workflow.py`)

Parallel execution where all agents process the same task simultaneously.

```python
workflow = ConcurrentWorkflow(
    agents=[agent1, agent2, agent3],
    show_dashboard=True
)
```

**Implementation:**
- `ThreadPoolExecutor` with `max_workers = cpu_count * 0.95`
- Dashboard mode with real-time status display
- `agent_statuses` dict tracks: pending/running/completed/error

#### AgentRearrange (`agent_rearrange.py`)

Most sophisticated orchestration - supports mixed sequential/concurrent flows.

```python
flow = "researcher -> writer, editor -> reviewer"
swarm = AgentRearrange(
    agents=[researcher, writer, editor, reviewer],
    flow=flow,
    team_awareness=True
)
```

**Flow Syntax:**
- `->` Sequential: "A -> B" means A runs, then B
- `,` Concurrent: "A, B" means A and B run in parallel
- `H` Human-in-the-loop: Pause for human input

**Team Awareness:** Each agent receives context about who runs before/after them.

#### MixtureOfAgents (`mixture_of_agents.py`)

Layered parallel processing with aggregation.

```python
moa = MixtureOfAgents(
    agents=[agent1, agent2, agent3],
    aggregator_agent=aggregator,
    layers=3
)
```

**Flow per layer:**
1. All agents process task concurrently
2. Outputs collected into shared conversation
3. Next layer receives full conversation history
4. After N layers, aggregator synthesizes final answer

**Key insight:** Full conversation history passed between layers, not just last output.

#### HierarchicalSwarm (`hiearchical_swarm.py`)

Director-led hierarchy with feedback loops.

```python
swarm = HierarchicalSwarm(
    director=llm_with_swarm_spec_format,
    agents=[worker1, worker2],
    max_loops=3
)
```

**Flow:**
1. Director receives task, creates execution plan via structured output (Pydantic)
2. Director delegates subtasks to workers
3. Workers report back
4. Director evaluates, may issue new tasks (up to max_loops)
5. Final synthesis

### Other Patterns

| File | Pattern |
|------|---------|
| `groupchat.py` | Round-robin agent discussion |
| `majority_voting.py` | Agents vote on answer |
| `council_as_judge.py` | Expert panel deliberation |
| `debate_with_judge.py` | Agents debate, judge decides |
| `round_robin.py` | Cyclic task distribution |
| `heavy_swarm.py` | High-capacity multi-agent |
| `planner_worker_swarm.py` | Separate planner/worker roles |
| `board_of_directors_swarm.py` | Corporate governance pattern |

### Shared Infrastructure

- **Conversation:** Shared memory object across agents
- **History Output Formatter:** `history_output_formatter()` converts conversation to various formats
- **Autosave:** All swarms support workspace-based conversation persistence
- **Multi-Agent Collaboration Prompts:** `MULTI_AGENT_COLLAB_PROMPT` injected to improve coordination

### Technical Observations

1. **AgentRearrange is foundational:** SequentialWorkflow delegates to it
2. **Inconsistent patterns:** Each swarm implements its own `run()` signature
3. **Dashboard coupling:** ConcurrentWorkflow and HierarchicalSwarm have built-in terminal UIs
4. **Flow string parsing:** Custom DSL in AgentRearrange, could be generalized

---

## Priority 3: AutoSwarmBuilder

**File:** `swarms/structs/auto_swarm_builder.py`

### Overview

An autonomous system that generates multi-agent configurations from natural language task descriptions using a "boss agent" pattern.

```python
builder = AutoSwarmBuilder(
    name="my-swarm",
    model_name="gpt-4.1",
    execution_type="return-agents",
    max_loops=1
)
agents = builder.run("Create a research team to analyze market trends")
```

### Three Execution Modes

| Mode | Output |
|------|--------|
| `return-agents` | Returns agent specification dict |
| `return-swarm-router-config` | Returns SwarmRouter config |
| `return-agents-objects` | Returns instantiated Agent objects |

### Process Flow

#### Phase 1: Agent Creation

The boss agent (GPT-4.1) analyzes the task and generates JSON agent specifications via a comprehensive 130+ line system prompt.

#### Phase 2: Agent Instantiation

Agent specs are mapped to Agent objects, handling field name translation (`description` -> `agent_description`).

#### Phase 3: Swarm Router Initialization

The boss generates SwarmRouter configuration including swarm type, flow, and rules.

### Configuration Classes

**AgentSpec:**
```python
class AgentSpec(BaseModel):
    agent_name: str = None
    description: str = None
    system_prompt: str = None
    model_name: str = "gpt-4.1"
    max_tokens: int = 8192
    temperature: float = 0.5
    role: str = "worker"
    max_loops: int = 1
```

**SwarmRouterConfig:**
```python
class SwarmRouterConfig(BaseModel):
    name: str
    description: str
    agents: List[AgentSpec]
    swarm_type: SwarmType
    rearrange_flow: str = None
    rules: str = None
```

### LiteLLM Integration

Uses LiteLLM's `response_format` for structured output (Pydantic enforcement):

```python
def build_llm_agent(self, config: BaseModel):
    return LiteLLM(
        model_name=self.model_name,
        system_prompt=BOSS_SYSTEM_PROMPT,
        temperature=0.5,
        response_format=config,  # Pydantic-constrained output
        max_tokens=self.max_tokens
    )
```

### Limitations

1. **Hardcoded boss model:** Always uses `gpt-4.1`
2. **Single boss:** No multi-boss or hierarchical structure
3. **No agent pool:** Creates fresh agents each run
4. **No retry on LLM failure:** Single attempt at creation
5. **Console print debugging:** `print(swarm_spec)` left in code

---

## Priority 4: Multi-Model Provider Support

**File:** `swarms/utils/litellm_wrapper.py` (59KB, 1458 lines)

### Overview

Vendor-agnostic architecture supporting OpenAI, Anthropic, Google, Groq, Cohere, DeepSeek, Ollama, OpenRouter, XAI, and Llama4 through LiteLLM.

### Architecture

```
LiteLLM Wrapper
    |
    +---> LiteLLM Library (External)
    |       +---> OpenAI, Anthropic, Google, Groq, Cohere, DeepSeek, Ollama, OpenRouter, XAI, Llama4
    |
    +---> Vision Processing (anthropic_vision_processing, openai_vision_processing)
    +---> Audio Processing (audio_processing, get_audio_base64)
    +---> Tool/Function Calling (output_for_tools, tools_list_dictionary)
    +---> Reasoning Models (reasoning_check, reasoning_effort)
```

### Key Implementation Patterns

#### Model-Agnostic Request Processing

Parameter priority cascade (lines 1191-1196):
1. Runtime kwargs (passed to run method) - HIGHEST
2. Runtime args (if dictionary)
3. Init kwargs (passed to __init__)
4. Init args (if dictionary)
5. Default parameters

#### Vision Processing Strategy

```python
def vision_processing(self, task: str, image: str, messages: Optional[list] = None):
    if "anthropic" in self.model_name.lower() or "claude" in self.model_name.lower():
        return self.anthropic_vision_processing(task, image, messages)
    else:
        return self.openai_vision_processing(task, image, messages)
```

The `_should_use_direct_url()` method intelligently decides between:
- Direct URL passing (efficient, for remote images)
- Base64 encoding (for local files, URLs requiring download, or local models)

Local model detection checks for: "ollama", "llama-cpp", "localhost", "127.0.0.1", etc.

#### Error Handling with Network Awareness

```python
if self.is_local_model(self.model_name, self.base_url):
    # Ollama-specific troubleshooting
    raise NetworkConnectionError("Network error connecting to local model...")
else:
    # Checks internet connectivity
    has_internet = self.check_internet_connection()
    if not has_internet:
        raise NetworkConnectionError("No internet connection detected...")
```

#### System Prompt Normalization

Handles orchestrator-style "System:\n\nHuman:" prompts and strips empty system blocks to prevent Anthropic API errors.

### Technical Debt

1. **Batched Run Incomplete:** `batched_run()` references `_process_batch()` which is never defined
2. **Hardcoded Model Exclusions:** Temperature exclusion for specific models is fragile
3. **Model Name Detection is String-Based:** Simple substring matching could produce false positives/negatives
4. **SSL Verification Disabled by Default:** `ssl_verify: bool = False` is a security concern

---

## Priority 5: Agent Tools and Memory Systems

### Overview

Extensive library of tools and multiple memory systems for agent capability extension.

### Tool System Architecture

```
BaseTool (base_tool.py)
    |
    +---> Schema Conversion
    |       +---> func_to_dict() - Function to OpenAI schema
    |       +---> base_model_to_dict() - Pydantic to OpenAI schema
    |       +---> multi_base_models_to_dict() - Batch conversion
    |
    +---> Execution
    |       +---> execute_tool() - Tool execution
    |       +---> parse_and_execute_json() - JSON-based execution
    |
    +---> Caching
            +---> _make_hashable() - Cache key generation

ToolStorage (tool_registry.py)
    |
    +---> add_tool(), add_many_tools(), get_tool(), list_tools()

HandoffsTool (handoffs_tool.py)
    |
    +---> handoff_task() - Delegate to other agents
    +---> ThreadPoolExecutor - Parallel execution
```

### Memory System Architecture

The framework does not have a unified memory module. Memory is implemented through several patterns:

**1. Conversation History (swarms/structs/conversation.py)**
- In-memory message storage with token-aware truncation
- Methods: `add()`, `get_history()`, `save()`, `load()`

**2. REACT Agent Memory (swarms/agents/react_agent.py)**
```python
class ReactAgent:
    def run(self, task: str, *args, **kwargs) -> List[str]:
        self.memory = []
        for i in range(self.max_loops):
            step_result = self.step(current_task)
            self.memory.append(step_result)
            memory_context = "\n\nMemory of previous steps:\n" + \
                "\n".join(f"Step {j+1}:\n{step}" for j, step in enumerate(self.memory))
```

**3. Iterative Reflective Expansion (swarms/agents/i_agent.py)**
- Maintains `memory_pool` for historical paths
- Score-based revision loop
- Path selection and synthesis

**4. Agent Long-Term Memory**
- `long_term_memory: BaseVectorDatabase` parameter in Agent
- Pluggable interface expecting external vector database

### Notable Observations

1. **No Explicit Memory Interface:** Framework lacks unified memory abstraction
2. **Memory Pattern Inconsistency:** ReactAgent uses `List[str]`, IAgent uses `Conversation` + `memory_pool`, Agent expects `BaseVectorDatabase`
3. **Tool Execution Decoupled:** BaseTool provides schema conversion but not direct execution
4. **Handoffs Limited to Same-Process:** agent_registry dict limits to same-process orchestration

---

## Priority 6: Model Context Protocol (MCP) Integration

**File:** `swarms/tools/mcp_client_tools.py` (42KB, 900+ lines)

### Overview

Standardized protocol for AI agents to interact with external tools and services through MCP servers.

### Architecture

```
MCP Client (mcp_client_tools.py)
    |
    +---> Tool Discovery
    |       +---> aget_mcp_tools() - Async tool fetching
    |       +---> get_mcp_tools_sync() - Sync wrapper
    |       +---> get_tools_for_multiple_mcp_servers() - Parallel multi-server
    |
    +---> Tool Execution
    |       +---> _execute_tool_call_simple() - Async execution
    |       +---> execute_tool_call_simple() - Async wrapper
    |       +---> execute_multiple_tools_on_multiple_mcp_servers_sync() - Batch
    |
    +---> Protocol Translation
    |       +---> transform_mcp_tool_to_openai_tool() - Schema conversion
    |       +---> transform_openai_tool_call_request_to_mcp_tool_call_request()
    |       +---> call_openai_tool() - OpenAI-format to MCP
    |
    +---> Transport Handling
            +---> get_mcp_client() - Context manager selection
            +---> auto_detect_transport() - HTTP vs stdio
```

### Transport Auto-Detection

```python
def auto_detect_transport(url: str) -> str:
    parsed = urlparse(url)
    scheme = parsed.scheme.lower()
    if scheme in ("http", "https"):
        return "streamable_http"
    elif "stdio" in url or scheme == "":
        return "stdio"
    else:
        return "streamable-http"
```

### Schema Transformation

**MCP to OpenAI:**
```python
def transform_mcp_tool_to_openai_tool(mcp_tool: MCPTool, verbose: bool = False):
    return ChatCompletionToolParam(
        type="function",
        function=FunctionDefinition(
            name=mcp_tool.name,
            description=mcp_tool.description or "",
            parameters=mcp_tool.inputSchema,
            strict=False,
        ),
    )
```

### Multi-Server Parallel Tool Fetching

Uses `ThreadPoolExecutor` to fetch tools from multiple servers concurrently with error isolation.

### Retry Logic with Exponential Backoff

```python
@retry_with_backoff(retries=3, backoff_in_seconds=1)
async def aget_mcp_tools(...):
    # Implementation with automatic retry
```

### Notable Observations

1. **No Server-Side MCP Implementation:** Project only implements MCP client, not servers
2. **StreamingHTTP is Default:** Appropriate for production but limits stdio use cases
3. **Tool Name Collision Risk:** When multiple servers expose same-named tools, dict overwrites entries
4. **Complex Async/Sync Mixing:** Both async and sync functions with event loop management

---

## Priority 7: SwarmRouter

**File:** `swarms/structs/swarm_router.py` (1017 lines)

### Overview

Universal orchestrator providing a single interface to run any swarm type with dynamic selection.

### Architecture

Factory-pattern-based dispatcher that routes tasks to different swarm implementations based on `swarm_type` parameter. O(1) lookup via pre-initialized factory dictionary.

### Supported Swarm Types (16)

| Type | Default |
|------|---------|
| SequentialWorkflow | Yes |
| ConcurrentWorkflow | No |
| MixtureOfAgents | No |
| AgentRearrange | No |
| HierarchicalSwarm | No |
| GroupChat | No |
| MultiAgentRouter | No |
| HeavySwarm | No |
| CouncilAsAJudge | No |
| DebateWithJudge | No |
| MajorityVoting | No |
| LLMCouncil | No |
| BatchedGridWorkflow | No |
| RoundRobin | No |
| PlannerWorkerSwarm | No |
| AutoSwarmBuilder | Reference only |

### SwarmFactory Pattern

```python
def _initialize_swarm_factory(self) -> Dict[str, Callable]:
    return {
        "HeavySwarm": self._create_heavy_swarm,
        "AgentRearrange": self._create_agent_rearrange,
        "CouncilAsAJudge": self._create_council_as_judge,
        # ... 13 more
    }
```

### SwarmCache

Implements caching for created swarms:
```python
self._swarm_cache = {}
cache_key = f"{self.swarm_type}_{hash(str(args) + str(kwargs))}"
```

### Execution Flow

1. User calls `run(task)` with `swarm_type`
2. `reliability_check()` validates configuration
3. `setup()` applies hooks (APE, shared memory, rules, collaboration prompts)
4. `_create_swarm()` looks up factory, creates swarm, caches it
5. Swarm's `run()` method is invoked

### Setup Hooks

```python
def setup(self):
    if self.auto_generate_prompts is True:
        self.activate_ape()
    if self.shared_memory_system is not None:
        self.activate_shared_memory()
    if self.rules is not None:
        self.handle_rules()
    if self.multi_agent_collab_prompt is True:
        self.update_system_prompt_for_agent_in_swarm()
```

### Technical Debt

1. **Deprecated Parameters:** Many parameters passed through but unused
2. **Inconsistent Agent Requirements:** Some swarms need exactly N agents but not validated
3. **Missing AutoSwarmBuilder:** Listed in SwarmType but not in factory
4. **Duplicate Parameter Assignment:** `self.autosave` assigned twice (lines 200 and 271)
5. **Circular Reference:** Imports from `swarms.structs.*` modules

---

## Priority 8: Agent Orchestration Protocol (AOP)

**File:** `swarms/structs/aop.py` (2949 lines)

### Overview

Framework for deploying and managing agents as distributed services with discovery and management. Transforms agents into MCP tools.

### Architecture

```
AOP
    |
    +---> TaskQueue (Thread-safe priority queue)
    |       +---> Priority-based task scheduling
    |       +---> Worker thread pool
    |       +---> Automatic retry with delay
    |       +---> Task status tracking
    |
    +---> MCP Server (FastMCP)
    |       +---> Agent registration as tools
    |       +---> Agent discovery tools
    |       +---> Queue management tools
    |
    +---> Network Resilience
            +---> Automatic restart on shutdown
            +---> Network error detection and retry
```

### TaskQueue

```python
class TaskQueue:
    def __init__(self, agent_name, agent, max_workers=1,
                 max_queue_size=1000, processing_timeout=30,
                 retry_delay=1.0, verbose=False)
```

Features:
- Priority-based scheduling (higher priority = first)
- Worker thread pool for parallel processing
- Automatic retry with configurable delay
- Task status: PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED

### Task Dataclass

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

### Queue-Based Execution

When queue_enabled, tool invocation:
1. Adds task to TaskQueue with priority
2. Returns task_id immediately (if wait_for_completion=False)
3. Worker loop processes task asynchronously
4. Polls for completion with timeout

### Agent Discovery Tools

| Tool | Purpose |
|------|---------|
| `discover_agents` | List all agents or get specific agent info |
| `get_agent_details` | Detailed single agent information |
| `get_agents_info` | Batch agent information lookup |
| `list_agents` | Simple agent name list |
| `search_agents` | Keyword search across agents |
| `get_server_info` | Comprehensive server metadata |

### Queue Management Tools

| Tool | Purpose |
|------|---------|
| `get_queue_stats` | Per-agent or aggregate queue statistics |
| `pause_agent_queue` | Pause specific agent queue |
| `resume_agent_queue` | Resume specific agent queue |
| `clear_agent_queue` | Clear pending tasks |
| `get_task_status` | Check individual task status |
| `cancel_task` | Cancel a queued task |

### Network Resilience

**Persistence Mode:**
- Automatic restart on shutdown (up to max_restart_attempts)
- Failsafe protection against infinite restart loops

**Network Error Detection:**
```python
def _is_network_error(self, error: Exception) -> bool:
    network_errors = (ConnectionError, ConnectionRefusedError,
                      ConnectionResetError, TimeoutError, socket.gaierror, ...)
```

### AOPCluster

Manages multiple MCP servers for distributed deployment:
```python
class AOPCluster:
    def __init__(self, urls: List[str], transport="streamable-http")
    def get_tools(self, output_type="dict") -> List[Dict[str, Any]]
    def find_tool_by_server_name(self, server_name: str) -> Dict[str, Any]
```

### Technical Debt

1. **Synchronous Threading in Async Context:** Uses `threading.RLock()` in async methods, potential deadlock
2. **Task Polling:** `_wait_for_task_completion` uses sleep polling (line 1276)
3. **Agent Type Hint:** Uses `AgentType` but stores as `Agent`
4. **Error Swallowing:** Many catch blocks silently return without full error context
5. **No Graceful Degradation:** If one agent fails, discovery tools may return partial results

---

## Priority 9: Prompt Management

**Directory:** `swarms/prompts/` (69 modules)

### Overview

Large library of prompt templates for various agent specializations and tasks.

### Organization Structure

| Category | Files | Purpose |
|----------|-------|---------|
| Agent-specific | `agent_prompts.py`, `agent_judge_prompt.py`, `agent_system_prompts.py` | Base agent prompts |
| Swarm patterns | `moa_prompt.py`, `multi_agent_collab_prompt.py`, `agent_orchestration_prompt.py` | Multi-agent orchestration |
| Domain-specific | `finance_agent_prompt.py`, `legal_agent_prompt.py`, `sales_prompts.py` | Industry vertical prompts |
| Functional | `react.py`, `reasoning_prompt.py`, `code_interpreter.py` | Capability-specific |
| Generator/Builder | `prompt_generator.py`, `autonomous_agent_prompt.py`, `agent_self_builder_prompt.py` | Prompt creation utilities |

### Prompt Class Architecture

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

Key methods:
- `edit_prompt()`: Updates content with history tracking
- `rollback()`: Revert to previous version
- `get_prompt()`: Returns current content
- `add_tools()`: Appends tool schemas to prompt
- `_autosave()`: Persists to JSON in workspace

### Prompt Template Patterns

#### 1. String Constants (Most Common)
```python
# From moa_prompt.py
MOA_RANKER_PROMPT = """
You are a highly efficient assistant who evaluates and selects...
"""
```

#### 2. Function-Generated Prompts
```python
# From autonomous_agent_prompt.py
def get_autonomous_agent_prompt(task: str, context: str = None) -> str:
    ...
def get_autonomous_agent_prompt_with_context(task, context, ...):
    ...
```

#### 3. Pydantic Model Wrappers
```python
class Prompt(BaseModel):
    content: str = Field(...)
    edit_history: List[str] = []
```

### Multi-Agent Collaboration Prompts

Three versions in `multi_agent_collab_prompt.py`:

1. **MULTI_AGENT_COLLAB_PROMPT** (Full, 154 lines) - Comprehensive protocols
2. **MULTI_AGENT_COLLAB_PROMPT_TWO** (Medium, 157 lines) - Condensed version
3. **MULTI_AGENT_COLLAB_PROMPT_TWO** (Short, 22 lines) - Minimal summary

**Note:** Variable shadowing - the same name is redefined. The final version (lines 316-338) is what gets used.

### Integration Points

**SwarmRouter Integration:**
```python
from swarms.prompts.multi_agent_collab_prompt import MULTI_AGENT_COLLAB_PROMPT_TWO

for agent in self.agents:
    agent.system_prompt += MULTI_AGENT_COLLAB_PROMPT_TWO
```

### Technical Debt

1. **Inconsistent Organization:** Some files export single constants, others multiple
2. **No Validation:** String constant prompts have no schema validation
3. **Variable Shadowing:** `MULTI_AGENT_COLLAB_PROMPT_TWO` redefined
4. **Partial Exports:** Only subset of 69 modules exposed in __init__.py
5. **No Prompt Testing:** No visible test infrastructure for prompt quality
6. **Limited Documentation:** Most prompt files lack docstrings

---

## Priority 10: CLI and SDK Tools

**Primary Files:**
- `swarms/cli/main.py` (57KB, ~1676 lines)
- `swarms/cli/utils.py` (20KB)
- `swarms/structs/agent_loader.py` (210 lines)
- `swarms/utils/` (29 utility modules)

### CLI Architecture

Uses `argparse` with custom help action for rich terminal output.

**Command Structure:**
```python
command_choices = [
    "onboarding",        # Environment setup check
    "get-api-key",       # Open browser for API keys
    "check-login",       # Verify authentication
    "run-agents",        # Execute from YAML config
    "load-markdown",     # Load from markdown files
    "agent",             # Create/run custom agent
    "chat",              # Interactive chat mode
    "upgrade",           # Update swarms package
    "autoswarm",         # Auto-generate swarm config
    "setup-check",       # Comprehensive env check
    "llm-council",       # Multi-agent collaboration
    "heavy-swarm",       # Complex task analysis
]
```

### AgentLoader

Unified loader supporting multiple file formats:

```python
class AgentLoader:
    def load_agents_from_markdown(...)   # Markdown with YAML frontmatter
    def load_agents_from_yaml(...)       # YAML configuration
    def load_agents_from_csv(...)        # CSV-based agents
    def auto(...)                        # Auto-detect by extension
    def load_multiple_agents(...)         # Batch loading
```

### SDK Utilities

| File | Size | Purpose |
|------|------|---------|
| `litellm_wrapper.py` | 59KB | Unified LLM interface |
| `formatter.py` | 31KB | Output formatting |
| `agent_loader_markdown.py` | 18KB | Markdown-based agent loading |
| `fetch_prompts_marketplace.py` | 6KB | Swarms marketplace integration |
| `swarm_autosave.py` | 12KB | Agent state persistence |

### Notable Patterns

**Lazy Imports:** Used extensively to avoid circular dependencies:
```python
def load_agent_from_markdown(self, file_path: str, **kwargs) -> "Agent":
    from swarms.structs.agent import Agent  # Lazy import
```

**Auto-format Detection:** `AgentLoader.auto()` routes based on file extension

**Concurrent Loading:** Uses `ThreadPoolExecutor` with configurable workers

### Technical Debt

1. **File location:** `agent_loader_markdown.py` in `swarms/agents/` but logically belongs in `swarms/utils/`
2. **Inconsistent naming:** `create_agents_from_yaml` imported from `swarms.agents`
3. **mcp_url type:** `MarkdownAgentConfig.mcp_url` typed as `Optional[int]` but URLs are strings
4. **No validation on file paths:** Limited sanitization

---

## Priority 11: X402 Payment Protocol

### Implementation Status: Documentation/Examples Only

The X402 payment protocol is **not implemented in the core swarms package**. It is purely external integration demonstrated through examples.

### Files

| File | Purpose |
|------|---------|
| `examples/guides/x402_examples/README.md` | Usage documentation |
| `examples/guides/x402_examples/research_agent_x402_example.py` | Working example |
| `examples/guides/x402_examples/agent_integration/x402_agent_buying.py` | Agent purchasing example |
| `docs/examples/x402_payment_integration.md` | Integration guide |

### Integration Pattern

```python
from x402.fastapi.middleware import require_payment
from swarms import Agent

# 1. Create agent with tools
research_agent = Agent(
    agent_name="Research-Agent",
    system_prompt="You are an expert research analyst...",
    model_name="gpt-5.4",
    max_loops=1,
    tools=[exa_search],
)

# 2. Apply payment middleware
app.middleware("http")(
    require_payment(
        path="/research",
        price="$0.01",
        pay_to_address="0xYourWalletAddressHere",
        network_id="base-sepolia",
    )
)
```

### Observations

1. **No core implementation:** X402 is purely example/documentation
2. **Minimal swarms coupling:** Agents are used as-is; x402 is orthogonal
3. **Pattern is generalizable:** Any FastAPI endpoint can wrap any swarms component
4. **Documentation is thorough:** Examples cover testnet, production, and troubleshooting

---

## Priority 12: Agent Skills (Anthropic-compatible)

**Primary Files:**
- `swarms/agents/agent_loader_markdown.py` (498 lines)
- `swarms/structs/agent_loader.py` (210 lines)

### Format Specification

Agent skills use **markdown files with YAML frontmatter** following Claude Code sub-agent format:

```markdown
---
name: skill-name
description: Skill description
model_name: claude-sonnet-4-20250514
temperature: 0.7
max_loops: 1
---

# Skill Name

System prompt content goes here as markdown...
```

### MarkdownAgentConfig Schema

```python
class MarkdownAgentConfig(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    model_name: Optional[str] = DEFAULT_MODEL
    temperature: Optional[float] = Field(default=0.1, ge=0.0, le=2.0)
    mcp_url: Optional[int] = None  # BUG: should be Optional[str]
    system_prompt: Optional[str] = None
    max_loops: Optional[int] = Field(default=1, ge=1)
    # ... more fields
```

### Loading Flow

```python
# Via CLI
swarms load-markdown --markdown-path ./agents/

# Via AgentLoader
loader = AgentLoader()
agents = loader.load_agents_from_markdown("./path/to/skills/")

# Via convenience function
from swarms.utils.agent_loader_markdown import load_agent_from_markdown
agent = load_agent_from_markdown("code-review.md", max_loops=2)
```

### Example Skills

**Code Review Skill:**
- Comprehensive review checklist (quality, security, performance)
- OWASP Top 10 vulnerability checks
- DRY/SOLID principles verification

**Financial Analysis Skill:**
- DCF modeling methodology
- Financial ratio analysis framework
- Valuation models

### Anthropic Compatibility

The format is explicitly designed for Claude Code sub-agent YAML frontmatter compatibility:
- Same field names (`name`, `description`, `model_name`, `temperature`)
- Same markdown content structure
- Pattern allows importing Claude Code skills directly into swarms

### Bug: mcp_url Type Error

```python
# In agent_loader_markdown.py, line 31:
mcp_url: Optional[int] = None  # WRONG - should be Optional[str]
```

### Technical Debt

1. **File location:** `agent_loader_markdown.py` in `swarms/agents/` instead of `swarms/utils/`
2. **Type bug:** `mcp_url` typed as `int` instead of `str`
3. **No validation:** System prompt validated but `mcp_url` has no validation
4. **Limited error recovery:** Only logs warnings, no retry for transient failures

---

## Cross-Feature Analysis

### Integration Hierarchy

```
User Task
    |
    v
Agent.run()  [Priority 1]
    |
    +---> LiteLLM [Priority 4]
    |       +---> 100+ model providers
    |
    +---> Tools via BaseTool [Priority 5]
    |       +---> MCP Client [Priority 6]
    |       +---> HandoffsTool
    |
    +---> Memory [Priority 5]
            +---> Conversation
            +---> VectorDB (external)
    |
    v
Multi-Agent Orchestration [Priority 2]
    |
    +---> AgentRearrange (foundational)
    +---> SequentialWorkflow
    +---> ConcurrentWorkflow
    +---> MixtureOfAgents
    +---> HierarchicalSwarm
    |
    v
SwarmRouter [Priority 7]
    |
    +---> Factory dispatch to 16 swarm types
    +---> Setup hooks (APE, shared memory, rules)
    |
    v
AutoSwarmBuilder [Priority 3]
    |
    +---> Boss agent generates agents + config
    +---> Returns to SwarmRouter for execution
```

### Shared Patterns

| Pattern | Used By |
|---------|---------|
| `Conversation` memory | Agent, all swarms |
| `LiteLLM` wrapper | Agent, AutoSwarmBuilder, HierarchicalSwarm |
| `history_output_formatter` | All swarms |
| Autosave workspace | All swarms |
| Pydantic configuration | All components |
| Multi-agent collab prompts | SequentialWorkflow, HierarchicalSwarm, AgentRearrange, SwarmRouter |
| `ThreadPoolExecutor` | ConcurrentWorkflow, AOP TaskQueue, HandoffsTool |

### Feature Synergies

| Feature 7 (SwarmRouter) | Feature 8 (AOP) | Feature 9 (Prompts) |
|-------------------------|-----------------|---------------------|
| Uses MULTI_AGENT_COLLAB_PROMPT_TWO | Transforms agents into discoverable MCP tools | Provides 69 pre-built prompt templates |
| Routes to 16 swarm types | TaskQueue provides reliability layer | Prompt class enables versioning |
| Factory pattern for O(1) lookup | Network resilience for distributed deployment | Domain-specific prompts |

### Potential Integration Points

1. **AOP could use SwarmRouter:** Instead of direct agent.run(), AOP could route through SwarmRouter for complex orchestration
2. **Prompt versioning in AOP:** Agent discovery could expose prompt edit history
3. **SwarmRouter with AOP:** Deploy SwarmRouter-managed swarms as MCP tools

---

## Technical Debt Summary

### Critical Issues

| Feature | Issue | Impact |
|---------|-------|--------|
| Agent (P1) | 252KB monolithic class | Maintenance difficulty |
| Agent (P1) | 50+ init parameters | Complex instantiation |
| Agent (P1) | Saved state path from UUID | Non-deterministic replay |
| LiteLLM (P4) | `_process_batch()` undefined | Batched runs fail |
| LiteLLM (P4) | SSL verification disabled | Security vulnerability |
| MCP (P6) | Tool name collision | Multi-server tool conflicts |
| AOP (P8) | Threading in async context | Potential deadlock |
| AOP (P8) | Sleep polling | Inefficient waiting |

### Medium Issues

| Feature | Issue | Impact |
|---------|-------|--------|
| SwarmRouter (P7) | Duplicate autosave assignment | Code confusion |
| SwarmRouter (P7) | AutoSwarmBuilder not in factory | Incomplete implementation |
| Prompts (P9) | Variable shadowing | Unexpected behavior |
| Prompts (P9) | Partial exports | Missing functionality |
| Agent Skills (P12) | mcp_url typed as int | Type errors |

### Minor Issues

| Feature | Issue | Impact |
|---------|-------|--------|
| AgentRearrange (P2) | Flow DSL parsing | Limited extensibility |
| AutoSwarmBuilder (P3) | Console print debugging | Cleanup needed |
| Agent Skills (P12) | File in wrong directory | Organization issue |

---

## Security Considerations

1. **SSL Verification Disabled by Default** - `ssl_verify: bool = False` in LiteLLM
2. **No MCP Authentication Validation** - Connection tokens accepted without validation
3. **Tool Injection Risk** - Dynamic tool loading from MCP servers could introduce malicious tools
4. **Agent Registry Handoffs** - No sandboxing between agents in handoff scenario
5. **No File Path Sanitization** - Limited validation of user-provided paths in AgentLoader

---

## Performance Characteristics

1. **LiteLLM Caching:** Response caching via `caching: bool` parameter
2. **Parallel MCP Fetching:** `ThreadPoolExecutor` with `max_workers = min(32, os.cpu_count() + 4)`
3. **Token-Aware Truncation:** Conversation implements `dynamic_context_window`
4. **Streaming Support:** Both LiteLLM and Agent support streaming responses
5. **Swarm Caching:** SwarmRouter caches created swarms to avoid recreation overhead

---

## Recommendations

### High Priority

1. **Memory Interface Standardization:** Create unified `Memory` interface that different implementations can satisfy
2. **Agent Refactoring:** Break monolithic Agent class into composable components
3. **AOP Async Redesign:** Replace threading with proper async primitives
4. **SSL by Default:** Enable SSL verification unless explicitly disabled

### Medium Priority

1. **MCP Tool Collision Handling:** Add namespace prefixing for multi-server tool names
2. **Prompt Testing Framework:** Add infrastructure for prompt quality validation
3. **AutoSwarmBuilder Retry:** Add retry logic for LLM failures during agent creation
4. **SwarmRouter Validation:** Add agent count requirements per swarm type

### Low Priority

1. **Documentation:** Add docstrings to all prompt files
2. **File Organization:** Move `agent_loader_markdown.py` to `swarms/utils/`
3. **Variable Shadowing Fix:** Rename duplicate prompt constants
4. **Console Debug Statements:** Remove print statements from AutoSwarmBuilder

---

## File Summary Table

| File | Size | Priority | Purpose |
|------|------|----------|---------|
| `swarms/structs/agent.py` | 252KB | 1 | Main Agent class |
| `swarms/tools/base_tool.py` | 107KB | 5 | Tool schema conversion |
| `swarms/structs/auto_swarm_builder.py` | - | 3 | Autonomous agent generation |
| `swarms/utils/litellm_wrapper.py` | 59KB | 4 | Multi-model LLM wrapper |
| `swarms/cli/main.py` | 57KB | 10 | CLI entry point |
| `swarms/structs/aop.py` | - | 8 | Distributed agent deployment |
| `swarms/structs/swarm_router.py` | - | 7 | Universal swarm dispatcher |
| `swarms/tools/mcp_client_tools.py` | 42KB | 6 | MCP client protocol |
| `swarms/prompts/` | 69 modules | 9 | Prompt templates |
| `swarms/structs/` | 65+ modules | 2 | Orchestration patterns |

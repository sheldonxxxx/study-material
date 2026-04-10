# Design Patterns in Swarms

## Pattern Index

| # | Pattern | Location | Purpose |
|---|---------|----------|---------|
| 1 | Strategy | `structs/groupchat.py` | Speaker selection, stopping conditions |
| 2 | Template Method | `structs/base_swarm.py`, `structs/base_structure.py` | Skeleton methods with default implementations |
| 3 | Decorator/AOP | `structs/aop.py` (102KB) | Cross-cutting concerns separation |
| 4 | Registry | `structs/agent_registry.py` | Dynamic agent lookup |
| 5 | Factory | `structs/auto_swarm_builder.py`, `structs/agent_loader.py` | Agent and swarm creation |
| 6 | Builder | Specialized swarm classes | Fluent swarm construction |
| 7 | Observer/Event | `structs/base_swarm.py` | Task callbacks, status updates |
| 8 | Promise/Future | `structs/multi_agent_exec.py` | Async execution |
| 9 | Hexagonal/Ports-and-Adapters | `tools/base_tool.py` | Tool system abstraction |
| 10 | Pipeline | `structs/sequential_workflow.py` | Sequential agent chaining |
| 11 | Master-Worker | `structs/hierarchical_swarm.py` | Hierarchical task distribution |
| 12 | Graph/Visitor | `structs/graph_workflow.py` | DAG-based task orchestration |

---

## 1. Strategy Pattern

**Purpose:** Encapsulate interchangeable algorithms for speaker selection and stopping conditions.

### Speaker Selection Strategies

**Location:** `structs/groupchat.py:20-100`

```python
# Strategy interface
SpeakerFunction = Callable[[List[str], "Agent"], bool]

# Concrete implementations
def random_speaker(agents: List[str], current: Agent) -> bool:
    """Select random speaker"""
    return random.choice(agents) == current.name

def round_robin_speaker(agents: List[str], current: Agent) -> bool:
    """Select next speaker in rotation"""
    current_idx = agents.index(current.name)
    next_idx = (current_idx + 1) % len(agents)
    return agents[next_idx] == current.name

def priority_speaker(agents: List[str], current: Agent) -> bool:
    """Select based on agent priority"""
    # Implementation checks agent.priority attribute
    pass

def expertise_based(agents: List[str], current: Agent) -> bool:
    """Select based on task expertise matching"""
    pass
```

### Stopping Conditions

**Location:** `structs/stopping_conditions.py`

```python
class StoppingCondition(Protocol):
    def should_stop(self, state: SwarmState) -> bool: ...

class MaxIterationsStopping:
    def __init__(self, max_iterations: int):
        self.max_iterations = max_iterations

class TimeoutStopping:
    def __init__(self, timeout_seconds: float):
        self.timeout_seconds = timeout_seconds

class ConsensusStopping:
    """Stop when agents reach consensus"""
    pass
```

---

## 2. Template Method Pattern

**Purpose:** Define skeleton algorithms in base classes, let subclasses override specific steps.

### BaseStructure

**Location:** `structs/base_structure.py`

```python
class BaseStructure:
    """Base class with async, threading, serialization templates"""

    # Template method - calls abstract _save implementation
    def save_to_file(self, path: str):
        data = self._prepare_save()
        with open(path, 'w') as f:
            f.write(data)

    # Hook methods for subclasses
    def _prepare_save(self) -> dict:
        """Override in subclass for custom serialization"""
        return self.to_dict()

    # Async variants with same interface
    async def asave_to_file(self, path: str):
        """Async version of save_to_file"""
        data = self._prepare_save()
        async with aiofiles.open(path, 'w') as f:
            await f.write(data)

    # Threading variants
    def run_in_thread(self, func: Callable, *args, **kwargs):
        """Execute function in thread pool"""
        with ThreadPoolExecutor() as executor:
            return executor.submit(func, *args, **kwargs)
```

### BaseSwarm

**Location:** `structs/base_swarm.py:191-214`

```python
class BaseSwarm:
    """Template method for swarm execution"""

    def run(self, task: str) -> str:
        """Skeleton execution algorithm"""
        self._initialize()
        result = self._execute_task(task)
        self._finalize()
        return result

    def _initialize(self):
        """Hook for subclass initialization"""
        pass

    def _execute_task(self, task: str) -> str:
        """Abstract - must be implemented by subclass"""
        raise NotImplementedError

    def _finalize(self):
        """Hook for cleanup"""
        pass

    # Async variant
    async def arun(self, task: str) -> str:
        """Async execution template"""
        await self._ainitialize()
        result = await self._aexecute_task(task)
        await self._afinalize()
        return result
```

---

## 3. Decorator/AOP Pattern

**Purpose:** Separate cross-cutting concerns from business logic.

**Location:** `structs/aop.py` (102KB)

```python
# Aspect-oriented programming module
# Separates logging, caching, metrics from core logic

class Aspect:
    """Base aspect class"""
    def before(self, method: Callable, *args, **kwargs):
        pass

    def after(self, method: Callable, result, *args, **kwargs):
        pass

    def around(self, method: Callable, *args, **kwargs):
        return method(*args, **kwargs)

class LoggingAspect(Aspect):
    """Cross-cutting logging"""
    def before(self, method, *args, **kwargs):
        logger.debug(f"Calling {method.__name__}")

    def after(self, method, result, *args, **kwargs):
        logger.debug(f"Completed {method.__name__}")

class MetricsAspect(Aspect):
    """Cross-cutting metrics collection"""
    def after(self, method, result, *args, **kwargs):
        metrics.increment(f"{method.__name__}.success")
```

---

## 4. Registry Pattern

**Purpose:** Centralized dynamic agent lookup by name or ID.

### AgentRegistry

**Location:** `structs/agent_registry.py`

```python
class AgentRegistry:
    """Central agent registry for dynamic lookup"""

    _agents: Dict[str, Agent] = {}
    _name_index: Dict[str, str] = {}  # name -> id
    _type_index: Dict[type, List[str]] = {}  # type -> [ids]

    @classmethod
    def register(cls, agent: Agent) -> None:
        """Register an agent in the registry"""
        cls._agents[agent.id] = agent
        cls._name_index[agent.name] = agent.id
        cls._type_index.setdefault(type(agent), []).append(agent.id)

    @classmethod
    def get(cls, identifier: str) -> Agent:
        """Get agent by name or ID"""
        if identifier in cls._agents:
            return cls._agents[identifier]
        if identifier in cls._name_index:
            return cls._agents[cls._name_index[identifier]]
        raise AgentNotFoundError(identifier)

    @classmethod
    def get_by_type(cls, agent_type: type) -> List[Agent]:
        """Get all agents of a specific type"""
        ids = cls._type_index.get(agent_type, [])
        return [cls._agents[id] for id in ids]
```

### SubagentRegistry

**Location:** `structs/async_subagent.py`

```python
class SubagentRegistry:
    """Registry for async subagents with status tracking"""
    _subagents: Dict[str, 'AsyncSubagent'] = {}

    @classmethod
    def register_subagent(cls, subagent: 'AsyncSubagent'):
        cls._subagents[subagent.task_id] = subagent

    @classmethod
    def get_status(cls, task_id: str) -> TaskStatus:
        return cls._subagents[task_id].status
```

---

## 5. Factory Pattern

**Purpose:** Create agents and swarms from configurations without coupling to concrete classes.

### AgentLoader

**Location:** `structs/agent_loader.py`

```python
class AgentLoader:
    """Factory for creating agents from configurations"""

    @staticmethod
    def from_config(config: Dict[str, Any]) -> Agent:
        """Create agent from configuration dict"""
        agent_type = config.get('type', 'Agent')

        if agent_type == 'ReActAgent':
            return ReActAgent(
                model=config['model'],
                system_prompt=config['system_prompt'],
                tools=config.get('tools', [])
            )
        elif agent_type == 'OpenAIAssistant':
            return OpenAIAssistant(
                api_key=config['api_key'],
                assistant_id=config['assistant_id']
            )
        # ... other types

    @classmethod
    def from_yaml(cls, path: str) -> Agent:
        """Create agent from YAML file"""
        with open(path, 'r') as f:
            config = yaml.safe_load(f)
        return cls.from_config(config)
```

### AutoSwarmBuilder

**Location:** `structs/auto_swarm_builder.py`

```python
class AutoSwarmBuilder:
    """Factory for auto-generating swarm configurations"""

    @classmethod
    def build(cls, task_description: str, available_agents: List[Agent]) -> BaseSwarm:
        """Automatically configure swarm based on task"""
        # Analyze task requirements
        # Select appropriate agents
        # Configure communication patterns
        # Return configured swarm
        pass
```

---

## 6. Builder Pattern

**Purpose:** Fluent interface for constructing complex swarm configurations.

```python
# Example from specialized swarm classes

# SpreadSheetSwarm builder pattern
swarm = (
    SpreadSheetSwarm()
    .with_agent(agent1)
    .with_agent(agent2)
    .with_output_format("csv")
    .with_max_rows(1000)
    .build()
)

# MixtureOfAgents fluent construction
moa = (
    MixtureOfAgents()
    .add_agent(expert1, weight=0.5)
    .add_agent(expert2, weight=0.3)
    .add_agent(expert3, weight=0.2)
    .with_aggregation("weighted_vote")
    .build()
)

# SkillOrchestra builder
orchestra = (
    SkillOrchestra()
    .register_skill("data_analysis", data_agent)
    .register_skill("writing", writer_agent)
    .register_skill("review", reviewer_agent)
    .with_router(router_agent)
    .build()
)
```

---

## 7. Observer/Event Pattern

**Purpose:** Notify interested parties about task status changes.

### Task Callbacks

**Location:** `structs/base_swarm.py`

```python
class BaseSwarm:
    def __init__(self, callbacks: Sequence[callable] = None):
        self.callbacks = callbacks or []

    def _notify(self, event: str, data: dict):
        """Notify all registered callbacks"""
        for callback in self.callbacks:
            callback(event, data)

    def run(self, task: str) -> str:
        self._notify("task_started", {"task": task})
        try:
            result = self._execute_task(task)
            self._notify("task_completed", {"task": task, "result": result})
            return result
        except Exception as e:
            self._notify("task_failed", {"task": task, "error": str(e)})
            raise
```

### Task Status Updates

**Location:** `structs/async_subagent.py`

```python
class TaskStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

class AsyncSubagent:
    def update_status(self, status: TaskStatus):
        """Observer notification of status change"""
        self.status = status
        for observer in self._observers:
            observer.on_status_changed(self.task_id, status)
```

---

## 8. Promise/Future Pattern

**Purpose:** Non-blocking async execution with result retrieval.

### Async Execution

**Location:** `structs/multi_agent_exec.py`

```python
async def run_agent_async(agent: Agent, task: str) -> str:
    """Execute agent task asynchronously, return immediately"""
    loop = asyncio.get_event_loop()
    future = loop.create_task(agent.arun(task))
    return await future

async def run_agents_concurrently_async(agents: List[Agent], tasks: List[str]) -> List[str]:
    """Launch multiple agents concurrently"""
    futures = [
        loop.create_task(agent.arun(task))
        for agent, task in zip(agents, tasks)
    ]
    return await asyncio.gather(*futures)

# With ThreadPoolExecutor for sync code
def run_with_executor(agent: Agent, task: str) -> Future:
    """Submit sync work to thread pool"""
    with ThreadPoolExecutor() as executor:
        return executor.submit(agent.run, task)
```

---

## 9. Hexagonal/Ports-and-Adapters Pattern

**Purpose:** Abstract tool system with swappable implementations.

### Tool Port Interface

**Location:** `tools/base_tool.py`

```python
class BaseTool:
    """
    Port: Abstract tool interface
    Adapter: Concretes implement execute()
    """

    def __init__(
        self,
        name: str,
        description: str,
        func: Callable = None,
        args_schema: type = None,
        cache_enabled: bool = False,
        cache_ttl: int = 3600
    ):
        self.name = name
        self.description = description
        self.func = func
        self.args_schema = args_schema  # Pydantic model
        self.cache_enabled = cache_enabled
        self.cache_ttl = cache_ttl

    def get_schema(self) -> dict:
        """Port: Get OpenAI function schema"""
        if self.args_schema:
            return self.pydantic_to_openai_schema(self.args_schema)
        return self.func_to_dict(self.func)

    def execute_tool(self, tool_input: dict) -> Any:
        """Port: Execute tool with given input"""
        if self.cache_enabled:
            return self._get_cached(tool_input)
        return self._execute(tool_input)

    def _execute(self, tool_input: dict) -> Any:
        """Adapter: Actual tool execution"""
        kwargs = self._validate_inputs(tool_input)
        return self.func(**kwargs)
```

### MCP Adapter

**Location:** `tools/mcp_client_tools.py`

```python
class MCPToolAdapter(BaseTool):
    """Adapter: MCP server as tool source"""

    def __init__(self, mcp_client: MCPClient, server_tool_name: str):
        self.mcp_client = mcp_client
        self.server_tool_name = server_tool_name
        super().__init__(
            name=server_tool_name,
            description=mcp_client.get_tool_description(server_tool_name),
        )

    def _execute(self, tool_input: dict) -> Any:
        """Call MCP server to execute tool"""
        return self.mcp_client.call_tool(self.server_tool_name, tool_input)
```

### Schema Conversion (Port)

```python
# From BaseTool - converting Python/Pydantic to OpenAI schema
def pydantic_to_openai_schema(self, model: type) -> dict:
    """Convert Pydantic model to OpenAI function schema"""
    schema = {
        "type": "function",
        "function": {
            "name": model.__name__,
            "description": model.__doc__ or "",
            "parameters": {
                "type": "object",
                "properties": {},
                "required": []
            }
        }
    }
    for field_name, field in model.__fields__.items():
        schema["function"]["parameters"]["properties"][field_name] = {
            "type": field.outer_type_.__name__,
            "description": field.field_info.description or ""
        }
        if field.required:
            schema["function"]["parameters"]["required"].append(field_name)
    return schema
```

---

## 10. Pipeline Pattern

**Purpose:** Sequential processing where each agent transforms output for the next.

**Location:** `structs/sequential_workflow.py`

```python
class AgentRearrange:
    """Pipeline of agents where output flows to next input"""

    def __init__(self, agents: List[Agent]):
        self.agents = agents

    def run(self, initial_input: str) -> str:
        """Execute pipeline sequentially"""
        current_output = initial_input

        for agent in self.agents:
            current_output = agent.run(current_output)

        return current_output

# Example: research -> analyze -> write -> review
pipeline = AgentRearrange([
    research_agent,
    analysis_agent,
    writing_agent,
    review_agent
])

final_output = pipeline.run(initial_topic)
```

---

## 11. Master-Worker Pattern

**Purpose:** Hierarchical task distribution with supervisor/worker relationship.

**Location:** `structs/hierarchical_swarm.py`

```python
class HierarchicalSwarm:
    """
    Master-worker hierarchy for task decomposition
    """

    def __init__(self, master_agent: Agent, worker_agents: List[Agent]):
        self.master = master_agent
        self.workers = {w.name: w for w in worker_agents}

    def run(self, task: str) -> str:
        """Master decomposes task, assigns to workers, synthesizes results"""

        # Master plans decomposition
        decomposition = self.master.run(
            f"Decompose: {task}\nWorkers: {list(self.workers.keys())}"
        )

        # Parse worker assignments
        subtasks = self._parse_subtasks(decomposition)

        # Execute subtasks on workers (potentially in parallel)
        results = {}
        for subtask, worker_name in subtasks:
            worker = self.workers[worker_name]
            results[worker_name] = worker.run(subtask)

        # Master synthesizes final output
        synthesis_prompt = f"Original task: {task}\nWorker results: {results}"
        return self.master.run(synthesis_prompt)
```

---

## 12. Graph/Visitor Pattern

**Purpose:** Flexible DAG-based task orchestration with configurable node/edge behavior.

**Location:** `structs/graph_workflow.py`

```python
class GraphWorkflow:
    """
    Graph-based workflow orchestration
    Uses NetworkX for DAG representation
    """

    def __init__(self, backend: GraphBackend = None):
        self.graph = nx.DiGraph()
        self.backend = backend or NetworkXBackend()

    def add_node(self, node_id: str, agent: Agent = None, task: str = None):
        """Add agent or task node to graph"""
        self.graph.add_node(node_id, agent=agent, task=task)

    def add_edge(self, from_node: str, to_node: str, condition: Callable = None):
        """Add dependency edge with optional conditional execution"""
        self.graph.add_edge(from_node, to_node, condition=condition)

    def execute(self, start_nodes: List[str] = None) -> dict:
        """Execute graph in topological order"""
        if start_nodes:
            # Filter to subgraph starting from start_nodes
            pass

        results = {}
        for node in nx.topological_sort(self.graph):
            node_data = self.graph.nodes[node]

            # Get inputs from predecessor nodes
            predecessors = list(self.graph.predecessors(node))
            inputs = [results[p] for p in predecessors if p in results]

            # Execute node
            if node_data.get('agent'):
                result = node_data['agent'].run(*inputs)
            else:
                result = node_data.get('task', '')

            results[node] = result

        return results

# GraphBackend abstraction for different implementations
class GraphBackend(Protocol):
    def topological_sort(self) -> List[str]: ...
    def get_predecessors(self, node: str) -> List[str]: ...

class NetworkXBackend(GraphBackend):
    def __init__(self):
        self.graph = nx.DiGraph()

    def topological_sort(self) -> List[str]:
        return list(nx.topological_sort(self.graph))

class RustworkXBackend(GraphBackend):
    """Rust-based backend for performance"""
    pass
```

---

## Pattern Usage Summary

| Pattern | Frequency | Key Benefit |
|---------|-----------|-------------|
| Strategy | High | Swap algorithms without changing structure |
| Template Method | High | Code reuse with customization points |
| Factory | Medium | Decouple creation from usage |
| Builder | Medium | Fluent complex object construction |
| Registry | Medium | Dynamic discovery and lookup |
| Observer | Medium | Decoupled event notification |
| Promise/Future | Medium | Non-blocking async execution |
| Hexagonal | Medium | Tool extensibility |
| Pipeline | High | Sequential processing flow |
| Master-Worker | Low | Hierarchical decomposition |
| Graph/Visitor | Low | Complex dependency orchestration |
| AOP | Low | Cross-cutting concern separation |

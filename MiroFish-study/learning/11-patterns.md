# MiroFish Design Patterns

Design patterns identified in the MiroFish codebase with specific file references and code evidence.

---

## 1. Service Layer Pattern

**Purpose:** Encapsulate business logic in focused classes, separate from HTTP handling.

**Evidence:** Each service class has a single responsibility with clear public API.

### SimulationManager
**File:** `backend/app/services/simulation_manager.py`

```python
class SimulationManager:
    """
    模拟管理器

    核心功能：
    1. 从Zep图谱读取实体并过滤
    2. 生成OASIS Agent Profile
    3. 使用LLM智能生成模拟配置参数
    4. 准备预设脚本所需的所有文件
    """
    def create_simulation(self, project_id: str, graph_id: str, ...) -> SimulationState: ...
    def prepare_simulation(self, simulation_id: str, simulation_requirement: str, ...) -> SimulationState: ...
    def get_simulation(self, simulation_id: str) -> Optional[SimulationState]: ...
```

### GraphBuilderService
**File:** `backend/app/services/graph_builder.py`

```python
class GraphBuilderService:
    """图谱构建服务 - 负责调用Zep API构建知识图谱"""
    def build_graph_async(self, text: str, ontology: Dict, ...) -> str: ...
    def create_graph(self, name: str) -> str: ...
    def set_ontology(self, graph_id: str, ontology: Dict[str, Any]): ...
    def add_text_batches(self, graph_id: str, chunks: List[str], ...) -> List[str]: ...
```

### ZepEntityReader
**File:** `backend/app/services/zep_entity_reader.py`

```python
class ZepEntityReader:
    """Zep实体读取与过滤服务"""
    def filter_defined_entities(self, graph_id: str, defined_entity_types: Optional[List[str]] = None, ...) -> FilteredEntities: ...
    def get_entity_with_context(self, graph_id: str, entity_uuid: str) -> Optional[EntityNode]: ...
```

---

## 2. Manager / Singleton Pattern

**Purpose:** Centralized access to instance lifecycle with class-level state.

**Evidence:** Manager classes with `@classmethod` decorators providing singleton-like access to shared state.

### ProjectManager
**File:** `backend/app/models/project.py`

```python
class ProjectManager:
    """项目管理器 - 负责项目的持久化存储和检索"""

    # 项目存储根目录
    PROJECTS_DIR = os.path.join(Config.UPLOAD_FOLDER, 'projects')

    @classmethod
    def _ensure_projects_dir(cls):
        os.makedirs(cls.PROJECTS_DIR, exist_ok=True)

    @classmethod
    def create_project(cls, name: str = "Unnamed Project") -> Project: ...

    @classmethod
    def save_project(cls, project: Project) -> None: ...

    @classmethod
    def get_project(cls, project_id: str) -> Optional[Project]: ...

    @classmethod
    def list_projects(cls, limit: int = 50) -> List[Project]: ...
```

### SimulationRunner
**File:** `backend/app/services/simulation_runner.py`

```python
class SimulationRunner:
    """模拟运行器"""

    # 内存中的运行状态
    _run_states: Dict[str, SimulationRunState] = {}
    _processes: Dict[str, subprocess.Popen] = {}
    _action_queues: Dict[str, Queue] = {}
    _monitor_threads: Dict[str, threading.Thread] = {}

    @classmethod
    def get_run_state(cls, simulation_id: str) -> Optional[SimulationRunState]: ...

    @classmethod
    def start_simulation(cls, simulation_id: str, platform: str = "parallel", ...) -> SimulationRunState: ...

    @classmethod
    def stop_simulation(cls, simulation_id: str) -> SimulationRunState: ...
```

---

## 3. Factory Pattern

**Purpose:** Create different types of responses or objects through a unified interface.

**Evidence:** LLMClient provides factory methods for different response types.

### LLMClient
**File:** `backend/app/utils/llm_client.py`

```python
class LLMClient:
    """OpenAI兼容的LLM客户端"""

    def chat(self, messages: List[Dict[str, str]], ...) -> str:
        """发送聊天请求，返回原始文本响应"""
        response = self.client.chat.completions.create(
            model=self.model_name,
            messages=messages,
            ...
        )
        return response.choices[0].message.content

    def chat_json(self, messages: List[Dict[str, str]], ...) -> Dict[str, Any]:
        """发送聊天请求并返回JSON - Factory method for JSON responses"""
        response = self.chat(
            messages=messages,
            response_format={"type": "json_object"}
        )
        # 清理markdown代码块标记
        cleaned_response = re.sub(r'^```(?:json)?\s*\n?', '', response, flags=re.IGNORECASE)
        cleaned_response = re.sub(r'\n?```\s*$', '', cleaned_response)
        return json.loads(cleaned_response.strip())
```

---

## 4. Observer Pattern

**Purpose:** Notify dependents of state changes via callback functions.

**Evidence:** Progress callbacks in simulation preparation and graph building.

### SimulationManager.prepare_simulation
**File:** `backend/app/services/simulation_manager.py`

```python
def prepare_simulation(
    self,
    simulation_id: str,
    simulation_requirement: str,
    document_text: str,
    progress_callback: Optional[callable] = None,
    ...
) -> SimulationState:
    # Stage 1: Reading entities
    if progress_callback:
        progress_callback("reading", 0, "正在连接Zep图谱...")
    # ...
    if progress_callback:
        progress_callback("reading", 30, "正在读取节点数据...")
    # ...

    # Stage 2: Generating profiles
    if progress_callback:
        progress_callback("generating_profiles", 0, "开始生成...")
    # ...

    # Stage 3: Generating config
    if progress_callback:
        progress_callback("generating_config", 0, "正在分析模拟需求...")
```

### GraphBuilderService._build_graph_worker
**File:** `backend/app/services/graph_builder.py`

```python
def add_text_batches(
    self,
    graph_id: str,
    chunks: List[str],
    batch_size: int = 3,
    progress_callback: Optional[Callable] = None
) -> List[str]:
    for i in range(0, total_chunks, batch_size):
        if progress_callback:
            progress = (i + len(batch_chunks)) / total_chunks
            progress_callback(f"发送第 {batch_num}/{total_batches} 批数据...", progress)
```

---

## 5. Strategy Pattern

**Purpose:** interchangeable algorithms for different platforms.

**Evidence:** Twitter and Reddit have distinct action spaces and execution strategies.

### Platform-specific Simulation
**File:** `backend/app/config.py`

```python
# Platform action definitions
OASIS_TWITTER_ACTIONS = ['CREATE_POST', 'LIKE_POST', 'REPOST', 'REPLY', 'FOLLOW', 'QUOTE_POST']
OASIS_REDDIT_ACTIONS = ['CREATE_POST', 'CREATE_COMMENT', 'LIKE_POST', 'LIKE_COMMENT', 'REPLY']
```

**File:** `backend/app/services/simulation_runner.py`

```python
# Platform selection in start_simulation
if platform == "twitter":
    script_name = "run_twitter_simulation.py"
    state.twitter_running = True
elif platform == "reddit":
    script_name = "run_reddit_simulation.py"
    state.reddit_running = True
else:  # parallel
    script_name = "run_parallel_simulation.py"
    state.twitter_running = True
    state.reddit_running = True
```

### Profile Generation Strategy
**File:** `backend/app/services/oasis_profile_generator.py`

Platform-specific profile formats with `output_platform` parameter controlling format.

---

## 6. Retry with Backoff Pattern

**Purpose:** Handle transient failures with exponential backoff and jitter.

### Backend Implementation
**File:** `backend/app/utils/retry.py`

```python
def retry_with_backoff(
    max_retries: int = 3,
    initial_delay: float = 1.0,
    max_delay: float = 30.0,
    backoff_factor: float = 2.0,
    jitter: bool = True,
    exceptions: Tuple[Type[Exception], ...] = (Exception,),
    on_retry: Optional[Callable[[Exception, int], None]] = None
):
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            last_exception = None
            delay = initial_delay

            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    if attempt == max_retries:
                        logger.error(f"函数 {func.__name__} 在 {max_retries} 次重试后仍失败")
                        raise

                    current_delay = min(delay, max_delay)
                    if jitter:
                        current_delay = current_delay * (0.5 + random.random())  # Jitter

                    logger.warning(f"函数 {func.__name__} 第 {attempt + 1} 次尝试失败, {current_delay:.1f}秒后重试...")
                    time.sleep(current_delay)
                    delay *= backoff_factor  # Exponential backoff

            raise last_exception
        return wrapper
    return decorator
```

### Frontend Implementation
**File:** `frontend/src/api/index.js`

```javascript
export const requestWithRetry = async (requestFn, maxRetries=3, delay=1000) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await requestFn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await delay * Math.pow(2, i);  // Exponential backoff
    }
  }
};
```

### ZepEntityReader Retry
**File:** `backend/app/services/zep_entity_reader.py`

```python
def _call_with_retry(self, func: Callable[[], T], operation_name: str, max_retries: int = 3, initial_delay: float = 2.0) -> T:
    last_exception = None
    delay = initial_delay

    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            last_exception = e
            if attempt < max_retries - 1:
                logger.warning(f"Zep {operation_name} 第 {attempt + 1} 次尝试失败: {str(e)[:100]}, {delay:.1f}秒后重试...")
                time.sleep(delay)
                delay *= 2  # Exponential backoff
            else:
                logger.error(f"Zep {operation_name} 在 {max_retries} 次尝试后仍失败")

    raise last_exception
```

---

## 7. File-based IPC Pattern

**Purpose:** Inter-process communication via filesystem for simulation control.

**File:** `backend/app/services/simulation_ipc.py`

### Architecture

```
Flask Process                          Simulation Process
     |                                        |
     |  write {command_id}.json               |
     +-------------------> ipc_commands/ -----+
     |                                        |
     |                           poll commands/
     | <-------------------------------------+
     |                                        |
     |                           process command
     |                                        |
     |  write {command_id}.json               |
     +------------------- ipc_responses/ <---+
     |                                        |
     |  read response / timeout               |
     v                                        v
```

### IPCCommand Dataclass
```python
@dataclass
class IPCCommand:
    """IPC命令"""
    command_id: str
    command_type: CommandType  # INTERVIEW, BATCH_INTERVIEW, CLOSE_ENV
    args: Dict[str, Any]
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())
```

### SimulationIPCClient
```python
class SimulationIPCClient:
    def send_command(self, command_type: CommandType, args: Dict[str, Any], timeout: float = 60.0, poll_interval: float = 0.5) -> IPCResponse:
        command_id = str(uuid.uuid4())
        command = IPCCommand(command_id=command_id, command_type=command_type, args=args)

        # Write command file
        command_file = os.path.join(self.commands_dir, f"{command_id}.json")
        with open(command_file, 'w') as f:
            json.dump(command.to_dict(), f)

        # Poll for response
        response_file = os.path.join(self.responses_dir, f"{command_id}.json")
        while time.time() - start_time < timeout:
            if os.path.exists(response_file):
                with open(response_file, 'r') as f:
                    return IPCResponse.from_dict(json.load(f))
            time.sleep(poll_interval)

        raise TimeoutError(f"等待命令响应超时 ({timeout}秒)")
```

### IPC File Structure
```
backend/uploads/simulations/{simulation_id}/
  ipc_commands/           # Commands from Flask to simulation
    {command_id}.json
  ipc_responses/          # Responses from simulation to Flask
    {command_id}.json
  env_status.json         # Environment alive status
```

---

## 8. Dataclass Value Objects

**Purpose:** Rich domain objects with methods for serialization and domain logic.

### SimulationState
**File:** `backend/app/services/simulation_manager.py`

```python
@dataclass
class SimulationState:
    """模拟状态"""
    simulation_id: str
    project_id: str
    graph_id: str
    enable_twitter: bool = True
    enable_reddit: bool = True
    status: SimulationStatus = SimulationStatus.CREATED
    entities_count: int = 0
    profiles_count: int = 0
    entity_types: List[str] = field(default_factory=list)
    config_generated: bool = False
    current_round: int = 0
    # ... more fields

    def to_dict(self) -> Dict[str, Any]:
        """完整状态字典（内部使用）"""
        return { "simulation_id": self.simulation_id, ... }

    def to_simple_dict(self) -> Dict[str, Any]:
        """简化状态字典（API返回使用）"""
        return { "simulation_id": self.simulation_id, ... }
```

### Project
**File:** `backend/app/models/project.py`

```python
@dataclass
class Project:
    """项目数据模型"""
    project_id: str
    name: str
    status: ProjectStatus
    created_at: str
    updated_at: str
    files: List[Dict[str, str]] = field(default_factory=list)
    ontology: Optional[Dict[str, Any]] = None
    graph_id: Optional[str] = None
    # ... more fields

    def to_dict(self) -> Dict[str, Any]: ...
    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> 'Project': ...
```

### EntityNode and FilteredEntities
**File:** `backend/app/services/zep_entity_reader.py`

```python
@dataclass
class EntityNode:
    """实体节点数据结构"""
    uuid: str
    name: str
    labels: List[str]
    summary: str
    attributes: Dict[str, Any]
    related_edges: List[Dict[str, Any]] = field(default_factory=list)
    related_nodes: List[Dict[str, Any]] = field(default_factory=list)

    def to_dict(self) -> Dict[str, Any]: ...
    def get_entity_type(self) -> Optional[str]: ...

@dataclass
class FilteredEntities:
    """过滤后的实体集合"""
    entities: List[EntityNode]
    entity_types: Set[str]
    total_count: int
    filtered_count: int

    def to_dict(self) -> Dict[str, Any]: ...
```

### AgentAction and SimulationRunState
**File:** `backend/app/services/simulation_runner.py`

```python
@dataclass
class AgentAction:
    """Agent动作记录"""
    round_num: int
    timestamp: str
    platform: str  # twitter / reddit
    agent_id: int
    agent_name: str
    action_type: str
    action_args: Dict[str, Any] = field(default_factory=dict)
    result: Optional[str] = None
    success: bool = True

    def to_dict(self) -> Dict[str, Any]: ...

@dataclass
class SimulationRunState:
    """模拟运行状态（实时）"""
    simulation_id: str
    runner_status: RunnerStatus = RunnerStatus.IDLE
    current_round: int = 0
    # ... more fields

    def add_action(self, action: AgentAction): ...
    def to_dict(self) -> Dict[str, Any]: ...
    def to_detail_dict(self) -> Dict[str, Any]: ...
```

---

## Pattern Summary Table

| Pattern | Files | Key Classes |
|---------|-------|-------------|
| Service Layer | `services/*.py` | SimulationManager, GraphBuilderService, ReportAgent |
| Manager/Singleton | `models/project.py`, `services/simulation_runner.py` | ProjectManager, SimulationRunner, ReportManager |
| Factory | `utils/llm_client.py` | LLMClient.chat(), LLMClient.chat_json() |
| Observer | `services/simulation_manager.py`, `services/graph_builder.py` | progress_callback parameter |
| Strategy | `services/simulation_runner.py`, `config.py` | Platform-specific scripts and action sets |
| Retry with Backoff | `utils/retry.py`, `services/zep_entity_reader.py` | @retry_with_backoff decorator, _call_with_retry |
| File-based IPC | `services/simulation_ipc.py` | SimulationIPCClient, SimulationIPCServer |
| Dataclass Value Objects | Multiple files | SimulationState, Project, EntityNode, AgentAction |

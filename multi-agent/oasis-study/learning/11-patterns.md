# OASIS Design Patterns

**Repo:** `/Users/sheldon/Documents/claw/reference/oasis`
**Generated:** 2026-03-27

---

## Pattern 1: Factory Pattern

**Intent:** Create families of related objects (agents, environments) without specifying concrete classes.

**Evidence:**

### Environment Factory
**File:** `oasis/environment/make.py`
```python
from oasis.environment.env import OasisEnv

def make(*args, **kwargs):
    obj = OasisEnv(*args, **kwargs)
    return obj
```

### Agent Graph Generators
**File:** `oasis/social_agent/agents_generator.py` lines 34-68
```python
async def generate_agents(
    agent_info_path: str,
    channel: Channel,
    model: Union[BaseModelBackend, List[BaseModelBackend]],
    start_time,
    recsys_type: str = "twitter",
    twitter: Platform = None,
    available_actions: list[ActionType] = None,
    neo4j_config: Neo4jConfig | None = None,
) -> AgentGraph:
    agent_graph = (AgentGraph() if neo4j_config is None else AgentGraph(
        backend="neo4j",
        neo4j_config=neo4j_config,
    ))
    # ... creates agents and signs them up
    return agent_graph
```

### Platform Factory
**File:** `oasis/environment/env.py` lines 71-98
```python
if isinstance(platform, DefaultPlatformType):
    if platform == DefaultPlatformType.TWITTER:
        self.channel = Channel()
        self.platform = Platform(
            db_path=database_path,
            channel=self.channel,
            recsys_type="twhin-bert",
            ...
        )
    elif platform == DefaultPlatformType.REDDIT:
        self.channel = Channel()
        self.platform = Platform(
            db_path=database_path,
            channel=self.channel,
            recsys_type="reddit",
            ...
        )
```

---

## Pattern 2: Strategy Pattern

**Intent:** Define a family of algorithms, encapsulate each one, and make them interchangeable.

**Evidence:**

### Recommendation System Selection
**File:** `oasis/social_platform/platform.py` lines 88-89
```python
self.recsys_type = RecsysType(recsys_type)
```

**File:** `oasis/social_platform/recsys.py` lines 322-330
```python
if self.recsys_type == RecsysType.RANDOM:
    new_rec_matrix = rec_sys_random(post_table, rec_matrix, max_rec_post_len)
elif self.recsys_type == RecsysType.REDDIT:
    new_rec_matrix = rec_sys_reddit(post_table, rec_matrix, max_rec_post_len)
elif self.recsys_type == RecsysType.TWITTER:
    new_rec_matrix = rec_sys_personalized_with_trace(
        user_table, post_table, trace_table, rec_matrix, max_rec_post_len, ...)
elif self.recsys_type == RecsysType.TWHIN:
    new_rec_matrix = rec_sys_personalized_twh(
        user_table, post_table, latest_post_count, max_rec_post_len, ...)
```

**RecsysType enum:**
**File:** `oasis/social_platform/typing.py`
```python
class RecsysType(Enum):
    RANDOM = "random"
    REDDIT = "reddit"
    TWITTER = "twitter"
    TWHIN = "twhin-bert"
```

---

## Pattern 3: Template Method Pattern

**Intent:** Define the skeleton of an algorithm; let subclasses override specific steps without changing structure.

**Evidence:**

### SocialEnvironment Prompt Building
**File:** `oasis/social_agent/agent_environment.py`
```python
class SocialEnvironment:
    env_template = Template(
        "$groups_env\n$posts_env\npick one you want to perform action..."
    )

    async def to_text_prompt(self, ...):
        followers_env = await self.get_followers_env()
        follows_env = await self.get_follows_env()
        posts_env = await self.get_posts_env()
        return self.env_template.substitute(
            groups_env=followers_env + follows_env,
            posts_env=posts_env,
            ...
        )
```

### Platform Action Handler Template
**File:** `oasis/social_platform/platform.py` lines 200-207
```python
async def running(self):
    while True:
        message_id, data = await self.channel.receive_from()
        agent_id, message, action = data
        action_function = getattr(self, action.value, None)
        result = await action_function(**params)
        await self.channel.send_to((message_id, agent_id, result))
```

---

## Pattern 4: Actor / Message Queue Pattern

**Intent:** Decouple sender and receiver through asynchronous message passing via queues.

**Evidence:**

### Channel Implementation
**File:** `oasis/social_platform/channel.py` lines 41-72
```python
class Channel:
    def __init__(self):
        self.receive_queue = asyncio.Queue()
        self.send_dict = AsyncSafeDict()

    async def write_to_receive_queue(self, action_info):
        message_id = str(uuid.uuid4())
        await self.receive_queue.put((message_id, action_info))
        return message_id

    async def read_from_send_queue(self, message_id):
        while True:
            if message_id in await self.send_dict.keys():
                message = await self.send_dict.pop(message_id, None)
                if message:
                    return message
            await asyncio.sleep(0.1)
```

### AsyncSafeDict (Thread-Safe Storage)
**File:** `oasis/social_platform/channel.py` lines 18-38
```python
class AsyncSafeDict:
    def __init__(self):
        self.dict = {}
        self.lock = asyncio.Lock()

    async def put(self, key, value):
        async with self.lock:
            self.dict[key] = value

    async def pop(self, key, default=None):
        async with self.lock:
            return self.dict.pop(key, default)
```

### Action Routing Through Channel
**File:** `oasis/social_agent/agent_action.py` lines 146-151
```python
async def perform_action(self, message, type):
    message_id = await self.channel.write_to_receive_queue(
        (self.agent_id, message, type))
    response = await self.channel.read_from_send_queue(message_id)
    return response[2]
```

---

## Pattern 5: Proxy / Wrapper Pattern

**Intent:** Provide a surrogate that controls access to another object.

**Evidence:**

### Neo4jHandler (Wrapper for Neo4j Driver)
**File:** `oasis/social_agent/agent_graph.py` lines 25-36
```python
class Neo4jHandler:
    def __init__(self, nei4j_config: Neo4jConfig):
        self.driver = GraphDatabase.driver(
            nei4j_config.uri,
            auth=(nei4j_config.username, nei4j_config.password),
        )
        self.driver.verify_connectivity()

    def close(self):
        self.driver.close()
```

**Neo4j Operations wrapped to match igraph interface:**
```python
def add_edge(self, src_agent_id: int, dst_agent_id: int):
    with self.driver.session() as session:
        session.write_transaction(
            self._add_and_return_edge,
            src_agent_id,
            dst_agent_id,
        )

def get_all_edges(self) -> list[tuple[int, int]]:
    with self.driver.session() as session:
        return session.read_transaction(self._get_all_edges)
```

---

## Pattern 6: Extension via Inheritance

**Intent:** Extend functionality of a base class through inheritance.

**Evidence:**

### SocialAgent extends ChatAgent
**File:** `oasis/social_agent/agent.py` lines 55-100
```python
class SocialAgent(ChatAgent):
    r"""Social Agent."""

    def __init__(self,
                 agent_id: int,
                 user_info: UserInfo,
                 channel: Channel | None = None,
                 model: Optional[Union[BaseModelBackend,
                                       List[BaseModelBackend],
                                       ModelManager]] = None,
                 agent_graph: "AgentGraph" = None,
                 available_actions: list[ActionType] = None,
                 tools: Optional[List[Union[FunctionTool, Callable]]] = None,
                 max_iteration: int = 1,
                 interview_record: bool = False):
        self.social_agent_id = agent_id
        self.user_info = user_info
        self.channel = channel or Channel()
        self.env = SocialEnvironment(SocialAction(agent_id, self.channel))
        # ... calls super().__init__() with system message
```

**Inherits from CAMEL's ChatAgent:**
- Message handling
- Memory management
- Tool calling infrastructure
- LLM interaction

---

## Pattern 7: Command Pattern

**Intent:** Encapsulate requests as objects, allowing parameterization and queuing.

**Evidence:**

### ActionType as Command Identifiers
**File:** `oasis/social_platform/typing.py`
```python
class ActionType(Enum):
    LIKE_POST = "like_post"
    CREATE_POST = "create_post"
    CREATE_COMMENT = "create_comment"
    FOLLOW = "follow"
    SEARCH_USER = "search_user"
    # ... 23 total actions
```

### Dynamic Dispatch via getattr
**File:** `oasis/social_platform/platform.py` lines 200-207
```python
action_function = getattr(self, action.value, None)
if action_function is None:
    raise ValueError(f"Unknown action: {action.value}")
result = await action_function(**params)
```

---

## Pattern 8: Observer Pattern

**Intent:** Define a one-to-many dependency so changes notify dependents.

**Evidence:**

### Agent Graph Visualization Callback
**File:** `oasis/social_agent/agent_graph.py` (igraph backend)
```python
def visualize(self, output_path: str = None):
    import matplotlib.pyplot as plt
    ...
    # Observes graph state changes and renders
```

### Trace/Audit Logging (implicit observer)
**File:** `oasis/social_platform/platform.py`
Actions write to `trace` table, which recsys observes for personalization.

---

## Pattern 9: Dataclass/Value Object Pattern

**Intent:** Group related data into immutable structures.

**Evidence:**

### Action Dataclasses
**File:** `oasis/environment/env_action.py`
```python
@dataclass
class ManualAction:
    action_type: ActionType
    action_args: Dict[str, Any]

@dataclass
class LLMAction:
    pass  # Marker class for LLM-generated actions
```

### UserInfo Configuration
**File:** `oasis/social_platform/config/__init__.py`
```python
@dataclass
class UserInfo:
    name: str
    description: str
    profile: Dict[str, Any]
    recsys_type: str = "twitter"
```

### Neo4jConfig
**File:** `oasis/social_platform/config/__init__.py`
```python
@dataclass
class Neo4jConfig:
    uri: str
    username: str
    password: str
```

---

## Pattern 10: Repository Pattern

**Intent:** Mediate between domain and data mapping layers via a collection-like interface.

**Evidence:**

### Database Helper Functions
**File:** `oasis/social_platform/database.py`
```python
def create_db(db_path) -> (conn, cursor):
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    # Create tables...
    return conn, cursor

def fetch_table_from_db(cursor, table_name) -> List[Dict]:
    cursor.execute(f"SELECT * FROM {table_name}")
    columns = [desc[0] for desc in cursor.description]
    return [dict(zip(columns, row)) for row in cursor.fetchall()]

def fetch_rec_table_as_matrix(cursor) -> List[List[int]]:
    cursor.execute("SELECT user_id, rec_post_ids FROM rec")
    # Returns matrix format for recsys
```

---

## Pattern 11: Semaphore / Resource Pooling Pattern

**Intent:** Limit concurrent access to a shared resource.

**Evidence:**

### LLM Request Throttling
**File:** `oasis/environment/env.py` lines 69-70
```python
self.llm_semaphore = asyncio.Semaphore(semaphore)
```

**Usage in step():**
```python
async with self.llm_semaphore:
    result = await agent.perform_action_by_llm(...)
```

---

## Pattern 12: Tool-Augmented LLM Agent

**Intent:** Extend LLM capabilities through function tool calling.

**Evidence:**

### SocialAction Exposes 23 Tools
**File:** `oasis/social_agent/agent_action.py`
```python
class SocialAction:
    def get_openai_function_list() -> List[FunctionTool]:
        # Returns tools for:
        # - create_post, like_post, dislike_post
        # - follow, unfollow, mute
        # - create_comment, like_comment
        # - search_user, search_post
        # - refresh, do_nothing
        # ... 23 total
```

### Tools Registered with ChatAgent
**File:** `oasis/social_agent/agent.py` lines 87-100
```python
self.action_tools = self.env.action.get_openai_function_list()
# Filter by available_actions if specified
```

---

## Pattern 13: Lazy Initialization / Singleton Cache

**Intent:** Defer expensive object creation until first use.

**Evidence:**

### Global Model Caching
**File:** `oasis/social_platform/recsys.py` lines 38-41, 64-80
```python
model = None
twhin_tokenizer = None
twhin_model = None

def get_twhin_tokenizer():
    global twhin_tokenizer
    if twhin_tokenizer is None:
        from transformers import AutoTokenizer
        twhin_tokenizer = AutoTokenizer.from_pretrained(
            pretrained_model_name_or_path="Twitter/twhin-bert-base",
            model_max_length=512)
    return twhin_tokenizer

def get_twhin_model(device):
    global twhin_model
    if twhin_model is None:
        from transformers import AutoModel
        twhin_model = AutoModel.from_pretrained(
            pretrained_model_name_or_path="Twitter/twhin-bert-base").to(device)
    return twhin_model
```

---

## Pattern 14: Pluggable Backend / Bridge Pattern

**Intent:** Separate abstraction from implementation so both can vary independently.

**Evidence:**

### AgentGraph Backend Abstraction
**File:** `oasis/social_agent/agent_graph.py` lines 157-168
```python
class AgentGraph:
    def __init__(
        self,
        backend: Literal["igraph", "neo4j"] = "igraph",
        neo4j_config: Neo4jConfig | None = None,
    ):
        if backend == "neo4j":
            self.backend_handler = Neo4jHandler(neo4j_config)
        else:
            self.backend_handler = None
        self.graph = ig.Graph(directed=True)
```

**Same public interface, different backends:**
```python
def add_edge(self, src_agent_id: int, dst_agent_id: int):
    if self.backend_handler:
        self.backend_handler.add_edge(src_agent_id, dst_agent_id)
    else:
        self.graph.add_edge(src_agent_id, dst_agent_id)
```

---

## Pattern Summary Table

| Pattern | File(s) | Key Mechanism |
|---------|---------|---------------|
| Factory | `make.py`, `agents_generator.py` | Factory functions |
| Strategy | `recsys.py` | if/elif algorithm selection |
| Template Method | `agent_environment.py`, `platform.py` | Base class + override points |
| Actor/MQ | `channel.py` | asyncio.Queue + AsyncSafeDict |
| Proxy/Wrapper | `agent_graph.py` | Neo4jHandler wrapper |
| Inheritance | `agent.py` | extends ChatAgent |
| Command | `typing.py`, `platform.py` | ActionType enum + getattr |
| Observer | `agent_graph.py` (visualize) | Callback rendering |
| Dataclass | `env_action.py`, `config/` | @dataclass |
| Repository | `database.py` | fetch_table_from_db() |
| Semaphore | `env.py` | asyncio.Semaphore |
| Tool-Augmented LLM | `agent_action.py` | FunctionTool list |
| Lazy Init | `recsys.py` | global + None check |
| Pluggable Backend | `agent_graph.py` | backend handler abstraction |

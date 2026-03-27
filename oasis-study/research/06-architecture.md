# OASIS Architecture Analysis

**Repo:** `/Users/sheldon/Documents/claw/reference/oasis`
**Generated:** 2026-03-27

---

## Architectural Pattern

**Primary Pattern:** Agent-Based Discrete Event Simulation with Layered Architecture

OASIS implements a **social media simulation platform** using CAMEL-AI agents. The architecture follows a **layered simulation pattern** with clear separation between:

1. **Simulation Orchestration Layer** (`OasisEnv`)
2. **Agent Layer** (`SocialAgent`, `AgentGraph`)
3. **Platform Layer** (`Platform`, `Channel`)
4. **Data Layer** (SQLite database, recommendation systems)

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Code                                 │
│  (examples/quick_start.py, twitter_simulation_openai.py, etc.)  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    OasisEnv (Environment)                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ - Coordinates simulation loop (reset, step, close)          │ │
│  │ - Manages asyncio concurrency with semaphore (128 limit)    │ │
│  │ - Routes actions to Platform and agents                    │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
         │                                    │
         ▼                                    ▼
┌─────────────────────┐          ┌─────────────────────────────────┐
│    Platform         │          │        AgentGraph               │
│  ┌───────────────┐  │          │  ┌───────────────────────────┐  │
│  │ - Database    │  │          │  │ - igraph/Neo4j backend   │  │
│  │ - Recsys      │  │          │  │ - Agent storage/mappings │  │
│  │ - Action      │  │          │  │ - Edge management        │  │
│  │   handlers    │  │          │  │ - Visualization         │  │
│  └───────────────┘  │          │  └───────────────────────────┘  │
│         │            │          │              │                  │
│         ▼            │          │              ▼                  │
│  ┌───────────────┐  │          │  ┌─────────────────────────────────┐
│  │   Channel     │  │◄─────────┼──│      SocialAgent               │
│  │ (Async queues)│  │          │  │  ┌───────────────────────────┐ │
│  └───────────────┘  │          │  │  │ - Extends ChatAgent      │ │
│                      │          │  │  │ - LLM-powered decisions  │ │
└──────────────────────┘          │  │  │ - Tool-based actions    │ │
                                  │  │  │ - SocialEnvironment     │ │
                                  │  │  └───────────────────────────┘ │
                                  │  └─────────────────────────────────┘
                                  │              │
                                  │              ▼
                                  │  ┌─────────────────────────────────┐
                                  │  │      SocialAction               │
                                  │  │  (23 action types as tools)    │
                                  │  └─────────────────────────────────┘
                                  └─────────────────────────────────────┘
```

---

## Component Details

### 1. Simulation Core (`oasis/environment/`)

#### OasisEnv (`env.py`)
The central orchestrator implementing the **simulation loop pattern**:

```python
class OasisEnv:
    async def reset()     # Initialize platform + sign up agents
    async def step()      # Update recsys + execute agent actions
    async def close()     # Graceful shutdown
```

**Key characteristics:**
- Uses `asyncio.Semaphore(128)` to limit concurrent LLM requests
- Dispatches actions via `asyncio.gather()` for parallel execution
- Supports both `ManualAction` (predefined) and `LLMAction` (LLM-generated)
- **Action dispatch pattern:** Maps `SocialAgent -> Action` pairs to async tasks

#### Action Types (`env_action.py`)
```python
@dataclass
class ManualAction:
    action_type: ActionType
    action_args: Dict[str, Any]

@dataclass
class LLMAction:
    # Marker class for LLM-generated actions
    pass
```

---

### 2. Agent System (`oasis/social_agent/`)

#### SocialAgent (`agent.py`)
**Inherits from:** `camel.agents.ChatAgent`

```python
class SocialAgent(ChatAgent):
    def __init__(
        self,
        agent_id: int,
        user_info: UserInfo,
        channel: Channel,
        model: BaseModelBackend,
        agent_graph: AgentGraph,
        available_actions: list[ActionType],
        tools: List[FunctionTool],
    )
```

**Key methods:**
- `perform_action_by_llm()` - LLM decides and executes action
- `perform_action_by_data()` - Execute predefined action
- `perform_interview()` - Query agent with custom prompt
- `perform_agent_graph_action()` - Sync follow/unfollow to graph

**Design pattern:** Tool-augmented LLM agent. Actions are exposed as OpenAI function tools.

#### SocialAction (`agent_action.py`)
Exposes 23 action types as callable async methods:

```python
class SocialAction:
    def get_openai_function_list() -> List[FunctionTool]
    # Returns tools for: create_post, like_post, follow, search_user, etc.

    async def perform_action(message, type) -> dict
    # Routes action through Channel to Platform
```

**Action routing via Channel:**
```python
async def perform_action(self, message, type):
    message_id = await self.channel.write_to_receive_queue(
        (self.agent_id, message, type))
    response = await self.channel.read_from_send_queue(message_id)
    return response[2]
```

#### AgentGraph (`agent_graph.py`)
Manages agent social network with **pluggable backend**:

```python
class AgentGraph:
    def __init__(
        self,
        backend: Literal["igraph", "neo4j"] = "igraph",
        neo4j_config: Neo4jConfig | None = None,
    )
```

**Backend abstraction:**
- `igraph` - In-memory directed graph (default, fast)
- `Neo4j` - Graph database via `Neo4jHandler` wrapper

**Operations:**
- `add_agent()` / `remove_agent()`
- `add_edge()` / `remove_edge()` (follow/unfollow)
- `get_agents()` / `get_edges()`
- `visualize()` (igraph only)

#### SocialEnvironment (`agent_environment.py`)
Builds prompt context for LLM decision-making:

```python
class SocialEnvironment:
    async def to_text_prompt() -> str
    # Generates: "I have X followers. After refreshing, you see posts..."
```

Uses **Template pattern** for composing environment description.

---

### 3. Platform Layer (`oasis/social_platform/`)

#### Platform (`platform.py`)
**Role:** Central message broker + database + recommendation system

```python
class Platform:
    async def running()  # Main event loop consuming from Channel
```

**Message handling pattern:**
```python
async def running(self):
    while True:
        message_id, data = await self.channel.receive_from()
        agent_id, message, action = data
        action_function = getattr(self, action.value, None)
        result = await action_function(**params)
        await self.channel.send_to((message_id, agent_id, result))
```

**Action handlers:** 30+ async methods (`sign_up`, `create_post`, `like_post`, `follow`, `search_user`, etc.)

**Key components:**
- `db` / `db_cursor` - SQLite connection
- `sandbox_clock` - Simulation time abstraction
- `pl_utils` - `PlatformUtils` helper class
- `recsys_type` - Recommendation algorithm selector

#### Channel (`channel.py`)
**Pattern:** Actor-style async message queue

```python
class Channel:
    receive_queue: asyncio.Queue      # Platform consumes from here
    send_dict: AsyncSafeDict         # Response storage by message_id
```

**Communication flow:**
1. Agent writes to `receive_queue` via `write_to_receive_queue()`
2. Platform reads, processes, writes response to `send_dict`
3. Agent polls `send_dict` via `read_from_send_queue()`

```python
async def read_from_send_queue(self, message_id):
    while True:
        if message_id in await self.send_dict.keys():
            message = await self.send_dict.pop(message_id, None)
            if message:
                return message
        await asyncio.sleep(0.1)
```

---

### 4. Recommendation System (`oasis/social_platform/recsys.py`)

**Algorithm types:**

| Type | Description |
|------|-------------|
| `RANDOM` | Uniform random selection |
| `REDDIT` | Hot score ranking (likes - dislikes + time decay) |
| `TWITTER` | Bio-post cosine similarity via SentenceTransformer |
| `TWHIN` | TWHIN-BERT embeddings + like history + recency |

**Core functions:**
```python
def rec_sys_random(post_table, rec_matrix, max_rec_post_len)
def rec_sys_reddit(post_table, rec_matrix, max_rec_post_len)
def rec_sys_personalized(user_table, post_table, trace_table, ...)
def rec_sys_personalized_twh(user_table, post_table, latest_post_count, ...)
def rec_sys_personalized_with_trace(...)
```

**Hot score algorithm** (Reddit-style):
```python
def calculate_hot_score(num_likes, num_dislikes, created_at):
    s = num_likes - num_dislikes
    order = log(max(abs(s), 1), 10)
    sign = 1 if s > 0 else -1 if s < 0 else 0
    seconds = epoch_seconds(created_at) - 1134028003
    return round(sign * order + seconds / 45000, 7)
```

---

### 5. Data Layer

#### Database Schema (`oasis/social_platform/schema/`)

**Core tables:**

| Table | Purpose |
|-------|---------|
| `user` | Agent profiles (user_id, agent_id, user_name, name, bio) |
| `post` | Content (post_id, user_id, content, num_likes, num_dislikes, num_shares) |
| `follow` | Social graph edges (follower_id, followee_id) |
| `like` / `dislike` | Post interactions |
| `comment` | Reply threads (post_id, user_id, content) |
| `comment_like` / `comment_dislike` | Comment interactions |
| `rec` | Recommendation cache (user_id, post_id) |
| `trace` | Action audit log (user_id, action, info, created_at) |
| `chat_group` / `group_members` / `group_messages` | Group functionality |
| `product` / `report` | E-commerce and moderation |

#### Database Helper (`database.py`)
```python
def create_db(db_path) -> (conn, cursor)
def fetch_table_from_db(cursor, table_name) -> List[Dict]
def fetch_rec_table_as_matrix(cursor) -> List[List[int]]
```

---

## Design Patterns in Use

### 1. Factory Pattern
**Location:** `agents_generator.py`, `environment/make.py`

```python
def make(*args, **kwargs):
    obj = OasisEnv(*args, **kwargs)
    return obj

async def generate_twitter_agent_graph(profile_path, model, available_actions) -> AgentGraph
async def generate_reddit_agent_graph(profile_path, model, available_actions) -> AgentGraph
```

### 2. Strategy Pattern
**Location:** `recsys.py`, `platform.py`

Recommendation algorithm selectable at runtime:
```python
if self.recsys_type == RecsysType.RANDOM:
    new_rec_matrix = rec_sys_random(...)
elif self.recsys_type == RecsysType.TWITTER:
    new_rec_matrix = rec_sys_personalized_with_trace(...)
elif self.recsys_type == RecsysType.TWHIN:
    new_rec_matrix = rec_sys_personalized_twh(...)
elif self.recsys_type == RecsysType.REDDIT:
    new_rec_matrix = rec_sys_reddit(...)
```

### 3. Template Method Pattern
**Location:** `agent_environment.py`

```python
class SocialEnvironment:
    env_template = Template(
        "$groups_env\n$posts_env\npick one you want to perform action..."
    )

    async def to_text_prompt(self, ...):
        followers_env = await self.get_followers_env()
        follows_env = await self.get_follows_env()
        posts_env = await self.get_posts_env()
        return self.env_template.substitute(...)
```

### 4. Actor / Message Queue Pattern
**Location:** `channel.py`

Agents and Platform communicate via async queues, never directly calling each other.

### 5. Proxy / Wrapper Pattern
**Location:** `agent_graph.py`

```python
class Neo4jHandler:
    # Wraps Neo4j driver, exposes same interface as igraph operations
    def add_edge(self, src_agent_id, dst_agent_id):
        with self.driver.session() as session:
            session.write_transaction(self._add_and_return_edge, ...)
```

### 6. Extension via Inheritance
**Location:** `agent.py`

```python
class SocialAgent(ChatAgent):
    # Extends CAMEL's ChatAgent with social media-specific behavior
    # Inherits: memory, model, messages, tool calling
```

### 7. Command Pattern
**Location:** `platform.py`

Each `ActionType` maps to an async method handler:
```python
action_function = getattr(self, action.value, None)
result = await action_function(**params)
```

---

## Communication Flow

### Agent Action Execution

```
1. SocialAgent.perform_action_by_llm()
   │
   ├─► LLM decides action (e.g., "follow user_id=5")
   │
   ├─► agent.env.action.follow(5)
   │      │
   │      └─► Channel.write_to_receive_queue((agent_id, 5, "follow"))
   │
2. Platform.running() consumes from Channel
   │
   ├─► getattr(platform, "follow")(agent_id=agent_id, followee_id=5)
   │      │
   │      └─► SQL: INSERT INTO follow VALUES (...)
   │
   └─► Platform writes result to Channel.send_dict

3. SocialAgent reads response from Channel.send_dict
```

### Recommendation Update Cycle

```
OasisEnv.step()
   │
   └─► Platform.update_rec_table()
          │
          ├─► fetch_table_from_db("user")
          ├─► fetch_table_from_db("post")
          ├─► fetch_table_from_db("trace")
          ├─► rec_sys_*() algorithm
          └─► DELETE + INSERT INTO rec
```

---

## Concurrency Model

- **Async I/O:** All database, network, and LLM calls are async
- **Semaphore limiting:** `asyncio.Semaphore(128)` caps concurrent LLM requests
- **Parallel execution:** `asyncio.gather(*tasks)` executes agent actions concurrently
- **Thread safety:** `AsyncSafeDict` with `asyncio.Lock()` for Channel's send dictionary

---

## Module Responsibilities

| Module | Responsibility |
|--------|----------------|
| `oasis/environment/` | Simulation orchestration, action dispatch |
| `oasis/social_agent/` | LLM agents, social graph, action tools |
| `oasis/social_platform/platform.py` | Platform state, database, action handlers |
| `oasis/social_platform/channel.py` | Async message passing between agents and platform |
| `oasis/social_platform/recsys.py` | Recommendation algorithms |
| `oasis/social_platform/database.py` | SQLite schema creation, queries |
| `oasis/social_platform/typing.py` | Enums: ActionType, RecsysType, DefaultPlatformType |
| `oasis/clock/` | Simulation time abstraction |

---

## Scalability Considerations

1. **1M Agent Support:** `generate_agents_100w()` uses batch SQL inserts instead of individual operations
2. **Neo4j Backend:** Optional graph database for persistent social graphs beyond memory
3. **Coarse Filtering:** TWHIN recommendation filters to 4000 posts before embedding computation
4. **Batch Operations:** `_execute_many_db_command()` for bulk inserts

---

## Dependencies

| Package | Role |
|---------|------|
| `camel-ai` | LLM agent framework (ChatAgent, tools, memory) |
| `igraph` | In-memory social graph |
| `neo4j` | Graph database (optional) |
| `sentence-transformers` | Bio/post embedding (Twitter recsys) |
| `transformers` | TWHIN-BERT model (advanced recsys) |
| `torch` | ML computations |
| `pandas` | CSV parsing for agent generation |
| `sqlite3` | Built-in database |

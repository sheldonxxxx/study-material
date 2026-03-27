# OASIS Architecture

**Repo:** `/Users/sheldon/Documents/claw/reference/oasis`
**Generated:** 2026-03-27

---

## Architectural Overview

OASIS implements a **layered agent-based discrete event simulation** for social media platforms. The architecture separates concerns into four distinct layers, each with defined responsibilities and communication protocols.

```
┌─────────────────────────────────────────────────────────────┐
│                    User Code                                 │
│   (examples/quick_start.py, twitter_simulation_openai.py)   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│         Simulation Orchestration Layer (OasisEnv)            │
│   -reset() / step() / close()                               │
│   - asyncio.Semaphore(128) concurrency limiting             │
│   - Action dispatch to Platform + AgentGraph                │
└─────────────────────────────────────────────────────────────┘
           │                                    │
           ▼                                    ▼
┌─────────────────────┐          ┌─────────────────────────────────┐
│  Platform Layer     │          │         Agent Layer             │
│  - Database (SQLite) │          │  - SocialAgent (LLM-powered)   │
│  - Recsys (4 types)  │          │  - AgentGraph (igraph/Neo4j)   │
│  - Action handlers   │          │  - SocialAction (23 tools)      │
│  - Channel (queue)   │◄────────▶│  - SocialEnvironment (prompts)  │
└─────────────────────┘          └─────────────────────────────────┘
```

---

## Architectural Decisions

### AD-1: CAMEL-AI Foundation

**Decision:** Build on CAMEL-AI framework rather than creating agents from scratch.

**Rationale:** CAMEL-AI provides proven LLM agent primitives (ChatAgent, tool calling, memory). OASIS extends these with social media-specific behavior.

**Evidence:** `oasis/social_agent/agent.py` line 55:
```python
class SocialAgent(ChatAgent):
    r"""Social Agent."""
```

### AD-2: Actor-Style Message Passing

**Decision:** Agents and Platform communicate via async message queues, not direct method calls.

**Rationale:** Decouples agent execution from platform processing. Enables simulation of network latency, platform rate limiting, and concurrent request handling.

**Evidence:** `oasis/social_platform/channel.py` lines 41-72:
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

### AD-3: Pluggable Graph Backend

**Decision:** AgentGraph supports both igraph (in-memory) and Neo4j (persistent) backends.

**Rationale:** igraph enables fast prototyping and small-to-medium simulations. Neo4j supports million-agent scale with persistent social graphs.

**Evidence:** `oasis/social_agent/agent_graph.py` lines 157-168:
```python
class AgentGraph:
    def __init__(
        self,
        backend: Literal["igraph", "neo4j"] = "igraph",
        neo4j_config: Neo4jConfig | None = None,
    )
```

### AD-4: Strategy Pattern for Recommendations

**Decision:** Four distinct recommendation algorithms selectable at runtime.

**Rationale:** Different platforms (Twitter, Reddit) require different algorithms. Strategy pattern allows swapping without modifying platform code.

**Evidence:** `oasis/social_platform/platform.py` lines 88-89:
```python
self.recsys_type = RecsysType(recsys_type)
```
Selection in `oasis/social_platform/recsys.py` lines 322-330:
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

### AD-5: Semaphore-Based LLM Concurrency Control

**Decision:** Cap concurrent LLM requests with asyncio.Semaphore(128).

**Rationale:** Prevents API rate limiting, manages costs, avoids overwhelming LLM providers.

**Evidence:** `oasis/environment/env.py` lines 69-70:
```python
self.llm_semaphore = asyncio.Semaphore(semaphore)
```

### AD-6: SQLite for Platform State

**Decision:** SQLite for all platform state (users, posts, follows, likes).

**Rationale:** Simple deployment, ACID compliance, sufficient for simulation scale (up to 1M agents).

**Evidence:** `oasis/social_platform/platform.py` line 83:
```python
self.db, self.db_cursor = create_db(self.db_path)
```

---

## Module Responsibilities

### `oasis/environment/` - Simulation Orchestration

| File | Responsibility |
|------|----------------|
| `env.py` | OasisEnv: main simulation loop (reset/step/close), action dispatch, semaphore management |
| `env_action.py` | LLMAction / ManualAction dataclasses |
| `make.py` | Factory function for OasisEnv creation |

**Public API:** `oasis/environment/make.py`:
```python
from oasis.environment.env import OasisEnv

def make(*args, **kwargs):
    obj = OasisEnv(*args, **kwargs)
    return obj
```

### `oasis/social_agent/` - LLM Agent System

| File | Responsibility |
|------|----------------|
| `agent.py` | SocialAgent: extends ChatAgent, tool-augmented LLM decisions |
| `agent_action.py` | SocialAction: exposes 23 actions as OpenAI function tools |
| `agent_environment.py` | SocialEnvironment: builds LLM prompt context |
| `agent_graph.py` | AgentGraph: manages social graph (igraph/Neo4j) |
| `agents_generator.py` | Factory functions: generate_twitter_agent_graph, generate_reddit_agent_graph |

**Key Interface:** `oasis/social_agent/agent.py` lines 58-70:
```python
def __init__(self,
             agent_id: int,
             user_info: UserInfo,
             channel: Channel,
             model: BaseModelBackend,
             agent_graph: AgentGraph,
             available_actions: list[ActionType],
             tools: List[FunctionTool], ...)
```

### `oasis/social_platform/` - Platform Simulation

| File | Responsibility |
|------|----------------|
| `platform.py` | Platform: message broker, database, action handlers (30+ methods) |
| `channel.py` | Channel: async message queue (Actor pattern) |
| `recsys.py` | Four recommendation algorithms |
| `database.py` | SQLite schema creation, query helpers |
| `typing.py` | Enums: ActionType, RecsysType, DefaultPlatformType |
| `config/` | UserInfo, Neo4jConfig dataclasses |

**Platform Action Handlers:** `oasis/social_platform/platform.py`:
```python
async def sign_up(self, user_id, name, description, profile): ...
async def create_post(self, user_id, content, action_id): ...
async def follow(self, follower_id, followee_id): ...
async def like_post(self, user_id, post_id): ...
# 26+ more handlers
```

### `oasis/clock/` - Simulation Time

| File | Responsibility |
|------|----------------|
| `clock.py` | Clock: simulation time abstraction (time dilation factor) |

---

## Communication Patterns

### Pattern 1: Request-Response via Channel

```
SocialAgent                           Platform
     │                                     │
     │  write_to_receive_queue(action)     │
     │────────────────────────────────────▶│
     │                                     │  receive_from()
     │                                     │  getattr(handler)()
     │                                     │  SQL operations
     │  send_to((message_id, result))      │
     │◀────────────────────────────────────│
     │                                     │
     │  read_from_send_queue(message_id)  │
     │────────────────────────────────────▶│
     │  return result                      │
     │◀────────────────────────────────────│
```

**Evidence:** `oasis/social_agent/agent_action.py` lines 146-151:
```python
async def perform_action(self, message, type):
    message_id = await self.channel.write_to_receive_queue(
        (self.agent_id, message, type))
    response = await self.channel.read_from_send_queue(message_id)
    return response[2]
```

### Pattern 2: Action Routing via getattr

**Mechanism:** ActionType enum values map to method names via getattr.

**Evidence:** `oasis/social_platform/platform.py` lines 200-207:
```python
async def running(self):
    while True:
        message_id, data = await self.channel.receive_from()
        agent_id, message, action = data
        action_function = getattr(self, action.value, None)
        result = await action_function(**params)
        await self.channel.send_to((message_id, agent_id, result))
```

### Pattern 3: Simulation Step Loop

```
OasisEnv.step()
     │
     ├──▶ Platform.update_rec_table()
     │         │
     │         ├── fetch user/post/trace tables
     │         ├── rec_sys_*() algorithm
     │         └── UPDATE rec table
     │
     └──▶ AgentGraph parallel execution
              │
              └──▶ asyncio.gather(*agent_tasks)
                        │
                        └──▶ SocialAgent.perform_action_by_llm()
                                  │
                                  └──▶ Channel.write_to_receive_queue()
```

**Evidence:** `oasis/environment/env.py` (step method dispatches to platform and agents)

### Pattern 4: Graph Backend Abstraction

```
AgentGraph (public interface)
     │
     ├──▶ igraph backend (default)
     │         │
     │         └── ig.Graph operations
     │
     └──▶ Neo4j backend (optional)
              │
              └──▶ Neo4jHandler wrapper
                        │
                        └──▶ Cypher queries
```

**Evidence:** `oasis/social_agent/agent_graph.py` - Neo4jHandler wraps igraph-like operations with Cypher

---

## Data Flow

### Agent Action Flow

```
1. User calls: env.step()
2. OasisEnv dispatches agent actions via asyncio.gather()
3. SocialAgent.perform_action_by_llm()
   - LLM decides action (e.g., "follow user_id=5")
   - agent.env.action.follow(5)
   - Channel.write_to_receive_queue((agent_id, 5, "follow"))
4. Platform.running() consumes from Channel
   - getattr(platform, "follow")(agent_id=agent_id, followee_id=5)
   - SQL: INSERT INTO follow VALUES (...)
5. Platform writes result to Channel.send_dict
6. SocialAgent reads response from Channel.send_dict
```

### Recommendation Update Flow

```
OasisEnv.step()
   │
   └─▶ Platform.update_rec_table()
          │
          ├─ fetch_table_from_db("user")
          ├─ fetch_table_from_db("post")
          ├─ fetch_table_from_db("trace")
          ├─ rec_sys_*() algorithm (strategy pattern)
          └─ DELETE + INSERT INTO rec
```

---

## Concurrency Model

| Mechanism | Purpose | Evidence |
|-----------|---------|----------|
| `asyncio.Semaphore(128)` | Limit concurrent LLM requests | `env.py:70` |
| `asyncio.gather(*tasks)` | Parallel agent execution | Used in `step()` |
| `AsyncSafeDict` | Thread-safe response storage | `channel.py:18-38` |
| `asyncio.Queue` | Decouple agent/platform | `channel.py:44` |

**AsyncSafeDict implementation:** `oasis/social_platform/channel.py` lines 18-38:
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

---

## Scalability Mechanisms

| Technique | Scale | Evidence |
|-----------|-------|----------|
| Batch SQL inserts | 100w+ agents | `agents_generator.py` batch operations |
| Neo4j backend | Persistent graphs | `agent_graph.py` Neo4jHandler |
| Coarse filtering (4000 posts) | TWHIN recsys | `recsys.py` pre-filter |
| Semaphore limiting | API protection | `env.py:70` |

---

## Dependency Map

```
camel-ai
    └── ChatAgent (base for SocialAgent)
    └── BaseModelBackend (LLM interface)
    └── FunctionTool (action tools)

igraph
    └── AgentGraph (igraph backend)

neo4j
    └── Neo4jHandler (Neo4j backend)

sentence-transformers
    └── rec_sys_personalized_with_trace (Twitter recsys)

transformers + torch
    └── TWHIN-BERT (twhin-bert recsys)

pandas
    └── CSV parsing for agent generation

sqlite3 (stdlib)
    └── Platform state
```

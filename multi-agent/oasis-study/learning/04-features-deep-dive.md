# OASIS Feature Deep Dive

**Project:** OASIS: Open Agent Social Interaction Simulations with One Million Agents
**Repository:** `/Users/sheldon/Documents/claw/reference/oasis`
**Analysis Date:** 2026-03-27
**Source Documents:** 05a-features-batch-1.md, 05b-features-batch-2.md, 05c-features-batch-3.md, 05d-features-batch-4.md

---

## Executive Summary

OASIS implements a million-agent social media simulation with LLM-powered decision making. The system comprises 12 major features organized into Core (essential simulation capabilities) and Secondary (extended functionality) tiers. The architecture is async-first with SQLite persistence, channel-based IPC, and CAMEL framework integration for agent management.

**Key Implementation Patterns:**
- Async/await throughout for concurrent operations
- Semaphore-based LLM request throttling (default 128 concurrent)
- Batch SQL operations for scale
- Channel-based platform-agent communication
- Global recsys state (technical debt)

---

## Core Features

### Feature 1: Massive-Scale Agent Simulation

**Priority:** Tier 1 (Critical)
**Claim:** One million LLM-powered agents
**Status:** Verified - `examples/experiment/twitter_simulation_1M_agents/` exists

#### Implementation

**Primary Entry Point:** `generate_agents_100w()` (agents_generator.py:179-345)

The million-agent path explicitly bypasses `AgentGraph` for performance:

```python
# Line 206-212: TODO comment reveals the compromise
# TODO when setting 100w agents, the agentgraph class is too slow.
# I use the list.
agent_graph = []
```

**Optimization Techniques:**

1. **Pre-computation of expensive operations** (lines 222-228):
   ```python
   previous_tweets_lists = agent_info["previous_tweets"].apply(ast.literal_eval)
   following_id_lists = agent_info["following_agentid_list"].apply(ast.literal_eval)
   ```

2. **Batch SQL operations** instead of individual inserts:
   ```python
   twitter.pl_utils._execute_many_db_command(user_insert_query, sign_up_list, commit=True)
   twitter.pl_utils._execute_many_db_command(follow_insert_query, follow_list, commit=True)
   ```

3. **List-based agent storage** instead of graph operations:
   ```python
   agent_graph.append(agent)  # Direct list append
   ```

4. **Activity thresholding** reduces LLM API calls - only agents with `active_threshold` below random roll act per timestep.

**AgentGraph Class** (agent_graph.py:175-293) supports dual backends:
- `igraph` for in-memory graphs
- `Neo4j` for distributed graph storage

#### Technical Debt

| Issue | Location | Impact |
|-------|----------|--------|
| Graph edges NOT added at 100W scale | agents_generator.py:294 | `perform_agent_graph_action()` broken at scale |
| API divergence | `generate_agents()` vs `generate_agents_100w()` | No shared interface |
| Missing column guards | lines 263-266 | Defensive coding indicates fragile data assumptions |

---

### Feature 2: Multi-Platform Simulation (Twitter + Reddit)

**Priority:** Tier 1 (Critical)
**Status:** Verified - `DefaultPlatformType` enum + platform implementations

#### Platform Configuration Differences

| Parameter | Twitter | Reddit |
|-----------|---------|--------|
| recsys_type | "twhin-bert" | "reddit" |
| refresh_rec_post_count | 2 | 5 |
| max_rec_post_len | 2 | 100 |
| following_post_count | 3 | N/A |
| show_score | False | True |
| Clock handling | `get_time_step()` | `time_transfer()` |

#### Action Space Differences

**Twitter (6 actions):**
```python
[CREATE_POST, LIKE_POST, REPOST, FOLLOW, DO_NOTHING, QUOTE_POST]
```

**Reddit (13 actions):**
```python
[LIKE_POST, DISLIKE_POST, CREATE_POST, CREATE_COMMENT, LIKE_COMMENT,
 DISLIKE_COMMENT, SEARCH_POSTS, SEARCH_USER, TREND, REFRESH, DO_NOTHING,
 FOLLOW, MUTE]
```

#### Time Abstraction

Platform handles time differently but exposes same interface:
- **Twitter**: Integer timesteps via `sandbox_clock.get_time_step()`
- **Reddit**: Wall-clock to simulation time via `sandbox_clock.time_transfer()`

#### Technical Debt

1. **Platform-specific conditionals scattered throughout** - Every action method has `if self.recsys_type == RecsysType.REDDIT` checks. Violates DRY.
2. **Magic numbers hardcoded** - `refresh_rec_post_count=2`, `max_rec_post_len=2` in env.py without explanation.
3. **No abstract base for platform-specific behavior** - Strategy pattern not used.

---

### Feature 3: LLM-Powered Agent Decision Making

**Priority:** Tier 1 (Critical)
**Status:** Verified - SocialAgent extends CAMEL ChatAgent

#### Architecture

```python
class SocialAgent(ChatAgent):  # Extends CAMEL's ChatAgent
    def __init__(self, agent_id, user_info, model, available_actions, tools, ...):
        system_message_content = self.user_info.to_system_message()
        self.action_tools = self.env.action.get_openai_function_list()
        all_tools = (tools or []) + (self.action_tools or [])
        super().__init__(system_message=system_message, model=model, tools=all_tools)
```

#### Action Decision Flow

1. **LLMAction marker** routes to `_perform_llm_action()`
2. **Context gathering** via `SocialEnvironment.to_text_prompt()`:
   ```python
   async def to_text_prompt(self, include_posts=True, include_followers=True, include_follows=True):
       followers_env = await self.get_followers_env()
       follows_env = await self.get_follows_env()
       posts_env = await self.get_posts_env()
       groups_env = await self.get_group_env()
   ```
3. **LLM call** via CAMEL's `astep()` method
4. **Tool call processing** extracts and executes actions

#### Model Support

- **Single model**: `model: BaseModelBackend`
- **Multiple models with random selection**: `model: List[BaseModelBackend]`
- **ModelManager** for complex strategies

Supported platforms: OpenAI, DeepSeek, Azure, Anthropic, Ollama, vLLM, Stub

#### Technical Debt

| Issue | Location | Impact |
|-------|----------|--------|
| `perform_agent_graph_action()` commented out for 100W | agent.py:150 | Follow/unfollow won't update graph at scale |
| No prompt length control | agent_environment.py | Token overflow risk |
| Global `db_path` access | agent_environment.py:71,89 | Testing difficulties |
| Exception as return value | agent.py:153-155 | Callers may not handle gracefully |

---

### Feature 4: Diverse Social Action Space (23 Actions)

**Priority:** Tier 1 (Critical)
**Actual Count:** 33 distinct action types (exceeds claim)

#### Action Categories

**Posting & Content Creation:**
- `CREATE_POST`, `CREATE_COMMENT`, `REPOST`, `QUOTE_POST`, `PURCHASE_PRODUCT`

**Social Engagement:**
- `LIKE_POST`, `UNLIKE_POST`, `DISLIKE_POST`, `UNDO_DISLIKE_POST`
- `LIKE_COMMENT`, `UNLIKE_COMMENT`, `DISLIKE_COMMENT`, `UNDO_DISLIKE_COMMENT`
- `FOLLOW`, `UNFOLLOW`, `MUTE`, `UNMUTE`

**Discovery & Information:**
- `SEARCH_POSTS`, `SEARCH_USER`, `REFRESH`, `TREND`

**Social Graph Management:**
- `CREATE_GROUP`, `JOIN_GROUP`, `LEAVE_GROUP`, `SEND_TO_GROUP`, `LISTEN_FROM_GROUP`

**Meta Actions:**
- `SIGNUP`, `REPORT_POST`, `INTERVIEW`, `DO_NOTHING`, `EXIT`

#### Implementation Pattern

Actions are async methods on `Platform` class with consistent pattern:

```python
async def <action_name>(self, agent_id: int, [additional_params]):
    # 1. Calculate current simulation time
    # 2. Validate and execute database operations
    # 3. Record action in trace table
    # 4. Return success/error result
```

**Key observation:** `agent_id` parameter always maps to `user_id` directly (platform.py:196). Assumes sequential registration.

#### Action Routing

`Platform.running()` uses `getattr()` for dynamic dispatch:

```python
action_function = getattr(self, action.value, None)
if action_function:
    result = await action_function(**params)
```

Adding new actions requires only defining a new async method.

#### Notable Implementation Details

1. **Repost chain handling**: `_get_post_type()` determines if a post is repost/quote and redirects likes/comments to root post.

2. **Duplicate detection varies by action**:
   - Repost: Allows quoting same post with different comment
   - Like/Dislike: Strict one-per-post-per-user

3. **`PURCHASE_PRODUCT` stands alone** - E-commerce action that doesn't fit social media theme.

4. **Report tracking is primitive** - Reports counted but only appends warning to content when threshold reached (platform_utils.py:157-160).

---

### Feature 5: Recommendation Systems

**Priority:** Tier 1 (Critical)
**Status:** Verified - recsys.py ~750 lines with 4 algorithms

#### Algorithm Types

**1. Random Recsys (`rec_sys_random`)**
- Baseline: randomly selects posts for users
- Used for Reddit when no personalization needed

**2. Reddit Hot Score (`rec_sys_reddit`)**
- Implements Reddit's "hot score" ranking algorithm
- Formula combines engagement with time decay:
  ```python
  s = num_likes - num_dislikes
  order = log(max(abs(s), 1), 10)
  sign = 1 if s > 0 else -1 if s < 0 else 0
  seconds = epoch_seconds - 1134028003  # Reddit epoch offset
  return sign * order + seconds / 45000
  ```

**3. Twitter Personalized with Trace (`rec_sys_personalized_with_trace`)**
- Encodes user bio and post content using embedding model
- Calculates cosine similarity
- Adjusts based on like/dislike history
- Swaps 10% with random posts for diversity

**4. Twhin-Bert Personalized (`rec_sys_personalized_twh`)** - Default for Twitter

**Architecture:**
- Loads `Twitter/twhin-bert-base` transformer model
- Caches model, tokenizer, user profiles, post content globally
- Filters to 4000 posts max for memory constraints

**Vector generation pipeline:**
```python
def generate_post_vector(model, tokenizer, texts, batch_size):
    all_outputs = []
    for i in range(0, len(texts), batch_size):
        batch_texts = texts[i:i + batch_size]
        batch_outputs = process_batch(model, tokenizer, batch_texts)
        all_outputs.append(batch_outputs)
    return torch.cat(all_outputs, dim=0)
```

**OpenAI fallback:**
```python
def generate_post_vector_openai(texts, batch_size=100):
    openai_embedding = OpenAIEmbedding(
        model_type=EmbeddingModelType.TEXT_EMBEDDING_3_SMALL)
```

#### Global State Issues (Critical)

```python
model = None
twhin_tokenizer = None
twhin_model = None
user_previous_post_all = {}
user_previous_post = {}
user_profiles = []
t_items = {}  # post_id -> content
u_items = {}  # user_id -> follower_count
date_score = []
```

**Issues:**
1. Memory accumulates across timesteps (`date_score` grows unbounded)
2. User profile updates append to existing profiles
3. `t_items` only grows, never clears old posts
4. `reset_globals()` exists but never called

#### Technical Debt

| Issue | Location | Severity |
|-------|----------|----------|
| Global recsys state never reset | recsys.py | High |
| Profile text grows unbounded | recsys.py:519 | Medium |
| Hardcoded 4000 post filter | recsys.py:525 | Medium |
| No GPU memory management | recsys.py | Medium |
| Missing `reset_globals()` call | recsys.py | Medium |
| Hardcoded magic numbers | recsys.py | Medium |

---

### Feature 6: Dynamic Environment State Management

**Priority:** Tier 1 (Critical)
**Status:** Verified - OasisEnv coordinates all components

#### Architecture

```
User Code
    |
    v
generate_*_agent_graph() --> AgentGraph with profiles
    |
    v
oasis.make() --> OasisEnv (creates Platform internally)
    |
    v
env.reset() --> Platform.running() task + agent registration
    |
    v
env.step(actions) --> Execute all actions concurrently
    |
    +--> Platform (handles DB, recsys, actions via channel)
    +--> AgentGraph (manages agent execution)
    +--> Clock (simulation time)
```

#### Simulation Clock

**Dual-mode design:**

1. **Twitter mode** (`get_time_step()`): Integer timesteps
   ```python
   self.platform.sandbox_clock.time_step += 1
   ```

2. **Reddit mode** (`time_transfer()`): Wall-clock to simulation time
   ```python
   time_diff = now_time - self.real_start_time
   adjusted_diff = self.k * time_diff
   return start_time + adjusted_diff
   ```

**Default clock magnification is 60x** (platform.py:77: `Clock(60)`)

#### OasisEnv Step Execution

```python
async def step(self, actions: dict[SocialAgent, Union[ManualAction, LLMAction, List[...]]]):
    # 1. Update recommendation table
    await self.platform.update_rec_table()

    # 2. Create task list for all agent actions
    tasks = []
    for agent, action in actions.items():
        if isinstance(single_action, LLMAction):
            tasks.append(self._perform_llm_action(agent))

    # 3. Execute all concurrently
    await asyncio.gather(*tasks)

    # 4. Advance clock for Twitter
    if self.platform_type == DefaultPlatformType.TWITTER:
        self.platform.sandbox_clock.time_step += 1
```

**Concurrency control:** `llm_semaphore = asyncio.Semaphore(semaphore=128)` limits concurrent LLM requests.

#### Channel-Based Communication

```python
# In platform.running():
while True:
    message_id, data = await self.channel.receive_from()
    agent_id, message, action = data
    result = await action_function(**params)
    await self.channel.send_to((message_id, agent_id, result))
```

#### Strengths

- Clean separation between environment, platform, and agents
- Async-first design enables high concurrency
- Semaphore-based backpressure prevents API overload
- Complete action trace enables offline analysis

#### Limitations

| Issue | Location | Impact |
|-------|----------|--------|
| Single-threaded platform task bottleneck | platform.py | Scalability ceiling |
| No checkpointing | env.py | Simulation loss on crash |
| Global state in recsys | recsys.py | Persists across runs |
| Synchronous OFF on SQLite | platform.py:84 | Corruption risk |
| No step error recovery | env.py:193 | Single action failure fails entire step |

---

### Feature 7: Database-Backed Persistence (SQLite)

**Priority:** Tier 1 (Critical)
**Status:** Verified - 16 tables with raw sqlite3

#### Schema Architecture

**Core Social Tables:**
- `user` - Agent profiles
- `post` - Posts with engagement metrics, original_post_id for reposts/quotes
- `comment` - Comments on posts
- `follow` - Follow relationships
- `mute` - Muted users
- `like` / `dislike` - Post ratings
- `comment_like` / `comment_dislike` - Comment ratings

**Recommendation System:**
- `rec` - User-post recommendation mappings
- `trace` - Action history

**Community:**
- `chat_group`, `group_members`, `group_message` - Group chat

**Other:**
- `product` - Product catalog
- `report` - Content reports

#### Performance Optimization

```python
# platform.py - __init__()
self.db.execute("PRAGMA synchronous = OFF")
```

#### Error Handling Pattern

```python
async def some_action(self, agent_id: int, ...) -> dict:
    try:
        return {"success": True, ...}
    except Exception as e:
        return {"success": False, "error": str(e)}
```

**Missing:** No transaction rollback, no retry logic, no connection pooling.

#### Trace Table Design Issue

The trace table uses a composite primary key that can silently fail:
```sql
PRIMARY KEY(user_id, created_at, action, info)
```

If the same user performs the same action at the same time with identical info, the insert fails or overwrites silently.

#### Technical Debt

| Issue | Location | Severity |
|-------|----------|----------|
| No schema versioning | database.py | High |
| No migrations | database.py | High |
| SQL injection risk | platform_utils.py | High |
| Composite PK in trace table | schema/trace.sql | Medium |
| In-memory database special case | platform.py:115-119 | Low |

---

## Secondary Features

### Feature 8: Group Chat / Community Features

**Priority:** Tier 2 (Extended)

#### Action Types

- `CREATE_GROUP` - Create a new group
- `JOIN_GROUP` - Join an existing group
- `LEAVE_GROUP` - Leave a group
- `SEND_TO_GROUP` - Send message to group
- `LISTEN_FROM_GROUP` - Retrieve group messages

#### Database Schema

**chat_group.sql:**
```sql
CREATE TABLE IF NOT EXISTS chat_group (
    group_id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**group_members.sql:**
```sql
CREATE TABLE IF NOT EXISTS group_members (
    group_id INTEGER NOT NULL,
    agent_id INTEGER NOT NULL,
    joined_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (group_id, agent_id),
    FOREIGN KEY (group_id) REFERENCES chat_group(group_id)
);
```

**group_message.sql:**
```sql
CREATE TABLE IF NOT EXISTS group_messages (
    message_id INTEGER PRIMARY KEY AUTOINCREMENT,
    group_id INTEGER NOT NULL,
    sender_id INTEGER NOT NULL,
    content TEXT NOT NULL,
    sent_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (group_id) REFERENCES chat_group(group_id),
    FOREIGN KEY (sender_id) REFERENCES user(agent_id)
);
```

#### Missing Functionality

1. No group message limits - All historical messages returned
2. No message editing/deletion - Immutable messages
3. No group admin controls
4. No offline message queue
5. No maximum group size limits

#### Technical Debt

| Issue | Location | Impact |
|-------|----------|--------|
| No atomic transactions | platform.py | Inconsistent state possible |
| No pagination | listen_from_group | Memory issues with large groups |

---

### Feature 9: Per-Agent Customization (Models, Tools, Prompts)

**Priority:** Tier 2 (Extended)

#### Customization Parameters

| Parameter | Type | Purpose |
|-----------|------|---------|
| `model` | `BaseModelBackend \| List[BaseModelBackend] \| ModelManager` | LLM backend(s) |
| `tools` | `List[FunctionTool \| Callable]` | Additional tools beyond social actions |
| `available_actions` | `list[ActionType]` | Subset of actions this agent can perform |
| `user_info_template` | `TextPrompt \| None` | Custom system prompt template |

#### Available Actions Filtering

```python
if not available_actions:
    self.action_tools = self.env.action.get_openai_function_list()
else:
    all_tools = self.env.action.get_openai_function_list()
    self.action_tools = [
        tool for tool in all_tools if tool.func.__name__ in [
            a.value if isinstance(a, ActionType) else a
            for a in available_actions
        ]
    ]
```

#### Limitations

1. No per-action model selection - Model is global per agent
2. No prompt versioning
3. Tools merged but not isolated
4. No tool permission system
5. `max_iteration` shared across all tools

---

### Feature 10: Interview Action (Agent-to-Agent QA)

**Priority:** Tier 2 (Extended)

#### Implementation

**Action Definition** (typing.py:44):
```python
INTERVIEW = "interview"
```

NOT included in default Twitter or Reddit actions - external-only.

**Handler** (platform.py:1348-1392):
```python
async def interview(self, agent_id: int, interview_data):
    # Dual Format Support:
    # - Old format (string): Just the prompt
    # - New format (dict): Contains prompt and response
```

**Interview ID:** `interview_id = f"{current_time}_{user_id}"`

**Trace Recording:** Stored in `trace` table with `{prompt, response?, interview_id}`.

#### Limitations

1. Response Handling: When `interview_data` is string, response is `None`
2. No validation for dict format - missing keys result in empty strings
3. Single-agent focus - no batch interview mechanism
4. Trace-only storage - no dedicated interview table

---

### Feature 11: Content Reporting / Moderation

**Priority:** Tier 2 (Extended)

#### Implementation

**Action Definition** (typing.py:27):
```python
REPORT_POST = "report_post"
```

**Handler** (platform.py:1394-1446):
```python
async def report_post(self, agent_id: int, report_message: tuple):
    post_id, report_reason = report_message
    # Increments num_reports on post
    # Stores in report table
```

**Duplicate Prevention:** Checks if report record already exists.

#### Critical Gap

**No Report Resolution Mechanism** - Reports stored but never acted upon. No mechanism to hide, delete, or penalize reported content.

```python
# The only consequence is a warning appended to content:
# platform_utils.py:157-160
if num_reports >= report_threshold:
    post.content += " [Warning: This content has been reported]"
```

#### Limitations

1. Post-only reporting - Cannot report comments
2. Freeform report reasons - No categories for analysis
3. No threshold actions - No auto-hide at N reports
4. Missing index on `post_id` in report table

---

### Feature 12: Agent Profile Generation & Visualization

**Priority:** Tier 2 (Extended)

#### Part A: Profile Generation

**Twitter Pipeline** (`generator/twitter/`):

1. **Demographics** (gen.py):
   - Age: ["13-17", "18-24", "25-34", "35-49", "50+"] with realistic distributions
   - MBTI: 16 types with realistic distribution
   - Gender: ["male", "female", "other"]

2. **RAG Enhancement** (rag.py):
   - Vector Store: Chroma with BGE-m3 embeddings
   - Profile Schema: `realname`, `username`, `bio`, `persona`
   - Topic Selection: 9 topics (Politics, Urban Legends, Business, etc.)

3. **BA Network Generation** (ba.py, network.py):
   - Barabasi-Albert preferential attachment model
   - Scale-free follow relationship generation

**Reddit Pipeline** (`generator/reddit/user_generate.py`):
- Simpler: Direct GPT-3.5-turbo calls (no RAG)
- Parallel generation with ThreadPoolExecutor (100 workers)

#### Part B: Visualization Tools

**1. Dynamic Follow Network** (`visualization/dynamic_follow_network/`):
- Neo4j export from SQLite
- Temporal views via `follow-timestamp` filter

**2. Twitter Propagation Analysis** (`visualization/twitter_simulation/align_with_real_world/`):
- `prop_graph` class builds repost propagation graph
- Metrics: depth, scale, max_breadth, structural virality

**3. Group Polarization** (`visualization/twitter_simulation/group_polarization/`):
- Compares answer extremism across simulation rounds
- Uses GPT-4o-mini for ranking

**4. Reddit Analysis** (`visualization/reddit_simulation_*`):
- Score analysis (upvoted/downvoted/control groups)
- Counterfactual analysis

#### Technical Debt

| Issue | Location | Impact |
|-------|----------|--------|
| Hardcoded paths | visualization/*.py | Brittle |
| Missing dependencies | generator/twitter/ | Cannot run |
| API key in code | generator/reddit/user_generate.py | Security risk |
| No unified pipeline | visualization/ | Fragmented |
| No real-time visualization | all | Post-hoc only |
| RAG model hardcoded to cuda:0 | rag.py | No CPU fallback |

---

## Cross-Cutting Analysis

### Architecture Strengths

1. **Async-first design** - Heavy `asyncio` usage enables high concurrency
2. **Semaphore-based backpressure** - Prevents API rate limit issues
3. **Channel abstraction** - Decouples platform from agents
4. **Batch database operations** - Critical for million-agent scale
5. **CAMEL integration** - Leverages proven agent framework
6. **Clean separation** - Agent, Platform, Environment are distinct

### Recurring Technical Debt Themes

| Theme | Manifestations |
|-------|----------------|
| Global state | recsys module, db_path access |
| Magic numbers | 4000 post limit, 128 semaphore, clock magnification |
| No error recovery | Step fails on any action failure |
| Platform conditionals | If-else chains throughout platform.py |
| Missing migrations | Schema changes require DB deletion |
| Hardcoded paths/values | Visualization, generators |

### Critical Gaps by Feature

| Feature | Critical Gap |
|---------|--------------|
| 1. Massive Scale | Graph edges not added at 100W |
| 5. Recommendation | Global state never reset |
| 7. Persistence | No schema versioning |
| 11. Reporting | No moderation action on reports |
| 12. Visualization | No real-time dashboard |

---

## Files Analyzed

| File | Lines | Purpose |
|------|-------|---------|
| `oasis/social_agent/agents_generator.py` | ~350 | Agent creation |
| `oasis/social_agent/agent_graph.py` | ~120 | Graph management |
| `oasis/social_agent/agent.py` | ~200 | SocialAgent class |
| `oasis/social_agent/agent_environment.py` | ~150 | Context gathering |
| `oasis/social_agent/agent_action.py` | ~100 | Action tools |
| `oasis/social_platform/platform.py` | ~1643 | Platform + actions |
| `oasis/social_platform/typing.py` | ~91 | Enums |
| `oasis/social_platform/recsys.py` | ~798 | Recommendation algorithms |
| `oasis/social_platform/database.py` | ~150 | DB interface |
| `oasis/social_platform/platform_utils.py` | ~263 | DB utilities |
| `oasis/environment/env.py` | ~209 | OasisEnv class |
| `oasis/environment/make.py` | ~20 | Factory |
| `oasis/clock/clock.py` | ~34 | Simulation clock |
| `oasis/social_platform/schema/*.sql` | 16 files | Schema definitions |
| `generator/twitter/gen.py`, `rag.py`, `ba.py`, `network.py` | Various | Profile generation |
| `generator/reddit/user_generate.py` | ~200 | Reddit profiles |
| `visualization/dynamic_follow_network/*.py` | Various | Neo4j export |
| `visualization/twitter_simulation/align_with_real_world/code/graph.py` | ~300 | Propagation analysis |

---

## Summary

OASIS provides a capable million-agent simulation platform with sophisticated recommendation systems and flexible agent customization. The async-first architecture and CAMEL integration are strong foundation choices. However, significant technical debt exists around global state management, error recovery, and database schema evolution that should be addressed before production use at scale.

The secondary features (group chat, interviews, reporting) appear designed for specific research studies rather than general-purpose simulation, suggesting they may need expansion for broader use cases.

---

*Deep-dive synthesis completed 2026-03-27*

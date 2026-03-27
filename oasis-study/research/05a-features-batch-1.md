# OASIS Feature Deep Dive: Batch 1 (Features 1-3)

**Project:** OASIS: Open Agent Social Interaction Simulations with One Million Agents
**Repository:** `/Users/sheldon/Documents/claw/reference/oasis`
**Analysis Date:** 2026-03-27
**Batch:** Features 1-3

---

## Feature 1: Massive-Scale Agent Simulation

### Overview
Supports simulations of up to one million LLM-powered agents, enabling studies of social media dynamics at real-world platform scale.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `oasis/social_agent/agents_generator.py` | Agent creation with 100W-optimized path |
| `oasis/social_agent/agent_graph.py` | AgentGraph class with igraph/Neo4j backends |
| `examples/experiment/twitter_simulation_1M_agents/twitter_simulation_1m.py` | 1M agent demonstration |

### Implementation Analysis

#### Primary Entry Point: `generate_agents_100w()` (agents_generator.py:179-345)

This function is specifically optimized for million-agent scale. Key design decisions:

```python
# Line 206-212: Explicitly bypasses AgentGraph for 100W scale
# TODO when setting 100w agents, the agentgraph class is too slow.
# I use the list.
agent_graph = []
```

**Optimization Techniques:**

1. **Pre-computation of expensive operations** (lines 222-228):
```python
# Before loop: parse all ast.literal_eval calls once
previous_tweets_lists = agent_info["previous_tweets"].apply(ast.literal_eval)
following_id_lists = agent_info["following_agentid_list"].apply(ast.literal_eval)
```

2. **Batch SQL operations** instead of individual inserts:
```python
# Line 307-341: Single batch insert for all users, follows, posts
twitter.pl_utils._execute_many_db_command(user_insert_query, sign_up_list, commit=True)
twitter.pl_utils._execute_many_db_command(follow_insert_query, follow_list, commit=True)
```

3. **List-based agent storage** instead of graph operations:
```python
# Line 257: Direct list append instead of AgentGraph.add_edge()
agent_graph.append(agent)
```

4. **Progress tracking** with tqdm (line 230):
```python
for agent_id in tqdm.tqdm(range(len(agent_info))):
```

#### AgentGraph Class (agent_graph.py:175-293)

Dual-backend architecture supporting both in-memory (igraph) and distributed (Neo4j) graph storage:

```python
def __init__(self, backend: Literal["igraph", "neo4j"] = "igraph", neo4j_config=None):
    if self.backend == "igraph":
        self.graph = ig.Graph(directed=True)
    else:
        self.graph = Neo4jHandler(neo4j_config)
```

Key methods: `add_agent()`, `add_edge()`, `remove_agent()`, `get_agents()`, `get_num_nodes()`, `visualize()`.

### 1M Agent Example Flow (twitter_simulation_1m.py)

```python
# Lines 127-135: Generation with 100W function
agent_graph = await generate_agents_100w(
    agent_info_path=csv_path,
    channel=twitter_channel,
    start_time=start_time,
    recsys_type=recsys_type,
    twitter=infra,
    model=models,
    available_actions=available_actions,
)

# Lines 138-162: Simulation loop with activity thresholding
for timestep in range(1, num_timesteps + 1):
    for agent in agent_graph:
        if agent.user_info.is_controllable is False:
            agent_ac_prob = random.random()
            threshold = agent.user_info.profile['other_info']['active_threshold'][...]
            if agent_ac_prob < threshold:
                tasks.append(agent.perform_action_by_llm())
```

### Clever Solutions

1. **Activity thresholding**: Only a fraction of agents act per timestep based on `active_threshold` profile data, reducing LLM API calls at scale.

2. **Controllable agents distinction**: First 197 agents (lines 153-154) have special handling with 10% action probability vs. user-defined thresholds for others.

3. **Async semaphore control**: `OasisEnv` uses `asyncio.Semaphore(semaphore=128)` to limit concurrent LLM requests (env.py:70).

### Technical Debt / Shortcuts

1. **Agent graph edges NOT added at 100W scale** (line 294 commented out):
```python
# agent_graph.add_edge(agent_id, follow_id)  # Commented out for scale
```
This means `perform_agent_graph_action()` (agent.py:296-317) which relies on graph edges will not work correctly for 100W agents.

2. **API divergence**: `generate_agents()` (line 34) uses proper AgentGraph, while `generate_agents_100w()` uses a plain list. No shared interface.

3. **Missing column guards** (lines 263-266):
```python
if 'following_count' not in agent_info.columns:
    agent_info['following_count'] = 0
if 'followers_count' not in agent_info.columns:
    agent_info['followers_count'] = 0
```
These were added as defensive coding but indicate fragile data assumptions.

### Error Handling

- Database errors caught in `sign_up()`, `create_post()`, all action methods
- Missing columns handled with default values
- Batch operations use try-except with rollback

---

## Feature 2: Multi-Platform Simulation (Twitter + Reddit)

### Overview
Simulates both Twitter and Reddit platforms with platform-specific action sets, interfaces, and social dynamics.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `oasis/social_platform/platform.py` | Platform class with Twitter/Reddit implementations |
| `oasis/social_platform/typing.py` | ActionType, DefaultPlatformType, RecsysType enums |
| `oasis/environment/env.py` | OasisEnv coordinating platform initialization |

### Implementation Analysis

#### Platform Type Enum (typing.py:88-90)

```python
class DefaultPlatformType(Enum):
    TWITTER = "twitter"
    REDDIT = "reddit"
```

#### Platform Configuration Differences

| Parameter | Twitter | Reddit |
|-----------|---------|--------|
| recsys_type | "twhin-bert" | "reddit" |
| refresh_rec_post_count | 2 | 5 |
| max_rec_post_len | 2 | 100 |
| following_post_count | 3 | N/A |
| show_score | False | True |
| allow_self_rating | True | True |
| Clock handling | `get_time_step()` | `time_transfer()` |

#### Action Space Differences (typing.py:51-78)

**Twitter default actions:**
```python
@classmethod
def get_default_twitter_actions(cls):
    return [
        cls.CREATE_POST, cls.LIKE_POST, cls.REPOST,
        cls.FOLLOW, cls.DO_NOTHING, cls.QUOTE_POST,
    ]
```

**Reddit default actions:**
```python
@classmethod
def get_default_reddit_actions(cls):
    return [
        cls.LIKE_POST, cls.DISLIKE_POST, cls.CREATE_POST,
        cls.CREATE_COMMENT, cls.LIKE_COMMENT, cls.DISLIKE_COMMENT,
        cls.SEARCH_POSTS, cls.SEARCH_USER, cls.TREND,
        cls.REFRESH, cls.DO_NOTHING, cls.FOLLOW, cls.MUTE,
    ]
```

#### Platform Initialization (platform.py:56-109)

The `Platform.__init__()` method accepts `recsys_type` which determines behavior:

```python
def __init__(self, recsys_type: str | RecsysType = "reddit", ...):
    self.recsys_type = RecsysType(recsys_type)
    # Reddit shows score (likes - dislikes), Twitter shows separate counts
    self.show_score = show_score
    # Allow self-rating on posts
    self.allow_self_rating = allow_self_rating
```

#### Recommendation System Dispatch (platform.py:328-381)

```python
async def update_rec_table(self):
    if self.recsys_type == RecsysType.RANDOM:
        new_rec_matrix = rec_sys_random(...)
    elif self.recsys_type == RecsysType.TWITTER:
        new_rec_matrix = rec_sys_personalized_with_trace(...)
    elif self.recsys_type == RecsysType.TWHIN:
        new_rec_matrix = rec_sys_personalized_twh(...)
    elif self.recsys_type == RecsysType.REDDIT:
        new_rec_matrix = rec_sys_reddit(...)
```

#### OasisEnv Platform Setup (env.py:67-117)

```python
if platform == DefaultPlatformType.TWITTER:
    self.channel = Channel()
    self.platform = Platform(
        db_path=database_path,
        channel=self.channel,
        recsys_type="twhin-bert",
        refresh_rec_post_count=2,
        max_rec_post_len=2,
        following_post_count=3,
    )
elif platform == DefaultPlatformType.REDDIT:
    self.platform = Platform(
        db_path=database_path,
        channel=self.channel,
        recsys_type="reddit",
        allow_self_rating=True,
        show_score=True,
        max_rec_post_len=100,
        refresh_rec_post_count=5,
    )
```

### Clever Solutions

1. **Time abstraction**: Platform handles time differently per platform type but exposes same interface to agents.

2. **Configurable recsys**: Different algorithms can be plugged in without changing platform code.

3. **Action filtering per platform**: `SocialAgent` constructor filters available actions based on platform type (agent.py:85-104).

### Technical Debt / Shortcuts

1. **Platform-specific conditionals scattered throughout**: Every action method has `if self.recsys_type == RecsysType.REDDIT` checks (e.g., lines 180-184, 258-264). This violates DRY and makes adding new platforms expensive.

2. **Magic numbers for platform configs**: Values like `refresh_rec_post_count=2`, `max_rec_post_len=2` are hardcoded in `env.py` without explanation.

3. **No abstract base for platform-specific behavior**: Platform differences are handled with if-else chains rather than strategy pattern.

### Error Handling

- All action methods wrapped in try-except, returning `{"success": False, "error": str(e)}`
- Duplicate action checks (e.g., follow already exists) return early with specific error messages
- Post/comment existence verified before operations

---

## Feature 3: LLM-Powered Agent Decision Making

### Overview
Agents use large language models (GPT-4, GPT-4O-MINI, vLLM) to autonomously decide social media actions based on their profiles and context.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `oasis/social_agent/agent.py` | SocialAgent class extending CAMEL ChatAgent |
| `oasis/environment/env_action.py` | LLMAction, ManualAction dataclasses |
| `oasis/social_agent/agent_environment.py` | SocialEnvironment for context gathering |
| `oasis/social_agent/agent_action.py` | SocialAction with all platform action methods |

### Implementation Analysis

#### SocialAgent Class Hierarchy (agent.py:55)

```python
class SocialAgent(ChatAgent):  # Extends CAMEL's ChatAgent
    def __init__(self, agent_id, user_info, model, available_actions, tools, ...):
        # System prompt from user profile
        system_message_content = self.user_info.to_system_message()
        system_message = BaseMessage.make_assistant_message(
            role_name="system",
            content=system_message_content,
        )
        # Action tools from SocialAction
        self.action_tools = self.env.action.get_openai_function_list()
        # Combine custom tools + action tools
        all_tools = (tools or []) + (self.action_tools or [])
        super().__init__(system_message=system_message, model=model, tools=all_tools)
```

#### Action Decision Flow

**1. LLMAction marker** (env_action.py:40-45):
```python
@dataclass
class LLMAction:
    """Represents actions generated by a Language Learning Model (LLM)."""
    def init(self):
        pass
```

**2. OasisEnv.step() routes LLMAction** (env.py:136-193):
```python
async def step(self, actions: dict[SocialAgent, Union[ManualAction, LLMAction, ...]]):
    for agent, action in actions.items():
        if isinstance(single_action, LLMAction):
            tasks.append(self._perform_llm_action(agent))

async def _perform_llm_action(self, agent):
    async with self.llm_semaphore:
        return await agent.perform_action_by_llm()
```

**3. perform_action_by_llm()** (agent.py:125-155):
```python
async def perform_action_by_llm(self):
    # Gather environment context
    env_prompt = await self.env.to_text_prompt()
    # Build user message
    user_msg = BaseMessage.make_user_message(
        role_name="User",
        content=f"Please perform social media actions after observing..."
    )
    # Call CAMEL agent
    response = await self.astep(user_msg)
    # Process tool calls
    for tool_call in response.info['tool_calls']:
        action_name = tool_call.tool_name
        args = tool_call.args
        # self.perform_agent_graph_action(action_name, args)  # Commented for 100W
    return response
```

#### SocialEnvironment Context (agent_environment.py)

The `to_text_prompt()` method (lines 118-135) assembles context from multiple sources:

```python
async def to_text_prompt(self, include_posts=True, include_followers=True, include_follows=True):
    followers_env = await self.get_followers_env()
    follows_env = await self.get_follows_env()
    posts_env = await self.get_posts_env()
    groups_env = await self.get_group_env()
    return self.env_template.substitute(
        followers_env=followers_env,
        follows_env=follows_env,
        posts_env=posts_env,
        groups_env=groups_env,
    )
```

Template (lines 49-53):
```
$groups_env
$posts_env
pick one you want to perform action that best reflect your current inclination
based on your profile and posts content. Do not limit your action in just `like`
to like posts
```

#### Action Tools Conversion (agent_action.py:28-61)

```python
def get_openai_function_list(self) -> list[FunctionTool]:
    return [
        FunctionTool(func) for func in [
            self.create_post, self.like_post, self.repost, self.quote_post,
            self.unlike_post, self.dislike_post, self.undo_dislike_post,
            self.search_posts, self.search_user, self.trend, self.refresh,
            self.do_nothing, self.create_comment, self.like_comment,
            self.dislike_comment, self.unlike_comment, self.undo_dislike_comment,
            self.follow, self.unfollow, self.mute, self.unmute,
            self.purchase_product, self.interview, self.report_post,
            self.join_group, self.leave_group, self.send_to_group,
            self.create_group, self.listen_from_group,
        ]
    ]
```

#### SocialAction.perform_action() (agent_action.py:63-67)

Channel-based communication with Platform:
```python
async def perform_action(self, message: Any, type: str):
    message_id = await self.channel.write_to_receive_queue(
        (self.agent_id, message, type))
    response = await self.channel.read_from_send_queue(message_id)
    return response[2]
```

### Model Support

- **Single model**: `model: BaseModelBackend`
- **Multiple models with random selection**: `model: List[BaseModelBackend]` with `'random_model'` scheduling (agent.py:109)
- **ModelManager for complex strategies**: `model: ModelManager`

```python
super().__init__(
    system_message=system_message,
    model=model,
    scheduling_strategy='random_model',  # Line 109
    tools=all_tools,
)
```

### Clever Solutions

1. **CAMEL framework integration**: Leverages CAMEL's ChatAgent for tool calling, memory management, and message handling.

2. **Channel-based platform communication**: Agents communicate with Platform via async channels, decoupling agent logic from platform.

3. **Flexible model configuration**: Supports single model, model lists, or ModelManager for different scaling scenarios.

4. **Action filtering**: Agents can be initialized with subset of actions via `available_actions` parameter (agent.py:85-104).

### Technical Debt / Shortcuts

1. **perform_agent_graph_action() commented out for 100W scale** (agent.py:150):
```python
# Abort graph action for if 100W Agent
# self.perform_agent_graph_action(action_name, args)
```
This means follow/unfollow won't update the graph at million-agent scale.

2. **No prompt length control**: `to_text_prompt()` doesn't limit context length, which could cause token overflow with many posts or long action histories.

3. **Direct db_path access in SocialEnvironment** (agent_environment.py:71, 89):
```python
db_path = get_db_path()  # Global state access
```
Uses global `get_db_path()` instead of injected dependency.

4. **Exception swallowed in perform_action_by_llm** (agent.py:153-155):
```python
except Exception as e:
    agent_log.error(f"Agent {self.social_agent_id} error: {e}")
    return e  # Returns exception as result, caller must handle
```

5. **TODO comments indicating incomplete implementation**:
   - Line 44 in agents_generator.py: "TODO: need update the description of args"
   - Line 206: "TODO when setting 100w agents, the agentgraph class is too slow"
   - Line 286: "TODO If we simulate 1 million agents, we can not use agent_graph class"

### Error Handling

- Try-except in `perform_action_by_llm()` logs error and returns exception
- Platform actions return `{"success": False, "error": str(e)}` dictionaries
- Channel operations can timeout or fail silently

---

## Cross-Cutting Observations

### Patterns

1. **Async-first architecture**: Heavy use of `asyncio` for concurrent operations
2. **Semaphore-based concurrency control**: Limits LLM API calls to prevent rate limiting
3. **Channel-based IPC**: Decoupled communication between components
4. **Batch database operations**: Critical for scale (1M agents)

### Technical Debt Summary

| Issue | Location | Impact |
|-------|----------|--------|
| Graph edges not added at 100W | agents_generator.py:294 | `perform_agent_graph_action()` broken at scale |
| Exception as return value | agent.py:155 | Callers may not handle gracefully |
| Global db_path access | agent_environment.py:71,89 | Testing difficulties, coupling |
| Platform if-else chains | platform.py:180,258,etc. | Adding platforms expensive |
| Magic numbers in config | env.py:82-96 | Undocumented assumptions |

### Architecture Strengths

1. **Clean separation**: Agent, Platform, Environment are distinct
2. **CAMEL integration**: Leverages proven agent framework
3. **Channel abstraction**: Enables async, decoupled communication
4. **Flexible model support**: Single, multi, or managed model strategies

---

*Analysis generated for OASIS project feature documentation*

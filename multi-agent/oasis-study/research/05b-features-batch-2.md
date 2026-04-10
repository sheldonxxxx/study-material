# OASIS Feature Analysis: Features 4-6

**Repository:** `/Users/sheldon/Documents/claw/reference/oasis`
**Analysis Date:** 2026-03-27
**Features Analyzed:** Feature 4 (Diverse Social Action Space), Feature 5 (Recommendation Systems), Feature 6 (Dynamic Environment State Management)

---

## Feature 4: Diverse Social Action Space (23 Actions)

### Overview

The OASIS simulation implements a rich social media action space with **33 distinct action types** (exceeding the 23 claimed in documentation). Actions are defined in `oasis/social_platform/typing.py` via the `ActionType` enum and implemented as async methods in `oasis/social_platform/platform.py`.

### Action Architecture

#### Action Type Hierarchy

The 33 actions break down into functional categories:

**Posting & Content Creation:**
- `CREATE_POST` - Create original posts
- `CREATE_COMMENT` - Reply to posts with comments
- `REPOST` - Share others' posts without comment
- `QUOTE_POST` - Share with added commentary
- `PURCHASE_PRODUCT` - E-commerce action (seems experimental)

**Social Engagement:**
- `LIKE_POST` / `UNLIKE_POST` / `DISLIKE_POST` / `UNDO_DISLIKE_POST`
- `LIKE_COMMENT` / `UNLIKE_COMMENT` / `DISLIKE_COMMENT` / `UNDO_DISLIKE_COMMENT`
- `FOLLOW` / `UNFOLLOW`
- `MUTE` / `UNMUTE`

**Discovery & Information:**
- `SEARCH_POSTS` - Full-text search across posts
- `SEARCH_USER` - Find users by name/username/bio
- `REFRESH` - Get personalized feed from recommendation system
- `TREND` - View trending posts

**Social Graph Management:**
- `CREATE_GROUP` / `JOIN_GROUP` / `LEAVE_GROUP`
- `SEND_TO_GROUP` / `LISTEN_FROM_GROUP`

**Meta Actions:**
- `SIGNUP` - Agent registration
- `REPORT_POST` - Content moderation
- `INTERVIEW` - Agent-to-agent Q&A
- `DO_NOTHING` - Idling action
- `EXIT` - Shutdown signal

#### Platform-Specific Action Sets

The `ActionType.get_default_twitter_actions()` and `ActionType.get_default_reddit_actions()` class methods define which actions each platform uses by default:

```python
# Twitter (6 actions)
[CREATE_POST, LIKE_POST, REPOST, FOLLOW, DO_NOTHING, QUOTE_POST]

# Reddit (13 actions)
[LIKE_POST, DISLIKE_POST, CREATE_POST, CREATE_COMMENT, LIKE_COMMENT,
 DISLIKE_COMMENT, SEARCH_POSTS, SEARCH_USER, TREND, REFRESH, DO_NOTHING,
 FOLLOW, MUTE]
```

### Action Implementation Pattern

Actions are implemented as async methods on the `Platform` class with a consistent pattern:

```python
async def <action_name>(self, agent_id: int, [additional_params]):
    # 1. Calculate current simulation time
    if self.recsys_type == RecsysType.REDDIT:
        current_time = self.sandbox_clock.time_transfer(datetime.now(), self.start_time)
    else:
        current_time = self.sandbox_clock.get_time_step()

    # 2. Validate and execute database operations
    # 3. Record action in trace table
    # 4. Return success/error result
```

**Key observation:** The `agent_id` parameter always maps to `user_id` directly (line 196 in platform.py: `user_id = agent_id`). This assumes agent registration is sequential with `user_id=agent_id+1`, which the code explicitly notes can cause issues if registration order is disrupted (comment in recsys.py lines 55-57).

### Action Routing Mechanism

The `Platform.running()` method uses `getattr()` to dynamically dispatch actions:

```python
action_function = getattr(self, action.value, None)
if action_function:
    # Build params dict and call
    result = await action_function(**params)
```

This allows adding new actions by simply defining a new async method without explicit routing logic. However, it silently fails if an action method doesn't exist (raises `ValueError`).

### Post Type Handling

A notable complexity is the handling of "repost chains" where a post can reference another post which may itself be a repost. The `_get_post_type()` utility (platform_utils.py lines 242-262) determines post type:

```python
def _get_post_type(self, post_id: int):
    # Returns: {"type": "common"|"repost"|"quote", "root_post_id": ...}
```

When liking or commenting on a repost, actions are automatically redirected to the root post:

```python
post_type_result = self.pl_utils._get_post_type(post_id)
if post_type_result['type'] == 'repost':
    post_id = post_type_result['root_post_id']  # Redirect to original
```

### Error Handling Patterns

Actions follow a consistent error handling pattern:

1. **Duplicate prevention:** Check for existing records before inserting
2. **Self-action prevention:** Optional `allow_self_rating` flag prevents self-likes/dislikes
3. **Atomic operations:** All state changes happen within try-except blocks
4. **Trace recording:** Every action is logged to the `trace` table regardless of outcome

### Interesting Implementation Details

**Duplicate detection is not idempotent for some actions:**
- Repost: Checks if user already reposted the *same* content (allows quoting same post with different comment)
- Like/Dislike: Strict duplicate prevention (one like per post per user)

**The `PURCHASE_PRODUCT` action is a standalone feature** (lines 220-256) that doesn't fit the social media theme - seems like an e-commerce prototype.

**Report tracking is primitive:** Reports are stored and counted, but there's no automatic moderation action - just a warning appended to post content when `num_reports >= report_threshold` (platform_utils.py line 157-160).

---

## Feature 5: Recommendation Systems

### Overview

OASIS implements **four distinct recommendation algorithms** in `oasis/social_platform/recsys.py` (~750 lines), with additional vector generation utilities in `oasis/social_platform/process_recsys_posts.py`.

### Recommendation Algorithm Types

#### 1. Random Recsys (`rec_sys_random`)
The simplest algorithm - randomly selects posts for users.

```python
def rec_sys_random(post_table, rec_matrix, max_rec_post_len):
    post_ids = [post['post_id'] for post in post_table]
    if len(post_ids) <= max_rec_post_len:
        new_rec_matrix = [post_ids] * len(rec_matrix)
    else:
        for _ in range(len(rec_matrix)):
            new_rec_matrix.append(random.sample(post_ids, max_rec_post_len))
```

**Use case:** Baseline comparison, Reddit when no personalization needed.

#### 2. Reddit Hot Score (`rec_sys_reddit`)
Implements Reddit's famous "hot score" ranking algorithm (referenced comment: https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9).

```python
def calculate_hot_score(num_likes, num_dislikes, created_at):
    s = num_likes - num_dislikes
    order = log(max(abs(s), 1), 10)
    sign = 1 if s > 0 else -1 if s < 0 else 0
    epoch = datetime(1970, 1, 1)
    td = created_at - epoch
    epoch_seconds_result = td.days * 86400 + td.seconds + (float(td.microseconds) / 1e6)
    seconds = epoch_seconds_result - 1134028003  # Reddit's epoch offset
    return round(sign * order + seconds / 45000, 7)
```

The score combines engagement (likes-dislikes) with time decay. All users receive the same "trending" list.

#### 3. Twitter Personalized with Trace (`rec_sys_personalized_with_trace`)
Personalized recommendations using user bio similarity, adjusted by like/dislike history.

Key steps:
1. Encode user bio and post content using embedding model
2. Calculate cosine similarity between user and posts
3. Adjust similarity based on trace (liked posts boost similar content, disliked reduces)
4. Swap 10% of recommendations with random posts for diversity

**Limitation:** Uses global `model` variable which can cause state issues.

#### 4. Twhin-Bert Personalized (`rec_sys_personalized_twh`)
The most sophisticated algorithm, using Twitter's Twhin-Bert embeddings. This is the **default for Twitter simulations**.

**Architecture:**
- Loads `Twitter/twhin-bert-base` transformer model
- Uses global caching for model, tokenizer, user profiles, post content
- Implements coarse filtering (4000 posts max) for memory constraints
- Supports optional "like score" enhancement

**The embedding pipeline:**
```python
# In process_recsys_posts.py
def generate_post_vector(model, tokenizer, texts, batch_size):
    all_outputs = []
    for i in range(0, len(texts), batch_size):
        batch_texts = texts[i:i + batch_size]
        batch_outputs = process_batch(model, tokenizer, batch_texts)
        all_outputs.append(batch_outputs)
    return torch.cat(all_outputs, dim=0)
```

**OpenAI embedding fallback:**
```python
def generate_post_vector_openai(texts, batch_size=100):
    openai_embedding = OpenAIEmbedding(
        model_type=EmbeddingModelType.TEXT_EMBEDDING_3_SMALL)
    # Processes in batches, handles empty strings
```

### Recommendation Table Architecture

The `rec` table stores user-post recommendations as a matrix. `update_rec_table()` (platform.py lines 328-398) rebuilds this table each timestep:

```python
async def update_rec_table(self):
    user_table = fetch_table_from_db(self.db_cursor, "user")
    post_table = fetch_table_from_db(self.db_cursor, "post")
    trace_table = fetch_table_from_db(self.db_cursor, "trace")
    rec_matrix = fetch_rec_table_as_matrix(self.db_cursor)

    # Route to appropriate algorithm based on recsys_type
    if self.recsys_type == RecsysType.TWHIN:
        new_rec_matrix = rec_sys_personalized_twh(...)
    elif self.recsys_type == RecsysType.REDDIT:
        new_rec_matrix = rec_sys_reddit(...)
    # etc.

    # Rebuild rec table
    sql_query = "DELETE FROM rec"
    self.pl_utils._execute_db_command(sql_query, commit=True)
    insert_values = [(user_id, post_id) for user_id in range(len(new_rec_matrix))
                                     for post_id in new_rec_matrix[user_id]]
    self.pl_utils._execute_many_db_command("INSERT INTO rec...", insert_values)
```

### Global State Issues

**Major concern:** The recsys module uses extensive global variables:

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

These are initialized once and persist across simulation runs. The `reset_globals()` function exists but is never called in the main code path.

**Issues:**
1. Memory accumulates across timesteps (date_score grows unbounded)
2. User profile updates append to existing profiles, creating unbounded growth
3. `t_items` only grows, never clears old posts
4. Parallel simulation runs could interfere

### Twhin-Bert Algorithm Details

The Twhin-Bert recommendation flow (recsys.py lines 419-606):

1. **Update user post history:**
   ```python
   user_previous_post_all[post['user_id']].append(post['content'])
   user_previous_post[post['user_id']] = post['content']
   # Append "# Recent post:" to bio for recency signal
   ```

2. **Coarse filtering:** Random sample of 4000 posts due to memory constraints

3. **Vector generation:**
   ```python
   corpus = user_profiles + filtered_posts
   all_post_vector_list = generate_post_vector(twhin_model, twhin_tokenizer, corpus)
   user_vector = all_post_vector_list[:len(user_profiles)]
   posts_vector = all_post_vector_list[len(user_profiles):]
   ```

4. **Similarity calculation:**
   ```python
   cosine_similarities = cosine_similarity(user_vector, posts_vector)
   cosine_similarities = cosine_similarities * scores  # Apply time decay
   ```

5. **Top-k selection:**
   ```python
   value, indices = torch.topk(cosine_similarities, max_rec_post_len, dim=1)
   # Map back from filtered indices to real post_ids
   ```

### Clever Optimizations

1. **Coarse filtering (4000 post limit):** Prevents O(n*m) similarity calculations from exploding
2. **Like-based boosting:** Uses trace history to boost similar content (optional feature)
3. **Recent post appended to profile:** Adds recency signal without retraining
4. **Batch insertion:** Uses `executemany` for database writes

### Technical Debt

1. **Global state:** Models and caches leak between runs
2. **No GPU memory management:** All vectors kept in memory
3. **Profile text grows unbounded:** `# Recent post:` appended each timestep
4. **Missing `reset_globals()` call:** Would prevent memory leaks
5. **Hardcoded magic numbers:** 4000 post limit, 5 like history items, 90 timestep limit

---

## Feature 6: Dynamic Environment State Management

### Overview

The environment state system coordinates agents, platform, and simulation time. Core files are `oasis/environment/env.py` (OasisEnv class) and `oasis/environment/make.py` (factory function).

### Architecture Overview

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

### Simulation Clock System

**File:** `oasis/clock/clock.py`

The `Clock` class implements simulation time acceleration:

```python
class Clock:
    def __init__(self, k: int = 1):
        self.real_start_time = datetime.now()
        self.k = k  # Time magnification factor
        self.time_step = 0

    def time_transfer(self, now_time: datetime, start_time: datetime):
        time_diff = now_time - self.real_start_time
        adjusted_diff = self.k * time_diff
        return start_time + adjusted_diff

    def get_time_step(self) -> str:
        return str(self.time_step)
```

**Key insight:** Reddit uses `time_transfer()` (wall-clock to simulation time), while Twitter uses `get_time_step()` (integer timesteps). This dual-mode design supports different platform semantics.

**Default clock magnification is 60x** (platform.py line 77: `Clock(60)`), meaning 1 real second = 60 simulation seconds.

### OasisEnv State Management

**File:** `oasis/environment/env.py`

#### Initialization (`__init__`)

```python
def __init__(self, agent_graph, platform, database_path=None, semaphore=128):
    self.agent_graph = agent_graph
    self.llm_semaphore = asyncio.Semaphore(semaphore)  # Concurrency control
```

The `semaphore` parameter limits concurrent LLM requests (default 128), critical for preventing API rate limits with large agent populations.

#### Reset Phase (`reset()`)

```python
async def reset(self):
    self.platform_task = asyncio.create_task(self.platform.running())
    self.agent_graph = await generate_custom_agents(
        channel=self.channel, agent_graph=self.agent_graph)
```

Starts the platform as a background task and registers all agents asynchronously.

#### Step Execution (`step()`)

```python
async def step(self, actions: dict[SocialAgent, Union[ManualAction, LLMAction, List[...]]]):
    # 1. Update recommendation table
    await self.platform.update_rec_table()

    # 2. Create task list for all agent actions
    tasks = []
    for agent, action in actions.items():
        if isinstance(action, list):
            for single_action in action:
                if isinstance(single_action, ManualAction):
                    if single_action.action_type == ActionType.INTERVIEW:
                        tasks.append(self._perform_interview_action(agent, prompt))
                    else:
                        tasks.append(agent.perform_action_by_data(...))
                elif isinstance(single_action, LLMAction):
                    tasks.append(self._perform_llm_action(agent))
        # ... handle non-list actions similarly

    # 3. Execute all concurrently
    await asyncio.gather(*tasks)

    # 4. Advance clock for Twitter
    if self.platform_type == DefaultPlatformType.TWITTER:
        self.platform.sandbox_clock.time_step += 1
```

**Key design:** All actions execute concurrently via `asyncio.gather()`, with LLM calls throttled by semaphore.

### Channel-Based Communication

**File:** `oasis/social_platform/channel.py`

The platform and agents communicate via an async channel:

```python
# In platform.running():
while True:
    message_id, data = await self.channel.receive_from()
    agent_id, message, action = data
    # Dispatch to action method
    result = await action_function(**params)
    await self.channel.send_to((message_id, agent_id, result))
```

This decouples the platform (which runs as a task) from the calling code.

### State Persistence

All state lives in SQLite:

| Table | Purpose |
|-------|---------|
| `user` | Agent profiles (id, name, bio, follower counts) |
| `post` | All posts with engagement metrics |
| `comment` | Comments on posts |
| `like` / `dislike` | Post ratings |
| `comment_like` / `comment_dislike` | Comment ratings |
| `follow` | Social graph edges |
| `mute` | Muted users |
| `trace` | Complete action log |
| `rec` | Recommendation matrix |
| `report` | Content reports |
| `chat_group` / `group_members` / `group_messages` | Group chat |

### Platform State Transitions

The `Platform` class maintains:

```python
self.db_path
self.recsys_type
self.sandbox_clock
self.start_time
self.channel
self.show_score  # Reddit-style combined score
self.allow_self_rating
self.refresh_rec_post_count
self.max_rec_post_len
self.following_post_count
```

### Dynamic Adaptation Patterns

1. **Rec table rebuilt each step:** `update_rec_table()` called at start of every `step()`
2. **Social graph updates:** Follow/unfollow immediately reflected in queries
3. **Engagement metrics:** Like counts updated atomically
4. **Clock progression:** Only Twitter advances integer timesteps; Reddit uses wall-clock

### Environment Factory

**File:** `oasis/environment/make.py`

```python
def make(*args, **kwargs):
    obj = OasisEnv(*args, **kwargs)
    return obj
```

Trivial wrapper - the real logic is in `OasisEnv.__init__()`.

### Key Observations

**Strengths:**
- Clean separation between environment, platform, and agents
- Async-first design enables high concurrency
- Semaphore-based backpressure prevents API overload
- Complete action trace enables offline analysis

**Limitations:**
- Single-threaded platform task could become bottleneck with many agents
- No checkpointing mechanism - simulation loss on crash
- Global state in recsys module persists across runs
- SQLite synchronous mode disabled (`PRAGMA synchronous = OFF`) for speed - risk of corruption

### Error Handling in Step

The `step()` method has no explicit error handling around `asyncio.gather()`. If any action fails, the entire step fails. This is a robustness issue for production simulations.

---

## Cross-Feature Interactions

### Action -> Recommendation Flow

When an agent creates a post:
1. `platform.create_post()` inserts into `post` table
2. `update_rec_table()` sees the new post
3. Twhin-Bert generates embedding and adds to corpus
4. Next `step()` recommendations include new post

### Recommendation -> Action Flow

When agent calls `refresh`:
1. `platform.refresh()` queries `rec` table for user's recommendations
2. Returns posts ordered by recsys relevance
3. Agent's LLM uses this context to decide next action

### Environment -> All

The `OasisEnv.step()` coordinates:
1. Rec table refresh
2. Action execution (through platform)
3. Clock advancement

---

## Technical Debt Summary

| Issue | Location | Severity |
|-------|----------|----------|
| Global recsys state never reset | recsys.py | High |
| Profile text grows unbounded | recsys.py:519 | Medium |
| Hardcoded 4000 post filter | recsys.py:525 | Medium |
| No step error recovery | env.py:193 | Medium |
| Agent/USER_ID assumption fragile | platform.py:196 | Medium |
| Synchronous OFF on SQLite | platform.py:84 | Low |
| Commented-out pdb imports | recsys.py:564, 579 | Low |

---

## Files Analyzed

| File | Purpose | Lines |
|------|---------|-------|
| `oasis/social_platform/typing.py` | ActionType/RecsysType enums | 91 |
| `oasis/social_platform/platform.py` | Platform + action implementations | 1643 |
| `oasis/social_platform/recsys.py` | Recommendation algorithms | 798 |
| `oasis/social_platform/process_recsys_posts.py` | Vector generation | 82 |
| `oasis/social_platform/platform_utils.py` | DB utilities | 263 |
| `oasis/environment/env.py` | OasisEnv main class | 209 |
| `oasis/environment/make.py` | Factory function | 20 |
| `oasis/clock/clock.py` | Simulation clock | 34 |

---

*Analysis completed for OASIS batch feature review (features 4-6)*

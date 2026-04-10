# OASIS Feature Deep Dive: Features 7-9

**Repository:** `/Users/sheldon/Documents/claw/reference/oasis`
**Analysis Date:** 2026-03-27
**Features Analyzed:**
- Feature 7: Database-Backed Persistence (SQLite)
- Feature 8: Group Chat / Community Features
- Feature 9: Per-Agent Customization (Models, Tools, Prompts)

---

## Feature 7: Database-Backed Persistence (SQLite)

### Overview

SQLite provides persistent storage for all simulation data. The implementation uses raw sqlite3 with SQL schema files, no ORM or migration framework.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `oasis/social_platform/database.py` | Database interface - creation, connection, queries |
| `oasis/social_platform/schema/*.sql` | 16 SQL schema definition files |
| `oasis/social_platform/platform_utils.py` | Database utility methods |

### Schema Architecture

The database consists of 16 tables organized by domain:

**Core Social Tables:**
- `user` - Agent profiles (user_id, agent_id, user_name, name, bio, created_at, num_followings, num_followers)
- `post` - Posts with content, likes, dislikes, shares, reports, and original_post_id for reposts/quotes
- `comment` - Comments on posts
- `follow` - Follow relationships (follower_id, followee_id)
- `mute` - Muted users
- `like` / `dislike` - Post ratings
- `comment_like` / `comment_dislike` - Comment ratings

**Recommendation System:**
- `rec` - User-post recommendation mappings (populated by recsys)
- `trace` - Action history for recsys training

**Community:**
- `chat_group` - Group metadata
- `group_members` - Group membership (composite PK: group_id, agent_id)
- `group_message` - Group messages

**Other:**
- `product` - Product catalog (for PURCHASE_PRODUCT action)
- `report` - Content reports

### Key Implementation Details

**Database Creation Pattern:**
```python
# database.py - create_db()
conn = sqlite3.connect(db_path)
cursor = conn.cursor()
# Reads each .sql file and executes via executescript()
user_sql_path = osp.join(schema_dir, USER_SCHEMA_SQL)
with open(user_sql_path, "r") as sql_file:
    user_sql_script = sql_file.read()
cursor.executescript(user_sql_script)
```

**Database Path Resolution (Line 62-74):**
```python
def get_db_path() -> str:
    env_db_path = os.environ.get("OASIS_DB_PATH")
    if env_db_path:
        return env_db_path
    # Default: {package_dir}/data/social_media.db
```

**Performance Optimization:**
```python
# platform.py - __init__()
self.db.execute("PRAGMA synchronous = OFF")
```

**Batch Operations for Recsys:**
```python
# database.py - insert_matrix_into_rec_table()
insert_values = [(user_id, post_id)
                 for user_id in range(len(matrix))
                 for post_id in new_rec_matrix[user_id]]
self.pl_utils._execute_many_db_command(
    "INSERT INTO rec (user_id, post_id) VALUES (?, ?)",
    insert_values,
    commit=True,
)
```

### Error Handling Patterns

All platform action methods follow a consistent pattern:
```python
async def some_action(self, agent_id: int, ...) -> dict:
    try:
        # Execute DB operations
        return {"success": True, ...}
    except Exception as e:
        return {"success": False, "error": str(e)}
```

**Missing:** No transaction rollback on partial failures, no retry logic, no connection pooling.

### Notable Technical Debt

1. **No schema versioning** - Schema files are applied via `executescript()` with no version tracking
2. **No migrations** - Schema changes require database deletion
3. **SQL injection risk** - String formatting in some queries (e.g., `f"SELECT * FROM {table_name}"`)
4. **In-memory database handling** - Special case in `running()` method:
   ```python
   if self.db_path == ":memory:":
       dst = sqlite3.connect("mock.db")
       with dst:
           self.db.backup(dst)
   ```
5. **Composite primary key in trace table** - `PRIMARY KEY(user_id, created_at, action, info)` can cause conflicts

### Trace Table Design

The trace table uses a composite primary key that can silently fail on duplicate actions:
```sql
PRIMARY KEY(user_id, created_at, action, info)
```

If the same user performs the same action at the same time with identical info, the insert fails silently (or overwrites). This is by design for recsys but can lose data.

---

## Feature 8: Group Chat / Community Features

### Overview

Groups enable agents to communicate in dedicated chat rooms. The implementation supports creating groups, joining/leaving, sending messages, and listening for group activity.

### Action Types

Defined in `oasis/social_platform/typing.py` (lines 45-49):
- `CREATE_GROUP` - Create a new group
- `JOIN_GROUP` - Join an existing group
- `LEAVE_GROUP` - Leave a group
- `SEND_TO_GROUP` - Send message to group
- `LISTEN_FROM_GROUP` - Retrieve group messages

### Database Schema

**chat_group.sql:**
```sql
CREATE TABLE IF NOT EXISTS `chat_group` (
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

### Implementation Flow

**create_group (platform.py lines 1497-1529):**
1. Insert group into `chat_group` table
2. Automatically add creator as first member in `group_members`
3. Record action in trace

**join_group (platform.py lines 1531-1573):**
1. Check group exists
2. Check user not already a member (prevents duplicate joins)
3. Insert into `group_members`
4. Record action in trace

**leave_group (platform.py lines 1575-1602):**
1. Verify user is a member
2. Delete from `group_members`
3. Record action in trace

**send_to_group (platform.py lines 1448-1495):**
1. Verify sender is a group member
2. Insert message into `group_messages`
3. Query other members to return in result
4. Record action in trace

**listen_from_group (platform.py lines 1604-1642):**
1. Fetch all groups (not just joined)
2. Fetch joined group IDs for current user
3. Fetch all messages from joined groups
4. Return: `{all_groups, joined_groups, messages}`

### Usage Example

From `examples/group_chat_simulation.py`:
```python
# Create group (manual action)
group_result = await env.platform.create_group(1, "AI Group")
group_id = group_result["group_id"]

# Join group (LLM-driven)
actions_2 = {
    agent: LLMAction()
    for _, agent in env.agent_graph.get_agents([1, 3, 5, 7, 9])
}
await env.step(actions_2)

# Send message (manual action)
actions_3[env.agent_graph.get_agent(1)] = ManualAction(
    action_type=ActionType.SEND_TO_GROUP,
    action_args={"group_id": group_id, "message": "DeepSeek is amazing!"},
)

# Listen and respond (LLM-driven)
actions_4 = {agent: LLMAction() for _, agent in env.agent_graph.get_agents()}
await env.step(actions_4)
```

### Missing Functionality

1. **No group message limits** - All historical messages returned, no pagination
2. **No message editing/deletion** - Immutable messages
3. **No group admin controls** - Anyone can add/remove members (via join/leave)
4. **No offline message queue** - Messages only retrieved via explicit LISTEN_FROM_GROUP
5. **No group name uniqueness enforcement**
6. **No maximum group size limits**

### Error Handling Issues

1. **send_to_group membership check bug** (line 1458):
   ```python
   check_query = ("SELECT * FROM group_members WHERE group_id = ? AND agent_id = ?")
   ```
   Uses wrong table name `group_members` instead of `group_members` (verify actual table reference)

2. **No atomic transactions** - If message insert succeeds but trace fails, inconsistent state

---

## Feature 9: Per-Agent Customization (Models, Tools, Prompts)

### Overview

Each SocialAgent can be configured independently with custom models, tools, available actions, and system prompts.

### Customization Parameters

From `oasis/social_agent/agent.py` `__init__` (lines 58-114):

| Parameter | Type | Purpose |
|-----------|------|---------|
| `model` | `BaseModelBackend \| List[BaseModelBackend] \| ModelManager` | LLM backend(s) |
| `tools` | `List[FunctionTool \| Callable]` | Additional tools beyond social actions |
| `available_actions` | `list[ActionType]` | Subset of actions this agent can perform |
| `user_info_template` | `TextPrompt \| None` | Custom system prompt template |

### Model Customization

**Per-agent model selection:**
```python
# From examples/custom_model_simulation.py
agent_1 = SocialAgent(
    agent_id=0,
    user_info=UserInfo(...),
    model=openai_model,  # Custom model per agent
    ...
)
agent_2 = SocialAgent(
    agent_id=1,
    user_info=UserInfo(...),
    model=deepseek_model,  # Different model
    ...
)
```

**Supported model platforms** (via CAMEL ModelFactory):
- OpenAI
- DeepSeek
- Azure
- Anthropic
- Ollama
- vLLM
- Stub (for testing)

**Scheduling strategy** (set in `__init__` line 109):
```python
scheduling_strategy='random_model'  # When model is a list
```

### Tool Customization

Agents can be given additional tools beyond the social media action tools:

```python
# From examples/search_tools_simulation.py
agent_2 = SocialAgent(
    agent_id=1,
    user_info=UserInfo(...),
    tools=[SearchToolkit().search_duckduckgo],  # Additional tool
    available_actions=[ActionType.CREATE_COMMENT],  # Limited actions
    max_iteration=5,
)
```

The tools are merged with action tools:
```python
all_tools = (tools or []) + (self.action_tools or [])
super().__init__(..., tools=all_tools)
```

### Prompt Customization

**Custom user_info_template:**
```python
# From examples/custom_model_simulation.py
seller_template = TextPrompt('Your aim is: {aim} Your task is: {task}')

agent_1 = SocialAgent(
    ...
    user_info_template=seller_template,
    ...
)
```

The template receives profile fields as variables:
```python
profile = {
    "aim": "Persuade people to buy `GlowPod` lamp.",
    "task": "Using roleplay to tell some story about the product.",
}
```

**UserInfo class** (`oasis/social_platform/config/user_info.py`) provides two methods:
- `to_system_message()` - Default prompt generation
- `to_custom_system_message(template)` - Template-based prompt

### Available Actions Filtering

Agents can be restricted to a subset of actions:

```python
available_actions = [
    ActionType.LIKE_POST,
    ActionType.CREATE_POST,
    ActionType.FOLLOW,
]
agent = SocialAgent(
    ...
    available_actions=available_actions,
)
```

The filtering logic (agent.py lines 85-104):
```python
if not available_actions:
    self.action_tools = self.env.action.get_openai_function_list()
else:
    all_tools = self.env.action.get_openai_function_list()
    all_possible_actions = [tool.func.__name__ for tool in all_tools]
    # Filter to only requested actions
    self.action_tools = [
        tool for tool in all_tools if tool.func.__name__ in [
            a.value if isinstance(a, ActionType) else a
            for a in available_actions
        ]
    ]
```

Warning is logged if requested action not supported:
```python
if action_name not in all_possible_actions:
    agent_log.warning(f"Action {action_name} is not supported...")
```

### Agent Configuration Summary

```python
SocialAgent(
    agent_id=0,
    user_info=UserInfo(...),           # Profile data
    user_info_template=TextPrompt,     # Custom prompt format
    channel=Channel(),                  # Message channel
    model=BaseModelBackend,            # LLM backend
    agent_graph=AgentGraph,             # Graph reference
    available_actions=[ActionType],     # Action subset
    tools=[FunctionTool],               # Additional tools
    max_iteration=1,                   # Max LLM calls per step
    interview_record=False,             # Record interview to memory
)
```

### Limitations

1. **No per-action model selection** - Model is global per agent, cannot use different models for different actions
2. **No prompt versioning** - Cannot update prompt without recreating agent
3. **Tools merged but not isolated** - Action tools + custom tools share the same tool context
4. **No tool permission system** - If agent has the tool, it can use it freely
5. **max_iteration shared** - All tools use same iteration limit

---

## Cross-Feature Interactions

### Database + Group Chat

Group chat data persists via SQLite tables (`chat_group`, `group_members`, `group_messages`). The `listen_from_group` action queries these tables to retrieve group state.

### Per-Agent Customization + Actions

The `available_actions` parameter filters which database operations an agent can perform. An agent with only `LISTEN_FROM_GROUP` in their actions cannot create posts or like content.

---

## Summary of Findings

| Feature | Strengths | Issues |
|---------|-----------|--------|
| **Persistence** | Simple, fast, schema-based | No migrations, SQL injection risks, no transaction safety |
| **Group Chat** | Complete CRUD, membership enforcement | No limits, no pagination, inconsistent error handling |
| **Per-Agent Customization** | Flexible model/tool/prompt selection | No per-action models, limited tool isolation |

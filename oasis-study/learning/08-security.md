# OASIS Security Practices

## Secrets Management

### API Key Handling

OASIS requires an LLM API key (typically OpenAI) for running simulations. The project uses **environment variables** for secrets, never hardcoded values.

### Environment Variable Pattern

```bash
# For Bash shell (Linux, macOS, Git Bash on Windows):
export OPENAI_API_KEY=<insert your OpenAI API key>

# For Windows Command Prompt:
set OPENAI_API_KEY=<insert your OpenAI API key>
```

### Database Path Configuration

The database path can be configured via environment variable:

```python
# From oasis/social_platform/database.py
def get_db_path() -> str:
    # First check if the database path is set in environment variables
    env_db_path = os.environ.get("OASIS_DB_PATH")
    if env_db_path:
        return env_db_path

    # If no environment variable is set, use the original default path
    curr_file_path = osp.abspath(__file__)
    parent_dir = osp.dirname(osp.dirname(curr_file_path))
    db_dir = osp.join(parent_dir, DB_DIR)
    os.makedirs(db_dir, exist_ok=True)
    db_path = osp.join(db_dir, DB_NAME)
    return db_path
```

## Authentication Model

### Simulated Authentication

OASIS is a **research simulator** with no real user authentication. All agents are programmatic LLM agents with predefined or generated profiles.

### Agent Identification

Agents are identified by integer IDs assigned during signup:

```python
# From oasis/social_platform/platform.py
async def sign_up(self, agent_id, user_message):
    user_name, name, bio = user_message
    # ...
    user_insert_query = (
        "INSERT INTO user (user_id, agent_id, user_name, name, "
        "bio, created_at, num_followings, num_followers) VALUES "
        "(?, ?, ?, ?, ?, ?, ?, ?)")
    # agent_id is used as both user_id and agent_id
    self.pl_utils._execute_db_command(
        user_insert_query,
        (agent_id, agent_id, user_name, name, bio, current_time, 0, 0),
        commit=True,
    )
```

## Input Validation

### SQL Injection Prevention

The codebase uses **parameterized queries** exclusively for database operations:

```python
# Good - parameterized query
like_check_query = ("SELECT * FROM 'like' WHERE post_id = ? AND user_id = ?")
self.pl_utils._execute_db_command(like_check_query, (post_id, user_id))

# The pattern is consistent across all database operations
```

### Action Argument Validation

Actions validate arguments before processing:

```python
# From platform.py - interview action
async def interview(self, agent_id: int, interview_data):
    if isinstance(interview_data, str):
        # Old format: just the prompt
        prompt = interview_data
        response = None
    else:
        # New format: dict with prompt and response
        prompt = interview_data.get("prompt", "")
        response = interview_data.get("response", "")
```

### File Path Handling

File paths are constructed safely using `os.path`:

```python
# From database.py
curr_file_path = osp.abspath(__file__)
parent_dir = osp.dirname(osp.dirname(curr_file_path))
db_dir = osp.join(parent_dir, DB_DIR)
os.makedirs(db_dir, exist_ok=True)
```

## Data Privacy

### Simulation Data Only

OASIS operates entirely on simulated data. No real user data is processed unless explicitly provided by researchers as agent profiles.

### Agent Profiles

Agent profiles are loaded from JSON files and define agent behavior:

```python
# From config/user.py
@dataclass
class UserInfo:
    user_name: str | None = None
    name: str | None = None
    description: str | None = None
    profile: dict[str, Any] | None = None
    recsys_type: str = "twitter"
    is_controllable: bool = False
```

### System Prompts

Agents receive structured system prompts that define their behavior scope:

```python
def to_twitter_system_message(self) -> str:
    system_content = f"""
# OBJECTIVE
You're a Twitter user, and I'll present you with some tweets. After you see the tweets, choose some actions from the following functions.

# SELF-DESCRIPTION
Your actions should be consistent with your self-description and personality.
{description}

# RESPONSE METHOD
Please perform actions by tool calling.
    """
    return system_content
```

## Content Safety

### Report Mechanism

The platform includes a content reporting system:

```python
async def report_post(self, agent_id: int, report_message: tuple):
    post_id, report_reason = report_message
    # Check if report already exists
    check_report_query = (
        "SELECT * FROM report WHERE user_id = ? AND post_id = ?")
    # Update report count on post
    update_reports_query = (
        "UPDATE post SET num_reports = num_reports + 1 WHERE post_id = ?")
    # Record in report table
    report_insert_query = (
        "INSERT INTO report (post_id, user_id, report_reason, "
        "created_at) VALUES (?, ?, ?, ?)")
```

### Report Threshold

A configurable report threshold exists for content moderation:

```python
# From platform.py __init__
self.report_threshold = 2
```

## Async Concurrency Control

### Semaphore for Rate Limiting

The environment uses asyncio semaphores to control concurrent LLM requests:

```python
# From environment/env.py
def __init__(
    self,
    agent_graph: AgentGraph,
    platform: Union[DefaultPlatformType, Platform],
    database_path: str = None,
    semaphore: int = 128,
) -> None:
    # Use a semaphore to limit the number of concurrent requests
    self.llm_semaphore = asyncio.Semaphore(semaphore)

async def _perform_llm_action(self, agent):
    r"""Send the request to the llm model and execute the action."""
    async with self.llm_semaphore:
        return await agent.perform_action_by_llm()
```

## Error Handling

### Graceful Error Recovery

All platform actions return consistent success/error responses:

```python
async def create_post(self, agent_id: int, content: str):
    try:
        # ... operation ...
        return {"success": True, "post_id": post_id}
    except Exception as e:
        return {"success": False, "error": str(e)}
```

### Database Cleanup

The platform properly closes database connections on exit:

```python
async def running(self):
    while True:
        message_id, data = await self.channel.receive_from()
        # ...
        if action == ActionType.EXIT:
            if self.db_path == ":memory:":
                dst = sqlite3.connect("mock.db")
                with dst:
                    self.db.backup(dst)
            self.db_cursor.close()
            self.db.close()
            break
```

## Logging for Audit

### Action Tracing

All agent actions are recorded in the trace table:

```python
# From platform_utils.py
def _record_trace(self, user_id, action, action_info, current_time=None):
    # Records every action for audit/replay
```

### Structured Logging

The project uses Python's logging module with structured output:

```python
# From agent.py
agent_log = logging.getLogger(name="social.agent")
agent_log.setLevel("DEBUG")
file_handler = logging.FileHandler(
    f"./log/social.agent-{str(now)}.log")
file_handler.setFormatter(
    logging.Formatter(
        "%(levelname)s - %(asctime)s - %(name)s - %(message)s"))
```

## Third-Party Dependencies

### Dependency Security

The project pins specific versions of dependencies:

```toml
# From pyproject.toml
[tool.poetry.dependencies]
python = ">=3.10.0,<3.12"
pandas = "2.2.2"
camel-ai = "0.2.78"
neo4j = "5.23.0"
# ...
```

### Known Dependencies

| Dependency | Version | Security Note |
|------------|---------|---------------|
| camel-ai | 0.2.78 | Core LLM agent framework |
| requests_oauthlib | 2.0.0 | OAuth handling (if used) |
| slack_sdk | 3.31.0 | Slack integration |

## Development Security Practices

### Pre-commit Security

The pre-commit configuration includes license checking:

```yaml
- repo: local
  hooks:
    - id: check-license
      name: Check License
      entry: python licenses/update_license.py . licenses/license_template.txt
      language: system
      types: [python]
```

### GitHub Actions

The project uses GitHub Actions for CI/CD with encrypted secrets management for any deployment credentials.

## Limitations

### Research Tool Warning

OASIS is explicitly a **research prototype**:
- Not audited for production security
- No warranty or support guarantees
- Users must assess suitability for their research context

### No Real Authentication

The simulator does not implement:
- User authentication (OAuth, JWT, etc.)
- Role-based access control
- API key management for end users
- Rate limiting for external consumers

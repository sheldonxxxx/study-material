# OASIS Project Topology

**Repo:** `/Users/sheldon/Documents/claw/reference/oasis`
**Generated:** 2026-03-27

## Project Overview

OASIS (Open Agent Social Interaction Simulations) is a Python-based social media simulator using CAMEL-AI agents. It supports simulations of up to one million agents on platforms like Twitter and Reddit.

**Package:** `camel-oasis` (PyPI)
**Python:** >=3.10, <3.12
**Build:** Poetry

---

## Directory Structure

```
oasis/
├── .container/              # Docker/container config
├── .github/                # GitHub workflows
├── assets/                 # Images, banners, logos
├── data/                   # Sample data (reddit/, twitter/)
│   ├── reddit/
│   └── twitter/
├── deploy.py               # Deployment script
├── docs/                   # Documentation (mdx format)
├── generator/              # User profile generators
│   ├── reddit/
│   │   └── user_generate.py
│   └── twitter/
│       ├── __init__.py
│       ├── ba.py           # (ba = brand? agent?)
│       ├── gen.py          # Main generator
│       ├── network.py
│       ├── rag.py
│       └── requirement.txt
├── licenses/               # License files
├── log/                    # Runtime logs
├── oasis/                  # MAIN PACKAGE
│   ├── __init__.py         # Package entry (exports public API)
│   ├── clock/              # Simulation clock
│   ├── environment/        # Core env (OasisEnv)
│   │   ├── __init__.py
│   │   ├── env.py          # OasisEnv class
│   │   ├── env_action.py   # LLMAction, ManualAction
│   │   └── make.py         # Factory function
│   ├── social_agent/       # Agent system
│   │   ├── __init__.py
│   │   ├── agent.py        # SocialAgent
│   │   ├── agent_action.py  # SocialAction
│   │   ├── agent_environment.py
│   │   ├── agent_graph.py  # AgentGraph
│   │   └── agents_generator.py
│   ├── social_platform/    # Platform simulation
│   │   ├── __init__.py
│   │   ├── channel.py
│   │   ├── config/         # UserInfo config
│   │   ├── database.py
│   │   ├── platform.py     # Platform base class
│   │   ├── platform_utils.py
│   │   ├── process_recsys_posts.py
│   │   ├── recsys.py       # Recommendation systems
│   │   ├── schema/         # DB schema definitions
│   │   └── typing.py       # ActionType, DefaultPlatformType, etc.
│   └── testing/            # Testing utilities
├── examples/               # Usage examples
│   ├── experiment/         # Experiment configs
│   ├── quick_start.py
│   ├── twitter_*.py        # Twitter simulation examples
│   ├── reddit_*.py         # Reddit simulation examples
│   └── group_chat_*.py    # Group chat examples
├── test/                   # Test suite
│   ├── agent/              # Agent tests
│   ├── conftest.py
│   ├── infra/
│   │   ├── database/      # DB tests
│   │   └── recsys/         # Recsys tests
│   └── test_data/
├── visualization/          # Data visualization tools
├── pyproject.toml
├── poetry.lock
└── README.md
```

---

## Key Entry Points

### 1. Package Import Entry
**File:** `oasis/__init__.py`

```python
from oasis.environment.make import make
from oasis.social_platform.platform import Platform
from oasis.social_platform.typing import ActionType, DefaultPlatformType
from oasis.environment.env_action import LLMAction, ManualAction
from oasis.social_agent.agent import SocialAgent
from oasis.social_agent.agent_graph import AgentGraph
from oasis.social_agent import generate_reddit_agent_graph, generate_twitter_agent_graph
```

**Public API:**
- `make()` - Factory to create OasisEnv
- `Platform` - Base platform class
- `ActionType` - Enum of available actions
- `DefaultPlatformType` - Platform type enum
- `LLMAction`, `ManualAction` - Action types
- `AgentGraph`, `SocialAgent` - Agent classes
- `generate_reddit_agent_graph()`, `generate_twitter_agent_graph()`

### 2. Environment Factory
**File:** `oasis/environment/make.py`

```python
from oasis.environment.env import OasisEnv

def make(*args, **kwargs):
    obj = OasisEnv(*args, **kwargs)
    return obj
```

**Role:** Creates `OasisEnv` instances. The main simulation environment.

### 3. Core Environment
**File:** `oasis/environment/env.py`

```python
class OasisEnv:
    def __init__(...)
    async def reset(...)
    async def step(...)
    async def close(...)
```

**Role:** Main simulation loop. Coordinates agents, platform, and recsys.

### 4. Platform Base
**File:** `oasis/social_platform/platform.py`

```python
class Platform:
    def __init__(self, platform_type, ...)
    # Handles database, recsys, user actions
```

**Role:** Platform simulation (Twitter/Reddit) with database and recommendation systems.

### 5. Agent Entry
**File:** `oasis/social_agent/agent.py`

```python
class SocialAgent:
    def __init__(self, agent_id, model, ...)
    async def step(...)
```

**Role:** Individual LLM-powered social media agent.

### 6. Graph Entry
**File:** `oasis/social_agent/agent_graph.py`

```python
class AgentGraph:
    # Manages multiple agents, generates graphs
```

**Role:** Manages multiple agents, handles graph-based agent relationships.

### 7. Agent Generators
**Files:** `oasis/social_agent/agents_generator.py`

Key functions:
- `generate_twitter_agent_graph()`
- `generate_reddit_agent_graph()`
- `generate_custom_agents()`

**Role:** Factory functions to create agent populations from profiles.

---

## Data Flow (High-Level)

```
User Code
    |
    v
generate_*_agent_graph()     [Creates AgentGraph with profiles]
    |
    v
oasis.make()                  [Creates OasisEnv]
    |
    v
env.reset()                  [Initializes Platform + agents]
    |
    v
env.step(actions)             [Runs simulation step]
    |
    +---> Platform            [Handles DB, recsys, user actions]
    +---> AgentGraph          [Manages agent execution]
    +---> Clock               [Simulation time]
```

---

## Platform Patterns

### Twitter
- Entry: `generate_twitter_agent_graph()`
- Platform: `oasis/social_platform/platform.py`
- Tests: `test/infra/database/test_*.py`

### Reddit
- Entry: `generate_reddit_agent_graph()`
- Same platform class, different config
- Tests: `test/infra/database/test_*.py`

---

## Action System

**File:** `oasis/social_platform/typing.py`

```python
class ActionType(Enum):
    LIKE_POST, DISLIKE_POST, CREATE_POST, CREATE_COMMENT, ...
    FOLLOW, MUTE, SEARCH_POSTS, SEARCH_USER, ...
    REFRESH, TREND, DO_NOTHING, ...
```

Agents can perform 23 actions across posts, comments, follows, searches.

---

## Database

- **Type:** SQLite
- **Location:** User-specified path
- **Schema:** `oasis/social_platform/schema/`
- **Creation:** `oasis/social_platform/database.py` - `create_db()`

---

## Configuration

**UserInfo:** `oasis/social_platform/config/`
- Defines user profile structure
- Used by generators

---

## Generator Tooling

**Reddit:** `generator/reddit/user_generate.py`

**Twitter:** `generator/twitter/`
- `gen.py` - Main generator
- `ba.py` - BA variant
- `network.py` - Network generation
- `rag.py` - RAG-based generation

---

## Testing

- **Framework:** pytest
- **Config:** `pyproject.toml`
- **Structure:** `test/{agent,infra}/`
- **Key Fixtures:** `test/conftest.py`

---

## Dependency Key

| Package | Purpose |
|---------|---------|
| camel-ai | LLM agent framework |
| pandas | Data handling |
| igraph | Graph/network operations |
| cairocffi | Visualization |
| neo4j | Graph database (optional) |
| slack_sdk | Slack integration (optional) |

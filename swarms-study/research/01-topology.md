# Swarms Project Topology Analysis

## Repository Root

```
/Users/sheldon/Documents/claw/reference/swarms/
```

### Root Directory Listing

| Item | Type | Purpose |
|------|------|---------|
| `swarms/` | Directory | Main source code package |
| `examples/` | Directory | Example scripts and use cases |
| `tests/` | Directory | Test suite |
| `docs/` | Directory | Documentation (MkDocs) |
| `scripts/` | Directory | Utility scripts (docker, tests) |
| `images/` | Directory | Static assets |
| `pyproject.toml` | File | Poetry project configuration |
| `requirements.txt` | File | Pip dependencies |
| `example.py` | File | Simple usage example |
| `hiearchical_swarm_example.py` | File | Example demonstrating hierarchical swarms |

## Main Source Organization (`swarms/`)

```
swarms/
├── __init__.py       # Package entry point - loads env, telemetry, imports all submodules
├── env.py            # Environment configuration loader
├── agents/           # Agent implementations (16 files)
├── artifacts/        # Artifact handling
├── cli/              # CLI entry point (main.py - 57KB)
├── prompts/          # Prompt templates (69 files)
├── schemas/          # Pydantic/data schemas
├── structs/          # Core data structures (65 files, agent.py is 252KB)
├── telemetry/        # Telemetry and monitoring
├── tools/            # Tool implementations (base_tool.py is 107KB)
└── utils/            # Utility functions (29 files)
```

### Key Subdirectories

| Directory | Files | Purpose |
|-----------|-------|---------|
| `swarms/structs/` | 65 | Core swarm structures, Agent class, workflow orchestrators |
| `swarms/agents/` | 16 | Specialized agent implementations (React, OpenAI Assistant, etc.) |
| `swarms/prompts/` | 69 | Prompt templates for various swarm types |
| `swarms/tools/` | 20 | Tool system (base_tool.py, MCP client, function calling utilities) |
| `swarms/utils/` | 29 | Helper utilities |
| `swarms/cli/` | 2 | CLI implementation (main.py 57KB) |
| `swarms/schemas/` | 13 | Data validation schemas |

## Entry Points

### Package Entry Point
**File:** `swarms/__init__.py`

```python
from swarms.env import load_swarms_env
load_swarms_env()

from swarms.telemetry.bootup import bootup
bootup()

from swarms.agents import *       # Agent implementations
from swarms.artifacts import *   # Artifact handling
from swarms.prompts import *      # Prompt templates
from swarms.schemas import *      # Data schemas
from swarms.structs import *      # Core structures
from swarms.telemetry import *    # Telemetry
from swarms.tools import *        # Tool system
from swarms.utils import *        # Utilities
```

### CLI Entry Point
**File:** `swarms/cli/main.py`
**Command:** `swarms` (defined in pyproject.toml)

```toml
[tool.poetry.scripts]
swarms = "swarms.cli.main:main"
```

### Example Entry Points
- `example.py` - Simple Agent usage demonstration
- `hiearchical_swarm_example.py` - Hierarchical swarm demonstration

## Notable File Patterns

### Large Files (indicates complex functionality)
| File | Size | Purpose |
|------|------|---------|
| `structs/agent.py` | 252KB | Main Agent class |
| `structs/aop.py` | 102KB | Aspect-oriented programming |
| `structs/graph_workflow.py` | 111KB | Graph-based workflow |
| `structs/board_of_directors_swarm.py` | 63KB | Board of directors swarm |
| `tools/base_tool.py` | 107KB | Base tool implementation |
| `cli/main.py` | 57KB | CLI implementation |

### Swarm Structure Types (structs/)
- `agent.py` - Base Agent class
- `agent_router.py` - Agent routing
- `base_swarm.py` - Base swarm
- `graph_workflow.py` - Graph-based workflow orchestration
- `groupchat.py` - Multi-agent group chat
- `hierarchical_structured_communication_framework.py` - Hierarchical communication
- `swarm_router.py` - Swarm-level routing
- `concurrent_workflow.py`, `sequential_workflow.py` - Workflow patterns

### Agent Types (agents/)
- `agent_judge.py` - Judge agent
- `react_agent.py` - ReAct pattern agent
- `openai_assistant.py` - OpenAI assistant
- `reasoning_agent_router.py` - Reasoning agent with routing

## Examples Organization (`examples/`)

```
examples/
├── single_agent/     # Single agent examples
├── multi_agent/     # Multi-agent orchestration (40 files)
├── reasoning_agents/ # Reasoning agent examples
├── tools/           # Tool usage examples
├── models/          # Model integration examples
├── voice_agents/    # Voice agent examples
├── cli/             # CLI usage examples
├── marketplace/     # Marketplace examples
├── mcp/             # MCP (Model Context Protocol) examples
├── aop_examples/    # Aspect-oriented programming examples
├── swarms_api/      # Swarms API examples
├── ui/              # UI examples
├── guides/          # How-to guides
└── utils/           # Utility examples
```

## Tests Organization (`tests/`)

```
tests/
├── artifacts/
├── tools/
├── utils/
├── structs/
└── benchmarks/
```

## Key Dependencies (from pyproject.toml)

- **LLM Integration:** `litellm` (v1.76.1) - Unified LLM interface
- **CLI:** `rich` - Terminal formatting
- **Data Validation:** `pydantic`
- **Async:** `asyncio`, `aiohttp`, `aiofiles`, `httpx`
- **Multi-agent:** `networkx` - Graph-based workflows
- **MCP:** `mcp` - Model Context Protocol support
- **Logging:** `loguru`
- **PDF:** `pypdf`

## Project Metadata

- **Name:** swarms
- **Version:** 10.0.1
- **Author:** Kye Gomez <kye@swarms.world>
- **License:** MIT
- **Python:** >=3.10, <4.0
- **Documentation:** https://docs.swarms.world
- **Repository:** https://github.com/kyegomez/swarms

# crewAI Repository Topology

## Repository Structure Overview

This is a **Python monorepo** using `uv` workspace with 4 packages:

```
crewAI/
├── pyproject.toml              # Workspace root (members: crewai, crewai-tools, crewai-files, devtools)
├── conftest.py                 # Pytest configuration
├── lib/
│   ├── crewai/                 # Main framework package
│   ├── crewai-tools/           # Tool integrations (75+ tools)
│   ├── crewai-files/           # File processing utilities
│   └── devtools/               # Development tools
├── docs/                       # Documentation
└── .github/                    # GitHub workflows
```

---

## Package: crewai (Main Framework)

**Location:** `lib/crewai/src/crewai/`
**Entry Point:** `crewai.cli.cli:crewai` (CLI script defined in `pyproject.toml`)

### Core Source Structure

```
crewai/
├── __init__.py                 # Package root - exports: Agent, Crew, Flow, Task, LLM, Knowledge, Memory
├── crew.py                     # Crew class (76KB) - main orchestrator
├── task.py                     # Task class (50KB)
├── llm.py                      # LLM class (99KB) - LLM abstraction
├── lite_agent.py               # LiteAgent class (38KB)
├── process.py                  # Process enum (hierarchical, sequential, etc)
├── context.py                  # Context utilities
│
├── agent/                      # Agent implementation
│   ├── core.py                 # Agent class (66KB)
│   ├── utils.py
│   └── planning_config.py
│
├── agents/                      # Multi-agent utilities
│   ├── agent_utils.py          # 59KB - agent orchestration utilities
│   ├── cache/
│   ├── crew/                    # Crew agent manager
│   ├── evaluators/
│   ├── exceptions/
│   └── ...
│
├── crews/                       # Crew output and utilities
│   ├── crew_output.py
│   └── ...
│
├── cli/                        # Command-line interface
│   ├── cli.py                  # Main CLI entry point (22KB)
│   ├── create_crew.py          # Crew creation
│   ├── create_flow.py          # Flow creation
│   ├── run_crew.py
│   ├── train_crew.py
│   ├── kickoff_flow.py
│   ├── crew_chat.py            # Interactive chat
│   ├── memory_tui.py           # Memory visualization TUI
│   ├── authentication/          # Auth commands
│   ├── deploy/                  # Deployment commands
│   ├── enterprise/             # Enterprise config
│   ├── organization/
│   ├── settings/
│   ├── shared/
│   ├── templates/               # Project templates
│   │   ├── crew/               # Crew project template
│   │   │   ├── main.py         # Entry point (run/train/test/replay)
│   │   │   ├── crew.py         # @CrewBase class
│   │   │   ├── config/
│   │   │   │   ├── agents.yaml
│   │   │   │   └── tasks.yaml
│   │   │   └── tools/
│   │   └── flow/               # Flow project template
│   │       ├── main.py
│   │       └── crews/
│   └── ...
│
├── flow/                        # Flow orchestration (newer)
│   ├── flow.py                 # Flow base class (129KB)
│   ├── flow_config.py
│   ├── flow_context.py
│   ├── flow_serializer.py
│   ├── human_feedback.py
│   ├── input_provider.py
│   ├── persistence/
│   ├── visualization/
│   └── ...
│
├── llms/                        # LLM providers
│   ├── base_llm.py             # Base LLM class
│   ├── config.py
│   ├── providers/              # Provider implementations (openai, anthropic, etc.)
│   ├── hooks/
│   └── third_party/
│
├── memory/                      # Memory system
│   ├── unified_memory.py        # Main memory class (39KB)
│   ├── memory_scope.py
│   ├── analyze.py
│   ├── encoding_flow.py
│   ├── recall_flow.py
│   ├── storage/                 # Storage backends
│   └── ...
│
├── knowledge/                   # RAG/Knowledge system
│   ├── knowledge.py             # Knowledge class
│   ├── knowledge_config.py
│   ├── source/                  # Knowledge sources (PDF, text, etc.)
│   └── storage/                 # Vector storage
│
├── tools/                       # Built-in tools
│   ├── base_tool.py
│   ├── structured_tool.py
│   ├── tool_usage.py           # 42KB - tool orchestration
│   ├── agent_tools/
│   ├── cache_tools/
│   ├── rag/
│   ├── mcp_tool_wrapper.py
│   ├── mcp_native_tool.py
│   └── [75+ tool directories]  # Individual tool implementations
│
├── tasks/                       # Task-related
│   ├── task_output.py
│   ├── conditional_task.py
│   ├── hallucination_guardrail.py
│   ├── llm_guardrail.py
│   └── ...
│
├── events/                      # Event system
│   ├── event_bus.py            # 26KB - async event bus
│   ├── event_listener.py       # 29KB
│   ├── event_types.py
│   ├── base_events.py
│   ├── base_event_listener.py
│   ├── handler_graph.py
│   ├── listeners/
│   │   └── tracing/             # OpenTelemetry tracing
│   ├── types/
│   └── utils/
│
├── rag/                         # RAG system
│   ├── ...
│
├── a2a/                         # Agent-to-Agent protocol
├── mcp/                         # Model Context Protocol
├── project/                     # Project decorators
├── skills/                      # Skills system
├── security/                     # Security utilities
├── telemetry/                    # Telemetry/analytics
├── utilities/                   # Shared utilities
│   ├── llm_utils.py
│   ├── tool_utils.py
│   ├── streaming.py
│   ├── planning_handler.py
│   ├── reasoning_handler.py
│   └── ...
└── types/                       # Type definitions
```

---

## Package: crewai-tools

**Location:** `lib/crewai-tools/src/crewai_tools/`

```
crewai_tools/
├── __init__.py                 # Exports tool classes
├── base_tool.py
├── generate_tool_specs.py
├── printer.py
├── adapters/
├── aws/
├── rag/
└── tools/                       # 75+ tool implementations
    ├── file_read_tool/
    ├── file_writer_tool/
    ├── pdf_search_tool/
    ├── selenium_scraping_tool/
    ├── serch/... etc.
    └── [75+ individual tools]
```

---

## Package: crewai-files

**Location:** `lib/crewai-files/`
File processing utilities for reading various file formats.

---

## Entry Points

### 1. CLI Entry Point
**Command:** `crewai`
**Source:** `crewai.cli.cli:crewai`

```python
# From pyproject.toml
[project.scripts]
crewai = "crewai.cli.cli:crewai"
```

Commands available:
- `crewai create crew <name>` - Create new crew
- `crewai create flow <name>` - Create new flow
- `crewai run` - Run a crew
- `crewai train` - Train a crew
- `crewai test` - Test a crew
- `crewai replay` - Replay execution
- `crewai chat` - Interactive chat mode
- `crewai reset-memories` - Reset crew memories
- `crewai uv` - UV wrapper with tool auth
- And more (auth, deploy, enterprise, settings, triggers, tools)

### 2. Python API Entry Point
**Source:** `lib/crewai/src/crewai/__init__.py`

```python
from crewai import Agent, Crew, Flow, Task, LLM, Knowledge, Memory, Process
```

Exports:
- `Agent` - AI agent
- `Crew` - Crew orchestrator
- `Flow` - Flow orchestrator (newer)
- `Task` - Task definition
- `LLM` - LLM configuration
- `Knowledge` - RAG knowledge base
- `Memory` - Memory system (lazy loaded)
- `Process` - Process enum (hierarchical, sequential)

### 3. User Project Entry Point
**Template:** `lib/crewai/src/crewai/cli/templates/crew/main.py`

```python
def run():
    {{crew_name}}().crew().kickoff(inputs=inputs)

def train():
    {{crew_name}}().crew().train(n_iterations=int(sys.argv[1]), filename=sys.argv[2])

def test():
    {{crew_name}}().crew().test(n_iterations=int(sys.argv[1]), eval_llm=sys.argv[2])

def replay():
    {{crew_name}}().crew().replay(task_id=sys.argv[1])
```

### 4. Flow Entry Point
**Template:** `lib/crewai/src/crewai/cli/templates/flow/main.py`

```python
class PoemFlow(Flow[PoemState]):
    @start()
    def generate_sentence_count(self, ...): ...

    @listen(generate_sentence_count)
    def generate_poem(self, ...): ...

def kickoff():
    PoemFlow().kickoff()

def plot():
    PoemFlow().plot()
```

---

## Key Classes and Their Relationships

```
Agent          - Individual AI agent with role, goal, backstory, tools
     ↓
Crew           - Orchestrates multiple agents + tasks (76KB core class)
     ↓
Process        - Execution process (sequential, hierarchical)

Task           - Work unit assigned to an agent
     ↓
TaskOutput     - Result of task execution

LLM            - LLM configuration and calls (99KB)
     ↓
BaseLLM        - Abstract base for LLM providers
     ↓
[Providers]    - OpenAI, Anthropic, Google GenAI, Azure, etc.

Flow           - Newer flow-based orchestration (129KB)
     ↓
FlowState       - State management for flows

Knowledge      - RAG-based knowledge management
     ↓
BaseKnowledgeSource - Abstract for knowledge sources

Memory         - Memory system for crew context
     ↓
[Storages]     - SQLite, LanceDB, etc.
```

---

## Directory Layout Patterns

| Pattern | Location | Purpose |
|---------|----------|---------|
| `src/{package}/` | `lib/*/src/*/` | Source code root |
| `tools/` | `crewai/tools/`, `crewai_tools/tools/` | Tool implementations |
| `cli/templates/` | `crewai/cli/templates/` | Project scaffolding |
| `tests/` | `lib/*/tests/` | Test suites |
| `config/` | Template dirs | YAML configuration |

---

## Version

**Current:** `1.13.0rc1` (from `__init__.py`)

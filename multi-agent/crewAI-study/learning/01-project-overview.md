# crewAI Project Overview

## What is crewAI?

crewAI is a standalone Python framework for orchestrating autonomous multi-AI agent systems, built entirely from scratch without dependencies on LangChain or other agent frameworks. It provides both high-level simplicity and low-level control for creating AI agents tailored to any scenario.

## Problem Solved

Building sophisticated AI automations requires coordinating multiple AI agents that can collaborate, delegate tasks, and make autonomous decisions. crewAI solves this by providing:

- **Crews**: Teams of AI agents with true autonomy, working together through role-based collaboration
- **Flows**: Event-driven workflows with precise control over execution paths and state management
- **Tools**: 75+ built-in tool integrations for agents to interact with external systems
- **Memory**: Persistent context and memory systems for crew continuity
- **Knowledge**: RAG-based knowledge management for grounding agent responses

## Target Users

- **Individual developers** building AI automations and agentic workflows
- **Enterprises** requiring production-grade, secure, scalable agent orchestration
- **Teams** creating collaborative AI systems where multiple specialized agents work together

## Core Abstractions

| Abstraction | Purpose | Key Files |
|-------------|---------|-----------|
| **Agent** | Individual AI agent with role, goal, backstory, and tools | `lib/crewai/src/crewai/agent/core.py` (66KB) |
| **Crew** | Orchestrates multiple agents and tasks | `lib/crewai/src/crewai/crew.py` (76KB) |
| **Flow** | Event-driven workflow orchestration with state | `lib/crewai/src/crewai/flow/flow.py` (129KB) |
| **Task** | Work unit assigned to an agent with expected output | `lib/crewai/src/crewai/task.py` (50KB) |
| **LLM** | Language model configuration and calls | `lib/crewai/src/crewai/llm.py` (99KB) |
| **Knowledge** | RAG-based knowledge sources | `lib/crewai/src/crewai/knowledge/` |
| **Memory** | Crew memory and context management | `lib/crewai/src/crewai/memory/` |

## Architecture

### Repository Structure

```
crewAI/ (Python monorepo with uv workspace)
├── lib/
│   ├── crewai/           # Main framework package
│   ├── crewai-tools/     # 75+ tool integrations
│   ├── crewai-files/     # File processing utilities
│   └── devtools/         # Development automation
├── docs/
└── .github/workflows/
```

### Key Relationships

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
[Providers]    - OpenAI, Anthropic, Google GenAI, Azure, Ollama, etc.

Flow           - Event-driven orchestration (129KB)
     ↓
FlowState      - State management for flows

Knowledge      - RAG-based knowledge management
     ↓
BaseKnowledgeSource - Abstract for knowledge sources

Memory         - Memory system for crew context
     ↓
[Storages]     - SQLite, LanceDB, etc.
```

## What Makes crewAI Distinctive

1. **Truly Standalone**: Built from scratch with no LangChain or other framework dependencies, resulting in faster execution and lighter resource demands

2. **Dual Paradigm**: Unique combination of Crews (autonomous agent collaboration) and Flows (precise event-driven control) that can be combined seamlessly

3. **Production-Ready Enterprise Suite**: AMP (Agent Management Platform) offers unified control plane, observability, advanced security, and 24/7 support

4. **Extensive Tool Ecosystem**: 75+ built-in tools in `crewai-tools` package covering file operations, RAG, scraping, database operations, and more

5. **MCP Integration**: Native support for Model Context Protocol, enabling connection to MCP servers with tool filtering capabilities

6. **Memory System**: Sophisticated memory management with short-term, long-term, and entity memory capabilities

7. **Guardrails**: Built-in hallucination and LLM guardrails for safer agent behavior

8. **Community Scale**: Over 100,000 developers certified through community courses at learn.crewai.com

## Entry Points

### CLI
```bash
crewai create crew <name>   # Create new crew project
crewai create flow <name>   # Create new flow project
crewai run                  # Run a crew
crewai train                # Train a crew
crewai chat                 # Interactive chat mode
```

### Python API
```python
from crewai import Agent, Crew, Flow, Task, LLM, Knowledge, Memory, Process
```

## Version

Current release: `1.13.0rc1`

## License

MIT License

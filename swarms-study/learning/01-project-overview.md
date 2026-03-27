# Project Overview: Swarms

## Project Purpose

**Swarms** is an enterprise-grade, production-ready multi-agent orchestration framework designed for building and deploying complex AI agent systems at scale. The framework enables developers to orchestrate multiple LLM-powered agents to work together through predefined workflows, handling everything from simple sequential tasks to sophisticated hierarchical decision-making processes.

The project is maintained by The Swarm Corporation and positioned as "the operating system for the agent economy," providing infrastructure for orchestrating millions of autonomous agents in production environments.

## Project Scope

### Core Domain
- **Multi-Agent Orchestration**: Framework for coordinating multiple AI agents
- **Workflow Management**: Various architectures including sequential, concurrent, hierarchical, and graph-based workflows
- **LLM Integration**: Unified interface to multiple language model providers via LiteLLM
- **Tool Ecosystem**: Extensible tool system for agents to interact with external services

### Key Technical Components

| Component | Purpose |
|-----------|---------|
| `swarms/agents/` | Agent implementations (Agent, ReasoningAgent, etc.) |
| `swarms/structs/` | Workflow orchestrators (SequentialWorkflow, ConcurrentWorkflow, etc.) |
| `swarms/tools/` | External tool integrations (web search, browser automation, etc.) |
| `swarms/cli/` | Command-line interface for swarm management |
| `swarms/prompts/` | Prompt templates and agent configurations |
| `swarms/utils/` | Utility functions and helpers |

### Supported Protocols
- **MCP (Model Context Protocol)**: Standardized tool integration
- **X402**: Cryptocurrency payment protocol for API endpoints
- **AOP (Agent Orchestration Protocol)**: Distributed agent deployment
- **Agent Skills**: Anthropic-style markdown-based skill definitions

## Project Metadata

| Attribute | Value |
|-----------|-------|
| Language | Python 3.10+ |
| Package Manager | Poetry |
| License | Apache 2.0 / MIT |
| Version | 10.0.1 |
| Homepage | https://swarms.world |
| Documentation | https://docs.swarms.world |
| Repository | https://github.com/kyegomez/swarms |

## Architecture Philosophy

The framework follows a modular architecture where:
1. **Agents** are autonomous entities powered by LLMs with optional tools and memory
2. **Workflows** define how agents interact (sequential, parallel, hierarchical, etc.)
3. **Tools** extend agent capabilities (search, browser, APIs)
4. **Prompts** define agent behavior and specialization

## Dependencies Overview

Key runtime dependencies include:
- `litellm` (1.76.1): Multi-model provider abstraction
- `pydantic`: Data validation and settings management
- `loguru`: Logging
- `networkx`: Graph-based workflow support
- `httpx`, `aiohttp`: HTTP clients
- `pypdf`: Document processing

Dev dependencies include:
- `black`, `ruff`: Code formatting and linting
- `pytest`: Testing framework
- `mypy-protobuf`: Type checking

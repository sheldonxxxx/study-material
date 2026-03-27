# Swarms Feature Index

## Core Features (Priority 1-5)

### 1. Agent Base Class
**Description:** The fundamental building block - an autonomous entity powered by an LLM with tools and memory.
**Key Files/Locations:**
- `swarms/agents/` - Agent implementation
- `swarms/agents/base_agent.py` - Core agent class
**Priority:** Core

### 2. Multi-Agent Orchestration Architectures
**Description:** Pre-built workflow patterns for coordinating multiple agents (sequential, concurrent, hierarchical, mixture-of-agents, etc.).
**Key Files/Locations:**
- `swarms/structs/` - 69 modules containing workflow implementations
- `swarms/structs/sequential_workflow.py`
- `swarms/structs/concurrent_workflow.py`
- `swarms/structs/hierarchical_swarm.py`
- `swarms/structs/moa.py` (MixtureOfAgents)
- `swarms/structs/swarm_router.py`
- `swarms/structs/agent_rearrange.py`
- `swarms/structs/group_chat.py`
- `swarms/structs/maker.py`
- `swarms/structs/forest_swarm.py`
- `swarms/structs/heavy_swarm.py`
**Priority:** Core

### 3. AutoSwarmBuilder
**Description:** Autonomous agent generation system that creates specialized agents and workflows from task descriptions.
**Key Files/Locations:**
- `swarms/structs/auto_swarm_builder.py`
**Priority:** Core

### 4. Multi-Model Provider Support
**Description:** Vendor-agnostic architecture supporting OpenAI, Anthropic, Groq, Cohere, DeepSeek, Ollama, OpenRouter, XAI, and Llama4.
**Key Files/Locations:**
- `swarms/utils/` - Utility modules for model interactions
- `swarms/agents/` - Agent configurations
**Priority:** Core

### 5. Agent Tools and Memory Systems
**Description:** Extensive library of tools and multiple memory systems for agent capability extension.
**Key Files/Locations:**
- `swarms/tools/` - Tool integrations
- `swarms/agents/` - Memory implementations
**Priority:** Core

---

## Secondary Features (Priority 6-10)

### 6. Model Context Protocol (MCP) Integration
**Description:** Standardized protocol for AI agents to interact with external tools and services through MCP servers.
**Key Files/Locations:**
- `swarms/examples/mcp/` - MCP examples
- `swarms/tools/` - MCP tool adapters
**Priority:** Secondary

### 7. SwarmRouter
**Description:** Universal orchestrator providing a single interface to run any swarm type with dynamic selection.
**Key Files/Locations:**
- `swarms/structs/swarm_router.py`
**Priority:** Secondary

### 8. Agent Orchestration Protocol (AOP)
**Description:** Framework for deploying and managing agents as distributed services with discovery and management.
**Key Files/Locations:**
- `swarms/structs/aop.py`
**Priority:** Secondary

### 9. Prompt Management
**Description:** Large library of prompt templates for various agent specializations and tasks.
**Key Files/Locations:**
- `swarms/prompts/` - 69 prompt template modules
**Priority:** Secondary

### 10. CLI and SDK Tools
**Description:** Command-line interface and developer tools for rapid deployment and management.
**Key Files/Locations:**
- `swarms/cli/` - CLI implementation
- `swarms/utils/` - SDK utilities
**Priority:** Secondary

---

## Protocol Integrations (Specialized)

### 11. X402 Payment Protocol
**Description:** Cryptocurrency payment protocol for API endpoint monetization with pay-per-use models.
**Key Files/Locations:** Documentation reference only (external protocol)
**Priority:** Secondary

### 12. Agent Skills (Anthropic-compatible)
**Description:** Markdown-based format for defining modular, reusable agent capabilities.
**Key Files/Locations:** `swarms/agents/agent_skills/` (referenced in docs)
**Priority:** Secondary

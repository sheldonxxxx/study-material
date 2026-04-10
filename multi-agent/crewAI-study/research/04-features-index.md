# CrewAI Feature Index

## Overview

This document maps CrewAI's documented features to their actual implementation locations in the codebase, prioritizing core functionality.

## Source Analysis

**Repository:** `/Users/sheldon/Documents/claw/reference/crewAI`
**Packages:**
- `lib/crewai/` - Core framework
- `lib/crewai-tools/` - Built-in tools (50+ integrations)
- `lib/crewai-files/` - File processing utilities
- `lib/devtools/` - Developer tools

---

## Priority Tier: CORE (Essential Features)

### 1. Multi-Agent Crew Orchestration

**Description:** Teams of AI agents with autonomy and collaborative intelligence, working together to accomplish complex tasks through role-based collaboration.

**Key Files:**
- `lib/crewai/src/crewai/crew.py` (76KB - main Crew class)
- `lib/crewai/src/crewai/agents/` - Agent execution logic
- `lib/crewai/src/crewai/agents/crew_agent_executor.py` (65KB)
- `lib/crewai/src/crewai/process.py`

**Evidence:** README emphasizes Crews as the primary abstraction. `@CrewBase` decorator, `Process.sequential` and `Process.hierarchical` supported.

---

### 2. Flow-Based Event-Driven Workflows

**Description:** Production-ready, event-driven architecture for precise control over complex automations with state management and conditional branching.

**Key Files:**
- `lib/crewai/src/crewai/flow/flow.py` (129KB - core Flow engine)
- `lib/crewai/src/crewai/flow/human_feedback.py` (26KB)
- `lib/crewai/src/crewai/flow/utils.py` (33KB)
- `lib/crewai/src/crewai/flow/visualization/` - Flow visualization

**Evidence:** README "Understanding Flows and Crews" section. Decorators: `@start()`, `@listen()`, `@router()`, `@or_()`, `@and_()`.

---

### 3. Autonomous Agent System

**Description:** Agents with specialized roles, goals, backstories, and natural autonomous decision-making capabilities.

**Key Files:**
- `lib/crewai/src/crewai/agent/` - Agent base classes
- `lib/crewai/src/crewai/agents/agent_builder/` - Agent construction
- `lib/crewai/src/crewai/lite_agent.py` (38KB - lightweight agent)
- `lib/crewai/src/crewai/llm.py` (99KB - LLM integration)

**Evidence:** `Agent` class with `role`, `goal`, `backstory` configuration. `BaseAgent` interface.

---

### 4. Task Management & Delegation

**Description:** Sophisticated task definition, coordination, and delegation between agents with output handling.

**Key Files:**
- `lib/crewai/src/crewai/task.py` (50KB - Task class)
- `lib/crewai/src/crewai/tasks/` - Task management
- `lib/crewai/src/crewai/agents/step_executor.py` (24KB)

**Evidence:** `Task` class with `expected_output`, `agent` assignment, `output_file` support.

---

### 5. Flexible LLM Integration

**Description:** Connect agents to various LLMs (OpenAI, Ollama, local models) with a unified interface.

**Key Files:**
- `lib/crewai/src/crewai/llm.py` (99KB)
- `lib/crewai/src/crewai/llms/` - LLM adapters
- `lib/crewai/src/crewai/utilities/` - LLM utilities

**Evidence:** `LLM` class, `BaseLLM` interface. README documents Ollama, LM Studio connections.

---

### 6. RAG (Retrieval-Augmented Generation) & Knowledge Management

**Description:** Vector-based knowledge retrieval for agents with multiple storage backends.

**Key Files:**
- `lib/crewai/src/crewai/rag/` (entire directory, 26KB+)
- `lib/crewai/src/crewai/rag/unified_memory.py` (39KB)
- `lib/crewai/src/crewai/rag/encoding_flow.py` (21KB)
- `lib/crewai/src/crewai/knowledge/` - Knowledge class

**Evidence:** `Knowledge` class exported in `__init__.py`. RAG storage: ChromaDB, Qdrant backends.

---

### 7. Agent Memory System

**Description:** Persistent memory capabilities for agents with various storage backends.

**Key Files:**
- `lib/crewai/src/crewai/memory/` (entire directory)
- `lib/crewai/src/crewai/memory/unified_memory.py` (39KB)
- `lib/crewai/src/crewai/memory/recall_flow.py` (14KB)
- `lib/crewai/src/crewai/memory/chromadb/` - ChromaDB adapter
- `lib/crewai/src/crewai/memory/qdrant/` - Qdrant adapter

**Evidence:** `Memory` lazy import in `__init__.py`. Memory types: short-term, long-term, working memory.

---

## Priority Tier: SECONDARY (Important but Distinct)

### 8. Built-in Tools & MCP Integration

**Description:** 50+ pre-built tools for agents (search, scraping, database, etc.) plus Model Context Protocol support.

**Key Files:**
- `lib/crewai-tools/src/crewai_tools/` (57 tool directories)
- `lib/crewai-tools/src/crewai_tools/tools/` - Individual tool implementations
- `lib/crewai/src/crewai/tools/` - Tool infrastructure
- `lib/crewai/src/crewai/mcp/` - MCP native tool support

**Evidence:** Tools include: SerperDevTool, BraveSearchTool, BrowserbaseLoadTool, DallETool, GithubSearchTool, SeleniumScrapingTool, and 50+ more.

---

### 9. CLI & Project Scaffolding

**Description:** Command-line interface for creating crews, installing dependencies, and running projects.

**Key Files:**
- `lib/crewai/src/crewai/cli/` (35 subdirectories)
- `lib/crewai/src/crewai/cli/templates/` - Project templates

**Evidence:** `crewai create crew <project_name>`, `crewai install`, `crewai run`, `crewai update` commands documented in README.

---

### 10. Telemetry & Observability

**Description:** Anonymous telemetry collection for usage insights, plus Crew Control Plane for enterprise.

**Key Files:**
- `lib/crewai/src/crewai/telemetry/` - Telemetry implementation
- `lib/crewai/src/crewai/events/` - Event tracking

**Evidence:** `Telemetry` class in `__init__.py`. OTEL_SDK_DISABLED env var support.

---

### 11. Human-in-the-Loop & Feedback

**Description:** Mechanisms for human intervention during agent execution.

**Key Files:**
- `lib/crewai/src/crewai/flow/human_feedback.py` (26KB)
- `lib/crewai/src/crewai/tools/async_feedback/` - Async feedback tools

**Evidence:** `human_feedback.py` with `HumanFeedback` class. README documents "Human input on the execution" example.

---

### 12. Security & Guardrails

**Description:** LLM guardrails for task outputs and security configurations.

**Key Files:**
- `lib/crewai/src/crewai/tasks/llm_guardrail.py`
- `lib/crewai/src/crewai/security/` - Security utilities

**Evidence:** `LLMGuardrail` exported in `__init__.py`. Input/output validation.

---

## Verification: README Claims vs. Actual Implementation

| README Claim | Verified | Implementation Location |
|-------------|----------|------------------------|
| Crews of AI Agents | Yes | `crew.py`, `agents/` |
| Flows for event-driven control | Yes | `flow/flow.py` |
| Sequential/Hierarchical processes | Yes | `process.py`, `crew.py` |
| YAML configuration | Yes | `cli/templates/`, `project/` |
| OpenAI/Ollama/LM Studio support | Yes | `llm.py`, `llms/` |
| RAG/Knowledge integration | Yes | `rag/`, `knowledge/` |
| 50+ built-in tools | Yes | `crewai-tools/tools/` |
| Human feedback support | Yes | `flow/human_feedback.py` |
| Enterprise AMP features | Partial | Telemetry only (open source) |

---

## File Count Summary

| Component | Files/Dirs | Location |
|-----------|------------|----------|
| Core Agents | 1 core file + 3 subdirs | `lib/crewai/src/crewai/agent/`, `lib/crewai/src/crewai/agents/` |
| Flow Engine | 8 files | `lib/crewai/src/crewai/flow/` |
| LLM Integration | 1 core file + 8 adapters | `lib/crewai/src/crewai/llm.py`, `lib/crewai/src/crewai/llms/` |
| Tools | 57 tool implementations | `lib/crewai-tools/src/crewai_tools/tools/` |
| Memory/RAG | 8+ storage implementations | `lib/crewai/src/crewai/memory/`, `lib/crewai/src/crewai/rag/` |
| CLI | 35 subcommands | `lib/crewai/src/crewai/cli/` |
| Tasks | 7 task-related modules | `lib/crewai/src/crewai/tasks/` |

---

## Notes

- The codebase is well-organized with clear separation between core (`crewai`) and tools (`crewai-tools`, `crewai-files`, `devtools`).
- Lazy imports are used for heavy dependencies (e.g., `Memory` imports lancedb on first access).
- The framework is Python 3.10-3.13 compatible, using Pydantic for data validation.
- Version: `1.13.0rc1` (release candidate)

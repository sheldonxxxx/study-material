# DeerFlow Feature Index

## Overview

DeerFlow is an open-source **super agent harness** that orchestrates sub-agents, memory, and sandboxes to execute complex tasks. Built on LangGraph and LangChain, version 2.0 is a ground-up rewrite.

**Repository:** https://github.com/bytedance/deer-flow
**Tech Stack:** Python 3.12+ (backend), Node.js 22+ (frontend), Next.js (UI), LangGraph/LangChain (orchestration), Docker (sandbox)

---

## Core Features

### 1. Skills & Tools System

**Description:** Extensible skill framework defining workflows, best practices, and resources. Ships with built-in skills for research, report generation, slide creation, web pages, image/video generation, and more. Supports custom skills via `.skill` archives and MCP servers.

**Priority:** CORE

**Key Files/Locations:**
- `skills/public/` - Built-in public skills (research, report-generation, slide-creation, image-generation, video-generation, podcast-generation, ppt-generation, deep-research, github-deep-research, data-analysis, frontend-design, web-design-guidelines, consulting-analysis, chart-visualization, skill-creator, find-skills, surprise-me, vercel-deploy-claimable, claude-to-deerflow)
- `backend/packages/harness/deerflow/skills/` - Skill loading and management
- `backend/packages/harness/deerflow/tools/` - Tool implementations
- `backend/packages/harness/deerflow/tools/builtins/` - Built-in tools (web search, web fetch, file operations, bash execution)
- `backend/packages/harness/deerflow/mcp/` - MCP server integration
- `skills/public/claude-to-deerflow/SKILL.md` - Claude Code CLI integration skill
- `backend/docs/MCP_SERVER.md` - MCP configuration guide

---

### 2. Sub-Agent Orchestration

**Description:** Lead agent decomposes complex tasks and spawns sub-agents on-the-fly, each with scoped context, tools, and termination conditions. Sub-agents run in parallel when possible, report back structured results, and the lead agent synthesizes everything.

**Priority:** CORE

**Key Files/Locations:**
- `backend/packages/harness/deerflow/agents/` - Agent system core
- `backend/packages/harness/deerflow/agents/lead_agent/` - Lead agent implementation
- `backend/packages/harness/deerflow/subagents/` - Sub-agent implementations
- `backend/packages/harness/deerflow/subagents/builtins/` - Built-in sub-agent types
- `backend/packages/harness/deerflow/agents/memory/` - Memory management for agents
- `backend/packages/harness/deerflow/agents/middlewares/` - Agent middleware
- `backend/packages/harness/deerflow/agents/checkpointer/` - State persistence

---

### 3. Sandbox Execution Environment

**Description:** Each task runs inside an isolated Docker container with a full filesystem (skills, workspace, uploads, outputs). Supports local execution, Docker containers, and Kubernetes pods via provisioner service.

**Priority:** CORE

**Key Files/Locations:**
- `backend/packages/harness/deerflow/sandbox/` - Sandbox implementations
- `backend/packages/harness/deerflow/sandbox/local/` - Local execution mode
- `backend/packages/harness/deerflow/community/aio_sandbox/` - Kubernetes sandbox provider
- `docker/` - Docker and Kubernetes deployment configs
- `docker/docker-compose.yaml` - Production compose
- `docker/docker-compose-dev.yaml` - Development compose
- `scripts/provisioner/` - Provisioner service for Kubernetes
- `backend/docs/CONFIGURATION.md#sandbox` - Sandbox configuration

---

### 4. Context Engineering

**Description:** Manages context aggressively through isolated sub-agent contexts, summarization of completed sub-tasks, and offloading intermediate results to filesystem. Prevents context window blowout across long multi-step tasks.

**Priority:** CORE

**Key Files/Locations:**
- `backend/packages/harness/deerflow/agents/memory/` - Context/memory management
- `backend/packages/harness/deerflow/reflection/` - Reflection and summarization
- `backend/docs/summarization.md` - Summarization implementation
- `frontend/src/components/ai-elements/chain-of-thought.tsx` - Chain of thought display
- `frontend/src/components/ai-elements/context.tsx` - Context visualization

---

### 5. Long-Term Memory

**Description:** Persistent memory across sessions storing user profile, preferences, and accumulated knowledge. Improves over time with usage. Memory is stored locally under user control.

**Priority:** CORE

**Key Files/Locations:**
- `backend/packages/harness/deerflow/agents/memory/` - Memory implementation
- `backend/docs/MEMORY_IMPROVEMENTS.md` - Memory improvements documentation
- `backend/docs/MEMORY_IMPROVEMENTS_SUMMARY.md` - Memory summary

---

### 6. Multi-Channel IM Integration

**Description:** Receive tasks from messaging apps (Telegram, Slack, Feishu/Lark) via WebSocket/bot APIs. Channels auto-start when configured; no public IP required.

**Priority:** CORE

**Key Files/Locations:**
- `backend/app/channels/` - Channel implementations
- `backend/app/channels/telegram.py` - Telegram Bot API integration
- `backend/app/channels/slack.py` - Slack Socket Mode integration
- `backend/app/channels/feishu.py` - Feishu/Lark WebSocket integration
- `backend/app/channels/manager.py` - Channel management
- `backend/docs/CONFIGURATION.md` - IM channel configuration (see IM Channels section)

---

### 7. Model Agnostic Architecture

**Description:** Works with any LLM implementing OpenAI-compatible API. Supports langchain_openai providers, CLI-backed providers (Codex CLI, Claude Code OAuth), and custom provider implementations.

**Priority:** CORE

**Key Files/Locations:**
- `backend/packages/harness/deerflow/models/` - Model provider implementations
- `backend/packages/harness/deerflow/models/claude_provider/` - Claude Code OAuth provider
- `backend/packages/harness/deerflow/models/openai_codex_provider/` - Codex CLI provider
- `config.example.yaml` - Model configuration examples
- `backend/docs/CONFIGURATION.md` - Model configuration guide

---

## Secondary Features

### 8. Claude Code CLI Integration

**Description:** The `claude-to-deerflow` skill enables terminal-based interaction with DeerFlow from Claude Code. Send research tasks, check status, manage threads, upload files.

**Priority:** SECONDARY

**Key Files/Locations:**
- `skills/public/claude-to-deerflow/SKILL.md` - Skill definition and API reference
- `frontend/src/components/ui/terminal.tsx` - Terminal UI component

---

### 9. Embedded Python Client

**Description:** Use DeerFlow as an embedded Python library without running full HTTP services. `DeerFlowClient` provides in-process access to agent and Gateway capabilities.

**Priority:** SECONDARY

**Key Files/Locations:**
- `backend/packages/harness/deerflow/client.py` - Embedded client implementation
- `backend/docs/ARCHITECTURE.md` - Client architecture documentation

---

### 10. Guardrails & Safety

**Description:** Safety middleware system for input/output validation, content filtering, and audit logging.

**Priority:** SECONDARY

**Key Files/Locations:**
- `backend/packages/harness/deerflow/guardrails/` - Guardrail implementations
- `backend/docs/GUARDRAILS.md` - Guardrails documentation

---

### 11. Gateway HTTP API

**Description:** HTTP Gateway API for external integrations, thread management, skill management, and file uploads.

**Priority:** SECONDARY

**Key Files/Locations:**
- `backend/app/gateway/` - Gateway implementation
- `backend/app/gateway/routers/` - API route handlers
- `backend/docs/API.md` - API reference

---

### 12. Artifact System

**Description:** Rich artifact rendering for code blocks, web previews, charts, visualizations, and interactive content. Includes code editor with syntax highlighting.

**Priority:** SECONDARY

**Key Files/Locations:**
- `frontend/src/components/workspace/artifacts/` - Artifact components
- `frontend/src/components/ai-elements/code-block.tsx` - Code block rendering
- `frontend/src/components/ai-elements/web-preview.tsx` - Web preview component
- `frontend/src/components/workspace/code-editor.tsx` - Code editor

---

### 13. InfoQuest Integration

**Description:** Intelligent search and crawling toolset from BytePlus for enhanced web research capabilities.

**Priority:** SECONDARY

**Key Files/Locations:**
- `backend/packages/harness/deerflow/community/infoquest/` - InfoQuest integration
- `backend/packages/harness/deerflow/community/jina_ai/` - Jina AI integration (for web content extraction)
- `backend/packages/harness/deerflow/community/tavily/` - Tavily search integration
- `backend/packages/harness/deerflow/community/firecrawl/` - Firecrawl web scraping

---

### 14. Frontend UI/UX

**Description:** Next.js-based web interface with workspace, chat, settings, agent gallery, and real-time streaming support.

**Priority:** SECONDARY

**Key Files/Locations:**
- `frontend/src/app/workspace/` - Main workspace page
- `frontend/src/components/workspace/` - Workspace components (messages, chats, settings, artifacts)
- `frontend/src/components/ui/` - Shadcn/ui component library
- `frontend/src/components/ai-elements/` - AI-specific elements (reasoning, prompts, suggestions)
- `frontend/src/components/landing/` - Landing page components

---

## Directory Structure Overview

```
deer-flow/
├── backend/
│   ├── app/
│   │   ├── channels/           # IM integrations (telegram, slack, feishu)
│   │   └── gateway/            # HTTP Gateway API
│   ├── docs/                   # Architecture and configuration docs
│   ├── packages/harness/deerflow/
│   │   ├── agents/             # Lead agent, memory, checkpointer
│   │   ├── subagents/           # Sub-agent system
│   │   ├── sandbox/            # Sandbox execution (local, docker, k8s)
│   │   ├── skills/             # Skill loading
│   │   ├── tools/               # Built-in tools
│   │   ├── mcp/                 # MCP server support
│   │   ├── models/              # Model providers
│   │   ├── community/           # External integrations (tavily, jina, infoquest)
│   │   ├── guardrails/          # Safety middleware
│   │   └── client.py           # Embedded Python client
│   └── tests/                   # Backend tests
├── frontend/
│   ├── src/
│   │   ├── app/                 # Next.js app router
│   │   ├── components/
│   │   │   ├── workspace/       # Main workspace UI
│   │   │   ├── ai-elements/      # AI-specific elements
│   │   │   ├── ui/               # Shadcn/ui component library
│   │   │   └── landing/         # Landing page
│   │   └── hooks/               # React hooks
│   └── public/                  # Static assets
├── skills/
│   └── public/                  # Built-in skills (16+ skill definitions)
├── docker/                       # Docker and Kubernetes configs
├── scripts/                      # Deployment and utility scripts
└── docs/                         # Additional documentation
```

---

## Summary

| Priority | Feature | Description |
|----------|---------|-------------|
| CORE | Skills & Tools | Extensible skill framework with 16+ built-in skills |
| CORE | Sub-Agent Orchestration | Parallel task decomposition and synthesis |
| CORE | Sandbox Execution | Isolated Docker/K8s execution environments |
| CORE | Context Engineering | Aggressive context management and summarization |
| CORE | Long-Term Memory | Persistent cross-session memory |
| CORE | Multi-Channel IM | Telegram, Slack, Feishu/Lark integrations |
| CORE | Model Agnostic | OpenAI-compatible API with CLI provider support |
| SECONDARY | Claude Code Integration | Terminal CLI integration |
| SECONDARY | Python Client | Embedded library for programmatic access |
| SECONDARY | Guardrails | Safety and content filtering |
| SECONDARY | Gateway API | HTTP API for external integrations |
| SECONDARY | Artifact System | Rich content rendering |
| SECONDARY | InfoQuest Integration | BytePlus search/crawling |
| SECONDARY | Frontend UI | Next.js workspace interface |

# GitClaw Feature Index

## Project Overview

GitClaw is a universal git-native multimodal AI Agent framework where the agent itself is a git repository. Identity, rules, memory, tools, and skills are all version-controlled files.

---

## Core Features (Priority Tier 1)

### 1. Git-Native Agent Architecture
**Description:** Agents are git repositories with version-controlled identity, rules, memory, tools, and skills. Enables branching, diffing, and history tracking of agent configuration.

**Key Files/Locations:**
- `src/loader.ts` - Agent directory loading/scaffolding
- `src/context.ts` - Agent context resolution
- `agent.yaml` - Model, tools, runtime config
- `SOUL.md`, `RULES.md`, `DUTIES.md` - Agent identity and behavior files

---

### 2. SDK Programmatic Interface
**Description:** TypeScript SDK for in-process agent execution with streaming responses, tool callbacks, and lifecycle hooks. Mirrors Claude Agent SDK pattern.

**Key Files/Locations:**
- `src/sdk.ts` - Main SDK implementation
- `src/sdk-types.ts` - Type definitions
- `src/sdk-hooks.ts` - Hook integration
- `src/exports.ts` - Public API exports

---

### 3. Multi-Model Provider Support
**Description:** Unified interface for multiple LLM providers (Anthropic, OpenAI, Google, xAI, Groq, Mistral) via pi-ai integration with fallback chains.

**Key Files/Locations:**
- `src/index.ts` - Model routing/integration
- `src/config.ts` - Provider configuration

---

### 4. Built-in Tools (cli, read, write, memory)
**Description:** Core tools for shell execution, file operations, and git-committed memory. These are the foundational capabilities agents use to interact with the filesystem and execute commands.

**Key Files/Locations:**
- `src/tools/cli.ts` - Shell command execution
- `src/tools/read.ts` - File reading with pagination
- `src/tools/write.ts` - File creation/writing
- `src/tools/memory.ts` - Git-committed memory operations
- `src/tools/shared.ts` - Shared tool utilities

---

### 5. Declarative Tool System
**Description:** YAML-defined tools with script implementations. Tools are defined declaratively and scripts receive args as JSON on stdin.

**Key Files/Locations:**
- `src/tool-loader.ts` - YAML tool loading
- `src/tool-utils.ts` - Tool utilities

---

### 6. Plugin System
**Description:** Reusable extensions providing tools, hooks, skills, prompts, and memory layers. Supports both declarative (YAML) and programmatic (TypeScript entry point) plugins.

**Key Files/Locations:**
- `src/plugins.ts` - Plugin discovery and loading
- `src/plugin-cli.ts` - Plugin CLI commands (install, list, enable, remove)
- `src/plugin-sdk.ts` - Plugin API (registerTool, registerHook, addPrompt, registerMemoryLayer)
- `src/plugin-types.ts` - Plugin type definitions

---

### 7. Lifecycle Hooks System
**Description:** Programmatic hooks for gating, logging, and controlling agent behavior at key points: pre_tool_use, post_response, on_session_start, on_error.

**Key Files/Locations:**
- `src/hooks.ts` - Hook execution engine

---

## Secondary Features (Priority Tier 2)

### 8. Voice UI
**Description:** Browser-based voice interface running at localhost:3333. Supports OpenAI Realtime API and Google Gemini Live for voice interactions.

**Key Files/Locations:**
- `src/voice/server.ts` - Voice server (115KB, large)
- `src/voice/ui.html` - Browser UI
- `src/voice/openai-realtime.ts` - OpenAI Realtime adapter
- `src/voice/gemini-live.ts` - Google Gemini Live adapter
- `src/voice/chat-history.ts` - Voice chat history
- `src/voice/adapter.ts` - Voice adapter abstraction

---

### 9. Skills System
**Description:** Composable instruction modules for agents. Skills contain SKILL.md with YAML frontmatter defining name/description, plus optional scripts.

**Key Files/Locations:**
- `src/skills.ts` - Skill loading and invocation
- `skills/example-skill/` - Example skill module
- `skills/gmail-email/` - Email skill example

---

### 10. Local Repo Mode (GitHub Integration)
**Description:** Clone a GitHub repo, run agent on it, auto-commit and push to a session branch. Supports session resumption.

**Key Files/Locations:**
- `src/session.ts` - Session management
- `examples/local-repo.ts` - Local repo usage example

---

### 11. Sandbox Execution
**Description:** Run agents in isolated sandbox VM for safety. Prevents destructive operations from affecting host system.

**Key Files/Locations:**
- `src/sandbox.ts` - Sandbox orchestration
- `src/tools/sandbox-cli.ts` - Sandbox command tool
- `src/tools/sandbox-read.ts` - Sandboxed file read
- `src/tools/sandbox-write.ts` - Sandboxed file write
- `src/tools/sandbox-memory.ts` - Sandboxed memory

---

### 12. Compliance & Audit Logging
**Description:** Built-in compliance validation, audit logging to .gitagent/audit.jsonl, and regulatory framework support (SOC2, GDPR).

**Key Files/Locations:**
- `src/compliance.ts` - Compliance validation
- `src/audit.ts` - Audit logging

---

### 13. Workflows
**Description:** Multi-step workflow definitions for complex agent tasks. Workflows chain tools and skills together.

**Key Files/Locations:**
- `src/workflows.ts` - Workflow execution engine

---

### 14. Agent Inheritance & Composition
**Description:** Agents can extend base agents from git URLs, mount shared dependencies, and delegate to sub-agents.

**Key Files/Locations:**
- `src/agents.ts` - Multi-agent support
- `src/loader.ts` - Inheritance resolution

---

### 15. Knowledge Base
**Description:** Knowledge entries for agent context. Files in knowledge/ directory indexed and available to agent.

**Key Files/Locations:**
- `src/knowledge.ts` - Knowledge loading

---

### 16. Schedules & Task Runner
**Description:** Scheduled task execution for agents. Define recurring tasks or delayed execution.

**Key Files/Locations:**
- `src/schedules.ts` - Schedule definitions
- `src/schedule-runner.ts` - Schedule execution

---

### 17. Learning & Skill Capture
**Description:** Learning system for agents to capture new skills from behavior. Includes task tracking and photo capture.

**Key Files/Locations:**
- `src/learning/` - Learning system
- `src/tools/task-tracker.ts` - Task tracking
- `src/tools/skill-learner.ts` - Skill learning
- `src/tools/capture-photo.ts` - Photo capture tool

---

### 18. Composio Integration
**Description:** Integration with Composio for extended tool actions.

**Key Files/Locations:**
- `src/composio/adapter.ts` - Composio adapter
- `src/composio/client.ts` - Composio client

---

## Feature Summary Table

| # | Feature | Priority | Key Files |
|---|---------|----------|-----------|
| 1 | Git-Native Agent Architecture | Core | loader.ts, context.ts |
| 2 | SDK Programmatic Interface | Core | sdk.ts, sdk-types.ts |
| 3 | Multi-Model Support | Core | index.ts, config.ts |
| 4 | Built-in Tools (cli/read/write/memory) | Core | tools/cli.ts, tools/read.ts, tools/write.ts, tools/memory.ts |
| 5 | Declarative Tool System | Core | tool-loader.ts |
| 6 | Plugin System | Core | plugins.ts, plugin-cli.ts, plugin-sdk.ts |
| 7 | Lifecycle Hooks | Core | hooks.ts |
| 8 | Voice UI | Secondary | voice/server.ts, voice/ui.html |
| 9 | Skills System | Secondary | skills.ts |
| 10 | Local Repo Mode | Secondary | session.ts |
| 11 | Sandbox Execution | Secondary | sandbox.ts |
| 12 | Compliance & Audit | Secondary | compliance.ts, audit.ts |
| 13 | Workflows | Secondary | workflows.ts |
| 14 | Agent Inheritance | Secondary | agents.ts, loader.ts |
| 15 | Knowledge Base | Secondary | knowledge.ts |
| 16 | Schedules | Secondary | schedules.ts, schedule-runner.ts |
| 17 | Learning System | Secondary | learning/, task-tracker.ts |
| 18 | Composio Integration | Secondary | composio/adapter.ts |

---

## Directory Structure Reference

```
gitclaw/
├── src/                    # Main source code
│   ├── index.ts            # Entry point
│   ├── sdk.ts              # SDK
│   ├── plugins.ts          # Plugin system
│   ├── hooks.ts            # Lifecycle hooks
│   ├── tools/              # Built-in tools
│   ├── voice/               # Voice UI
│   ├── composio/           # Composio integration
│   ├── learning/           # Learning system
│   └── ...
├── skills/                 # Skill modules
├── agents/                 # Sub-agent definitions
├── memory/                 # Git-committed memory
├── examples/               # Usage examples
├── agent.yaml              # Agent manifest
├── SOUL.md                 # Agent identity
├── RULES.md                # Behavioral rules
└── install.sh              # Installer
```

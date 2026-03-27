# Hermes Agent Feature Index

## Core Features

- **Self-Improving Agent with Learning Loop** — Built-in learning system that creates skills from experience, nudges itself to persist knowledge, and improves skills during use. Includes FTS5 session search with LLM summarization for cross-session recall. (`agent/`, `skills/`, `honcho_integration/`)
  *Priority: core*

- **Multi-Platform Messaging Gateway** — Unified gateway delivering conversations to Telegram, Discord, Slack, WhatsApp, Signal, DingTalk, Matrix, Mattermost, Email, SMS, and Home Assistant. Voice memo transcription and cross-platform continuity. (`gateway/`, `gateway/platforms/`)
  *Priority: core*

- **Terminal TUI with Streaming Output** — Full terminal interface with multiline editing, slash-command autocomplete, conversation history, interrupt-and-redirect, and real-time streaming tool output. (`hermes_cli/`, `cli.py`)
  *Priority: core*

- **Multi-Model LLM Support** — Use any model provider — Nous Portal, OpenRouter (200+ models), z.ai/GLM, Kimi/Moonshot, MiniMax, OpenAI, or custom endpoints. Switch models with `hermes model` command. (`model_tools.py`, `agent/models.py`)
  *Priority: core*

- **Multi-Environment Execution Backends** — Six terminal backends: local, Docker, SSH, Daytona, Singularity, and Modal. Daytona and Modal offer serverless persistence with hibernation when idle. (`tools/environments/`, `environments/`)
  *Priority: core*

- **Skills System with Procedural Memory** — Agent-curated skill creation after complex tasks. Skills self-improve during use. Compatible with agentskills.io open standard. Includes skill management, sync, and guardrails. (`skills/`, `tools/skills_*.py`, `tools/skill_manager_tool.py`)
  *Priority: core*

- **Natural Language Cron Scheduling** — Built-in cron scheduler with platform delivery. Define daily reports, nightly backups, weekly audits in natural language. (`cron/`, `tools/cronjob_tools.py`)
  *Priority: core*

## Secondary Features

- **Subagent Delegation & Parallelization** — Spawn isolated subagents for parallel workstreams. Write Python scripts that call tools via RPC, collapsing multi-step pipelines into zero-context-cost turns. (`tools/delegate_tool.py`)
  *Priority: secondary*

- **Research-Ready RL Training Tools** — Batch trajectory generation, Atropos RL environment integration, trajectory compression for training next-generation tool-calling models. (`rl_cli.py`, `batch_runner.py`, `trajectory_compressor.py`, `tinker-atropos/`)
  *Priority: secondary*

- **Persistent Memory & User Modeling** — Honcho dialectic user modeling. Persistent memory, user profiles across sessions. FTS5 session search with LLM summarization. (`honcho_integration/`, `tools/memory_tool.py`, `tools/session_search_tool.py`)
  *Priority: secondary*

- **MCP (Model Context Protocol) Integration** — Connect any MCP server for extended capabilities. Includes OAuth support for MCP authentication. (`tools/mcp_tool.py`, `agent/mcp_config.py`)
  *Priority: secondary*

- **Context Files for Project Shaping** — Project context files that shape every conversation. AGENTS.md workspace instructions. (`AGENTS.md`, `CONTRIBUTING.md`)
  *Priority: secondary*

- **OpenClaw Migration Path** — Automatic migration from OpenClaw including SOUL.md persona, memories, skills, command allowlist, messaging settings, API keys, and TTS assets. (`agent/claw.py`, `setup-hermes.sh`)
  *Priority: secondary*

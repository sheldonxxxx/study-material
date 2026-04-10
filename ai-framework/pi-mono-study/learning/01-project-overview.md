# pi-mono Project Overview

## What is pi-mono?

pi-mono is a monorepo containing tools for building AI agents and managing LLM deployments. The project provides a multi-provider LLM runtime with a unified API, an agent runtime with tool calling capabilities, and multiple user-facing interfaces (CLI, TUI, web UI, Slack bot).

**Repository:** `/Users/sheldon/Documents/claw/reference/pi-mono`
**Node.js:** >=20.0.0
**Type:** npm workspaces monorepo

## The Seven Packages

| Package | Purpose | Entry Point |
|---------|---------|-------------|
| `@mariozechner/pi-ai` | Unified multi-provider LLM API client | `src/index.ts` |
| `@mariozechner/pi-agent-core` | Agent runtime with tool calling and state management | `src/index.ts` |
| `@mariozechner/pi-coding-agent` | Interactive coding agent CLI (the main `pi` command) | `src/cli.ts` |
| `@mariozechner/pi-tui` | Terminal UI library with differential rendering | `src/index.ts` |
| `@mariozechner/pi-web-ui` | Web components for AI chat interfaces | `src/index.ts` |
| `@mariozechner/pi-mom` | Slack bot that delegates messages to the pi coding agent | `src/index.ts` |
| `@mariozechner/pi-pods` | CLI for managing vLLM deployments on GPU pods | `src/main.ts` |

## Architecture

The project follows a layered architecture:

```
pi-coding-agent (CLI)
    ├── pi-agent-core (Runtime)
    │       └── pi-ai (Multi-provider LLM API)
    ├── pi-tui (Terminal UI)
    └── pi-web-ui (Web UI)
```

The `pi-ai` package handles abstraction over 15+ LLM providers including Anthropic, OpenAI, Google, Azure, Mistral, and Amazon Bedrock. The `pi-agent-core` provides the agent loop with tool calling capabilities. The `pi-coding-agent` combines these with a polished CLI experience.

## Key Features

**Multi-Provider Support:** A single unified interface across Anthropic, OpenAI, Google Gemini, Azure OpenAI, Mistral, Amazon Bedrock, and more. Provider-specific optimizations exist for each.

**Interactive CLI:** The main `pi` command provides an interactive coding agent experience with session management, branching, compaction, and extensibility.

**Tool System:** The agent supports tools for Read, Write, Edit, Bash, Grep, Glob, and more. Tools are implemented with security considerations including output truncation and sanitization.

**Session Management:** Agent sessions support compaction (context summarization), branching, retry, and concurrent operations.

**Multiple Interfaces:** TUI (terminal), Web UI (web components), and Slack bot integration.

## Development Commands

```bash
npm install          # Install all dependencies
npm run build        # Build all packages
npm run dev          # Run all packages in dev mode
npm run check        # Lint, format, type check
npm run test         # Run tests across workspaces
```

## Project-Specific Configuration

The `.pi/` directory contains Claude agent configuration for the project including prompt templates in `.pi/prompts/` and VSCode extensions in `.pi/extensions/`.

## Build Order

Packages are built in dependency order:
1. `pi-tui` (foundation for UI)
2. `pi-ai` (LLM provider API)
3. `pi-agent-core` (runtime)
4. `pi-coding-agent` (CLI)
5. `pi-mom` (Slack bot)
6. `pi-web-ui` (web components)
7. `pi-pods` (GPU pod CLI)

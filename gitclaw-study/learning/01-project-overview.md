# Project Overview: GitClaw

## What is GitClaw?

GitClaw is a universal git-native AI agent framework where the agent itself lives inside a git repository. Identity, rules, memory, tools, and skills are all version-controlled files.

## Core Philosophy: Agents as Repos

Instead of scattered configuration files, GitClaw flips the model:

- **`agent.yaml`** -- model, tools, runtime config
- **`SOUL.md`** -- personality and identity
- **`RULES.md`** -- behavioral constraints
- **`memory/`** -- git-committed memory with full history
- **`tools/`** -- declarative YAML tool definitions
- **`skills/`** -- composable skill modules
- **`hooks/`** -- lifecycle hooks (script or programmatic)
- **`plugins/`** -- reusable extensions

Fork an agent. Branch a personality. `git log` your agent's memory. Diff its rules.

## Key Features

### SDK (`gitclaw` npm package)

Programmatic access to agents via `query()`:

```typescript
import { query } from "gitclaw";

for await (const msg of query({
  prompt: "Refactor the auth module",
  dir: "/path/to/agent",
  model: "anthropic:claude-sonnet-4-5-20250929",
})) {
  // msg.type: delta, assistant, tool_use, tool_result, system, user
}
```

### Multi-Model Support

Works with any LLM provider supported by `pi-ai`: anthropic, openai, google, xai, groq, mistral.

### Tools

**Built-in tools:** `cli`, `read`, `write`, `memory`

**Declarative tools:** YAML-defined in `tools/*.yaml` with script-based implementation

**Custom tools:** Register via SDK `tool()` helper

### Hooks

Lifecycle hooks for gating, logging, and control:

- `pre_tool_use` -- block/modify tool calls
- `post_response` -- run after responses
- `on_error` -- error handling
- `on_session_start` -- session initialization

### Skills

Composable instruction modules in `skills/<name>/SKILL.md`

### Plugins

Reusable extensions with `plugin.yaml` manifest providing tools, hooks, skills, prompts, and memory layers.

### Sandbox Mode

Run agents in isolated VM (via e2b) with `--sandbox` flag.

### Local Repo Mode

Clone a GitHub repo and work on a session branch:

```bash
gitclaw --repo https://github.com/org/repo --pat ghp_xxx "Fix the login bug"
```

### Compliance & Audit

Built-in compliance validation:

```yaml
compliance:
  risk_level: high
  human_in_the_loop: true
  data_classification: confidential
  regulatory_frameworks: [SOC2, GDPR]
```

Audit logs written to `.gitagent/audit.jsonl` with full tool invocation traces.

## Architecture

```
src/
├── index.ts          # CLI entry point
├── sdk.ts            # Core SDK (query function)
├── exports.ts        # Public API surface
├── loader.ts         # Agent config loader
├── tools/            # Built-in tools (cli, read, write, memory)
├── voice/            # Voice mode (OpenAI Realtime adapter)
├── hooks.ts          # Script-based hooks
├── sdk-hooks.ts      # Programmatic SDK hooks
├── skills.ts         # Skill expansion
├── workflows.ts      # Workflow metadata
├── agents.ts         # Sub-agent metadata
├── compliance.ts     # Compliance validation
├── audit.ts          # Audit logging
├── sandbox.ts        # Sandbox VM integration
└── plugins.ts        # Plugin discovery/loading
```

## Technical Stack

- **Runtime:** Node.js >= 20
- **Language:** TypeScript (strict mode)
- **Modules:** ESM (`"type": "module"`)
- **Core Dependencies:**
  - `@mariozechner/pi-agent-core` -- Agent runtime
  - `@mariozechner/pi-ai` -- Multi-model provider abstraction
  - `@sinclair/typebox` -- Runtime type validation
  - `js-yaml` / `yaml` -- YAML parsing
- **Optional:** `gitmachine` for sandbox mode

## Project Structure (Agent Directory)

```
my-agent/
├── agent.yaml          # Model, tools, runtime config
├── SOUL.md             # Agent identity & personality
├── RULES.md            # Behavioral rules & constraints
├── DUTIES.md           # Role-specific responsibilities
├── memory/
│   └── MEMORY.md       # Git-committed agent memory
├── tools/
│   └── *.yaml          # Declarative tool definitions
├── skills/
│   └── <name>/
│       ├── SKILL.md    # Skill instructions
│       └── scripts/    # Skill scripts
├── workflows/
│   └── *.yaml|*.md     # Multi-step workflow definitions
├── agents/
│   └── <name>/         # Sub-agent definitions
├── plugins/
│   └── <name>/         # Local plugins
├── hooks/
│   └── hooks.yaml      # Lifecycle hook scripts
├── knowledge/
│   └── index.yaml      # Knowledge base entries
├── config/
│   ├── default.yaml    # Default environment config
│   └── <env>.yaml      # Environment overrides
└── compliance/
    └── *.yaml          # Compliance & audit config
```

## License

MIT

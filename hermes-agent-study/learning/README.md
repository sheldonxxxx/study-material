# Hermes Agent — Learning Reference

## What This Is
Analysis of the [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) codebase for informing development of similar CLI-first AI agent projects.

## Key Takeaways
- **Registry pattern** for self-registering tools (~50 tools) enables zero-change extensibility
- **Base adapter pattern** for 12+ messaging platforms normalizes all platform differences
- **Security scanning at every layer**: skills guard (500+ regex patterns), Tirith binary scanner, supply-chain CI, memory/prompt injection scanning
- **Frozen snapshot pattern** preserves prompt caching benefits across sessions — mid-session memory writes update files but NOT cached system prompts
- **Skills as markdown files** with YAML frontmatter — git-friendly, no database, agentskills.io compatible
- **Comprehensive contributing docs** (27KB) with skill vs tool decision framework, security guidelines, and PR checklist

## Should Trust / Should Not Trust
- **Trust**: Architectural patterns, test isolation (HERMES_HOME redirection, 30s timeouts), security scanning depth, multi-backend abstraction
- **Do Not Trust**: README simplicity claims (codebase is 387KB run_agent.py, 329KB cli.py); linting is absent (no Ruff/Black/MyPy); fail-open threat scanning silently skips blocked content

## At a Glance
- **Language**: Python 3.11+ (CLI, agent, gateway) + TypeScript (docs website)
- **Architecture**: Modular monolith with plugin-based extensions (Registry/Adapter/Strategy patterns)
- **Key Libraries**: fire, prompt_toolkit, rich, httpx, pydantic, anthropic, openai, playwright, tenacity
- **Notable Patterns**: Self-registering tools, base platform adapter, frozen snapshot caching, atomic file writes (temp+os.replace), progressive disclosure loading
- **Stars / Activity**: Active — 2,156 commits in Feb 2026, 48 in Mar 2026; date-based versioning every 5-7 days
- **License**: MIT

## Directory Structure
```
hermes-agent-study/
├── learning/          # Synthesized learning documents
│   ├── 01-project-overview.md
│   ├── 02-architecture.md
│   ├── 03-tech-stack.md
│   ├── 04-features-deep-dive.md
│   ├── 05-code-quality.md
│   ├── 06-ci-cd.md
│   ├── 07-documentation.md
│   ├── 08-security.md
│   ├── 09-dependencies.md
│   ├── 10-community.md
│   ├── 11-patterns.md
│   ├── 12-lessons-learned.md
│   └── 13-my-action-items.md
└── research/          # Preserved subagent evidence
    ├── 01-topology.md
    ├── 02-tech-stack.md
    ├── 03-community.md
    ├── 04-features-index.md
    ├── 05a-e-features-batch-*.md
    ├── 06-architecture.md
    ├── 07-code-quality.md
    └── 08-security-perf.md
```

## Top 3 Action Items
1. Set up Ruff + MyPy + pre-commit hooks before writing code
2. Enforce 500-line file limit via CI (largest file is 387KB)
3. Build security scanning into skill installation, command execution, context file loading, and CI

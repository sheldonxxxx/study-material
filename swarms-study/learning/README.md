# Swarms — Learning Reference

## What This Is
Analysis of the swarms multi-agent orchestration framework codebase for informing development of similar agent-based projects.

## Key Takeaways
- **LiteLLM abstraction is central** — vendor-agnostic LLM access enables 100+ model support with single interface
- **Multi-layer orchestration patterns** — 16+ swarm types (Sequential, Concurrent, Hierarchical, MixtureOfAgents, etc.) with AgentRearrange DSL as foundational
- **Conversation as first-class citizen** — memory, token counting, and history management built into core Agent class
- **AutoSwarmBuilder demonstrates AI-generated architecture** — boss agent pattern creates agents/workflows from task descriptions
- **Strong CI/CD with 14 workflows** — but code quality enforcement is inconsistent (mypy configured but not active)

## Should Trust / Should Not Trust
- **Trust**: Overall architecture decisions, LiteLLM integration pattern, multi-agent orchestration patterns, CI/CD infrastructure
- **Do Not Trust**: README claims about code quality (verify with code) — type annotations exist but mypy not enforced; Black line-length 70 is too restrictive and mismatched with target-version

## At a Glance
- **Language:** Python 3.10+
- **Architecture:** Multi-agent orchestration with layered swarm patterns
- **Key Libraries:** litellm (LLM abstraction), pydantic (validation), networkx (graph workflows), asyncio (concurrency)
- **Notable Patterns:** Strategy, Template Method, Factory, Registry, Builder, Hexagonal/Ports-and-Adapters
- **Stars / Activity:** HIGH activity — ~1,620 commits in 2025, 140 version tags
- **License:** Apache 2.0
- **Maintainer:** Kye Gomez (@kyegomez) — single-maintainer model

## Repository
https://github.com/kyegomez/swarms

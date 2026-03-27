# OASIS — Learning Reference

## What This Is
Analysis of the [CAMEL-Oasis](https://github.com/camel-ai/oasis) codebase — a Python-based multi-agent social simulation platform supporting up to 1 million LLM-powered agents simulating Twitter and Reddit interactions.

## Key Takeaways
- **Architecture is solid**: Layered design (Orchestration → Agent → Platform → Data) with async-first concurrency using semaphore-based backpressure (default 128)
- **CAMEL integration is the foundation**: OASIS extends `camel-ai`'s ChatAgent rather than rebuilding agent infrastructure; this is the right call
- **Global state is the main technical debt**: Recommendation system has unbounded memory growth; reset function exists but is never called
- **100W scale path sacrifices graph edges**: The `generate_agents_100w()` bypasses AgentGraph entirely for performance, breaking follow/unfollow operations at that scale
- **Documentation vs code mismatch**: README claims 23 actions but code has 33; Interview and Report features require ManualAction only (not autonomous)

## Should Trust / Should Not Trust
- **Trust**: Layered architecture, async concurrency model, strategy pattern for recsys, pre-commit + CI setup
- **Do Not Trust**: README claims about "23 actions" (33 actual), "dynamic recsys reset" (never called), "content moderation" (reports stored but never acted on)

## At a Glance
- **Language:** Python 3.10-3.11
- **Architecture:** Agent-Based Discrete Event Simulation with Layered Architecture
- **Key Libraries:** camel-ai 0.2.78, sentence-transformers 3.0.0, igraph, Neo4j, SQLite
- **Notable Patterns:** Factory, Strategy, Template Method, Actor/MQ, Command, Extension via Inheritance, Lazy Initialization
- **Stars / Activity:** 3,864 stars, 401 forks, 9 open PRs, 467 commits in 2025
- **License:** Apache 2.0
- **Build:** Poetry + pre-commit + GitHub Actions + pytest
- **CI Quality:** Good (Ruff, Flake8, isort, license checks) but no mypy/static type checking, no security scanning

## Study Structure
```
learning/
├── 01-project-overview.md    # Project purpose, scope, key stats
├── 02-architecture.md         # Architectural decisions and module responsibilities
├── 03-tech-stack.md           # Core technologies and dependencies
├── 04-features-deep-dive.md   # All 12 features synthesized (core + secondary)
├── 05-code-quality.md         # Quality practices, testing, tooling
├── 06-ci-cd.md               # GitHub Actions pipeline and deployment
├── 07-documentation.md        # Mintlify docs structure and contribution guide
├── 08-security.md            # Secrets, SQL injection, input validation
├── 09-dependencies.md        # 19 direct dependencies with rationale
├── 10-community.md            # CAMEL-AI org, contributing, sprints
├── 11-patterns.md            # 14 design patterns with file evidence
├── 12-lessons-learned.md      # What to emulate, avoid, and surprises
└── 13-my-action-items.md     # Prioritized action items for similar builds

research/                      # PRESERVED — evidence of parallel subagent work
├── 01-topology.md            # Directory structure and entry points
├── 02-tech-stack.md          # Dependencies, build system, runtime
├── 03-community.md           # Stars, forks, contributors, activity
├── 04-features-index.md      # 12-feature prioritized index
├── 05a-features-batch-1.md   # Features 1-3 deep dive
├── 05b-features-batch-2.md   # Features 4-6 deep dive
├── 05c-features-batch-3.md   # Features 7-9 deep dive
├── 05d-features-batch-4.md   # Features 10-12 deep dive
├── 06-architecture.md        # Architectural pattern analysis
├── 07-code-quality.md        # Code quality assessment
└── 08-security-perf.md       # Security and performance patterns
```

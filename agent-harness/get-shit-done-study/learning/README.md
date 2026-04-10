# get-shit-done — Learning Reference

## What This Is
Analysis of the GSD (get-shit-done) codebase — a meta-prompting framework for AI coding agents — for the purpose of informing development of similar agent orchestration tools.

## Key Takeaways
- **Thin orchestrator pattern**: Workflows coordinate agents but never do heavy lifting — keeps main context at 30-40%
- **Zero runtime dependencies**: Pure CommonJS with Node.js built-ins only; deliberate security/reliability tradeoff
- **File-based state machine**: `.planning/` directory with structured Markdown/JSON prevents context rot across sessions
- **Fresh context per agent**: Each executor run gets 200k fresh tokens; eliminates context accumulation bugs
- **Defense in depth**: Path traversal, prompt injection, shell injection, JSON safety all validated at multiple layers

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions, security patterns, wave-based execution model, test coverage approach
- **Do Not Trust**: README claims about simplicity (codebase is 1000+ files with significant complexity)

## At a Glance
- **Language:** JavaScript (CommonJS/`.cjs`), Node.js >= 20.0.0
- **Architecture:** Multi-Agent Orchestration with Thin Coordinators (17 specialized agents, 46 workflows)
- **Key Libraries:** None (zero runtime dependencies) — all Node.js built-ins
- **Notable Patterns:** Thin Orchestrator, File-Based State, Fresh Context Per Agent, Wave-Based Execution, Atomic Commits, Defense in Depth
- **Stars / Activity:** 42,324 stars, ~488 commits/month, 69 open PRs
- **License:** MIT
- **Maintainer:** Solo (TACHES / @glittercowboy)
- **CI/CD:** GitHub Actions — Ubuntu + macOS + Windows x Node 22/24, security scanning (prompt injection, base64, secrets), 70% coverage enforced

## Repository Structure
```
get-shit-done/
├── bin/                      # CLI entry points
├── commands/gsd/             # User-invoked slash commands (/gsd:*)
├── get-shit-done/
│   ├── agents/               # 17 specialized agent definitions
│   ├── bin/                  # Core CLI tools (19 modules)
│   ├── workflows/            # 46 workflow orchestrations
│   └── references/           # Configuration and spec documents
├── hooks/                    # Git hooks (prompt guard)
├── scripts/                  # Build utilities
└── tests/                    # Test suite (43 test files)
```

## Key Files to Reference
| Purpose | File |
|---------|------|
| Security patterns | `hooks/gsd-prompt-guard.js`, `bin/lib/security.cjs` |
| Architecture | `bin/gsd-tools.cjs`, `bin/lib/home.cjs` |
| Agent definitions | `get-shit-done/agents/` |
| Workflows | `get-shit-done/workflows/` |
| Test patterns | `tests/helpers.cjs` |
| Git integration | `get-shit-done/references/git-integration.md` |

## Study Outputs
- `learning/01-project-overview.md` — High-level summary
- `learning/02-architecture.md` — Architectural decisions
- `learning/03-tech-stack.md` — Tech choices
- `learning/04-features-deep-dive.md` — Feature analysis
- `learning/05-code-quality.md` — Quality practices
- `learning/06-ci-cd.md` — CI/CD pipeline
- `learning/07-documentation.md` — Docs audit
- `learning/08-security.md` — Security hardening
- `learning/09-dependencies.md` — Dependency analysis
- `learning/10-community.md` — Community health
- `learning/11-patterns.md` — 17 enumerated design patterns
- `learning/12-lessons-learned.md` — What to emulate/avoid
- `learning/13-my-action-items.md` — Prioritized next steps

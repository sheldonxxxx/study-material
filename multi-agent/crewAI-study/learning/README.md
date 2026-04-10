# crewAI — Learning Reference

## What This Is
Analysis of the crewAI codebase for informing development of a similar AI agent orchestration framework.

## Key Takeaways
- **Pydantic v2 as backbone** — all config, agents, tasks, and tool schemas are Pydantic models; this is the single most impactful architectural choice
- **Dual paradigm** — crews (task-based agent collaboration) and flows (event-driven state machines) are two first-class, well-integrated models
- **Event-driven via metaclass** — Flow uses a metaclass to build execution graphs at class definition time; clever and Pythonic
- **Composite memory scoring** — memory uses semantic + recency + importance scoring with configurable weights; RLM-inspired recall
- **Security gaps in OSS** — HallucinationGuardrail is a no-op stub; SQL injection in NL2SQL tool; SandboxPython marked insecure

## Should Trust / Should Not Trust
- **Trust**: Architectural patterns, Flow decorator DSL, memory system design, LLM factory/provider abstraction
- **Do Not Trust**: README claims about simplicity (verify with code); HallucinationGuardrail (stub); `S608` SQL injection warning is ignored

## At a Glance
- **Language**: Python 3.10–3.13
- **Architecture**: Hybrid — task-based (Crew/Agent/Task) + event-driven (Flow)
- **Key Libraries**: Pydantic v2, Instructor, ChromaDB, LanceDB, Textual, OpenTelemetry, LiteLLM
- **Notable Patterns**: Factory (LLM providers), Observer (event bus), Decorator (Flow/@listen), Plugin (BaseTool), Metaclass (Flow graph)
- **Stars / Activity**: 100K+ certified developers, active GitHub, stable + nightly release cadence
- **License**: MIT (per LICENSE file)
- **Monorepo**: Yes — 4 packages (`crewai`, `crewai-tools`, `crewai-files`, `devtools`) under `lib/`
- **Package Manager**: `uv` with lockfile
- **Build System**: hatchling
- **Test Ratio**: 0.78:1 (crewai), 4.7:1 (crewai-tools), 0.08:1 (crewai-files — significant gap)
- **Type Checking**: mypy strict mode with pydantic plugin (317 `type: ignore` comments)
- **Linting**: Ruff (all-in-one: E, F, B, S, RUF, N, W, I, T, PERF, PIE, TID, ASYNC, RET, UP)

## File Map

| File | Contents |
|------|----------|
| `01-project-overview.md` | Elevator pitch, core abstractions, target users |
| `02-architecture.md` | System design, package relationships, extensibility matrix |
| `03-tech-stack.md` | Python versions, dependencies, build tooling |
| `04-features-deep-dive.md` | All 12 features with implementation details |
| `05-code-quality.md` | Testing, types, linting, error handling patterns |
| `06-ci-cd.md` | GitHub Actions pipeline, test matrix, publishing |
| `07-documentation.md` | Docs structure, Mintlify setup, gaps |
| `08-security.md` | Secrets, input validation, vulnerabilities found |
| `09-dependencies.md` | All dependencies by category |
| `10-community.md` | Stars, contributors, release cadence, governance |
| `11-patterns.md` | All design patterns with code references |
| `12-lessons-learned.md` | Architectural wins, concerns, surprises |
| `13-my-action-items.md` | Prioritized P0/P1/P2 action items |

## Critical Vulnerabilities Found

1. **SQL Injection** (`NL2SQL tool`) — f-string interpolation in SQL query; flagged by Ruff S608 but ignored
2. **Insecure SandboxPython** — explicitly marked "DO NOT USE" yet shipped
3. **HallucinationGuardrail stub** — no-op implementation in OSS (direct to crewAI cloud in production)

## Notable Technical Debt

- `flow.py`: 129KB, 3257 lines — exceeds reasonable single-file size
- Hardcoded context windows for 100+ LLM models (maintenance burden)
- Duplicate sync/async guardrail logic (~200 lines each in `task.py`)
- 30+ files with bare `except Exception:` antipattern
- `HallucinationGuardrail` always passes — promises don't match implementation

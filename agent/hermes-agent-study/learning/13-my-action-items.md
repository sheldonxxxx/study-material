# My Action Items: Building a Similar Project

**Based on:** Lessons learned from Hermes Agent reverse engineering
**Priority:** High to Low within each section
**Estimated effort:** S (same-day), M (1-3 days), L (1-2 weeks)

---

## Phase 1: Project Foundation (Start Here)

### Documentation

- [ ] **Create CONTRIBUTING.md (M)**
  - Architecture diagram with core loop
  - Skill vs Tool decision framework
  - Security considerations (injection, exfil, dangerous commands)
  - Cross-platform compatibility rules
  - PR checklist (code, docs, config, architecture, testing)
  - Target: 15-20KB comprehensive guide

- [ ] **Create SKILL.md standard (S)**
  - Define YAML frontmatter schema (name, description, platforms, prerequisites)
  - Define directory structure (SKILL.md, references/, templates/, scripts/)
  - Document agentskills.io compatibility if applicable

### Tooling Setup

- [ ] **Configure Ruff (S)**
  - Linter + formatter in one tool
  - Add to pyproject.toml
  - Configure rules (no asyncio.run in threadpool, etc.)

- [ ] **Configure MyPy (S)**
  - Add to pyproject.toml
  - Set `python_version = "3.11"`
  - Configure strictness level (start with non-strict, tighten over time)

- [ ] **Add pre-commit hooks (S)**
  - ruff-format + ruff check (auto-fix + fail on issues)
  - mypy (type checking)
  - trailing-whitespace, end-of-file-fixer
  - Add `.pre-commit-config.yaml`

- [ ] **Add supply-chain-audit CI (M)**
  - Scan for .pth files
  - Scan for base64+exec patterns
  - Scan for marshal/pickle/compile usage
  - Reference: `.github/workflows/supply-chain-audit.yml` in Hermes

### Code Organization

- [ ] **Enforce 500-line file limit (S)**
  - Add to pre-commit or CI: `find . -name "*.py" -exec wc -l {} + | awk '$1 > 500 {print}'`
  - Document the rule in CONTRIBUTING.md

- [ ] **Choose registry/plugin pattern for extensibility (S)**
  - Define `BaseTool` abstract class
  - Implement `ToolRegistry.register()` pattern
  - All tools self-register at import time

---

## Phase 2: Core Architecture

### Agent Core

- [ ] **Define AIAgent class interface (M)**
  - Core loop: receive message, call LLM, handle tool calls, return response
  - Tool dispatch via registry
  - Session context propagation

- [ ] **Implement frozen snapshot pattern for prompt caching (M)**
  - System prompt snapshot at session start
  - Memory/prompt writes during session do NOT mutate snapshot
  - Snapshot refresh only on session boundary

- [ ] **Implement base adapter pattern for multi-platform (M)**
  - `BasePlatformAdapter` ABC with connect, disconnect, send, get_chat_info
  - All platform implementations inherit from base
  - No platform-specific logic duplicated

### Tool System

- [ ] **Build progressive disclosure tiers (M)**
  - Tier 0: List categories only (names + counts)
  - Tier 1: List skills within category (name + description)
  - Tier 2+: Full skill content loaded on demand

- [ ] **Implement atomic file operations (S)**
  - Temp file + os.replace() for all persistent writes
  - Apply to: config, memory, skills, state

- [ ] **Build skills as markdown files (M)**
  - SKILL.md with YAML frontmatter
  - Optional references/, templates/, scripts/, assets/
  - Store in ~/.yourproject/skills/

- [ ] **Build Skills Guard with security scanning (L)**
  - Regex patterns for injection, exfil, persistence, destructive
  - LLM secondary scan for contextual analysis
  - Trust levels: builtin, trusted, community, agent-created
  - Install policy matrix per trust level

### Context & Memory

- [ ] **Implement context file system (M)**
  - Support .project.md at git root (walks up from CWD)
  - Priority: .project.md > PROJECT.md > AGENTS.md > CLAUDE.md
  - Threat scanning before injection
  - Head+tail truncation for long files (40% head, 50% tail, 10% truncated)

- [ ] **Implement memory store (M)**
  - Atomic file writes
  - Bounded character/token limits
  - Section sign delimiter between entries
  - Prompt injection scanning

- [ ] **Implement FTS session search (L)**
  - SQLite FTS5 virtual table
  - LLM summarization of matched sessions
  - Delegation chain resolution to avoid duplicates

---

## Phase 3: Security

### At Every Layer

- [ ] **Add dangerous command detection (S)**
  - Regex patterns for: sudo, rm -rf, drop database, >/dev/
  - Approval required before execution

- [ ] **Add prompt injection scanning (S)**
  - Pattern: `ignore previous instructions`, `you are now`, `do not tell`
  - Zero-width unicode detection
  - Block and log attempts

- [ ] **Add secret redaction (S)**
  - API key patterns (ghp_, sk-, Bearer, token=)
  - Sanitize error messages before LLM
  - Sanitize logs

### Secrets & Auth

- [ ] **Use .env for secrets (S)**
  - Never commit .env
  - 0600 permissions on .env file
  - 0700 permissions on config directory

- [ ] **Implement user allowlisting (M)**
  - Per-platform user ID allowlists
  - deny-by-default gate

### Supply Chain

- [ ] **Add binary pre-execution scanning (L)**
  - Reference Tirith pattern from Hermes
  - SHA-256 verification for downloaded binaries
  - Configurable fail-open/fail-closed

---

## Phase 4: Integration Features

### Multi-Backend Execution

- [ ] **Abstract execution environments (M)**
  - `BaseEnvironment` ABC with execute, cleanup
  - Implement: local, docker
  - Optional: SSH, Daytona, Modal, Singularity

- [ ] **Implement sandbox isolation (M)**
  - CPU, memory, disk limits
  - Network isolation where possible
  - Hermes env var blocklist for subprocesses

### CLI & TUI

- [ ] **Build slash command registry (M)**
  - `CommandDef` dataclass
  - Auto-complete from registry
  - Gateway + CLI unified commands

- [ ] **Implement streaming output (M)**
  - Server-sent events or WebSocket
  - Progressive message editing
  - Interrupt support

### Platform Adapters (if messaging gateway)

- [ ] **Start with Telegram + Discord (M)**
  - Both inherit from BasePlatformAdapter
  - MarkdownV2 escaping for Telegram
  - Slash commands for Discord

- [ ] **Add more platforms incrementally (L)**
  - Follow adapter pattern strictly
  - Each platform: one file, ~500 lines max

---

## Phase 5: Advanced Features

### RL Training Readiness

- [ ] **Design trajectory format early (M)**
  - Standard schema: prompt, conversations, metadata, tool_stats
  - JSONL output for HuggingFace datasets
  - Tool name normalization across runs

- [ ] **Implement trajectory compression (L)**
  - Protect first turns, last N turns
  - Compress middle with LLM summary
  - Token counting with appropriate tokenizer

### Subagent Delegation

- [ ] **Implement delegation pattern (M)**
  - Spawn child agents with restricted toolsets
  - Depth limiting (max 2 levels)
  - Credential inheritance options
  - Progress callback to parent

### Cron Scheduling

- [ ] **Build natural language cron parser (M)**
  - Support: "30m", "every 30m", "0 9 * * *", ISO timestamps
  - Use croniter for validation

- [ ] **Implement job delivery (M)**
  - local (file), origin (creator), platform-specific
  - Grace period for missed jobs
  - Skill loading at runtime (not at schedule time)

---

## Phase 6: Testing & Quality

### Test Infrastructure

- [ ] **Set up pytest (S)**
  - pytest-asyncio, pytest-xdist
  - Test isolation: redirect all filesystem ops to tmp_path
  - 30-second timeout per test

- [ ] **Write security regression tests (M)**
  - Dangerous command detection
  - Prompt injection blocking
  - Secret redaction verification

- [ ] **Write platform adapter tests (M)**
  - Mock platform APIs
  - Test message normalization
  - Test streaming and interrupts

### CI/CD

- [ ] **Add tests workflow (S)**
  - Run on Python 3.11, 3.12
  - Skip integration tests unless API keys present
  - Parallel execution with pytest-xdist

- [ ] **Add docs build check (S)**
  - If using Docusaurus: lint diagrams, build site
  - Check all links are valid

---

## Phase 7: Community (When Ready)

### For Open Source

- [ ] **Add CODEOWNERS file (S)**
  - Define code ownership by area
  - Require review from owners on changes

- [ ] **Create issue templates (S)**
  - Bug report: reproduction steps, component, platform, version
  - Feature request: problem, proposed solution, scope estimation

- [ ] **Create PR template (S)**
  - What/Why fields
  - Type of change checkboxes
  - How to test section
  - Checklist: tests pass, docs updated, cross-platform considered

### Release Process

- [ ] **Set up release cadence (S)**
  - Semantic versioning (v0.1.0, v0.2.0)
  - Date-based tags for continuous delivery
  - CHANGELOG.md generation

---

## Priority Summary

| Priority | Items | Phase |
|----------|-------|-------|
| P0 - Before code | Ruff, MyPy, pre-commit, 500-line limit, CONTRIBUTING.md | Phase 1 |
| P1 - Core | Registry pattern, base adapter, frozen snapshot, atomic writes | Phase 2 |
| P2 - Security | Command detection, injection scanning, secret redaction | Phase 3 |
| P3 - UX | Progressive disclosure, context files, streaming, slash commands | Phase 2-4 |
| P4 - Advanced | Skills Guard, RL trajectories, delegation, cron | Phase 5 |
| P5 - Later | Additional platforms, Nix flakes, community docs | Phase 6-7 |

---

## Anti-Patterns to Never Introduce

1. **Never** commit code without linting CI
2. **Never** allow files over 500 lines
3. **Never** use `print()` in production code
4. **Never** write directly to files (use atomic rename)
5. **Never** skip security scanning for "internal" tools
6. **Never** hardcode secrets in source
7. **Never** mutate cached content mid-session

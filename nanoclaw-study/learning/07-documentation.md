# Documentation

## Official Documentation Site

**Live site**: [docs.nanoclaw.dev](https://docs.nanoclaw.dev)

The official documentation is the primary source of truth for users. The `docs/` directory in the repository contains original design documents and developer references, which are published to the documentation site.

## Documentation Structure

### Repository Docs (`docs/`)

| File | Published Location | Purpose |
|------|-------------------|---------|
| `docs/README.md` | - | Index of design docs vs published docs |
| `docs/SPEC.md` | [Architecture](https://docs.nanoclaw.dev/concepts/architecture) | Full architecture specification |
| `docs/SECURITY.md` | [Security model](https://docs.nanoclaw.dev/concepts/security) | Security design |
| `docs/REQUIREMENTS.md` | [Introduction](https://docs.nanoclaw.dev/introduction) | Design requirements |
| `docs/skills-as-branches.md` | [Skills system](https://docs.nanoclaw.dev/integrations/skills-system) | Feature delivery via skills |
| `docs/DEBUG_CHECKLIST.md` | [Troubleshooting](https://docs.nanoclaw.dev/advanced/troubleshooting) | Debugging guide |
| `docs/docker-sandboxes.md` | [Docker Sandboxes](https://docs.nanoclaw.dev/advanced/docker-sandboxes) | Micro VM isolation |
| `docs/APPLE-CONTAINER-NETWORKING.md` | [Container runtime](https://docs.nanoclaw.dev/advanced/container-runtime) | Apple Container networking |
| `docs/SDK_DEEP_DIVE.md` | (Reference) | Claude Agent SDK internals |

### README Files

| File | Language | Purpose |
|------|----------|---------|
| `README.md` | English | Primary documentation |
| `README_zh.md` | Chinese (Simplified) | Translated documentation |
| `README_ja.md` | Japanese | Translated documentation |

### Contributing Guide (`CONTRIBUTING.md`)

Covers:
- Before-you-start checklist (searching existing PRs/issues)
- Source code change philosophy (accepted vs not accepted)
- Four skill types with examples
- SKILL.md format specification
- Testing requirements
- PR description guidelines

### CLAUDE.md

Internal developer reference at repo root. Contains:
- Quick context on architecture
- Key files inventory
- Skills taxonomy
- Development commands
- Troubleshooting tips

### Project Metadata

| File | Purpose |
|------|---------|
| `CHANGELOG.md` | Breaking changes and release history |
| `CONTRIBUTORS.md` | Contributor acknowledgments |
| `LICENSE` | MIT license |
| `repo-tokens/README.md` | Context token tracking |

## Skills Documentation

Skills follow the [Claude Code skills standard](https://code.claude.com/docs/en/skills).

### Skill Types

1. **Feature skills** (branch-based) - Located at `.claude/skills/<name>/SKILL.md` with code on `skill/*` branches
2. **Utility skills** - Self-contained with code files alongside SKILL.md
3. **Operational skills** - Instruction-only, no code (e.g., `/setup`, `/debug`)
4. **Container skills** - Loaded inside agent containers at runtime (`container/skills/`)

### SKILL.md Format

```markdown
---
name: my-skill
description: What this skill does and when to use it.
---

Instructions here...
```

Rules:
- Under 500 lines
- `name`: lowercase, alphanumeric + hyphens, max 64 chars
- `description`: Required for Claude to know when to invoke

## Architecture Documentation

### SPEC.md (25KB)

Comprehensive specification covering:
- Overview and goals
- User stories and workflows
- System architecture
- Channel system design
- IPC mechanism
- Group isolation
- Container model
- Security considerations
- Configuration
- API design

### REQUIREMENTS.md (8KB)

Design requirements including:
- Problem statement
- Goals and non-goals
- User requirements
- Functional requirements
- Non-functional requirements
- Acceptance criteria

### SECURITY.md (6.7KB)

Security model documentation covering:
- Threat model
- Security measures
- Credential handling
- Container isolation
- Access control

## Local Development Docs

Quick reference in `CLAUDE.md`:

```bash
npm run dev          # Run with hot reload
npm run build        # Compile TypeScript
./container/build.sh # Rebuild agent container
```

## External Links

- **Website**: [nanoclaw.dev](https://nanoclaw.dev)
- **Docs**: [docs.nanoclaw.dev](https://docs.nanoclaw.dev)
- **Discord**: [discord.gg/VDdww8qS42](https://discord.gg/VDdww8qS42)

# Documentation

## Repository Documentation

### README.md (Root)

- Project overview and description
- Package table with descriptions
- Quick start development commands
- Links to CONTRIBUTING.md and AGENTS.md

### CONTRIBUTING.md

**Core philosophy:** "You must understand your code."

**Key rules:**
- AI-generated code acceptable if contributor understands it
- Must be able to explain changes and interactions
- Issue-first approach before PRs (approval gate)
- `npm run check` and `./test.sh` must pass before PR
- Changelog entries added by maintainers only
- New providers to `packages/ai` require tests per AGENTS.md

### AGENTS.md

Project-specific rules for AI agents working in the codebase.

**Sections:**
- First message protocol (read relevant README.md files)
- Code quality rules (no `any`, no inline imports, no dynamic imports for types)
- Command restrictions (check but never run dev/build/test automatically)
- GitHub issues workflow (read all comments, use `gh` CLI)
- OSS weekend mode management
- PR workflow (analyze without pulling, feature branches, merge into main)
- Changelog format and rules
- **New LLM Provider guide** (comprehensive, multi-file changes required)
- Releasing process (lockstep versioning)
- Critical tool usage and git rules for parallel agents

## Per-Package Documentation

Each package has:
- `README.md` -- Installation, usage, configuration
- `CHANGELOG.md` -- Version history (Unreleased section for pending changes)
- Package-specific docs in `docs/` folder (e.g., `packages/coding-agent/docs/`)

## Issue Templates

Located in `.github/ISSUE_TEMPLATE/`

### bug.yml

Labels: `bug`

Fields:
- What happened? (required)
- Steps to reproduce (required)
- Expected behavior (optional)
- Version (optional)

### contribution.yml

**Required for new contributors before PR approval**

Fields:
- What do you want to change? (required)
- Why? (required)
- How? (optional)

Note: "Keep this short. If it doesn't fit on one screen, it's too long."

## Documentation Standards

**For humans:**
- Concise, technical prose
- No emojis in commits, issues, PR comments
- No fluff or cheerful filler text
- Kind but direct tone

**For AI agents (AGENTS.md):**
- Strict rules on import patterns
- Required test coverage
- Multi-file change protocols for new providers
- Git safety rules for parallel work

## License

MIT

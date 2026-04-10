# Community

## Overview

The Vercel AI SDK is an open-source project maintained by Vercel with contributions from the developer community. The project values community input through bug reports, feature suggestions, documentation improvements, and code contributions.

## Repository Information

| Attribute | Value |
|-----------|-------|
| **Repository** | https://github.com/vercel/ai |
| **Documentation** | https://ai-sdk.dev/docs |
| **License** | Apache-2.0 |
| **Package Manager** | pnpm |

## Contribution Channels

### Bug Reports

1. **Check existing issues** before creating new ones
2. **Create new issue** with:
   - Clear title describing the bug
   - Detailed description with reproduction steps
   - Relevant logs and error messages
   - Environment details (Node version, OS, etc.)
3. **Label as `bug`** for maintainer identification

### Feature Suggestions

1. **Search existing suggestions** in the issue tracker
2. **Create new issue** describing:
   - The enhancement and its benefits
   - Potential use cases
   - Alternative solutions considered
3. **Label as `enhancement`**

### Documentation Improvements

- **Location**: Docs are in `content/` directory
- **Small fixes**: Edit directly on GitHub or via github.dev (press `.`)
- **Larger changes**: Follow code contribution workflow
- **Prettier issues**: Run `ultracite fix` to resolve formatting

## Code Contributions

### Development Environment Requirements

| Requirement | Version |
|-------------|---------|
| Node.js | v22 (recommended) |
| pnpm | v10+ |

### Initial Setup

```bash
# 1. Fork the repository
# 2. Clone your fork
git clone https://github.com/<your-username>/ai.git

# 3. Install Node v22 (if not already)
# 4. Install pnpm
npm install -g pnpm@10
# or
brew install pnpm

# 5. Install dependencies (includes Husky setup)
pnpm install

# 6. Build all packages
pnpm build
```

### Local Development Workflow

#### Building Packages

```bash
cd packages/<package-name>
pnpm build              # Production build
pnpm build:watch        # Watch mode for development
```

Changes to `dist/` are picked up by examples automatically.

#### Testing

```bash
# Test a specific package
cd packages/<package-name>
pnpm test               # Run all tests (Node + Edge)

# Or test from root (excludes examples)
pnpm test
```

Some packages support:
- `pnpm test:watch` - Watch mode
- `pnpm test:node` - Node.js runtime only
- `pnpm test:edge` - Edge runtime only

### Pull Request Process

#### 1. Create a Branch

```bash
git checkout -b feat/package-name-my-feature
```

#### 2. Add a Changeset

Required for any changes to production packages:

```bash
pnpm changeset
```

- Select the packages you've modified
- Choose **patch** changeset type (unless told otherwise)
- Do NOT select example packages (not released)

**Changeset Types**:
- `patch` (default): Bug fixes, non-breaking changes
- `minor`: New features, backward-compatible
- `major`: Breaking changes (maintainers will advise)

#### 3. Add a Codemod (if applicable)

For deprecations or breaking changes, add a codemod:
- See `contributing/codemods.md` for guidance
- Helps users migrate automatically

#### 4. Commit Changes

```bash
git add .
git commit -m "feat(provider): description of changes"
```

**Commit Message Format**:
```
<type>(<package-name>): description
```

Types: `fix`, `feat`, `chore`, `docs`, `test`, `refactor`

**Pre-commit Hooks**:
- lint-staged auto-formats staged files
- If package.json staged, `pnpm install` runs automatically
- Skip hooks: `ARTISANAL_MODE=1 git commit -m "message"`

#### 5. Push and Create PR

```bash
git push origin feat/package-name-my-feature
```

Create PR via GitHub with:
- Clear title following the commit format
- Description linking related issues
- Reference any issues resolved

### PR Title Format

```
fix(ai): fix text streaming issue
feat(@ai-sdk/openai): add gpt-4o support
chore(docs): update provider docs
```

## Project Governance

### Maintainers

The project is maintained by Vercel's team:
- Core team manages releases, roadmaps, major decisions
- Community contributions reviewed by maintainers
- RFC process for significant architectural changes

### Decision Making

Architecture decisions are documented in ADRs:
- Location: `contributing/decisions/`
- Process: Proposed -> Accepted/Deprecated
- Purpose: Explain why decisions were made

Key decision areas:
- Provider architecture patterns
- API design choices
- Breaking change policies
- Package organization

## Release Process

Releases are automated via Changesets:

1. Changesets merged to main create version PRs
2. Version PR updates changelogs and versions
3. Merging version PR triggers npm publish
4. GitHub notifications sent for released packages

See `contributing/releases.md` and `contributing/pre-release-cycle.md`.

## Community Resources

### Examples (23+)

The repository includes extensive examples:

| Example | Description |
|---------|-------------|
| ai-functions | Core SDK usage |
| next | Next.js App Router |
| next-openai-pages | Next.js Pages Router |
| next-agent | AI agent patterns |
| sveltekit-openai | SvelteKit integration |
| nuxt-openai | Nuxt integration |
| express | Express.js server |
| fastify | Fastify server |
| hono | Hono server |
| anthropic | Anthropic-specific |
| google-vertex | Google Cloud Vertex |
| langchain | LangChain integration |

Examples are NOT published to npm but demonstrate SDK usage.

### Contributing Guides

| Guide | Purpose |
|-------|---------|
| `add-new-provider.md` | Add new AI providers |
| `add-new-model.md` | Add new models to providers |
| `add-new-tool-to-registry.md` | Extend tool registry |
| `testing.md` | Write tests and fixtures |
| `documentation.md` | Improve docs |
| `provider-architecture.md` | Understand provider pattern |

## Code of Conduct

As an open-source project under Vercel:
- Follow GitHub Community Guidelines
- Be respectful and constructive
- Focus on technical merit
- Help newcomers get started

## Issue Labels

Common labels used:
- `bug` - Bug reports
- `enhancement` - Feature suggestions
- `documentation` - Doc improvements
- `good first issue` - Beginner-friendly tasks
- `help wanted` - Seeking community assistance
- `provider` - Provider-specific issues
- `framework` - Framework integration issues

## Support Channels

- **GitHub Issues**: Bug reports and feature requests
- **GitHub Discussions**: Questions and community discussion
- **Discord**: Vercel community server (for general AI/Next.js help)
- **Twitter/X**: @vercel announcements

## Recognition

Contributors are recognized via:
- GitHub's contributor graph
- Release notes acknowledgment
- Potential future community spotlight features

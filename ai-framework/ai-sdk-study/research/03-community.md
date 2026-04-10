# Community Health and Project Metadata

## Repository Overview

**Project:** Vercel AI SDK
**Repository:** https://github.com/vercel/ai
**License:** Apache-2.0
**Monorepo:** Yes (pnpm workspaces + Turborepo)

## Repository Metrics

GitHub CLI could not retrieve live metrics (authentication not available). Based on codebase analysis:

- **Active development:** Yes - recent commits show continuous integration work
- **Release cadence:** Regular via Changesets with beta pre-release cycle
- **Node.js support:** ^18.0.0 || ^20.0.0 || ^22.0.0 || ^24.0.0

## Governance and Maintenance

### Ownership Files
| File | Status |
|------|--------|
| CODEOWNERS | Not present |
| MAINTAINERS | Not present |
| AUTHORS | Not present |

**Note:** The project appears to be maintained directly by Vercel, with no explicit CODEOWNERS or MAINTAINERS file. Community contributions are managed through standard GitHub PR workflows.

### Community Guidelines
- **Code of Conduct:** Vercel Community Code of Conduct (https://community.vercel.com/guidelines)
- **Security Reporting:** Dedicated SECURITY.md with security@vercel.com contact
- **Support:** GitHub Issues with support label

## Contributing Infrastructure

### GitHub Templates Found
| Template | Location |
|----------|----------|
| Support Request | `.github/ISSUE_TEMPLATE/1.support_request.yml` |
| Help Discussions | `.github/DISCUSSION_TEMPLATE/help.yml` |
| Ideas/Feedback | `.github/DISCUSSION_TEMPLATE/ideas-feedback.yml` |
| Polls | `.github/DISCUSSION_TEMPLATE/polls.yml` |
| Show and Tell | `.github/DISCUSSION_TEMPLATE/show-and-tell.yml` |

### Pull Request Template
Comprehensive PR template found at `.github/pull_request_template.md` with sections for:
- Background (why the change was necessary)
- Summary (what was changed)
- Manual Verification (end-to-end testing steps)
- Checklist (tests, docs, changeset)
- Future Work
- Related Issues

### Contributing Guidelines
Comprehensive `CONTRIBUTING.md` at repository root covering:
- Bug reporting process
- Enhancement suggestions
- Documentation contribution
- Code contribution workflow
- Environment setup (pnpm v10+, Node v22)
- PR submission process (branching, changeset, commit conventions)

## Versioning and Releases

### Changeset Configuration
- **Tool:** @changesets/cli (v2.27.10)
- **Configuration:** `.changeset/config.json`
- **Base branch:** main
- **Access:** public (published packages)
- **Update internal dependencies:** patch

### Release Process
The project follows a structured pre-release cycle documented in `contributing/pre-release-cycle.md`:
1. Maintenance branch created (e.g., `release-v6.0`)
2. npm dist-tag set for maintenance releases
3. Main enters changeset pre-release mode
4. Major changeset bumps all packages
5. New spec versions seeded
6. Mock test utilities created
7. Documentation site updated

**Backporting:** Uses `backport` label on merged PRs to automatically create backport PRs.

## Dependency Management

### Renovate Bot
Active `renovate.json5` configuration with:
- Production dependencies: Weekly updates (Fridays)
- Development dependencies: Monthly (first Friday)
- Other packages: Quarterly
- GitHub Actions: Monthly (3rd Friday)
- Extends: `config:recommended` with dependency dashboard disabled

## CI/CD Infrastructure

### GitHub Actions
Found `.github/workflows/` with 20 workflow files including:
- Standard lint, test, build workflows
- `release.yml` - runs on `release-v*` branches
- `tigent.yml` - custom workflow (5942 bytes)

### Pre-commit Hooks
- **Tool:** husky
- **lint-staged:** Runs oxlint/oxfmt on staged files
- **Auto-install:** Runs `pnpm install` when package.json changes

## Architecture Governance

### Architecture Decision Records (ADRs)
Located in `contributing/decisions/` with recent ADR:
- `2026-03-11-adopt-architecture-decision-records.md`

### Architecture Documentation
| Document | Purpose |
|----------|---------|
| `architecture/provider-abstraction.md` | Provider pattern conceptual overview |
| `architecture/stream-text-loop-control.md` | Stream control architecture |
| `architecture/message-layers.md` | Message layer architecture |
| `contributing/provider-architecture.md` | Provider development guide |
| `contributing/project-philosophies.md` | Core architectural philosophies |

### Project Philosophies
Documented principles include:
- Unified provider interface (adapter pattern)
- Lean, focused mission
- Stability and backward compatibility first
- Conservative API surface
- Rule of 3 for abstraction decisions
- Experimental prefixes for new features outside major releases
- DX through consistency (for developers AND agents)

## Recent Commit Activity

Last 20 commits show active development:
- Version Packages (beta) releases - frequent
- Provider/gateway features (spend reporting, model settings)
- Documentation additions (pricing API, stream pipeline)
- Stream part refactoring
- OTEL integration improvements
- PostHog observability provider documentation

**Pattern:** Mix of feature development, refactoring, and documentation with regular beta releases.

## Community Resources

| Resource | Location/Link |
|----------|---------------|
| Documentation | https://ai-sdk.dev/docs |
| Community | https://community.vercel.com |
| Security | security@vercel.com |

## Code Quality Infrastructure

| Tool | Version | Purpose |
|------|---------|---------|
| ultracite | 7.3.2 | Linting and formatting |
| oxlint | 1.56.0 | Linter |
| oxfmt | 0.41.0 | Formatter |
| vitest | 4.1.0 | Testing |
| playwright | 1.44.1 | E2E testing |
| publint | 0.2.12 | Package linting |
| typescript | 5.8.3 | Type checking |

## Summary

The Vercel AI SDK demonstrates healthy open-source practices:
- Well-documented contributing process
- Structured release workflow with changesets
- Active dependency management via Renovate
- Comprehensive CI/CD pipeline
- Architecture governance through ADRs and documented philosophies
- Multiple community engagement channels (Discussions, Issues, Polls)
- Professional security handling process

The project is maintained by Vercel with community contributions welcome through standard GitHub workflows. Lack of CODEOWNERS/MAINTAINERS files suggests direct Vercel team maintenance rather than delegated community governance.

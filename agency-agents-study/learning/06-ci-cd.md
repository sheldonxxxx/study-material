# CI/CD

## Overview

The agency-agents repository uses **GitHub Actions** for continuous integration with a single linting workflow that validates agent files on every pull request.

## GitHub Actions Workflow

### lint-agents.yml

**File**: `.github/workflows/lint-agents.yml`

**Trigger**: Pull requests modifying agent files in any category directory:
- `design/**/*.md`
- `engineering/**/*.md`
- `game-development/**/*.md`
- `marketing/**/*.md`
- `paid-media/**/*.md`
- `sales/**/*.md`
- `product/**/*.md`
- `project-management/**/*.md`
- `testing/**/*.md`
- `support/**/*.md`
- `spatial-computing/**/*.md`
- `specialized/**/*.md`

**Jobs**:
- `lint` — Validates agent frontmatter and structure

**Steps**:
1. Checkout with full history (`fetch-depth: 0`)
2. Get changed agent files using `git diff --name-only`
3. Run `scripts/lint-agents.sh` with changed files

### Lint Script (lint-agents.sh)

The linting script validates:

| Check | Severity | Description |
|-------|----------|-------------|
| Frontmatter opening `---` | ERROR | File must start with YAML delimiter |
| Frontmatter not empty | ERROR | Must have content between delimiters |
| Required fields | ERROR | Must have: `name`, `description`, `color` |
| Recommended sections | WARN | Should have: Identity, Core Mission, Critical Rules |
| Minimum body content | WARN | Body should be >= 50 words |

**Exit Codes**:
- Exit 1 if any ERRORs exist
- Exit 0 if only WARNings or clean

## Validation Process

```
PR Opened
   |
   v
GitHub Actions Triggered
   |
   v
Get Changed Files (git diff)
   |
   v
Run lint-agents.sh
   |
   +---> ERRORS exist? --Yes--> FAIL (exit 1)
   |                                              |
   No                                             Block PR
   v
PASS (exit 0)
   |
   v
PR Can Merge
```

## What Gets Validated

### Required Frontmatter Fields

```yaml
---
name: Agent Name              # Required - unique identifier
description: One-line desc    # Required - what the agent does
color: colorname or "#hex"    # Required - for visual identification
---
```

### Recommended Sections (Warnings Only)

- **Identity & Memory** — Role, personality, background
- **Core Mission** — Primary responsibilities
- **Critical Rules** — Boundaries and constraints

### File Structure Rules

1. First line must be `---` (frontmatter opening)
2. Frontmatter must have content (not just `---` on line 2)
3. Body must have meaningful content (50+ words)

## Contribution Flow

```
Fork Repository
   |
   v
Create Branch (git checkout -b add-agent-name)
   |
   v
Add/Modify Agent File
   |
   v
Commit (git commit -m "Add [Agent Name] specialist")
   |
   v
Push (git push origin add-agent-name)
   |
   v
Open Pull Request
   |
   v
Lint Workflow Runs
   |
   +---> FAIL --> Fix Errors --> Push (auto-re-runs)
   |
   PASS
   |
   v
Maintainer Review
   |
   v
Merge
```

## What Doesn't Trigger CI

The workflow only runs on changes to agent directories. The following are explicitly excluded:
- Documentation changes (README.md, CONTRIBUTING.md)
- Script modifications
- Integration file changes (these are generated, not committed)
- Root-level files

## Build Output

Generated files (integration outputs) are **gitignored** and never committed:

```
_site/
*.mdc
CONVENTIONS.md
.windsurfrules
```

Users run `convert.sh` locally to regenerate these.

## Continuous Improvement

The CI system focuses on:
- **Fast feedback** — Only lints changed files, not entire repo
- **Clear errors** — Provides specific line/file for each issue
- **Non-blocking warnings** — Recommendations don't fail the build
- **Helpful messages** — Explains what's missing and why it matters

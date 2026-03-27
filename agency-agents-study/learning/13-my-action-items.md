# My Action Items: Building a Similar Agent Knowledge Base

## Overview

Derived from lessons learned analyzing the agency-agents repository (63K stars, 144+ agents). These are prioritized concrete steps for building a similar multi-agent knowledge base with improved quality assurance.

---

## Phase 1: Foundation (Weeks 1-2)

### 1.1 Define Canonical Source Format

**Priority:** Critical

**Actions:**
- Design YAML frontmatter schema with JSON Schema validation
- Define markdown body section structure (Identity, Mission, Deliverables, Metrics, Workflow)
- Create example agent file demonstrating full template

**Schema requirements:**
```yaml
name: string (required)
description: string (required)
color: string (hex or named, validated)
emoji: string (optional)
vibe: string (optional)
version: string (semver, recommended)
```

**Deliverable:** `schema/agent-schema.json` + `examples/template-agent.md`

---

### 1.2 Build Conversion Pipeline

**Priority:** Critical

**Actions:**
- Create `convert.sh` with tool-specific converter functions
- Implement frontmatter extraction helpers (get_field, get_body)
- Add color normalization for named colors to hex
- Implement parallel execution with job count auto-detection

**Required tools to support initially:**
- Claude Code (native .md)
- Cursor (.mdc rules)
- Aider (accumulated CONVENTIONS.md)

**Non-goals for Phase 1:** All 10 tools. Start with 2-3, expand later.

**Deliverable:** `scripts/convert.sh` with --tool and --parallel flags

---

### 1.3 Write Comprehensive CONTRIBUTING.md

**Priority:** Critical

**Actions:**
- Document agent design philosophy (5 principles)
- Create complete markdown template with all sections
- Add style guide with do/don't examples
- Define external services policy
- Add contribution checklist

**Minimum length target:** 300+ lines (agency-agents is 428)

**Deliverable:** `CONTRIBUTING.md`

---

## Phase 2: Quality Assurance (Weeks 2-3)

### 2.1 Add Schema Validation to CI

**Priority:** Critical

**Actions:**
- Create JSON Schema for agent frontmatter
- Add schema validation to `lint-agents.sh`
- Add markdownlint configuration for body formatting
- Create GitHub Actions workflow running on PR

**Validation must catch:**
- Missing required fields (name, description)
- Invalid color values
- Section header inconsistencies
- Body content minimum length

**Deliverable:** `.github/workflows/validate-agents.yml` + `scripts/lint-agents.sh` (improved)

---

### 2.2 Add Link Checking

**Priority:** High

**Actions:**
- Add broken link detection to CI pipeline
- Verify internal references exist before merge
- Check external URLs are reachable (or mark as known-dead)

**Why:** Research found referenced files that don't exist. Prevent this.

**Deliverable:** Link validation step in CI

---

### 2.3 Enforce Minimum Content Standards

**Priority:** High

**Actions:**
- Define minimum line count per section (e.g., Identity: 10 lines, Deliverables: 20 lines)
- Add code example requirements (at least one per agent)
- Require at least 3 measurable success metrics

** enforcement:** Warning in CI, not blocking (agency-agents model)

**Deliverable:** Updated `lint-agents.sh` with content checks

---

### 2.4 Create Supporting File Templates

**Priority:** High

**Actions:**
- If agents reference scripts (e.g., `qa-playwright-capture.sh`), create stub versions
- If agents reference other documents, ensure they exist
- Document the "reference contract" -- referenced files must exist

**Agency-agents failure:** Referenced files that don't exist undermine agent credibility.

**Deliverable:** All files referenced in agents must exist in repo

---

## Phase 3: Community Infrastructure (Weeks 3-4)

### 3.1 Set Up Issue and PR Templates

**Priority:** High

**Actions:**
- Create bug report template (agent file, description, evidence)
- Create new agent request template (name, category, use cases)
- Create PR template with checklist (template adherence, testing, proofreading)

**Based on agency-agents templates but add:**
- Schema validation confirmation
- Link checking pass
- Minimum content standards met

**Deliverable:** `.github/ISSUE_TEMPLATE/` and `.github/PULL_REQUEST_TEMPLATE.md`

---

### 3.2 Create CODEOWNERS File

**Priority:** Medium

**Actions:**
- Define code owners per domain directory
- Enable automated reviewer assignment on PR

**Why:** Agency-agents has monolithic maintainership risk. Distribute review ownership.

**Deliverable:** `.github/CODEOWNERS`

---

### 3.3 Document Governance

**Priority:** Medium

**Actions:**
- Create MAINTAINERS.md defining decision-making process
- Document how new agents are approved
- Define tool integration criteria (what qualifies a new tool for conversion support)

**Deliverable:** `MAINTAINERS.md`

---

### 3.4 Add Conventional Commits Enforcement

**Priority:** Low (process improvement)

**Actions:**
- Add GitHub Actions step validating commit messages
- Use `@commitlint/config-conventional`

**Deliverable:** `.github/workflows/commit-lint.yml`

---

## Phase 4: Installation & UX (Weeks 4-5)

### 4.1 Build Interactive Installer

**Priority:** High

**Actions:**
- Create `install.sh` with auto-detection of installed tools
- Add interactive TUI with ASCII box-drawing
- Handle TTY detection (non-interactive fallback)
- Implement parallel installation workers

**Copy patterns from agency-agents:**
- Color output with NO_COLOR/TERM=dumb checks
- Visible length calculation for ANSI strings
- Worker process pattern for parallel output

**Deliverable:** `scripts/install.sh`

---

### 4.2 Add Uninstall Capability

**Priority:** High

**Actions:**
- Track installed files in manifest
- Create `uninstall.sh` or add --uninstall flag to install.sh
- Document uninstall process in README

**Agency-agents gap:** No way to remove installed agents. Don't repeat.

**Deliverable:** `scripts/uninstall.sh`

---

### 4.3 Create Examples Directory

**Priority:** Medium

**Actions:**
- Create 2-3 workflow examples showing multi-agent collaboration
- Include quality gates (Reality Checker pattern)
- Document sequential handoffs and parallel activations
- Show how context is passed between agents

**Start with:**
- Simple 2-agent workflow
- Complex 5+ agent workflow with parallelization

**Deliverable:** `examples/` directory with `workflow-*.md` files

---

## Phase 5: Content Development (Ongoing)

### 5.1 Seed Initial Agent Collection

**Priority:** Critical

**Actions:**
- Create 20-30 agents in 3-5 domains (don't spread thin)
- Focus on depth over breadth initially
- Prioritize measurable success metrics
- Include real code examples (not pseudo-code)

**Domain suggestions:**
- Engineering (5-8 agents): frontend, backend, DevOps, security, data
- Product (3-5 agents): PM, designer, analyst
- Testing (2-3 agents): QA, accessibility, performance

**Minimum per agent:**
- 100 lines of content
- 3+ code examples
- 5+ measurable metrics
- Complete frontmatter

---

### 5.2 Establish Content Standards

**Priority:** Critical

**Actions:**
- Define minimum depth (e.g., 100 lines per agent)
- Create agent depth checklist
- Set expectation: 400+ line agents preferred, 100 line minimum
- Document "depth tiers" (core, extended, specialist)

**Why:** Agency-agents has 33-line agents alongside 400-line agents. Inconsistent quality erodes trust.

---

### 5.3 Build Versioning Strategy

**Priority:** Medium

**Actions:**
- Implement semantic versioning for agent collection
- Create CHANGELOG tracking new agents and changes
- Tag releases in GitHub
- Provide version pinning for users

**Agency-agents gap:** Rolling release model. Users can't lock to known state.

**Deliverable:** `CHANGELOG.md`, GitHub releases

---

## Phase 6: Operational Excellence (Weeks 6+)

### 6.1 Add ShellCheck to CI

**Priority:** Low (code quality)

**Actions:**
- Add shellcheck validation to shell scripts
- Pin to specific version (not @latest)

**Deliverable:** Updated `lint-agents.yml`

---

### 6.2 Create Pre-commit Hooks

**Priority:** Medium

**Actions:**
- Create pre-commit configuration
- Run lint-agents.sh locally before commit
- Add pre-commit hook for common issues

**Benefit:** Catch issues before CI, reduce CI queue time.

**Deliverable:** `.pre-commit-config.yaml`

---

### 6.3 Implement Incremental Linting

**Priority:** Low (optimization)

**Actions:**
- Track modified files in PR
- Only lint changed files (not full repo)
- Full lint on demand or nightly

**Current state:** Agency-agents does full lint every PR.

**Deliverable:** Modified `lint-agents.sh` with CHANGED_FILES support

---

## Priority Summary

| Priority | Item | Phase | Effort |
|----------|------|-------|--------|
| Critical | Define schema + validation | 1 | Medium |
| Critical | Build convert pipeline | 1 | Medium |
| Critical | Write CONTRIBUTING.md | 1 | Medium |
| Critical | Seed initial agents | 5 | High |
| Critical | Content standards | 5 | Medium |
| High | CI schema validation | 2 | Low |
| High | Link checking | 2 | Low |
| High | Content standards enforcement | 2 | Low |
| High | Supporting file templates | 2 | Medium |
| High | Issue/PR templates | 3 | Low |
| High | Interactive installer | 4 | Medium |
| High | Uninstall capability | 4 | Low |
| Medium | CODEOWNERS | 3 | Low |
| Medium | MAINTAINERS.md | 3 | Low |
| Medium | Examples directory | 4 | Medium |
| Medium | Versioning strategy | 5 | Medium |
| Low | Conventional commits | 3 | Low |
| Low | ShellCheck | 6 | Low |
| Low | Pre-commit hooks | 6 | Low |
| Low | Incremental linting | 6 | Low |

---

## Anti-Patterns to Avoid

1. **Don't launch without schema validation** -- malformed agents will slip through
2. **Don't reference non-existent files** -- if agents mention scripts/docs, those must exist
3. **Don't allow 33-line agents** -- minimum depth standards prevent quality inconsistency
4. **Don't skip uninstall** -- users need cleanup paths
5. **Don't ignore parallel failure** -- xargs returns 0 even when children fail

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Agent count | 20-30 at launch, grow to 100+ |
| Minimum agent depth | 100 lines |
| Preferred agent depth | 400+ lines |
| CI validation time | < 60 seconds |
| Schema validation | All required fields + color format |
| Tool integrations | 2-3 at launch |
| Workflow examples | 2-3 at launch |

---

## Next Steps After Phase 1

1. **Validate the format** -- Have 3+ people create agents using CONTRIBUTING.md, identify gaps
2. **Test the conversion** -- Run convert.sh, verify output for each tool
3. **Review community response** -- Are contributions following the template?
4. **Iterate schema** -- Real usage reveals schema deficiencies

---

*Generated from agency-agents repository analysis*
*Analysis date: 2026-03-27*

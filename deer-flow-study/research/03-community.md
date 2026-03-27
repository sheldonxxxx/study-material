# Community Health Assessment: deer-flow

**Repository:** github.com/bytedance/deer-flow
**Last Updated:** 2026-03-26
**Analysis Date:** 2026-03-26

---

## Overview

| Metric | Value |
|--------|-------|
| Stars | 47,709 |
| Forks | 5,680 |
| Open Issues | 30 |
| Open PRs | 30 |
| Total Contributors | 30 |
| Primary Languages | Python (1.58M bytes), TypeScript (666K bytes), HTML (201K bytes) |
| License | MIT |

---

## Repository Metadata

**Description:** An open-source long-horizon SuperAgent harness that researches, codes, and creates. With the help of sandboxes, memories, tools, skill, subagents and message gateway, it handles different levels of tasks that could take minutes to hours.

**Topics:** agent, agentic, agentic-framework, agentic-workflow, ai, ai-agents, deep-research, langchain, langgraph, llm, multi-agent, nodejs, podcast, python, langmanus, typescript, harness, superagent

**Repository Settings:**
- Issues enabled: Yes
- Discussions enabled: No
- Archived: No
- Visibility: Public

---

## Contributor Analysis

### Top Contributors

| Rank | Contributor | Contributions | Role |
|------|-------------|---------------|------|
| 1 | MagicCube | 606 | Primary maintainer (Bytedance) |
| 2 | hetaoBackend | 210 | Contributor |
| 3 | henry-byted | 203 | Contributor (Bytedance) |

### Activity Metrics

| Time Period | Commits |
|-------------|---------|
| 2026 YTD | 1,133 |
| Last 6 months (Oct 2025 - Mar 2026) | 1,260 |
| First year (Jun 2024 - Jun 2025) | 327 |

**Observations:**
- Project started around April 2025
- Significant growth in activity since October 2025 (1,260 commits in ~6 months vs 327 in first year)
- Active development continues with commits as recent as March 26, 2026
- 84 unique contributors in 2026 alone

---

## Release and Versioning

**Formal Releases:** No GitHub Releases created
- No tagged releases found
- SECURITY.md states: "As deer-flow doesn't provide an official release yet, please use the latest version"
- Two maintenance branches:
  - `main` for deer-flow 2.x
  - `main-1.x` for deer-flow 1.x

**Versioning Strategy:** Rolling releases via git main branch

---

## Community Infrastructure

### Contributing Guidelines

- **CONTRIBUTING.md:** Present (8,695 bytes) - comprehensive developer onboarding guide
- **LICENSE:** MIT License with dual copyright (Bytedance 2025, DeerFlow Authors 2025-2026)

### Issue Templates

- **Runtime Information Template:** Detailed template requiring OS, Python/Node/pnpm versions, reproduction steps, and git state
- Uses `needs-triage` label for new issues

### Pull Request Templates

No custom PR template found in `.github/`.

### CI/CD Pipeline

**Workflows:**
1. **backend-unit-tests.yml:** Runs on PR (opened, synchronize, reopened, ready_for_review) and main push
   - Python 3.12, uv package manager
   - 15-minute timeout
   - Runs `make test` in backend/

2. **lint-check.yml:** Linting workflow for code quality

### Documentation

- Multi-language README support: English, Chinese, French, Japanese, Russian
- Comprehensive README (27,392 bytes)
- Co-pilot onboarding instructions (`.github/copilot-instructions.md`)

---

## Security

- **SECURITY.md:** Present with vulnerability reporting process via GitHub Security
- **Security advisories:** 0 published advisories
- Active security work visible in recent commits (e.g., "Add security alerts to documents", XSS mitigation fixes)

---

## Activity Assessment

**Commit Cadence:** Very active
- Multiple commits daily
- Recent activity (March 26, 2026) includes security fixes, feature development, and dependency updates
- Average ~60+ commits per month in 2026

**Community Engagement:**
- Responsive to issues (30 open, actively triaged)
- PR activity visible with recent merges
- International contributions (China region config support, multi-language docs)

**Health Indicators:**
- Active maintainer (MagicCube with 606 contributions)
- Growing contributor base
- Regular dependency updates (dependabot activity)
- Security-conscious development (recent XSS mitigation, safe download enforcement)
- Windows compatibility improvements

---

## Key Observations

### Strengths
1. Large and growing star count (47K+) indicates high community interest
2. Active development with substantial commit velocity
3. Good diversity in contributions from both internal (Bytedance) and external developers
4. Comprehensive onboarding documentation (CONTRIBUTING.md, copilot-instructions.md)
5. Security-conscious with dedicated SECURITY.md and recent security fixes
6. Multi-language documentation reaching global audience

### Areas for Improvement
1. No formal releases - users must track main branch
2. No GitHub Discussions enabled (limited async community discussion)
3. Only one issue template (could benefit from bug/feature request templates)
4. No CODEOWNERS file visible
5. No PR template for standardized PR descriptions
6. Limited CI coverage (backend tests + lint only, no frontend E2E or integration tests in CI)

---

## Metrics Summary

| Category | Rating | Notes |
|----------|--------|-------|
| Stars/Forks | Excellent | 47K stars, 5.6K forks - very popular |
| Commit Activity | Excellent | 1,260 commits in 6 months |
| Contributor Diversity | Good | Mix of internal + external |
| Issue Response | Good | 30 open, needs-triage labels |
| Documentation | Excellent | Multi-language, comprehensive |
| CI/CD | Moderate | Basic coverage only |
| Community Engagement | Good | No discussions, but active issues/PRs |

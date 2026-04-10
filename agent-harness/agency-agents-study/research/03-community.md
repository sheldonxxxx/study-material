# Community Health Report: agency-agents

**Repository:** https://github.com/msitarzewski/agency-agents
**Report Date:** 2026-03-27

## Overview

| Metric | Value |
|--------|-------|
| Stars | 63,112 |
| Forks | 9,486 |
| Open Issues | 114 |
| Open Pull Requests | Not publicly listed |
| Main Language | Markdown |
| License | MIT |

## Repository Activity

### Recent Commit Activity

The repository shows very active recent development with commits happening daily:

| Date | Commits |
|------|---------|
| 2026-03-10 | 45 |
| 2026-03-11 | 27 |
| 2026-03-12 | 23 |
| 2026-03-09 | 18 |
| 2026-03-15 | 16 |
| 2026-03-03 | 14 |

**Recent commits include:**
- Add AI Citation Strategist agent (AEO/GEO optimization) - 2026-03-15
- Add Academic Division with 5 storytelling-focused agents - 2026-03-15
- Add Korean Business Navigator, French Consulting Market Navigator, Salesforce Architect - 2026-03-15
- Parallelize scripts default mode PR merge - 2026-03-15

### Merged Pull Requests (Recent)

Recent merged PRs show active community contribution:
- #219: Academic agents (toniDefez)
- #209: Korean Business Navigator (sebastientang)
- #208: French Consulting Market (sebastientang)
- #207: Salesforce Architect (sebastientang)
- #188: Parallelize scripts (CagesThrottleUs)
- #203: Workflow Architect (tsanford01)
- #180: Agent Product Manager (Subhodip-Chatterjee)

## Contributors

### Top Contributors (by GitHub contributions API)

| Rank | Contributor | Contributions |
|------|-------------|---------------|
| 1 | msitarzewski | 101 |
| 2 | jnMetaCode | 21 |
| 3 | 4shil | 9 |
| 4 | CagesThrottleUs | 6 |
| 5 | aryanvr961 | 4 |

### Top Contributors (by commit log)

| Contributor | Commits (recent 30) |
|-------------|---------------------|
| Michael (msitarzewski) | 18 |
| CagesThrottleUs | 6 |
| Sebastien | 3 |
| Travis | 1 |
| Toni | 1 |
| Subhodip | 1 |

**Observation:** The repository has a clear primary maintainer (msitarzewski) with significant community contributions. The top 5 contributors account for ~150 contributions, indicating a healthy contributor ecosystem.

## Versioning and Releases

**GitHub Releases:** No formal releases found via API (no tagged releases published as GitHub Releases)

**Git Tags:** No tags found locally

**Versioning Approach:** The project does not appear to use formal releases or semantic versioning. It operates on a rolling release model where agents are added/updated directly on the main branch.

## Community Infrastructure

### Contributing Guidelines

**File:** CONTRIBUTING.md (428 lines)

The contributing guide is comprehensive and well-structured:

1. **Code of Conduct** - Explicitly defined expectations for respectful, inclusive, collaborative, professional behavior

2. **How to Contribute:**
   - Create a new agent (fork, choose category, create file, test, submit PR)
   - Improve existing agents
   - Share success stories
   - Report issues

3. **Agent Design Guidelines:**
   - Detailed markdown template with frontmatter (name, description, color, emoji, vibe)
   - Persona section (identity, memory, communication, rules)
   - Operations section (mission, deliverables, workflow, metrics)
   - Design principles emphasizing personality, concrete deliverables, measurable metrics

4. **Pull Request Process:**
   - Clear distinction between PR-ready items (single agent files) vs. discussion-first items (tooling, architecture)
   - Pre-submission checklist
   - PR template provided inline

5. **Style Guide:**
   - Specific, concrete, memorable writing style
   - Markdown formatting conventions
   - Code block requirements

### Issue Templates

**Bug Report Template (.github/ISSUE_TEMPLATE/bug-report.yml):**
- Fields: Agent file, description, suggested fix
- Labels: ["bug"]

**New Agent Request Template (.github/ISSUE_TEMPLATE/new-agent-request.yml):**
- Fields: Agent name, category (dropdown with 11 options), description, use cases
- Labels: ["enhancement", "new-agent"]

### Pull Request Template

**File:** .github/PULL_REQUEST_TEMPLATE.md

Contains:
- Brief description field
- Agent Information section (name, category, specialty)
- Checklist: follows template, YAML frontmatter, includes examples, tested, proofread

### CI/CD

**Lint Workflow (.github/workflows/lint-agents.yml):**
- Triggers on PR changes to agent directories
- Validates agent frontmatter and structure
- Uses custom `scripts/lint-agents.sh`
- Checks diff against base branch

### Funding

**File:** .github/FUNDING.yml

- GitHub Sponsor: msitarzewski

### No CODEOWNERS or MAINTAINERS file

The repository does not have:
- CODEOWNERS file (no automated review assignment)
- MAINTAINERS file (no explicit governance document)

## Community Health Assessment

### Strengths

1. **Active Development:** Daily commits, consistent activity over the past 2 weeks
2. **Clear Contribution Path:** Well-documented process for adding agents
3. **Structured Templates:** Agent template, PR template, issue templates all present
4. **Active Community PRs:** Multiple community members contributing regularly (sebastientang, CagesThrottleUs, tsanford01, Subhodip-Chatterjee)
5. **CI Pipeline:** Automated linting for agent files
6. **Responsive Maintainer:** msitarzewski merges PRs regularly

### Areas for Improvement

1. **No Formal Releases:** No tagged releases or CHANGELOG - users must track main branch
2. **No CODEOWNERS:** No automated assignment of reviewers
3. **No Governance Document:** No MAINTAINERS file defining decision-making process
4. **Monolithic Maintainership:** Primary maintainer accounts for ~67% of recent commits
5. **No Roadmap:** No public roadmap for planned agent categories or features
6. **Sparse GitHub Discussions:** Limited visible community discussion

### Overall Rating: **Healthy**

The repository shows strong community health with active development, clear contribution processes, and regular community involvement. The primary concern is maintainer concentration and lack of formal release/versioning, but the active PR review process and welcoming CONTRIBUTING guide compensate for these gaps.

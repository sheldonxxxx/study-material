# pi-mono Community Health Report

## Repository Overview

| Metric | Value |
|--------|-------|
| **Stars** | 28,137 |
| **Forks** | 2,981 |
| **Open Issues** | 7 |
| **Open PRs** | ~1 (pr-1724 branch) |
| **Default Branch** | main |
| **Last Push** | 2026-03-26 |

**Description:** AI agent toolkit: coding agent CLI, unified LLM API, TUI & web UI libraries, Slack bot, vLLM pods

## Activity Metrics

### Commit Velocity
- **Last 30 days:** 710 commits (~23/day average)
- **Last 7 days:** Highly active, multiple commits daily
- **Pattern:** Continuous integration-style commits (fix, chore, feat, test prefixed)

### Release Cadence
- **Recent releases:** v0.62.0 (2026-03-23), v0.61.1, v0.61.0, v0.60.0, v0.59.0
- **Frequency:** Every 2-4 weeks
- **Versioning:** Semantic versioning (major.minor.patch)

### Branch Strategy
- Single primary branch (`main`)
- One PR branch visible (`remotes/origin/pr-1724`)
- Feature branches merged via PRs

## Community Infrastructure

### Issue Templates (.github/ISSUE_TEMPLATE/)
- **bug.yml:** Bug reports with fields for description, reproduction steps, expected behavior, version
- **contribution.yml:** Contribution proposals (required for new contributors before PR)
- **config.yml:** Contains triage config

### Contributor Management
- **APPROVED_CONTRIBUTORS:** 137 GitHub handles approved to submit PRs
- **APPROVED_CONTRIBUTORS.vacation:** Vacation/absence tracking for maintainers
- **Approval gate:** New contributors must open an issue and receive `lgtm` before PR access

### CONTRIBUTING.md Guidelines
- Core philosophy: "You must understand your code"
- AI-generated code acceptable if contributor understands it
- Required checks: `npm run check` and `./test.sh` must pass
- New providers to `packages/ai` require tests per AGENTS.md
- Changelog entries added by maintainers only

### Workflows (.github/workflows/)
- 8 workflow files present
- Includes "openclaw labeler" (added recently)

## Community Health Indicators

### Positive Signals
1. **High velocity:** 710 commits/month indicates active development
2. **Large user base:** 28k stars suggests widely used toolkit
3. **Low issue count:** Only 7 open issues (well-maintained or new project)
4. **Structured onboarding:** Approval process filters quality
5. **Clear contribution path:** Issue-first approach before PRs
6. **Regular releases:** Consistent versioning cadence

### Observations
1. **Maintainer-heavy workflow:** Single maintainer (@badlogic) with curated contributor list
2. **Quality gate:** APPROVED_CONTRIBUTORS list suggests careful PR management
3. **AI-aware policies:** CONTRIBUTING explicitly addresses AI-generated code expectations
4. **Documentation focus:** AGENTS.md for provider tests, clear CONTRIBUTING.md

## Risks / Considerations

1. **Single-person oversight:** APPROVED_CONTRIBUTORS suggests limited maintainer capacity
2. **High volume of small commits:** May indicate rapid iteration but could affect review bandwidth
3. **Vacation tracking file:** Suggests small team managing large project

## Summary

pi-mono is a highly active, well-structured monorepo with strong community infrastructure despite being maintained by a small team. The approval-gate contributor model is notable for managing AI-generated contribution quality. With 28k stars and active development (23+ commits/day), the project demonstrates healthy community engagement and professional maintenance practices.

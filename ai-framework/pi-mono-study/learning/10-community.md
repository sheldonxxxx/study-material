# Community

## Repository Overview

| Metric | Value |
|--------|-------|
| **Stars** | 28,137 |
| **Forks** | 2,981 |
| **Open Issues** | 7 |
| **Open PRs** | ~1 |
| **Default Branch** | main |
| **License** | MIT |

## Activity Metrics

### Commit Velocity
- **Last 30 days:** 710 commits (~23/day average)
- **Last 7 days:** Highly active, multiple commits daily
- **Pattern:** Continuous integration-style commits (fix, chore, feat, test prefixed)

### Release Cadence
- **Recent releases:** v0.62.0 (2026-03-23), v0.61.1, v0.61.0, v0.60.0, v0.59.0
- **Frequency:** Every 2-4 weeks
- **Versioning:** Semantic versioning (major.minor.patch)

## Community Infrastructure

### Issue Templates (.github/ISSUE_TEMPLATE/)

- **bug.yml** -- Bug reports with description, reproduction steps, expected behavior, version
- **contribution.yml** -- Required for new contributors before PR (what, why, how)
- **config.yml** -- Contains triage config

### Contributor Management

**Approval Gate:**
- New contributors must open an issue describing the desired change
- Maintainer comments `lgtm` to approve
- Once approved, contributor can submit PRs
- 137 GitHub handles in `APPROVED_CONTRIBUTORS`

**Rationale:** Filters AI-generated low-quality contributions early.

**Files:**
- `.github/APPROVED_CONTRIBUTORS` -- List of approved GitHub handles
- `.github/APPROVED_CONTRIBUTORS.vacation` -- Vacation/absence tracking

### PR Gate Workflow (pr-gate.yml)

1. PR opened triggers workflow
2. Skip bots (dependabot, etc.)
3. Check if author has admin/maintain/write permission -- allow if yes
4. Check if author is in APPROVED_CONTRIBUTORS list
5. If OSS weekend active + approved contributor -- close with pause message
6. If not approved -- auto-close with instructions

### OSS Weekend Mode

**Purpose:** Periodic pause of external contributions.

**Files:**
- `.github/oss-weekend.json` -- Weekend state configuration
- `.github/workflows/oss-weekend-issues.yml` -- Auto-closes issues during weekend
- `.github/workflows/pr-gate.yml` -- Pauses PRs from approved non-maintainers

**Script:** `node scripts/oss-weekend.mjs` with `--mode=close/open` and `--end-date`

**Currently:** Weekend runs through March 30, 2026 (issue tracker reopens then).

## Contributing Guidelines

**The One Rule:** "You must understand your code."

**AI Policy:**
- AI-generated code acceptable if contributor understands it
- Must be able to explain what changes do and how they interact
- AI slop without understanding will be closed

**Before Submitting PR:**
```bash
npm run check  # must pass with no errors
./test.sh     # must pass
```

**Changelog:** Do not edit. Maintainers add entries.

**Questions:** Open issue or ask on Discord.

## Communication

- **Discord:** https://discord.com/invite/3cU7Bz4UPx
- **GitHub Issues:** For bugs and feature requests
- **GitHub PRs:** Only from approved contributors

## Health Indicators

### Positive Signals
1. High velocity (710 commits/month, 23+/day)
2. Large user base (28k stars)
3. Low issue count (7 open -- well maintained)
4. Structured onboarding with approval process
5. Clear contribution path (issue first, then PR)
6. Regular releases (2-4 week cadence)

### Observations
1. Single-person oversight with curated contributor list
2. Quality gate via APPROVED_CONTRIBUTORS
3. AI-aware policies explicitly addressing generated code
4. Professional maintenance practices despite small team

## Related Documentation

- CONTRIBUTING.md -- Contribution guidelines
- AGENTS.md -- Rules for AI agents working in the codebase

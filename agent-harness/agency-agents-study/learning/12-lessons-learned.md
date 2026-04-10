# Lessons Learned: agency-agents Repository Analysis

## Overview

This document synthesizes insights from analyzing the agency-agents repository (63K stars, 144+ AI agents) as a reference for building a similar project. The analysis covered architecture, tech stack, community health, code quality, security, and feature depth across 12 research documents.

---

## What to Emulate

### 1. Canonical Single-Source, Multi-Format Publishing

**Pattern:** One agent definition in markdown, converted to 10+ tool-specific formats via `convert.sh`.

**Why it works:** Separation of concerns -- content creators write once, tooling adapts to each AI tool's format requirements.

**Implementation:**
- YAML frontmatter for machine-readable metadata
- Markdown body for human-readable content
- Tool-specific converter functions in `convert.sh`
- OpenClaw splits by persona/operations semantics

**Verification:** 10 integration directories exist, each with tool-appropriate format.

---

### 2. Domain-Based Organization Over Technical Classification

**Pattern:** Agents grouped by business domain (engineering, marketing, design) rather than technical function.

**Why it works:** Mirrors how agencies staff projects. Non-technical stakeholders can discover relevant agents without understanding technical distinctions.

**Evidence:** Marketing division (29 agents) and Engineering (24 agents) are the largest -- reflecting real market demand.

**Surprise:** China market agents are clustered (10+ in marketing alone), enabling "China market team" deployments.

---

### 3. Comprehensive CONTRIBUTING.md as Foundation

**Size:** 428 lines of detailed guidance.

**Contents:**
- Agent design philosophy with 5 principles (Strong Personality, Clear Deliverables, Success Metrics, Proven Workflows, Learning Memory)
- Complete markdown template with all sections documented
- External services policy
- Style guide with concrete examples
- Recognition program for contributors

**Quality signal:** A well-crafted CONTRIBUTING.md signals project maturity and reduces low-quality contributions.

---

### 4. Shell Scripts with Modern Bash Practices

**Evidence from `convert.sh`:**
```bash
set -euo pipefail   # Strict error handling
trap 'rm -f "$AIDER_TMP"' EXIT   # Cleanup on crash
NO_COLOR and TERM=dumb checks for color output
```

**Parallel execution:**
```bash
parallel_tools=(antigravity gemini-cli opencode cursor openclaw qwen)
printf '%s\n' "${parallel_tools[@]}" | xargs -P "$parallel_jobs" -I {}
```

**Job count auto-detection:**
```bash
n=$(nproc 2>/dev/null) && echo "$n" && return
n=$(sysctl -n hw.ncpu 2>/dev/null) && echo "$n" && return
echo 4  # fallback
```

**Result:** Scripts are deploy-ready despite being bash-only.

---

### 5. Quality Gates in Multi-Agent Workflows

**Pattern from `examples/workflow-startup-mvp.md`:**

```
Week 2: Reality Check at midpoint (gating quality gate)
Week 4: Reality Checker final GO/NO-GO decision
```

**Evidence Collector philosophy:**
- "Screenshots don't lie" -- visual evidence is ground truth
- Mandatory process: generate evidence FIRST, then assess
- Explicit anti-pattern: "zero issues found" claims are automatic failures

**Honest rating scale:** C+/B-/B/B+ (NO A+ fantasies), default to FAILED for production readiness.

**This is rare:** Most prompt libraries assume success. These agents build in skepticism.

---

### 6. Measurable Success Metrics Over Aspirational Goals

**Example from Frontend Developer:**
- "Page load times under 3 seconds on 3G networks"
- "Lighthouse scores >90 for Performance and Accessibility"
- "Component reusability rate >80%"

**Example from Douyin Strategist:**
- "Average video completion rate > 35%"
- "DOU+ ROI > 1:3"
- "Monthly follower growth > 15%"

**Why it matters:** Agents can be evaluated objectively rather than by subjective impression.

---

### 7. Interactive Installer with Good UX

**From `install.sh`:**
- Auto-detects installed AI tools
- ASCII box-drawing UI with proper width calculation
- ANSI escape sequence handling (visible length vs raw length)
- Non-interactive fallback when not a TTY
- Worker process pattern to suppress duplicate headers in parallel mode

**Key insight:** Even infrastructure scripts benefit from thoughtful UX.

---

## What to Avoid

### 1. No Schema Validation for Content Structure

**Problem:** Agent files have YAML frontmatter but no JSON Schema validation.

**Evidence:** `lint-agents.sh` only checks for field presence, not content validity:
```bash
# Only checks if field exists, not if value is valid
if ! echo "$frontmatter" | grep -qE "^${field}:"; then
```

**Consequence:** Malformed YAML silently produces empty fields.

**Fix for similar project:** Add JSON Schema validation to CI pipeline.

---

### 2. Referenced Files That Don't Exist

**Found in research:**
- `qa-playwright-capture.sh` -- referenced by Evidence Collector and Reality Checker, not in repo
- `ai/agents/qa.md` -- referenced but doesn't exist
- `integrations/mcp-memory/README.md` -- referenced by multiple agents

**Impact:** Agents provide frameworks but the supporting scripts/templates are missing.

**Lesson:** If agents reference supporting files, those files must exist.

---

### 3. Inconsistent Agent Depth

**Variation observed:**
- `macos-spatial-metal-engineer.md`: ~330 lines with full Metal/Compositor code
- `xr-immersive-developer.md`: ~33 lines, no code examples
- `healthcare-marketing-compliance.md`: ~396 lines
- `specialized-mcp-builder.md`: ~64 lines

**Root cause:** Organic growth without contribution standards enforcement.

**Impact:** Users can't predict agent quality from directory position.

---

### 4. Silent Failure in Parallel Execution

**From `convert.sh` analysis:**
```bash
printf '%s\n' "${parallel_tools[@]}" | xargs -P "$parallel_jobs" -I {} \
  sh -c '"$AGENCY_CONVERT_SCRIPT" --tool "{}" ...'
```

**Problem:** `xargs` returns 0 even if child processes fail.

**Result:** Tool conversion failures are silently ignored.

---

### 5. No Formal Versioning or Releases

**Evidence:** No GitHub Releases, no CHANGELOG, rolling release model.

**Impact:** Users must track main branch for updates. No stable version pinning.

**Consequence for similar project:** No way for users to lock to a known-good state.

---

### 6. Monolithic Maintainership

**Evidence:**
- `msitarzewski`: 101 contributions (67% of recent commits)
- Next contributor: 21 contributions

**Risk:** Single point of failure for review/merge.

**Absence:** No CODEOWNERS file, no MAINTAINERS document.

---

### 7. No Uninstall Mechanism

**From `install.sh` analysis:**
- Installs agents to tool config directories
- No removal script exists
- Users must manually delete installed files

**Lesson:** Build removal/cleanup alongside installation.

---

## Surprises

### 1. Testing Agents Have the Most Substantial Code

**Expectation:** Agent definitions are personality prompts, not code.

**Reality:** Performance Benchmarker, API Tester, Tool Evaluator, and Workflow Optimizer all include substantial Python/JavaScript code blocks.

**Example from k6 load test:**
```javascript
export const options = {
  stages: [
    { duration: '2m', target: 10 },
    { duration: '5m', target: 50 },
    { duration: '2m', target: 100 },
  ],
};
```

**Implication:** The "agent" concept can include executable code, not just prompts.

---

### 2. Quality Rating Honesty

**Evidence from Reality Checker:**
- Quality ratings: C+/B-/B/B+ (NO A+)
- Default production readiness: FAILED
- Explicit anti-pattern: "zero issues found" claims

**This is distinctive:** Most AI tools overstate quality. These agents build in skepticism.

---

### 3. China Market Coverage is Deep and Specific

**Examples:**
- Baidu SEO Specialist: ICP备案 as "step zero", specific algorithm names (飓风, 细雨, 惊雷)
- China E-Commerce: T-60/T-30/T-7/T-day/T+7 campaign battle plan structure
- Douyin: Livestream product type breakdown (traffic driver 20%, profit item 50%)

**Not generic:** The agents demonstrate genuine understanding of Chinese platform mechanics.

---

### 4. nexus-spatial-discovery.md is a Real Document

**Evidence:** Dated March 5, 2026, references specific market data (Vision Pro ~1M units, 95% sales decline).

**Structure:** 8-agent parallel activation, 10 sections of synthesis, real budget figures ($121.5K-$155.5K).

**Interpretation:** This reads like an actual agency deliverable, not a template.

---

### 5. Minimal Attack Surface

**Because:** Documentation-only repository with no:
- Backend services or APIs
- Databases
- Authentication/authorization systems
- Secrets or credentials
- User input leading to injection

**Result:** Security assessment is straightforward -- the risk is content quality, not code exploitation.

---

## Key Metrics Summary

| Aspect | Finding |
|--------|---------|
| Total agents | 144+ across 12 divisions |
| Supported tools | 10 integration formats |
| Repository size | < 10 MB |
| CI pipeline | GitHub Actions (markdown linting) |
| Community health | Healthy (63K stars, active PRs) |
| Code quality | Good shell scripts, no content validation |
| Security risk | Minimal |
| Maintained by | 1 primary maintainer (67% commits) |

---

## Transferability to Similar Projects

**High transferability:**
- Canonical source + multi-format output architecture
- Domain-based organization
- CONTRIBUTING.md as quality foundation
- Shell script practices (set -euo pipefail, trap cleanup, parallel execution)

**Medium transferability:**
- Agent personality template (requires adaptation)
- Workflow examples (process patterns reusable)

**Low transferability:**
- China market specifics (domain knowledge)
- Tool integration specifics (convert.sh targets specific tools)

---

## Conclusion

The agency-agents repository succeeds as a curated knowledge base because it:

1. Solves a real problem (multi-tool agent deployment)
2. Maintains consistent structure (CONTRIBUTING.md enforcement)
3. Provides measurable quality (success metrics)
4. Builds in skepticism (Reality Checker, evidence-based verification)

The main risks for similar projects are:

1. Schema validation gaps (add JSON Schema)
2. Content depth inconsistency (enforce minimum standards)
3. Maintenance concentration (diversify contributors)
4. Supporting file drift (keep referenced files in sync)

The project demonstrates that a documentation-only repository can achieve significant utility through thoughtful structure, quality standards, and multi-format publishing -- without requiring traditional software engineering infrastructure.

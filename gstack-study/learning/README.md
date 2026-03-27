# gstack — Learning Reference

## What This Is
Analysis of the [gstack](https://github.com/garrytan/gstack) codebase (49,272 stars) — an AI-augmented development workflow system that transforms Claude Code into a virtual engineering team via 28 specialized skills.

## Key Takeaways
- **Daemon pattern wins for latency-critical tools** — persistent Chromium daemon with HTTP CLI achieves ~100ms/command vs 3-5s cold start
- **Skills are vertical slices, not horizontal layers** — each skill is a self-contained `SKILL.md` with embedded prompts, no shared runtime
- **Self-regulation is architectural** — explicit WTF-caps, 3-strike rules, hard-stops on completeness gaps prevent runaway AI behavior
- **Template generation with committed output** — `gen-skill-docs.ts` generates SKILL.md files at build time, committed to git to avoid runtime surprises
- **Diff-aware mode reduces friction to zero** — skills auto-detect affected scope from git diff, skipping irrelevant work

## Should Trust / Should Not Trust
- **Trust**: Architecture decisions (daemon pattern, circular buffers), test coverage claims, security audit methodology (CSO)
- **Do Not Trust**: README simplicity claims (verify with code — SKILL.md files are 500-1800 lines each)

## At a Glance
- **Language:** TypeScript (ES modules)
- **Architecture:** Daemon + Client-Server (browse daemon) + Skill Dispatch (28 command-based skills)
- **Key Libraries:** Playwright (browser), Bun (runtime/build), diff (text comparison)
- **Notable Patterns:** CircularBuffer (O(1) bounded logs), Ref System (ariaSnapshot locators), Template Generation (SKILL.md from .tmpl)
- **Stars / Activity:** 49,272 stars, daily commits, ~2 releases/week, 12 parallel E2E runners on PRs
- **License:** MIT
- **Maintainer:** Garry Tan (YC President/CEO) — single-maintainer, no formal governance

## Study Structure
```
gstack-study/
├── learning/          # Synthesized learning documents
│   ├── 01-project-overview.md
│   ├── 02-architecture.md
│   ├── 03-tech-stack.md
│   ├── 04-features-deep-dive.md
│   ├── 05-code-quality.md
│   ├── 06-ci-cd.md
│   ├── 07-documentation.md
│   ├── 08-security.md
│   ├── 09-dependencies.md
│   ├── 10-community.md
│   ├── 11-patterns.md
│   ├── 12-lessons-learned.md
│   └── 13-my-action-items.md
└── research/         # Preserved evidence from parallel subagent work
    ├── 01-topology.md
    ├── 02-tech-stack.md
    ├── 03-community.md
    ├── 04-features-index.md
    ├── 05a-features-batch-1.md  (through 05i — 9 batch files)
    ├── 06-architecture.md
    ├── 07-code-quality.md
    └── 08-security-perf.md
```

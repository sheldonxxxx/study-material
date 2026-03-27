# My Action Items: Building a Superpowers-Inspired System

## Context

Based on lessons learned from studying the Superpowers project (v5.0.6), these are prioritized action items for building a similar skill-based workflow system.

---

## Phase 1: Foundation (Week 1)

### 1.1 Choose Core Skill: Verification Before Completion
**Priority: P0 - Critical**

Verification Before Completion is the most universal discipline. Without it, nothing else matters.

**Actions:**
- [ ] Read `skills/verification-before-completion/SKILL.md` fully
- [ ] Write my own version: the Iron Law, the Gate Function, claim verification table
- [ ] Create a rationalization table from my own common excuses
- [ ] Test against my past failures to verify the excuses are accurate

**Deliverable:** My own `verification-before-completion.md` skill document

---

### 1.2 Choose Core Skill: Systematic Debugging
**Priority: P0 - Critical**

Debugging without process leads to random guessing and missed root causes.

**Actions:**
- [ ] Read `skills/systematic-debugging/SKILL.md` fully
- [ ] Extract the Four Phases: Root Cause Investigation, Pattern Analysis, Hypothesis and Testing, Implementation
- [ ] Adapt root-cause-tracing technique (backward stack trace analysis)
- [ ] Adapt defense-in-depth validation pattern
- [ ] Create my own find-polluter script for test pollution

**Deliverable:** My own `systematic-debugging.md` skill document

---

### 1.3 Establish TDD Practice
**Priority: P0 - Critical**

TDD is the foundation of quality. Without it, verification is补救 rather than preventive.

**Actions:**
- [ ] Read `skills/test-driven-development/SKILL.md` fully
- [ ] Read `skills/test-driven-development/testing-anti-patterns.md`
- [ ] Create my own TDD cheat sheet with Iron Law language
- [ ] Build a habit: before writing any production code, write a failing test

**Deliverable:** My TDD reference card with Iron Law and anti-rationalization arguments

---

## Phase 2: Planning & Execution (Week 2)

### 2.1 Create Writing Plans Skill
**Priority: P1 - High**

Plans without placeholders force complete thinking.

**Actions:**
- [ ] Read `skills/writing-plans/SKILL.md` fully
- [ ] Read `skills/writing-plans/plan-document-reviewer-prompt.md`
- [ ] Define my plan template with header: Goal, Architecture, Tech Stack, File Structure
- [ ] Implement no-placeholder policy: no "TBD", no "similar to", no undefined references
- [ ] Create self-review checklist
- [ ] Define task granularity: 2-5 minute execution steps

**Deliverable:** My own `writing-plans.md` skill document with template

---

### 2.2 Create Subagent-Driven Development Workflow
**Priority: P1 - High**

Parallel execution with fresh context is key to throughput.

**Actions:**
- [ ] Read `skills/subagent-driven-development/SKILL.md` fully
- [ ] Read the three prompt templates: implementer, spec-reviewer, code-quality-reviewer
- [ ] Design my version: fresh context construction, two-stage review order (spec BEFORE quality)
- [ ] Define status codes: DONE, DONE_WITH_CONCERNS, NEEDS_CONTEXT, BLOCKED
- [ ] Create escalation protocols

**Deliverable:** My own `subagent-driven-development.md` skill document with templates

---

### 2.3 Create Git Worktree Workflow
**Priority: P1 - High**

Isolated workspaces prevent main branch pollution.

**Actions:**
- [ ] Read `skills/using-git-worktrees/SKILL.md` fully
- [ ] Define directory selection priority: .worktrees/ (hidden) > worktrees/ > ask
- [ ] Implement .gitignore verification before worktree creation
- [ ] Define baseline test verification step
- [ ] Create auto-detect for project type (Node.js, Rust, Python, Go)

**Deliverable:** My own `using-git-worktrees.md` skill document

---

### 2.4 Create Finishing Development Branch Skill
**Priority: P1 - High**

Clean completion prevents forgotten branches and merge conflicts.

**Actions:**
- [ ] Read `skills/finishing-a-development-branch/SKILL.md` fully
- [ ] Define blocking test verification step
- [ ] Design exactly 4 options: merge locally, create PR, keep branch, discard
- [ ] Implement discard confirmation (typed "discard" required)
- [ ] Define worktree cleanup logic per option

**Deliverable:** My own `finishing-a-development-branch.md` skill document

---

## Phase 3: Review & Collaboration (Week 3)

### 3.1 Create Requesting Code Review Skill
**Priority: P2 - Medium**

Pre-review checklist ensures complete code goes to review.

**Actions:**
- [ ] Read `skills/requesting-code-review/SKILL.md` fully
- [ ] Read `skills/requesting-code-review/code-reviewer.md`
- [ ] Define issue severity tiers: Critical, Important, Minor
- [ ] Create subagent prompt template with git range context
- [ ] Document push-back guidance

**Deliverable:** My own `requesting-code-review.md` skill with reviewer template

---

### 3.2 Create Receiving Code Review Skill
**Priority: P2 - Medium**

Responding to feedback constructively prevents defensiveness.

**Actions:**
- [ ] Read `skills/receiving-code-review/SKILL.md` fully
- [ ] Define forbidden responses: no "You're absolutely right!", no "Great point!"
- [ ] Create verification-before-implementation rule
- [ ] Define YAGNI check process
- [ ] Document when and how to push back

**Deliverable:** My own `receiving-code-review.md` skill document

---

### 3.3 Create Brainstorming Skill
**Priority: P2 - Medium**

Design before implementation prevents wasted effort.

**Actions:**
- [ ] Read `skills/brainstorming/SKILL.md` fully
- [ ] Read `skills/brainstorming/spec-document-reviewer-prompt.md`
- [ ] Define HARD-GATE: no implementation before design approval
- [ ] Implement one-question-at-a-time protocol
- [ ] Define 9-step checklist (or adapt for my needs)
- [ ] Consider visual companion (WebSocket-based) - optional

**Deliverable:** My own `brainstorming.md` skill document with HARD-GATE

---

## Phase 4: Meta-Skills (Week 4)

### 4.1 Create Writing Skills Skill (Meta)
**Priority: P3 - Lower**

TDD for documentation ensures skills survive rationalization.

**Actions:**
- [ ] Read `skills/writing-skills/SKILL.md` fully
- [ ] Read `skills/writing-skills/testing-skills-with-subagents.md`
- [ ] Apply TDD to skill creation: RED (baseline without skill), GREEN (skill addresses failures), REFACTOR (close loopholes)
- [ ] Create pressure scenarios for discipline skills
- [ ] Define frontmatter schema: name, description (triggering conditions only, not workflow summary)

**Deliverable:** My own `writing-skills.md` meta-skill document

---

### 4.2 Create Using Superpowers Skill (Entry Point)
**Priority: P3 - Lower**

Entry point that explains how to use the skill system.

**Actions:**
- [ ] Read `skills/using-superpowers/SKILL.md` fully
- [ ] Define instruction priority: User > Skills > Default
- [ ] Implement 1% rule: if there's any chance a skill applies, invoke it
- [ ] Create rationalization red flags table
- [ ] Document skill types: Rigid (follow exactly) vs Flexible (adapt)

**Deliverable:** My own `using-superpowers.md` entry point skill

---

## Phase 5: Infrastructure (Ongoing)

### 5.1 Implement Zero-Dependency Where Possible
**Priority: P2 - Medium**

Before adding any dependency, check if built-ins suffice.

**Actions:**
- [ ] For any new server/daemon, use only Node.js built-ins
- [ ] For WebSocket needs, implement RFC 6455 from scratch (study `server.cjs`)
- [ ] For file watching, use native fs.watch with debouncing
- [ ] Document: zero-dependency philosophy in my project README

**Deliverable:** Built-in-only server patterns documented

---

### 5.2 Add Tests for Critical Code
**Priority: P2 - Medium**

Address the gap: plugin code has no test coverage.

**Actions:**
- [ ] Identify my critical path code (not just "main" code)
- [ ] Add tests for: config loading, skill discovery, hook injection
- [ ] Use Node.js native assert module (no testing framework needed initially)
- [ ] Cover error paths: malformed config, missing files, parsing errors

**Deliverable:** Test suite for critical path code

---

### 5.3 Consider TypeScript for Server Code
**Priority: P3 - Lower**

Return on investment: typed inputs/outputs with minimal new code.

**Actions:**
- [ ] Evaluate: does the server code (server.cjs equivalent) need types?
- [ ] If yes, add TypeScript with JSDoc annotations as stepping stone
- [ ] Document decision rationale in code

**Deliverable:** TypeScript types added to server code (if applicable)

---

## Quick Reference: Iron Laws to Internalize

Print or bookmark these:

```
VERIFICATION: NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE

TDD: NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST

DEBUGGING: NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST

SKILLS: NO SKILL WITHOUT A FAILING TEST FIRST

BRAINSTORMING: DO NOT INVOKE ANY IMPLEMENTATION SKILL UNTIL DESIGN IS APPROVED
```

---

## Quick Reference: Rationalization Tables (Adapt for Your Context)

**On Verification:**
| Excuse | Reality |
|--------|---------|
| "Should work now" | RUN the verification |
| "I'm confident" | Confidence != evidence |
| "Just this once" | No exceptions |

**On TDD:**
| Excuse | Reality |
|--------|---------|
| "Tests after achieve same goals" | Tests-after = "what does this do?" Tests-first = "what should this do?" |
| "Already manually tested" | Ad-hoc, no record, can't re-run |
| "TDD will slow me down" | TDD faster than debugging |

---

## Priority Summary

| Phase | Skill | Priority | Week |
|-------|-------|----------|------|
| 1 | Verification Before Completion | P0 | 1 |
| 1 | Systematic Debugging | P0 | 1 |
| 1 | TDD | P0 | 1 |
| 2 | Writing Plans | P1 | 2 |
| 2 | Subagent-Driven Development | P1 | 2 |
| 2 | Git Worktrees | P1 | 2 |
| 2 | Finishing Development Branch | P1 | 2 |
| 3 | Requesting Code Review | P2 | 3 |
| 3 | Receiving Code Review | P2 | 3 |
| 3 | Brainstorming | P2 | 3 |
| 4 | Writing Skills (meta) | P3 | 4 |
| 4 | Using Superpowers (entry) | P3 | 4 |
| 5 | Zero-Dependency Infrastructure | P2 | Ongoing |
| 5 | Critical Path Tests | P2 | Ongoing |
| 5 | TypeScript for Server | P3 | If applicable |

---

## Notes

- **Start with P0 skills first.** Everything else is compromised without verification and debugging discipline.
- **Build skills iteratively.** Create v1, use it, identify gaps, improve.
- **Test with pressure scenarios.** Skills survive contact with reality only if tested under pressure.
- **Match model to task.** Not every task needs the most capable model.
- **Fresh context beats inheritance.** When dispatching work, construct context fresh.

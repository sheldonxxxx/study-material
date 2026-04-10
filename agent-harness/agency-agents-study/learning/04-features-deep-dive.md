# Agency-Agents Repository: Features Deep Dive

**Repository:** `/Users/sheldon/Documents/claw/reference/agency-agents`
**Synthesized:** 2026-03-27
**Source:** Feature index + 4 batch research documents

---

## Executive Summary

The agency-agents repository contains 144+ AI agents organized into 12 functional divisions. The system provides deep domain expertise through personality-driven agents, multi-tool conversion infrastructure, and production-ready workflow templates. The architecture is markdown-based with YAML frontmatter, enabling transformation to 10+ AI coding tools via shell scripts.

**Key strengths:** Comprehensive domain coverage, concrete code deliverables, measurable success metrics, multi-tool portability.

**Key concerns:** Inconsistent agent depth, no schema validation, missing referenced files, manual handoff assumption.

---

## Priority 1: Core Features

### Feature 1: Multi-Agent Specialization System

**What it is:** 144+ AI agents with deep domain expertise, distinct personalities, and proven workflows, organized into 12 functional divisions.

**How it works:**

Each agent is a `.md` file with YAML frontmatter defining identity:

```yaml
---
name: Frontend Developer
description: Expert frontend developer specializing in modern web technologies...
color: cyan
emoji: 🖥️
vibe: Builds responsive, accessible web apps with pixel-perfect precision.
---
```

The agent body follows an 8-section structure:
1. Identity & Memory - Role, personality, background, experience
2. Core Mission - Primary responsibilities with deliverables
3. Critical Rules - Domain-specific constraints and approach
4. Technical Deliverables - Concrete code examples and templates
5. Workflow Process - Step-by-step methodology
6. Communication Style - Tone, voice, example phrases
7. Learning & Memory - Pattern recognition and improvement
8. Success Metrics - Measurable outcomes with benchmarks
9. Advanced Capabilities - Specialized techniques

**Key implementation details:**

- **Frontmatter extraction** (`convert.sh`): Uses awk state machine to parse YAML -- `fm==1` captures fields in first block only
- **Slugification**: Three sed passes handle non-alphanumeric to dash, collapse multiple dashes, trim leading/trailing
- **Accumulator pattern**: Aider/Windsurf accumulate into temp files, write once at end

**Clever solutions:**

1. **Persona/Operations Separation**: OpenClaw converter splits agent body into `SOUL.md` (persona), `AGENTS.md` (operations), `IDENTITY.md` (emoji + name + vibe)
2. **Color Normalization**: Maps 20+ named colors to #RRGGBB hex with validation and gray fallback
3. **Services Frontmatter**: Agents declare external dependencies without tight coupling:

```yaml
services:
  - name: Stripe
    url: https://stripe.com
    tier: paid
```

**Technical debt:**

- No JSON Schema validation for frontmatter -- malformed YAML silently produces empty fields
- `lint-agents.sh` only checks formatting, not semantic validity
- Parallel race condition: xargs returns 0 even if children fail
- Hardcoded `AGENT_DIRS` array must be manually updated

---

### Feature 2: Multi-Tool Integration Platform

**What it is:** Unified agent system working across 10+ AI coding tools via conversion and installation scripts.

**Architecture:**

```
Source: *.md agents (12 divisions)
           |
           v
    scripts/convert.sh
    (transforms to 9 formats)
           |
           v
    integrations/<tool>/
           |
           v
    scripts/install.sh
    (copies to tool config dirs)
```

**Tool conversion matrix:**

| Tool | Output Format | Output Location | Per-Agent? |
|------|-------------|----------------|------------|
| antigravity | SKILL.md | `~/.gemini/antigravity/skills/agency-<slug>/` | Yes |
| gemini-cli | SKILL.md + manifest | `~/.gemini/extensions/agency-agents/` | Yes |
| opencode | .md | `.opencode/agents/` | Yes |
| cursor | .mdc | `.cursor/rules/` | Yes |
| aider | CONVENTIONS.md | project root | No (accumulated) |
| windsurf | .windsurfrules | project root | No (accumulated) |
| openclaw | SOUL.md + AGENTS.md + IDENTITY.md | `~/.openclaw/agency-agents/<slug>/` | Yes |
| qwen | .md | `~/.qwen/agents/` | Yes |
| claude-code | .md (direct copy) | `~/.claude/agents/` | Yes |
| copilot | .md (direct copy) | `~/.github/agents/` + `~/.copilot/agents/` | Yes |

**Key implementation details:**

- **Parallel execution**: Aider/Windsurf excluded from parallel because they accumulate into temp files (requires sequential build)
- **Tool detection**: Dual detection -- tries command first (`command -v`), falls back to directory check
- **ANSI width handling**: `strip_ansi()` measures visible length for box-drawing, preventing terminal misalignment
- **Job count detection**: Linux (`nproc`) → macOS (`sysctl -n hw.ncpu`) → fallback chain

**Clever solutions:**

1. **Temp file cleanup with trap**: `trap 'rm -f "$AIDER_TMP" "$WINDSURF_TMP"' EXIT` ensures cleanup even on crash
2. **Parallel install workers**: Parent spawns workers with `AGENCY_INSTALL_WORKER=1` env var to suppress duplicate output
3. **Qwen minimal frontmatter**: Tool-specific stripping for platforms that can't handle full agent complexity

**Technical debt:**

- No uninstall mechanism -- must manually delete files
- OpenClaw registration uses `|| true` silently swallowing failures
- Project-scoped tools (Cursor, OpenCode, Aider, Windsurf, Qwen) install to `${PWD}` -- wrong directory = unexpected location
- Worker failures not detected by parent -- xargs returns 0

---

### Feature 3: Engineering Division

**What it is:** 24 engineering agents covering full-stack development, DevOps, security, data engineering, and specialized domains.

**Agent roster highlights:**

| Agent | Specialty |
|-------|-----------|
| Frontend Developer | React/Vue/Angular, Core Web Vitals, UI |
| Backend Architect | API design, database, scalability |
| DevOps Automator | CI/CD, Terraform, infrastructure |
| Security Engineer | Threat modeling, secure code review |
| AI Engineer | ML models, deployment, AI integration |
| Mobile App Builder | iOS/Android, React Native, Flutter |
| Solidity Smart Contract Engineer | EVM, gas optimization |
| Embedded Firmware Engineer | Bare-metal, RTOS, ESP32/STM32 |
| SRE | SLOs, error budgets, observability |
| Data Engineer | Data pipelines, lakehouse, ETL/ELT |
| WeChat Mini Program Developer | WeChat ecosystem, China market |
| Feishu Integration Developer | Feishu/Lark Open Platform |

**Architectural patterns:**

**Frontend Developer** provides concrete React patterns:
```tsx
import React, { memo, useCallback, useMemo } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface DataTableProps {
  data: Array<Record<string, any>>;
  columns: Column[];
  onRowClick?: (row: any) => void;
}

export const DataTable = memo<DataTableProps>(({ data, columns, onRowClick }) => {
  // Uses tanstack/react-virtual for virtualization
  // Memoized with React.memo for performance
  // Full accessibility with ARIA roles
});
```

**Success Metrics (Frontend Developer):**
- Page load times under 3 seconds on 3G networks
- Lighthouse scores >90 for Performance and Accessibility
- Component reusability rate >80%

**DevOps Automator** delivers GitHub Actions CI/CD with security scanning, Terraform templates, Prometheus/Grafana configs.

**Quality observations:**

- All agents follow 8-section structure consistently
- Real code examples (not pseudo-code), 100-300 lines per agent
- Measurable success metrics with specific benchmarks
- Some success metrics unrealistic: "Zero console errors in production"
- Duplicate Data Engineer appears in roster but file count suggests 24 unique files

---

### Feature 4: Agent Personality & Deliverable System

**What it is:** The foundational architecture giving each agent unique identity, workflow, and measurable outcomes.

**Template structure (from CONTRIBUTING.md):**

```
Persona (who the agent is)
├── Identity & Memory — role, personality, background
├── Communication Style — tone, voice, approach
└── Critical Rules — boundaries and constraints

Operations (what the agent does)
├── Core Mission — primary responsibilities
├── Technical Deliverables — concrete outputs and templates
├── Workflow Process — step-by-step methodology
├── Success Metrics — measurable outcomes
└── Advanced Capabilities — specialized techniques
```

**Personality traits:**

- Each agent has 4-6 distinct personality attributes
- Agents use first-person voice ("You are **GameDesigner**...")
- "Vibe line" provides one-line personality hook

**Critical Rules pattern:**

Agents enforce domain-specific rules marked **MANDATORY**:

```markdown
### Anti-Pattern Watchlist
- ❌ God MonoBehaviour with 500+ lines managing multiple systems
- ❌ `DontDestroyOnLoad` singleton abuse
- ❌ Tight coupling via `GetComponent<GameManager>()`
```

**Technical deliverables:**

Real, runnable code with:
- Language-specific syntax highlighting
- Comments explaining key concepts
- Modern best practices

**Clever solutions:**

1. **Services Frontmatter**: Declares external dependencies without coupling -- "The agent must stand on its own"
2. **Vibe Line**: One-line personality hook for memorability
3. **Memory Section**: Tracks what patterns worked/failed

**Technical debt:**

- Variable templating (`${variable}`) is Qwen-specific -- other tools don't use it
- Color normalization in convert.sh suggests inconsistency in source agent colors
- No schema validation -- tool relies on consistent contributor behavior
- No runtime error handling -- syntax errors in code blocks silently fail

---

### Feature 5: Division-Based Agent Organization

**What it is:** 144+ agents grouped into 12 functional divisions enabling coordinated multi-agent workflows.

**Directory structure:**

```
agency-agents/
├── engineering/           # 24 agents
├── design/               # 8 agents
├── marketing/            # 29 agents
├── paid-media/           # 7 agents
├── sales/                # 8 agents
├── product/              # 5 agents
├── project-management/    # 6 agents
├── testing/              # 8 agents
├── support/              # 6 agents
├── spatial-computing/    # 6 agents
├── specialized/          # 28 agents
├── game-development/     # 18 agents
├── academic/             # 5 agents
└── strategy/             # coordination/, playbooks/, runbooks/
```

**Division size distribution:**

| Division | Count | Percentage |
|----------|-------|------------|
| Marketing | 29 | 20.1% |
| Specialized | 28 | 19.4% |
| Engineering | 24 | 16.7% |
| Game Dev | 18 | 12.5% |
| Design | 8 | 5.6% |
| Testing | 8 | 5.6% |
| Sales | 8 | 5.6% |
| Support | 6 | 4.2% |
| Project Management | 6 | 4.2% |
| Spatial Computing | 6 | 4.2% |
| Academic | 5 | 3.5% |
| Product | 5 | 3.5% |

**Coordination infrastructure:**

```
strategy/
├── coordination/    # Multi-agent coordination patterns
├── playbooks/      # Playbook definitions
└── runbooks/       # Operational runbooks
```

**Clever solutions:**

1. **Domain Clustering**: China market agents grouped in Marketing (10+ agents)
2. **Engine Subdirectories**: Game Development uses engine-specific subdirs for targeted activation
3. **Division as Hiring Pool**: Mirrors actual agency staffing structures

**Technical debt:**

- Inconsistent directory depth: most divisions flat, Game Development 2-level
- No formal division metadata: no `divisions.json` describing responsibilities
- Strategy directory underdeveloped relative to 12 functional directories

---

### Feature 6: Multi-Engine Game Development

**What it is:** Game Development division with 18 agents across 5 major engines (Unity, Unreal Engine, Godot, Roblox Studio, Blender).

**Directory structure:**

```
game-development/
├── game-designer.md           # Generic — systems/mechanics design
├── level-designer.md          # Generic — level design and pacing
├── technical-artist.md        # Generic — art-engine bridge
├── game-audio-engineer.md     # Generic — audio implementation
├── narrative-designer.md      # Generic — story and dialogue
├── unity/                     # 4 agents
├── unreal-engine/             # 4 agents
├── godot/                     # 3 agents
├── roblox-studio/             # 3 agents
└── blender/                    # 1 agent
```

**Engine-specific technical depth:**

**Unity Architect** -- ScriptableObject-first architecture:
```csharp
public class GameEvent : ScriptableObject {
    private readonly List<GameEventListener> _listeners = new();
    public void Raise() { /* notify listeners */ }
}

public abstract class RuntimeSet<T> : ScriptableObject {
    public List<T> Items = new List<T>();
    public void Add(T item) { if (!Items.Contains(item)) Items.Add(item); }
}
```

**Godot Gameplay Scripter** -- GDScript 2.0 typed signals:
```gdscript
signal health_changed(new_health: float)

@onready var health: HealthComponent = $HealthComponent

signal player_died
```

**Unreal Systems Engineer** -- C++/Blueprint boundary:
```cpp
// MANDATORY: Any logic that runs every frame must be in C++
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health)
FGameplayAttributeData Health;

TWeakObjectPtr<APlayerController> CachedController;
```

**"MANDATORY" rule pattern:**

Each engine-specific agent has hard rules preventing common mistakes:
- Godot: "Every variable must be explicitly typed"
- Unreal: "All UObject pointers must use UPROPERTY()"
- Unity: "Never use GameObject.Find()"

**Cross-engine patterns:**

| Concept | Unity | Godot | Unreal |
|---------|-------|-------|--------|
| Event System | ScriptableObject events | Signal Bus Autoload | Event Dispatcher |
| Entity Tracking | RuntimeSet<T> | Typed arrays | TArray/TSubclassOf |
| Decoupled Architecture | SO-based messaging | Signal-driven composition | GAS-based ability system |

**Technical debt:**

- Blender underinvestment: only 1 agent vs. 4 each for Unity/Unreal
- No Godot 3.x/4.x version separation
- No "engine-agnostic gameplay programmer" -- missing cross-engine generic
- Unity Architect anti-pattern: "❌ God MonoBehaviour with 500+ lines"

---

### Feature 7: Production-Ready Workflow Examples

**What it is:** Pre-built multi-agent team configurations for common scenarios.

**File inventory:**

| File | Description | Lines |
|------|-------------|-------|
| `nexus-spatial-discovery.md` | 8-agent product discovery exercise (March 5, 2026) | ~850 |
| `workflow-startup-mvp.md` | 4-week Startup MVP workflow | ~155 |
| `workflow-with-memory.md` | MCP memory-enhanced startup-mvp | ~240 |
| `workflow-landing-page.md` | 1-day landing page sprint | ~120 |
| `workflow-book-chapter.md` | Book chapter writing workflow | ~60 |

**Workflow structure pattern:**

1. Agent table mapping names to roles
2. Timeboxed steps with agent activation prompts
3. Sequential handoffs (each output becomes next input)
4. Quality gates (Reality Checker) at midpoint and before launch
5. Parallel work identification

**Startup MVP workflow:**

```
Week 1: Discovery + Architecture
  - Sprint Prioritizer (sequential)
  - UX Researcher (parallel with Sprint Prioritizer)
  - Backend Architect (receives combined output)

Week 2: Build Core Features
  - Frontend Developer + Rapid Prototyper (parallel)
  - Reality Check at midpoint

Week 3: Polish + Landing Page
Week 4: Launch (Reality Checker final GO/NO-GO)
```

**Manual context passing:**

> "Always paste previous agent outputs into the next prompt -- agents don't share memory"

**Memory solution** (`workflow-with-memory.md`):

1. Tag everything with project name for recall
2. Tag deliverables for receiving agent
3. Reality Checker gets full visibility
4. Rollback replaces manual undo

**Nexus Spatial discovery** -- most detailed workflow:

8 agents deployed: Product Trend Researcher, Backend Architect, Brand Guardian, Growth Hacker, Support Responder, UX Researcher, Project Shepherd, XR Interface Architect

10 synthesis sections including real market data (Vision Pro: ~1M units, 95% sales decline), brand color system, 35-week timeline, $121.5K-$155.5K budget.

**Technical debt:**

- `workflow-book-chapter.md` sparse (~60 lines)
- No parameterized templates -- each is one-off
- Manual agent activation assumes human copy-paste
- No orchestration code -- workflows describe but don't execute
- `examples/README.md` empty

---

## Priority 2: Secondary Features

### Feature 8: China Market Agent Suite

**What it is:** 10+ specialized agents for Chinese platforms and markets.

**Agent roster:**

| Agent | Platform | Color |
|-------|----------|-------|
| Baidu SEO Specialist | Baidu search | blue |
| Douyin Strategist | Douyin/TikTok | #000000 |
| Xiaohongshu Specialist | Xiaohongshu/RED | #FF1B6D |
| Bilibili Content Strategist | Bilibili | -- |
| WeChat Official Account Manager | WeChat | -- |
| Kuaishou Strategist | Kuaishou | -- |
| Zhihu Strategist | Zhihu | -- |
| China E-Commerce Operator | Taobao/Tmall/Pinduoduo/JD/Douyin Shop | red |
| Cross-Border E-Commerce Specialist | Amazon/TikTok Shop/Temu | blue |
| WeChat Mini Program Developer | WeChat Mini Programs | purple |

**Douyin Strategist** -- algorithm-first approach:

Algorithm Priority: completion rate > like rate > comment rate > share rate

Viral Video Script Template:
```
Seconds 1-3: Golden Hook (conflict/value/suspense/relatability)
Seconds 4-20: Core Content (pain point, solution, demo, results)
```

Livestream Product Structure: 20% traffic driver, 50% profit item, 15% prestige, 15% flash deal

**Baidu SEO Specialist** -- compliance-first:

```markdown
## 基础合规 (Compliance Foundation)
- [ ] ICP备案 status: [Valid/Pending/Missing]
- [ ] Server location: [City, Provider] - Ping to Beijing: [ms]
- [ ] SSL certificate: [Domestic CA recommended]
```

Baidu Algorithm Mastery: 飓风 (Hurricane), 细雨 (Drizzle), 惊雷 (Thunder), 蓝天 (Blue Sky), 清风 (Breeze)

**China E-Commerce Operator** -- T-60 campaign battle plan:

```
T-60 Days: Strategic Planning (GMV target, platform slots)
T-30 Days: Preparation (creative assets, ad campaigns)
T-7 Days: Warm-Up (pre-sale listings, spend ramp)
T-Day: Campaign Execution (war room, real-time GMV)
T+1 to T+7: Post-Campaign (performance report, retention)
```

**Technical debt:**

- Inconsistent depth: Douyin/Baidu thorough, Bilibili/Kuaishou/Zhihu less detailed
- No actual Chinese content -- frameworks only
- Regulatory info may be outdated (frequent changes)
- No integration workflow between agents

---

### Feature 9: Testing & QA Division

**What it is:** 8 testing agents covering evidence collection, reality checking, performance benchmarking, API testing, accessibility auditing.

**Agent roster:**

| Agent | Focus | Color |
|-------|-------|-------|
| Evidence Collector | Screenshot-based QA | orange |
| Reality Checker | Integration verification | red |
| Performance Benchmarker | Load/speed testing | orange |
| API Tester | API validation | purple |
| Accessibility Auditor | WCAG compliance | #0077B6 |
| Tool Evaluator | Technology assessment | teal |
| Workflow Optimizer | Process improvement | green |
| Test Results Analyzer | Results analysis | -- |

**Evidence Collector philosophy: "Screenshots Don't Lie"**

Mandatory Process:
```bash
./qa-playwright-capture.sh http://localhost:8000 public/qa-screenshots
grep -r "luxury\|premium\|glass\|morphism" . --include="*.html" --include="*.css"
cat public/qa-screenshots/test-results.json
```

**Automatic FAIL triggers:**
- Any agent claiming "zero issues found"
- Perfect scores (A+, 98/100) on first implementation
- "Luxury/premium" claims without visual evidence

**Quality Rating Scale (Honest):**
- C+ / B- / B / B+ (NO A+ fantasies)
- Design Level: Basic / Good / Excellent
- Production Readiness: FAILED / NEEDS WORK / READY (default to FAILED)

**Performance Benchmarker** -- Core Web Vitals:

| Metric | Target | Description |
|--------|--------|-------------|
| LCP | < 2.5s | Largest Contentful Paint |
| FID | < 100ms | First Input Delay |
| CLS | < 0.1 | Cumulative Layout Shift |

k6 Load Test with stages: warm-up → normal load → peak load → sustained peak → stress test → cool down

**Accessibility Auditor** -- 70% manual:

> "Automated tools catch roughly 30% of accessibility issues -- you catch the other 70%"

Protocol: axe-core automated (30%) + manual screen reader/keyboard/voice control testing (70%)

**Technical debt:**

- `qa-playwright-capture.sh` referenced but doesn't exist in repo
- `ai/agents/qa.md` referenced but doesn't exist
- No actual test code files -- frameworks/templates only
- No CI/CD integration

---

### Feature 10: Spatial Computing / XR Division

**What it is:** 6 agents for AR/VR/XR development across Apple platforms, browser-based WebXR, cockpit systems, and terminal integration.

**Agent roster:**

| Agent | Specialization | Emoji |
|-------|---------------|-------|
| XR Interface Architect | Spatial UX/UI, HUDs, gesture input | bubble |
| visionOS Spatial Engineer | Native visionOS 26, SwiftUI, Liquid Glass | glasses |
| macOS Spatial/Metal Engineer | Swift/Metal, Vision Pro streaming, 90fps | apple |
| XR Immersive Developer | WebXR, A-Frame, Three.js | globe |
| XR Cockpit Interaction Specialist | Cockpit controls, seated XR | joystick |
| Terminal Integration Specialist | SwiftTerm, VT100/xterm, SSH | monitor |

**macOS Spatial/Metal Engineer** -- most technically detailed agent:

Critical Performance Requirements:
```yaml
Frame rate: 90fps minimum
GPU utilization: <80%
Draw calls: <100 per frame
Memory: <1GB
Gaze-to-selection latency: <50ms
```

Metal Rendering Architecture with instanced rendering, edge geometry shaders, Vision Pro Compositor integration with stereo configuration.

**visionOS Spatial Engineer** -- visionOS 26 specific:

- Liquid Glass materials, Spatial Widgets, WindowGroups
- SwiftUI volumetric APIs, RealityKit-SwiftUI integration
- References WWDC25 videos, beta features

**Technical debt:**

- XR Immersive Developer sparse (~33 lines, no code)
- macOS Spatial/Metal Engineer extremely detailed (~330 lines) -- inconsistent
- No fallback for non-Vision Pro users
- visionOS 26 beta features may not be stable

---

### Feature 11: Specialized Agents for Niche Domains

**What it is:** 28+ agents covering blockchain security, healthcare compliance, government digital presales, Salesforce, Korean business, ZK knowledge management, MCP development.

**Notable agents:**

**Blockchain Security Auditor** -- most detailed security agent:

Severity Classification: Critical > High > Medium > Low > Informational

Slither Integration:
```bash
slither . --detect reentrancy-eth,arbitrary-send-eth,suicidal,\
  controlled-delegatecall,uninitialized-state,unchecked-transfer,locked-ether
```

Reentrancy vulnerability patterns with before/after fixes.

**Salesforce Architect** -- Governor Limit Budget:

```markdown
Transaction Budget (Synchronous):
├── SOQL:     100 total
├── DML:      150 total
├── CPU:      10,000ms
├── Heap:     6,144 KB
├── Callouts: 100
└── Future:  50
```

**Healthcare Marketing Compliance** -- extensive China regulatory:

Advertising Law, Medical Advertisement Management Measures, PIPL. Prohibited terms (absolute claims, guarantees, inducements). Medical aesthetics red lines: no before-and-after photos.

**Government Digital Presales Consultant** -- China ToG procurement:

- Dengbao 2.0 (Classified Protection): Level 3/4 requirements
- Miping (Cryptographic Assessment): Guomi algorithms (SM2/SM3/SM4)
- Xinchuang: Domestic CPUs (Kunpeng/Phytium), OS (UnionTech UOS/Kylin)

**Korean Business Navigator** -- 품의 (consensus) timeline:

| Company Type | Timeline |
|-------------|----------|
| SME | 6-10 weeks |
| Mid-cap | 8-12 weeks |
| Chaebol | 12-16 weeks |
| Foreign expectation | 2-4 weeks |

**ZK Steward** -- Luhmann's Zettelkasten for AI:

Luhmann's Four Principles: Atomicity, Connectivity, Organic growth, Continued dialogue

Domain-Expert Mapping: Ogilvy (brand), Godin (growth), Munger (strategy), Porter (competitive), Feynman (learning).

**Technical debt:**

- Inconsistent depth: Healthcare/Government agents 400+ lines, MCP Builder ~64 lines
- Real-world knowledge may become stale (regulations, exploits, technology)
- No formal collaboration patterns between related agents

---

### Feature 12: Academic Division for World-Building

**What it is:** 5 agents for narrative and world-building design forming a complete academic team.

**Agent roster:**

| Agent | Specialization | Color |
|-------|---------------|-------|
| Anthropologist | Cultural systems, kinship, rituals | amber |
| Geographer | Physical/human geography, climate | emerald |
| Historian | Historical analysis, periodization | brown |
| Narratologist | Narrative theory, story structure | purple |
| Psychologist | Human behavior, personality, trauma | pink |

**Anthropologist** -- cultural systems:

Core Framework: Function before aesthetics, no culture salad, kinship is infrastructure, avoid Noble Savage

Deliverables include Cultural System Analysis with subsistence/economy, social organization (kinship, residence, political), belief systems (cosmology, ritual, specialists).

Key Theorists: Levi-Strauss, Geertz, Bourdieu, Malinowski, van Gennep, Polanyi.

**Geographer** -- geographic coherence:

Core Rules:
- Rivers don't split
- Climate is a system
- Geography is not decoration
- Avoid geographic determinism
- Scale matters
- Maps are arguments

Climate System Design: axial tilt, ocean currents, prevailing winds, continental position, rain shadows.

**Historian** -- period authenticity:

Core Rules: Name sources, history is not a monolith, challenge Eurocentrism, material conditions first, avoid presentism, myths are data.

Period Authenticity Report: material culture, social structure, anachronism flags.

**Narratologist** -- story structure:

Frameworks: Propp's morphology, Campbell's monomyth, Vogler's Writer's Journey, Todorov's equilibrium model, Genette's narratology, Barthes' five codes.

Character Arc Assessment: Arc Type, Want vs. Need, Ghost/Wound, Lie Believed, Arc Checkpoints.

**Psychologist** -- behavioral credibility:

Core Rules: No reducing characters to diagnoses, distinguish pop psychology from research, acknowledge cultural context, trauma responses are diverse, honest about limitations.

Deliverables include Psychological Profile (Big Five, Attachment, Defense Mechanisms) and Interpersonal Dynamics Analysis.

**Key observation:** Most coherent agent group -- five agents form self-checking system. Highest theoretical grounding with named frameworks. Least depth variation (all substantial).

---

## Cross-Cutting Technical Debt

| Issue | Severity | Affects |
|-------|----------|---------|
| No schema validation | Medium | All agents |
| Referenced files don't exist | High | Testing, Workflows |
| Inconsistent agent depth | Medium | Most divisions |
| No uninstall mechanism | Low | Multi-tool integration |
| Strategy/coordination underinvested | Medium | Division organization |
| Manual handoff assumption | High | Workflows |
| Real-world knowledge staleness | Medium | China, Blockchain, Healthcare |

---

## Clever Solutions Summary

1. **Persona/Operations Separation** (OpenClaw): SOUL.md + AGENTS.md + IDENTITY.md
2. **Services Frontmatter**: External dependencies without coupling
3. **MANDATORY Rule Pattern**: Hard constraints with anti-pattern watchlists
4. **Memory Workflow**: Tags + recall instead of copy-paste
5. **Academic Team Cross-Checking**: Five agents validate each other's work
6. **Governor Limit Budget**: Quantified Salesforce constraints
7. **品의 Timeline**: Explicit Korean vs. Western decision pace contrast
8. **Evidence-Based QA**: Screenshots as ground truth, not claims
9. **Honest Quality Ratings**: C+/B-/B/B+ instead of inflated A+

---

## Architectural Insights

**Strengths:**
- Single-source markdown enables multi-tool portability
- Consistent 8-section structure across all agents
- Concrete code deliverables with real examples
- Measurable success metrics throughout
- Domain clustering enables team-based deployment

**Weaknesses:**
- No executable code -- knowledge base only
- Manual orchestration required
- No schema validation
- Inconsistent depth across agents
- Missing cross-references between related agents

**Repository is well-organized** with clear division-based categorization. All README claims corroborated by actual folder structure.

---

*Deep dive synthesized from 4 batch research documents: 05a-features-batch-1.md, 05b-features-batch-2.md, 05c-features-batch-3.md, 05d-features-batch-4.md*

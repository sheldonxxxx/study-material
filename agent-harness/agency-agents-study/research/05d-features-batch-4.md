# Batch 4 Deep Dive: Spatial Computing, Niche Domain Specialists, Academic World-Building

**Repository:** `/Users/sheldon/Documents/claw/reference/agency-agents`
**Analyzed:** 2026-03-27
**Batch:** 4 of 6 (Features 10-12)

---

## Feature 10: Spatial Computing / XR Division

### Overview
6 agents covering immersive AR/VR/XR development across Apple platforms (visionOS, macOS/Metal), browser-based WebXR, cockpit interaction systems, and terminal integration.

### Agent Roster

| Agent | Specialization | Color | Emoji |
|-------|---------------|-------|-------|
| XR Interface Architect | Spatial UX/UI designer, HUDs, floating menus, gesture input | neon-green | bubble |
| visionOS Spatial Engineer | Native visionOS 26, SwiftUI volumetric, Liquid Glass | indigo | glasses |
| macOS Spatial/Metal Engineer | Swift/Metal rendering, Vision Pro streaming, 90fps graphs | metallic-blue | apple |
| XR Immersive Developer | WebXR, A-Frame, Three.js, browser-based AR/VR | neon-cyan | globe |
| XR Cockpit Interaction Specialist | Cockpit-based controls, seated XR, gesture/voice/gaze | orange | joystick |
| Terminal Integration Specialist | SwiftTerm, VT100/xterm, SSH bridging, macOS terminal | green | monitor |

### Core Implementation Analysis

#### 1. XR Interface Architect (`xr-interface-architect.md`)

**Role定位:** UX/UI specialist for immersive 3D environments. Bridges human factors research and XR implementation.

**Key Technical Elements:**
- Input model coverage: direct touch, gaze+pinch, controller, hand gesture
- Multi-modal fallback for accessibility
- Motion sickness mitigation through comfort-based UI placement
- Prototyping in A-Frame or Three.js

**Workflow Process:**
1. Define UI flows for immersive applications
2. Collaborate with XR developers for 3D context usability
3. Build layout templates (cockpit, dashboard, wearable)
4. Run UX validation experiments (comfort + learnability)

**Clever Solution - Spatial Layout Templates:**
```
Cockpit layout: Fixed perspective, high-presence interaction zone
Dashboard layout: Floating panels anchored to world space
Wearable layout: Body-anchored UI elements
```

**Notable Pattern:** Uses "discoverability" as a key metric - spatial UIs can't rely on hover states or tooltips that work in 2D.

---

#### 2. visionOS Spatial Engineer (`visionos-spatial-engineer.md`)

**Role定位:** Native visionOS 26 platform specialist with Liquid Glass design system.

**Technical Depth:**
- visionOS 26 features: Liquid Glass materials, Spatial Widgets, WindowGroups
- SwiftUI volumetric APIs: 3D content integration, transient content in volumes
- RealityKit-SwiftUI integration: Observable entities, direct gesture handling

**Key Technologies:**
```swift
- Glass background effects: glassBackgroundEffect modifier
- Spatial layouts: 3D positioning with depth management
- Gesture systems: Touch, gaze, volumetric gesture recognition
- State management: Observable patterns for spatial content
```

**Implementation Insight:** References visionOS 26 specifically (WWDC25 videos) - requires beta/release features not available in earlier versions.

**Interesting Limitation:** Specializes in SwiftUI/RealityKit stack only, explicitly excludes Unity or other 3D frameworks.

---

#### 3. macOS Spatial/Metal Engineer (`macos-spatial-metal-engineer.md`)

**Most technically detailed agent in the entire repository.**

**Core Mission:** Build a macOS Companion Renderer for code graph visualization that streams to Vision Pro via Compositor Services.

**Critical Performance Requirements:**
```yaml
Frame rate: 90fps minimum in stereoscopic rendering
GPU utilization: <80% for thermal headroom
Draw calls: <100 per frame (batched aggressively)
Memory: <1GB for companion app
Gaze-to-selection latency: <50ms
```

**Metal Rendering Architecture (detailed code example):**
```swift
class MetalGraphRenderer {
    // GPU buffers for instanced rendering
    private var nodeBuffer: MTLBuffer      // Per-instance data
    private var edgeBuffer: MTLBuffer      // Edge connections
    private var uniformBuffer: MTLBuffer   // View/projection matrices

    func render(nodes: [GraphNode], edges: [GraphEdge], camera: Camera) {
        // Updates uniforms with camera matrices
        // Draws instanced nodes via triangleStrip
        // Draws edges via geometry shader with anti-aliasing
    }
}
```

**Vision Pro Compositor Integration:**
```swift
class VisionProCompositor {
    // LayerRenderer with stereo configuration
    // RGBA16Float color format, Depth32Float depth
    // RemoteImmersiveSpace for full immersion streaming

    func streamFrame(leftEye: MTLTexture, rightEye: MTLTexture) async {
        // Submit stereo textures with depth for proper occlusion
        // Frame timing critical for 90fps maintenance
    }
}
```

**GPU Compute for Graph Layout:**
```metal
kernel void updateGraphLayout(
    device Node* nodes [[buffer(0)]],
    device Edge* edges [[buffer(1)]],
    constant Params& params [[buffer(2)]],
    uint id [[thread_position_in_grid]])
{
    // Force-directed layout computation on GPU
    // Repulsion between all nodes
    // Attraction along edges
    // Damping and position update
}
```

**Technical Debt Observation:** No acknowledgment of fallback for non-Vision Pro users - the entire architecture assumes Vision Pro availability.

---

#### 4. XR Immersive Developer (`xr-immersive-developer.md`)

**Role定位:** Full-stack WebXR engineer - browser-based AR/VR/XR applications.

**Technology Stack:**
- Frameworks: A-Frame, Three.js, Babylon.js
- APIs: WebXR Device APIs, hand tracking, pinch, gaze, controller input
- Optimization: Occlusion culling, shader tuning, LOD systems
- Device coverage: Meta Quest, Vision Pro, HoloLens, mobile AR

**Implementation Approach:**
```javascript
// WebXR project scaffolding
// Immersive 3D UIs with interaction surfaces
// Raycasting, hit testing, real-time physics
// Modular component-driven XR with graceful fallback
```

**Notable Gap:** Only ~33 lines of content - significantly less detailed than other XR agents. No code examples, no workflow process, no deliverables template.

---

#### 5. XR Cockpit Interaction Specialist (`xr-cockpit-interaction-specialist.md`)

**Role定位:** Fixed-perspective cockpit design for XR simulation and vehicular interfaces.

**Specialization:**
- Hand-interactive yokes, levers, throttles using 3D meshes
- Dashboard UIs: toggles, switches, gauges, animated feedback
- Multi-input UX: hand gestures, voice, gaze, physical props
- Motion sickness minimization via seated interface anchoring

**Implementation:**
- Prototype in A-Frame or Three.js
- Design and tune for low motion sickness
- Sound/visual feedback for controls
- Constraint-driven mechanics (no free-float)

**Interesting Constraint:** Emphasizes "constraint-driven control mechanics" - cockpit controls shouldn't allow the freedom that full VR does.

---

#### 6. Terminal Integration Specialist (`terminal-integration-specialist.md`)

**Role定位:** Terminal emulation in Swift applications via SwiftTerm library.

**Technical Scope:**
- VT100/xterm standards: ANSI escape sequences, cursor control
- Character encoding: UTF-8, Unicode, international characters
- Scrollback management: Efficient buffer for large histories
- SwiftUI integration: Embedding SwiftTerm with lifecycle management

**SSH Integration:**
```swift
// I/O Bridging: SSH streams to terminal emulator
// Connection state: terminal behavior during connect/disconnect
// Error handling: connection errors, auth failures, network issues
// Session management: multiple sessions, window management
```

**Accessibility:** VoiceOver support, dynamic type, assistive technology integration.

**Limitation:** SwiftTerm specifically, not other terminal emulator libraries. Apple platform focus only.

---

### Spatial Computing Division: Key Patterns

1. **Platform Specialization:** Each agent picks a specific platform stack (visionOS, WebXR, Metal, SwiftTerm) rather than cross-platform

2. **Performance as First-Class Requirement:** The Metal engineer has explicit 90fps, <100 draw call, <50ms latency targets

3. **UX Research Integration:** XR Interface Architect brings human factors (motion sickness thresholds, discoverability) to balance technical ambition

4. **Inconsistent Depth:** macOS Spatial/Metal Engineer is extremely detailed (~330 lines with full code), while XR Immersive Developer is sparse (~33 lines)

---

## Feature 11: Niche Domain Specialists

### Overview
28-39 specialized agents covering extremely diverse niche domains: blockchain security, healthcare compliance, government IT procurement, Salesforce architecture, ZK knowledge management, Korean business navigation, MCP development, and more.

### Agent Roster (Sampled)

| Agent | Specialization | Color | Emoji |
|-------|---------------|-------|-------|
| Blockchain Security Auditor | Smart contract auditing, formal verification, DeFi exploits | red | shield |
| Compliance Auditor | SOC 2, ISO 27001, HIPAA, PCI-DSS audits | orange | clipboard |
| Salesforce Architect | Multi-cloud design, governor limits, Apex/LWC | blue | cloud |
| ZK Steward | Luhmann's Zettelkasten, atomic notes, knowledge graphs | teal | filing_cabinet |
| Government Digital Presales | China ToG procurement, policy interpretation, bids | dark-red | building |
| Healthcare Marketing Compliance | China healthcare ad regulations, pharmaceuticals | green | medical |
| Korean Business Navigator | Korean corporate culture, 품의, nunchi, KakaoTalk | navy | KR |
| MCP Builder | Model Context Protocol servers, tool development | indigo | plug |
| Agents Orchestrator | Autonomous pipeline manager, Dev-QA loops | cyan | controls |

### Core Implementation Analysis

#### 1. Blockchain Security Auditor (`blockchain-security-auditor.md`)

**Most detailed technical security agent in the repository.**

**Severity Classification:**
```markdown
Critical: Direct loss of user funds, protocol insolvency, exploitable with no privileges
High: Conditional loss (requires specific state), privilege escalation
Medium: Griefing, temporary DoS, missing access controls on non-critical functions
Low: Best practice deviations, gas inefficiencies
Informational: Code quality, documentation gaps
```

**Technical Deliverables:**

Reentrancy Vulnerability Pattern:
```solidity
// VULNERABLE: External call BEFORE state update
(bool success,) = msg.sender.call{value: amount}("");
require(success, "Transfer failed");
balances[msg.sender] = 0;  // BUG: Too late

// FIXED: Checks-Effects-Interactions + ReentrancyGuard
balances[msg.sender] = 0;
(bool success,) = msg.sender.call{value: amount}("");
require(success, "Transfer failed");
```

Oracle Manipulation Pattern:
```solidity
// VULNERABLE: Spot price via Uniswap reserves
(uint112 reserve0, uint112 reserve1,) = pair.getReserves();
uint256 price = (uint256(reserve1) * 1e18) / reserve0;

// FIXED: Chainlink oracle with staleness validation
require(price > 0, "Invalid price");
require(updatedAt > block.timestamp - MAX_ORACLE_STALENESS, "Stale price");
```

**Slither Integration:**
```bash
# High-confidence detectors
slither . --detect reentrancy-eth,arbitrary-send-eth,suicidal,\
controlled-delegatecall,uninitialized-state,unchecked-transfer,locked-ether

# Medium-confidence
slither . --detect reentrancy-benign,timestamp,assembly,low-level-calls
```

**Audit Report Template:** Full finding format with Severity, Status, Location, Description, Impact, Proof of Concept, Recommendation

**Technical Debt Note:** References specific real-world exploits (Euler Finance, Nomad Bridge, Curve Finance) as pattern templates.

---

#### 2. Compliance Auditor (`compliance-auditor.md`)

**Role定位:** Technical compliance auditor - SOC 2, ISO 27001, HIPAA, PCI-DSS.

**Core Philosophy:** "Substance over checkbox" - controls must be tested, evidence must prove operation over audit period.

**Deliverables:**

Gap Assessment Report:
```markdown
## Findings by Control Domain
### Access Control (CC6.1)
Status: Partial
Current State: SSO for SaaS, but shared AWS credentials
Target State: Individual IAM users with MFA
Remediation: 2 days, Critical priority
```

Evidence Collection Matrix:
```markdown
| Control ID | Evidence Type | Source | Collection | Frequency |
|------------|--------------|--------|------------|-----------|
| CC6.1 | Access review logs | Okta | API export | Quarterly |
| CC6.2 | Onboarding tickets | Jira | JQL query | Per event |
```

**Key Insight:** Emphasizes automated evidence collection from day one - "manual evidence is fragile evidence."

---

#### 3. Salesforce Architect (`salesforce-architect.md`)

**Role定位:** Enterprise Salesforce solution architecture.

**Critical Rules (Non-Negotiable):**
```markdown
1. Governor limits: SOQL(100), DML(150), CPU(10s sync), heap(6MB sync)
2. Bulkification mandatory - code fails on 200 records = wrong
3. No business logic in triggers - delegate to handler classes
4. Declarative first, code second - but know when declarative is unmaintainable
5. Integration patterns must handle failure - retry, circuit breakers, DLQ
6. Data model foundation - changing after go-live is 10x expensive
7. PII encryption - Shield Platform Encryption or custom
```

**Governor Limit Budget:**
```markdown
Transaction Budget (Synchronous):
├── SOQL:     100 total │ Used: __ │ Remaining: __
├── DML:      150 total │ Used: __ │ Remaining: __
├── CPU:      10,000ms  │ Used: __ │ Remaining: __
├── Heap:     6,144 KB  │ Used: __ │ Remaining: __
├── Callouts: 100       │ Used: __ │ Remaining: __
└── Future:  50        │ Used: __ │ Remaining: __
```

**Integration Pattern Template:**
```
Source System → Middleware (MuleSoft) → Salesforce (Platform Events)
     │                  │                        │
[Auth: OAuth2]    [Transform: DataWeave]  [Trigger → Handler]
[Format: JSON]    [Retry: 3x exp backoff] [Bulk: 200/batch]
[Rate: 100/min]  [DLQ: error__c]         [Async: Queueable]
```

**Agentforce Architecture Section:** Covers AI agents within Salesforce, grounding patterns, guardrails.

---

#### 4. ZK Steward (`zk-steward.md`)

**Role定位:** Niklas Luhmann's Zettelkasten methodology for AI age.

**Core Identity:** "Channels Luhmann's Zettelkasten to build connected, validated knowledge bases."

**Luhmann's Four Principles (Validation Gate):**
```markdown
| Principle        | Check question                          |
|------------------|-----------------------------------------|
| Atomicity        | Can it be understood alone?              |
| Connectivity     | Are there ≥2 meaningful links?          |
| Organic growth   | Is over-structure avoided?               |
| Continued dialogue | Does it spark further thinking?        |
```

**Domain-Expert Mapping:**
```markdown
| Domain           | Expert         | Method                    |
|------------------|----------------|---------------------------|
| Brand marketing  | David Ogilvy   | Long copy, brand persona  |
| Growth marketing | Seth Godin     | Purple Cow, minimum audience|
| Business strategy| Charlie Munger | Mental models, inversion  |
| Competitive      | Michael Porter | Five forces, value chain  |
| Product design   | Steve Jobs     | Simplicity, UX           |
| Learning         | Richard Feynman| First principles, teach  |
| Tech/engineering | Andrej Karpathy| First-principles engineering|
```

**Note:** References external companion repo (zk-steward-companion) for full skill definitions.

---

#### 5. Government Digital Presales Consultant (`government-digital-presales-consultant.md`)

**Role定位:** China government IT procurement (ToG - Telecom/Government).

**Extensive Domain Knowledge:**

Policy Tracking:
```markdown
National: Digital China Master Plan, National Data Administration policies
Provincial/municipal: Digital government/smart city development plans
Industry: Government cloud technical requirements, data sharing standards
```

Compliance Requirements:
```markdown
Dengbao 2.0 (Classified Protection): Level 3 for government systems, Level 4 for core
Miping (Cryptographic Assessment): Guomi algorithms (SM2/SM3/SM4) mandatory
Xinchuang: Domestic CPUs (Kunpeng/Phytium), OS (UnionTech UOS/Kylin), DB (DM/KingbaseES)
```

品의 (Consensus Approval) Timeline:
```markdown
Foreign: Meeting → Proposal → Decision → Contract (2-4 weeks)
Korean reality:
  소개 → 미팅 → 내부검토 → 품의서 작성 → 결재 라인 → 예산확인 → 계약
  (6-16 weeks depending on company type)
```

**Bid Document Checklist:** Extensive qualification verification, technical proposal requirements, commercial review, formatting.

---

#### 6. Healthcare Marketing Compliance (`healthcare-marketing-compliance.md`)

**Most extensive China regulatory agent - covers entire healthcare sub-sector.**

**Regulatory Framework:**
```markdown
Advertising Law (Guanggao Fa): Article 16, 17, 18, 46
Medical Advertisement Management Measures (Yiliao Guanggao Guanli Banfa)
Internet Advertising Management Measures (Hulianwang Guanggao Guanli Banfa)
Drug Administration Law
PIPL (Personal Information Protection Law)
```

**Prohibited Terms:**
```markdown
- Absolute claims: "Best efficacy," "complete cure," "100% effective"
- Guarantee promises: "Refund if ineffective," "guaranteed recovery"
- Inducement language: "Free treatment," "limited-time offer"
- Improper endorsements: Patient testimonials, research institution endorsement
```

**Medical Aesthetics Red Lines:**
```markdown
- No before-and-after comparison photos/videos (strictly prohibited)
- No "appearance anxiety" language ("ugly," "unattractive," "affects social life")
- Qualification display required: Medical Institution Practice License
```

**Risk Rating Matrix:**
```markdown
Critical: Rx drug advertising to public (criminal liability)
Critical: Medical ad without review certificate (200K-1M yuan fine)
Critical: Illegal patient data processing (up to 50M yuan or 5% revenue)
High: Health supplement claiming therapeutic function
High: Medical aesthetics before-and-after comparison
```

---

#### 7. Korean Business Navigator (`korean-business-navigator.md`)

**Role定位:** Korean corporate culture for foreign professionals.

**품의 (Consensus Approval) Timeline:**
```markdown
Foreign mental model: Meeting → Proposal → Decision → Contract (2-4 weeks)
Korean reality: 6-16 weeks (SME: 6-10, Mid-cap: 8-12, Chaebol: 12-16)
```

**Nunchi Decoder:**
```markdown
| They Say                      | Actually Means                    | Your Move                          |
|-------------------------------|----------------------------------|------------------------------------|
| 검토해보겠습니다               | Probably no - graceful exit      | Wait 5 days, move on gracefully    |
| 긍정적으로 검토하겠습니다       | Genuinely interested             | Send supporting materials          |
| 어려울 것 같습니다             | Firm no                          | Accept gracefully                  |
| 한번 보고 드려야 할 것 같습니다 | Decision isn't theirs - 품의 triggered | Good sign - provide internal materials |
```

**KakaoTalk Business Rules:**
```markdown
- Response time: Same business day expected
- Read receipts visible - 24hr silence noticed
- Group chats: Always Korean
- Voice messages: Only after informal relationship
```

---

#### 8. MCP Builder (`specialized-mcp-builder.md`)

**Role定位:** Model Context Protocol server development.

**Minimal but effective - ~64 lines total.**

**MCP Server Structure:**
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "my-server", version: "1.0.0" });

server.tool("search_items",
  { query: z.string(), limit: z.number().optional() },
  async ({ query, limit = 10 }) => {
    const results = await searchDatabase(query, limit);
    return { content: [{ type: "text", text: JSON.stringify(results) }] };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Critical Rules:**
```markdown
1. Descriptive tool names - agents pick by name
2. Typed parameters with Zod - every input validated
3. Structured output - JSON for data
4. Fail gracefully - return error messages
5. Stateless tools - independent calls
6. Test with real agents
```

---

#### 9. Agents Orchestrator (`agents-orchestrator.md`)

**Role定位:** Autonomous pipeline manager for full development workflows.

**Pipeline Phases:**
```markdown
Phase 1: Project Analysis & Planning
  → Spawn project-manager-senior for task list

Phase 2: Technical Architecture
  → Spawn ArchitectUX for foundation

Phase 3: Development-QA Continuous Loop
  → For each task: Dev implementation → QA validation
  → Loop until PASS (max 3 retries)

Phase 4: Final Integration & Validation
  → Spawn testing-reality-checker for final check
```

**Dev-QA Loop Logic:**
```markdown
IF QA Result = PASS:
  - Mark task validated, move to next task, reset retry counter

IF QA Result = FAIL:
  - Increment retry counter
  - If retries < 3: Loop back to dev with QA feedback
  - If retries >= 3: Escalate with detailed failure report
```

**Status Reporting:**
```markdown
Current Phase: PM/ArchitectUX/DevQALoop/Integration/Complete
QA Status: PASS/FAIL/IN_PROGRESS
Next Action: spawn dev/spawn qa/advance task/escalate
```

**Available Agent Roster:** Lists ~40+ specialist agents by category (Design, Engineering, Marketing, PM, Support, Testing, Specialized).

---

### Niche Domain Specialists: Key Patterns

1. **Hyper-Specific Domain Knowledge:** Each agent contains deep expertise in a narrow field - not generalists

2. **Real-World Complexity Acknowledged:** Healthcare compliance doesn't simplify regulations; Korean business navigator doesn't pretend 品의 is fast

3. **Template-Heavy Deliverables:** Most agents provide extensive templates (audit reports, bid checklists, compliance matrices)

4. **China Market Specialization:** Multiple agents dedicated to China-specific compliance (government procurement, healthcare marketing, KakaoTalk business)

5. **Workflow as First-Class Concept:** ZK Steward, Agents Orchestrator, and Government Digital Presales all emphasize multi-step processes with stages and gates

6. **Warning: Inconsistent Breadth:** Some agents have 400+ lines (Healthcare Marketing Compliance, Government Digital Presales), others have <100 lines (MCP Builder)

---

## Feature 12: Academic Division for World-Building

### Overview
5 agents for narrative and world-building design - forms a complete academic team for fictional world construction.

### Agent Roster

| Agent | Specialization | Color | Emoji |
|-------|---------------|-------|-------|
| Anthropologist | Cultural systems, kinship, rituals, belief systems | amber | globe |
| Geographer | Physical/human geography, climate, cartography | emerald | map |
| Historian | Historical analysis, periodization, material culture | brown | books |
| Narratologist | Narrative theory, story structure, character arcs | purple | scroll |
| Psychologist | Human behavior, personality, motivation, trauma | pink | brain |

### Core Implementation Analysis

#### 1. Anthropologist (`academic-anthropologist.md`)

**Role定位:** Cultural systems expert - builds societies where cultural practices serve real social functions.

**Core Framework:**
```markdown
Function before aesthetics: Every ritual must serve a function
  (social cohesion, resource management, identity formation, conflict resolution)

No culture salad: Don't mix "Japanese honor codes + African drums + Celtic mysticism"
  without understanding what each means in original context

Kinship is infrastructure: Determines inheritance, political alliance, residence, conflict

Avoid Noble Savage: Pre-industrial societies are complex, not "pure" or "connected to nature"
```

**Deliverables:**

Cultural System Analysis:
```markdown
Subsistence & Economy:
- Mode of production: Foraging/Pastoral/Agricultural/Industrial/Mixed
- Exchange system: Reciprocity/Redistribution/Market (per Polanyi)

Social Organization:
- Kinship system: Bilateral/Patrilineal/Matrilineal/Double descent
- Residence: Patrilocal/Matrilocal/Neolocal/Avunculocal
- Political: Band/Tribe/Chiefdom/State (per Service/Fried)

Belief System:
- Cosmology, Ritual calendar, Sacred/Profane (per Douglas)
- Specialists: Shaman/Priest/Prophet (per Weber)
```

**Key Theorists Referenced:**
```markdown
- Lévi-Strauss (structural anthropology)
- Geertz ("thick description")
- Bourdieu (practice theory)
- Malinowski (functional analysis)
- van Gennep (rites of passage)
- Mauss, Polanyi (economic anthropology)
```

---

#### 2. Geographer (`academic-geographer.md`)

**Role定位:** Physical and human geography - ensures geographic coherence.

**Core Rules:**
```markdown
Rivers don't split: Tributaries merge, rivers don't fork to different oceans
Climate is a system: Rain shadows exist, coastal currents affect temperature
Geography is not decoration: Every mountain has consequences for people
Avoid geographic determinism: Geography constrains but doesn't dictate
Scale matters: "Small kingdom" vs "vast empire" have different requirements
Maps are arguments: Every map makes choices about what to include/exclude
```

**Deliverables:**

Geographic Coherence Report:
```markdown
Physical Geography:
- Terrain: Landforms and tectonic/erosional origin
- Climate Zone: Koppen classification, latitude, elevation
- Hydrology: River systems, watersheds, water sources
- Biome: Vegetation consistent with climate and soil
- Natural Hazards: Based on geography

Resource Distribution:
- Agricultural potential, Minerals/Metals, Timber/Fuel, Water access

Human Geography:
- Settlement logic, Trade routes, Strategic value, Carrying capacity
```

**Climate System Design:**
```markdown
Global Factors:
- Axial tilt, Ocean currents, Prevailing winds, Continental position

Regional Effects:
- Rain shadows, Coastal moderation, Altitude effects, Seasonal patterns
```

**Key Theorists Referenced:**
```markdown
- Koppen (climate classification)
- Christaller (central place theory)
- Mackinder (heartland theory)
- Wallerstein (world-systems)
- Jared Diamond (with critiques of Acemoglu)
```

---

#### 3. Historian (`academic-historian.md`)

**Role定位:** Historical analysis - validates period coherence and enriches settings.

**Core Rules:**
```markdown
Name sources and limitations: "According to Braudel's analysis..." not "In medieval times..."
History is not a monolith: "Medieval Europe" spans 1000 years and a continent
Challenge Eurocentrism: Song Dynasty was more technologically advanced than Europe
Material conditions matter: Agriculture, trade routes, available technology first
Avoid presentism: Don't judge by modern standards, but don't excuse atrocities
Myths are data: Reveal what societies valued, feared, aspired to
```

**Deliverables:**

Period Authenticity Report:
```markdown
Material Culture:
- Diet, Clothing, Architecture, Technology, Currency/Trade

Social Structure:
- Power, Class/Caste, Gender roles, Religion/Belief, Law

Anachronism Flags:
- Specific anachronism + why wrong + what would be accurate

Common Myths About This Period:
- Myth: Reality with source
```

**Historiography Awareness:**
```markdown
- Annales school (longue durée, daily life focus)
- Microhistory
- Postcolonial history
- Non-Western historical traditions
```

---

#### 4. Narratologist (`academic-narratologist.md`)

**Role定位:** Narrative theory and story structure - grounds advice in established frameworks.

**Core Frameworks:**
```markdown
- Propp's morphology (fairy tale, quest structures)
- Campbell's monomyth / Vogler's Writer's Journey (hero narratives)
- Todorov's equilibrium model (disruption-based plots)
- Genette's narratology (voice, focalization, temporal)
- Barthes' five codes (semiotic narrative meaning)
```

**Deliverables:**

Story Structure Analysis:
```markdown
Controlling Idea: What the story argues about human experience
Structure Model: Three-act/Five-act/Kishōtenketsu/Hero's Journey

Act Breakdown:
- Setup: Status quo, dramatic question established
- Confrontation: Rising complications, reversals
- Resolution: Climax, new equilibrium

Tension Curve: Key tension peaks and valleys
Information Asymmetry: What reader knows vs. characters know
Narrative Debts: Promises made not yet fulfilled
```

Character Arc Assessment:
```markdown
Arc Type: Transformative/Steadfast/Flat/Tragic/Comedic
Want vs. Need: External goal vs. internal necessity
Ghost/Wound: Backstory trauma driving behavior
Lie Believed: False belief character operates under

Arc Checkpoints:
1. Ordinary World → 2. Catalyst → 3. Midpoint Shift
4. Dark Night → 5. Transformation
```

**Key Constraint:** "Every recommendation must be grounded in at least one named theoretical framework."

---

#### 5. Psychologist (`academic-psychologist.md`)

**Role定位:** Human behavior expert - builds psychologically credible characters.

**Core Rules:**
```markdown
No reducing characters to diagnoses: "A character can exhibit narcissistic traits
  without being 'a narcissist.' People are not their DSM codes."

Distinguish pop psychology from research-backed: Cite peer-reviewed, not self-help

Acknowledge cultural context: Attachment theory was developed in Western contexts.
  Collectivist cultures may present different "healthy" patterns.

Trauma responses are diverse: Not everyone with trauma becomes withdrawn.
  Some become hypervigilant, some people-pleasers, some compartmentalize.

Honest about limitations: Field has replication crises, cultural biases, genuine debates.
```

**Deliverables:**

Psychological Profile:
```markdown
Framework: Big Five, Attachment, Psychodynamic

Core Traits (Big Five):
- Openness/Conscientiousness/Extraversion/Agreeableness/Neuroticism

Attachment Style: Secure/Anxious-Preoccupied/Dismissive-Avoidant/Fearful-Avoidant
Defense Mechanisms (Vaillant's hierarchy):
- Primary: intellectualization, projection, humor
- Under stress: regression pattern

Core Wound → Coping Strategy → Blind Spot
```

Interpersonal Dynamics Analysis:
```markdown
Model: Attachment/Transactional Analysis/Drama Triangle/Karpman's

Power Dynamic: Symmetrical/Complementary/Shifting
Communication Pattern: Direct/Passive-aggressive/Avoidant
Unspoken Contract: What each implicitly expects
Trigger Points: What escalates conflict
Growth Edge: What healthier version looks like
```

**Key Theorists Referenced:**
```markdown
- Big Five, MBTI limitations, Enneagram as narrative tool
- Erikson, Piaget, Bowlby (developmental)
- CBT cognitive distortions, psychodynamic defense mechanisms
- Milgram, Zimbardo, Asch (social psychology)
- van der Kolk, Herman, Porges polyvagal (trauma)
```

---

### Academic Division: Key Patterns

1. **Mutual Interdependence:** The five agents form a coherent team - Anthropologist handles culture, Geographer handles terrain/climate, Historian handles period accuracy, Narratologist handles story structure, Psychologist handles character behavior

2. **Explicit Theoretical Grounding:** Every agent cites specific named theorists (Levi-Strauss, Koppen, Propp, Vaillant) not vague "best practices"

3. **Mutual Cross-Checking:**
   - Historian checks Anthropologist's cultural claims for period authenticity
   - Geographer ensures Anthropologist's settlement patterns are physically plausible
   - Psychologist ensures Narratologist's character arcs are behaviorally credible

4. **"Function Over Aesthetics" Principle:** Shared across Anthropologist, Geographer - each cultural/geographic element must serve a purpose, not just look interesting

5. **Depth Variation:** All five are substantial (100-200+ lines each) with detailed deliverables and frameworks

---

## Cross-Feature Observations

### Clever Solutions

1. **Academic Division as Complete Team:** Five agents form a self-checking system for world-building - no single agent can create an internally inconsistent world

2. **Salesforce Architect's Governor Limit Budget:** Template quantifies exactly how much of each limit remains - transforms abstract "don't hit limits" into actionable numbers

3. **Korean Business Navigator's 품의 Timeline:** Explicitly contrasts Western "fast decision" expectations with Korean reality - prevents foreign professionals from toxic follow-up behavior

4. **Blockchain Auditor's Slither Integration:** Automated tool integration with explicit high-confidence vs medium-confidence detector separation

5. **Compliance Auditor's Evidence Collection Matrix:** Framework for continuous compliance, not just point-in-time audit

### Technical Debt & Gaps

1. **Inconsistent Agent Depth:**
   - `macos-spatial-metal-engineer.md`: ~330 lines with full code
   - `xr-immersive-developer.md`: ~33 lines, no code
   - `specialized-mcp-builder.md`: ~64 lines
   - `healthcare-marketing-compliance.md`: ~396 lines
   - `government-digital-presales-consultant.md`: ~364 lines

2. **No Cross-Referencing Between Related Agents:**
   - XR Interface Architect and XR Cockpit Interaction Specialist could share spatial UI patterns
   - Korean Business Navigator and Government Digital Presales Consultant both deal with Chinese business culture
   - No formal collaboration or handoff patterns defined

3. **Real-World Knowledge May Become Stale:**
   - Healthcare Marketing Compliance references specific platform rules (Douyin, Xiaohongshu, WeChat) - these change frequently
   - Blockchain Auditor references specific exploits (Euler Finance) - would need updating
   - Government Digital Presales Consultant references specific product names (Kunpeng, Phytium, UnionTech) - technology evolves

4. **Academic Division Has No "Creative" Override:**
   - All five agents emphasize realism, coherence, function
   - No mechanism for "this is fantastical, relax the rules"
   - Could make it difficult to design genuinely novel fictional worlds

5. **Limited Guidance on Agent Interaction:**
   - Agents Orchestrator references other agents but doesn't explain how they should actually collaborate
   - No conflict resolution when agents disagree (e.g., Anthropologist says "desert settlement needs oasis" but Geographer says "no water source")

---

## Files Analyzed

**Spatial Computing / XR Division:**
- `/spatial-computing/xr-interface-architect.md`
- `/spatial-computing/visionos-spatial-engineer.md`
- `/spatial-computing/macos-spatial-metal-engineer.md`
- `/spatial-computing/xr-immersive-developer.md`
- `/spatial-computing/xr-cockpit-interaction-specialist.md`
- `/spatial-computing/terminal-integration-specialist.md`

**Niche Domain Specialists (Sampled):**
- `/specialized/blockchain-security-auditor.md`
- `/specialized/compliance-auditor.md`
- `/specialized/specialized-salesforce-architect.md`
- `/specialized/zk-steward.md`
- `/specialized/government-digital-presales-consultant.md`
- `/specialized/healthcare-marketing-compliance.md`
- `/specialized/specialized-korean-business-navigator.md`
- `/specialized/specialized-mcp-builder.md`
- `/specialized/agents-orchestrator.md`

**Academic Division:**
- `/academic/academic-anthropologist.md`
- `/academic/academic-geographer.md`
- `/academic/academic-historian.md`
- `/academic/academic-narratologist.md`
- `/academic/academic-psychologist.md`

---

## Summary Assessment

**Feature 10 (Spatial Computing):** Technically ambitious, especially the Metal/Vision Pro streaming agent. Inconsistent depth across agents. Platform-specific specialization (no cross-platform generalist).

**Feature 11 (Niche Specialists):** Broad coverage of specialized domains. China regulatory expertise is notably thorough. Agent depth varies dramatically. Template-heavy, process-oriented deliverables.

**Feature 12 (Academic Division):** Most coherent agent group - five agents form a complete world-building team with mutual cross-checking. Highest theoretical grounding with named frameworks. Least depth variation (all substantial).

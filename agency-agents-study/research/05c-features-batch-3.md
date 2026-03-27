# Feature Batch 3 Deep Dive

**Repository:** `/Users/sheldon/Documents/claw/reference/agency-agents`
**Researcher:** Claude Code
**Date:** 2026-03-27
**Features:** Production-Ready Workflow Examples, China Market Agent Suite, Testing & QA Division

---

## Feature 7: Production-Ready Workflow Examples

**Priority:** CORE
**Key Files:** `examples/nexus-spatial-discovery.md`, `examples/workflow-startup-mvp.md`, `examples/workflow-landing-page.md`, `examples/workflow-with-memory.md`

### Overview

Pre-built multi-agent team configurations for common scenarios: Startup MVP, Marketing Campaign Launch, Enterprise Feature Development, Paid Media Account Takeover, Full Agency Product Discovery. Five workflow files exist in `examples/`.

### File Inventory

| File | Lines | Description |
|------|-------|-------------|
| `examples/nexus-spatial-discovery.md` | ~850 | 8-agent product discovery exercise (dated March 5, 2026) |
| `examples/workflow-startup-mvp.md` | ~155 | 4-week Startup MVP workflow |
| `examples/workflow-landing-page.md` | ~120 | 1-day landing page sprint |
| `examples/workflow-with-memory.md` | ~240 | MCP memory-enhanced version of startup-mvp |
| `examples/workflow-book-chapter.md` | ~60 | Book chapter writing workflow |
| `examples/README.md` | ~1 | Placeholder |

### Workflow Structure Analysis

**Pattern across all workflows:**

1. **Agent Table** at the top mapping agent names to roles
2. **Timeboxed steps** with clear agent activation prompts
3. **Sequential handoffs** where each agent's output becomes the next input
4. **Quality gates** (Reality Checker) at midpoint and before launch
5. **Parallel work** identification (independent tasks can run simultaneously)

**Startup MVP Workflow (workflow-startup-mvp.md):**

```
Week 1: Discovery + Architecture
  - Sprint Prioritizer (sequential)
  - UX Researcher (parallel with Sprint Prioritizer)
  - Backend Architect (receives Sprint Prioritizer + UX Researcher output)

Week 2: Build Core Features
  - Frontend Developer + Rapid Prototyper (parallel)
  - Reality Check at midpoint (gating quality gate)

Week 3: Polish + Landing Page
  - Frontend Developer continues
  - Growth Hacker starts

Week 4: Launch
  - Reality Checker final GO/NO-GO decision
```

**Key Pattern: Manual Context Passing**

The workflow explicitly acknowledges that agents don't share memory:
> "Always paste previous agent outputs into the next prompt -- agents don't share memory"

This is the problem that `workflow-with-memory.md` solves.

### workflow-with-memory.md: The Memory Solution

This workflow shows how to escape the copy-paste pattern using MCP memory servers. Key additions:

1. **Tag everything with the project name**: Enables recall via memory search
2. **Tag deliverables for the receiving agent**: e.g., Backend Architect tags output `frontend-developer` so Frontend Dev finds it
3. **Reality Checker gets full visibility**: Recalls all project context automatically
4. **Rollback replaces manual undo**: `Recall the Reality Checker's feedback and roll back to your last known-good schema`

### nexus-spatial-discovery.md: The Flagship Example

**Most detailed workflow in the repo.** A real exercise dated March 5, 2026 for "Nexus Spatial" -- an AI Agent Command Center in spatial computing.

**8 Agents deployed in parallel:**
- Product Trend Researcher
- Backend Architect
- Brand Guardian
- Growth Hacker
- Support Responder
- UX Researcher
- Project Shepherd
- XR Interface Architect

**10 sections of synthesis:**

| Section | Agent | Key Output |
|---------|-------|------------|
| 1. The Opportunity | (synthesis) | Concept: Nexus Spatial |
| 2. Market Validation | Product Trend Researcher | CONDITIONAL GO -- 2D-first, Spatial-second |
| 3. Technical Architecture | Backend Architect | 8-service architecture, Rust orchestration engine |
| 4. Brand Strategy | Brand Guardian | "Mission Control for the Agent Era" tagline |
| 5. Go-to-Market & Growth | Growth Hacker | 3-phase GTM, $8K-15K MRR at month 6 |
| 6. Customer Support Blueprint | Support Responder | AI-powered in-product Nexus Guide |
| 7. UX Research & Design Direction | UX Researcher | 7 design principles, 4-level semantic zoom |
| 8. Project Execution Plan | Project Shepherd | 35-week timeline, 5 squads, $121.5K-$155.5K budget |
| 9. Spatial Interface Architecture | XR Interface Architect | "Command Theater" layout, 3-layer depth system |
| 10. Cross-Agent Synthesis | (all) | 6 agreements, 5 tensions identified |

**Technical Depth in Backend Architect Section:**

```
+------------------------------------------------------------------+
|                     CLIENT TIER                                   |
|  VisionOS Native (Swift/RealityKit)  |  WebXR (React Three Fiber) |
+------------------------------------------------------------------+
                              |
+-----------------------------v------------------------------------+
|                      API GATEWAY (Kong / AWS API GW)              |
|  Rate limiting | JWT validation | WebSocket upgrade | TLS        |
+------------------------------------------------------------------+
                              |
+------------------------------------------------------------------+
|                      SERVICE TIER                                 |
|  Auth | Workspace | Workflow | Orchestration (Rust) |             |
|  Collaboration (Yjs CRDT) | Streaming (WS) | Plugin | Billing    |
+------------------------------------------------------------------+
```

**Real Market Data:**
- Vision Pro installed base: ~1M units globally, sales declined 95% from launch
- AI orchestration market: $13.5B (22.3% CAGR)
- Recommendation: "Do NOT lead with VisionOS. Lead with web, add WebXR, native VisionOS last."

**Brand Color System:**
```css
:root {
  --nxs-deep-space: #1B1F3B;
  --nxs-blue: #4A7BF7;
  --nxs-cyan: #00D4FF;
  --nxs-green: #00E676;
  --nxs-amber: #FFB300;
  --nxs-red: #FF3D71;
}
```

### workflow-landing-page.md: Minimal Example

A compressed 1-day sprint with 4 agents:

| Time | Activity | Agent |
|------|----------|-------|
| 9:00 | Copy + design kickoff | Content Creator + UI Designer (parallel) |
| 11:00 | Build starts | Frontend Developer |
| 14:00 | First version ready | -- |
| 14:30 | Conversion review | Growth Hacker |
| 15:30 | Apply feedback | Frontend Developer |
| 16:30 | Ship | Deploy to Vercel/Netlify |

### Clever Solutions

1. **Quality gates via Reality Checker**: Midpoint and final checkpoints prevent shipping broken code
2. **MCP memory integration**: Eliminates copy-paste handoffs, enables rollback
3. **Parallel kickoffs**: Independent agents start simultaneously to compress timeline
4. **Merge points made explicit**: Frontend Developer waits for both Content Creator + UI Designer outputs
5. **Timeboxing**: Each step has a clear timebox to prevent scope creep

### Technical Debt / Gaps

1. **workflow-book-chapter.md is sparse**: Only ~60 lines, much less detailed than other workflows
2. **No parameterized templates**: Each workflow is a one-off, not a reusable template
3. **Manual agent activation**: Prompts like "Activate Sprint Prioritizer" assume user copy-pastes between sessions
4. **No automation**: The workflows describe a process but don't execute it -- there's no orchestration code
5. **examples/README.md is empty**: No index or guide to help users select workflows

### Key Links

- Workflow examples reference `integrations/mcp-memory/README.md` for memory setup
- Nexus Spatial discovery references external sources (BigIdeasDB, Deloitte, G2, etc.)
- All workflows use agents from the broader agent pool (Backend Architect, Frontend Developer, etc.)

---

## Feature 8: China Market Agent Suite

**Priority:** SECONDARY
**Agent Count:** 10+
**Key Files:** `marketing/marketing-douyin-strategist.md`, `marketing/marketing-baidu-seo-specialist.md`, `marketing/marketing-xiaohongshu-specialist.md`, `marketing/marketing-china-ecommerce-operator.md`, `marketing/marketing-cross-border-ecommerce.md`, plus 5 more

### Agent Roster

| Agent | File | Color | Emoji |
|-------|------|-------|-------|
| WeChat Mini Program Developer | `engineering/engineering-wechat-mini-program-developer.md` | purple | N/A |
| Baidu SEO Specialist | `marketing/marketing-baidu-seo-specialist.md` | blue | CN |
| Xiaohongshu Specialist | `marketing/marketing-xiaohongshu-specialist.md` | #FF1B6D | flower |
| Bilibili Content Strategist | `marketing/marketing-bilibili-content-strategist.md` | N/A | N/A |
| WeChat Official Account Manager | `marketing/marketing-wechat-official-account.md` | N/A | N/A |
| Douyin Strategist | `marketing/marketing-douyin-strategist.md` | #000000 | music |
| Kuaishou Strategist | `marketing/marketing-kuaishou-strategist.md` | N/A | N/A |
| Zhihu Strategist | `marketing/marketing-zhihu-strategist.md` | N/A | N/A |
| China E-Commerce Operator | `marketing/marketing-china-ecommerce-operator.md` | red | cart |
| Cross-Border E-Commerce Specialist | `marketing/marketing-cross-border-ecommerce.md` | blue | globe |

### Douyin Strategist: Deep Dive

**Golden Rules:**

1. **Algorithm Priority Order**: completion rate > like rate > comment rate > share rate
2. **First 3 seconds decide everything** -- no buildup, lead with conflict/suspense/value
3. **No external platform links in-video** -- triggers throttling

**Viral Video Script Template:**
```
Seconds 1-3: Golden Hook (pick one)
  A. Conflict: "Never buy XXX unless you watch this first"
  B. Value: "Spent XX yuan to solve a problem that bugged me for 3 years"
  C. Suspense: "I discovered a secret the XX industry doesn't want you to know"
  D. Relatability: "Does anyone else lose it every time XXX happens?"

Seconds 4-20: Core Content
  - Amplify the pain point (2-3s)
  - Introduce the solution (3-5s)
  - Usage demo / results showcase (5-8s)
  - Key data / before-after comparison (3-5s)
```

**Livestream Product Structure:**
| Type | Share | Margin | Purpose |
|------|-------|--------|---------|
| Traffic driver | 20% | 0-10% | Build viewership |
| Profit item | 50% | 40-60% | Core revenue |
| Prestige item | 15% | 60%+ | Elevate brand |
| Flash deal | 15% | Loss-leader | Spike retention |

**Success Metrics:**
- Average video completion rate > 35%
- Organic reach per video > 10,000 views
- Livestream GPM > 500 yuan
- DOU+ ROI > 1:3
- Monthly follower growth > 15%

### Baidu SEO Specialist: Deep Dive

**Compliance-First Architecture:**

```markdown
## 基础合规 (Compliance Foundation)
- [ ] ICP备案 status: [Valid/Pending/Missing] - 备案号: [Number]
- [ ] Server location: [City, Provider] - Ping to Beijing: [ms]
- [ ] SSL certificate: [Domestic CA recommended]
- [ ] Baidu站长平台 (Webmaster Tools) verified: [Yes/No]
- [ ] Baidu Tongji (百度统计) installed: [Yes/No]
```

**Critical Rules (Non-Negotiable):**
- ICP Filing is Non-Negotiable: Sites without valid ICP备案 will be severely penalized
- China-Based Hosting: Servers must be in mainland China
- No Google Tools: Google Analytics, Fonts, reCAPTCHA are blocked -- use Baidu Tongji
- Simplified Chinese Only: For mainland China targeting

**Baidu Algorithm Mastery:**
| Algorithm | Purpose | What to Avoid |
|-----------|---------|---------------|
| 飓风 (Hurricane) | Content originality | Aggregation penalties, duplicate content |
| 细雨 (Drizzle) | B2B optimization | Keyword stuffing in titles |
| 惊雷 (Thunder) | Click manipulation | Click farms, artificial CTR |
| 蓝天 (Blue Sky) | News source quality | Editorial standards for Baidu News |
| 清风 (Breeze) | Anti-clickbait | Titles must accurately represent content |

**Cross-Search-Engine Coverage:**
- Sogou (搜狗): WeChat content integration
- 360 Search: Security-focused
- Shenma (神马): Mobile-only, Alibaba/UC Browser
- Toutiao Search: ByteDance ecosystem

### Xiaohongshu Specialist: Deep Dive

**Content Mix Mandate:**
```
70% organic lifestyle content
20% trend-participating
10% brand-direct
```

**Performance Benchmarks:**
- Engagement Rate: 5%+ target (2x Instagram due to platform culture)
- Comments Conversion: 30%+ of engagements as meaningful comments
- Share Rate: 2%+ monthly, 8%+ on viral content
- Collection Save Rate: 8%+ indicating bookmark value
- Click-Through Rate: 3%+ for CTAs

**Community Engagement Rules:**
- Post 3-5 times weekly (not oversaturated)
- Engage within 2 hours of posting for maximum visibility
- Use Xiaohongshu's native tools: collections, keywords, cross-platform promotion

### China E-Commerce Operator: Deep Dive

**Multi-Platform Dashboard:**
| Metric | Taobao/Tmall | Pinduoduo | JD | Douyin Shop |
|--------|-------------|-----------|-----|-------------|
| Monthly GMV | ¥___ | ¥___ | ¥___ | ¥___ |
| Order Volume | ___ | ___ | ___ | ___ |
| Conversion Rate | ___% | ___% | ___% | ___% |
| Store Rating | ___/5.0 | ___/5.0 | ___/5.0 | ___/5.0 |

**618 / Double 11 Campaign Battle Plan (T-60 to T+7):**

```
T-60 Days: Strategic Planning
  - Set GMV target, negotiate platform resource slots
  - Plan product lineup: 引流款 (traffic), 利润款 (profit), 活动款 (promo)
  - Confirm inventory requirements

T-30 Days: Preparation Phase
  - Finalize creative assets
  - Configure advertising campaigns (直通车, 万相台, 超级推荐)
  - Coordinate influencer seeding

T-7 Days: Warm-Up Phase (蓄水期)
  - Activate pre-sale listings
  - Ramp advertising spend
  - Push CRM messages to existing customers

T-Day: Campaign Execution (爆发期)
  - War room: real-time GMV dashboard
  - Execute hourly bid adjustments
  - Run live commerce marathon (8-12 hours)

T+1 to T+7: Post-Campaign
  - Compile performance report
  - Execute retention campaigns
```

**Advertising Stack (Taobao/Tmall):**
- 直通车 (Zhitongche): Search ads, keyword bidding, target ROAS 3:1 minimum
- 万相台 (Wanxiangtai): Smart advertising, 货品加速, 拉新快
- 超级推荐 (Super Recommendation): Feed ads for discovery traffic

### Cross-Border E-Commerce Specialist: Deep Dive

**Platform Strategy Comparison:**

| Dimension | Amazon NA | Amazon EU | Shopee SEA | TikTok Shop | Temu |
|-----------|----------|----------|------------|-------------|------|
| Core logic | Search + ads | Compliance + localization | Low price + campaigns | Content + social | Rock-bottom pricing |
| User mindset | Everything Store | Quality + fast delivery | Cheap + free shipping | Discovery shopping | Ultra-low-price |
| Logistics | FBA primary | FBA / Pan-EU | SLS / self-fulfilled | Platform logistics | Platform-fulfilled |
| Margin range | 20-35% | 15-30% | 10-25% | 15-30% | 5-15% |

**Amazon PPC Framework (3 phases):**

```
Launch Phase (Days 0-30):
  - SP Auto campaigns: 40% budget, harvest keyword data
  - SP Manual broad: 30% budget, 10-15 core keywords
  - SP Manual exact: 20% budget, 3-5 proven converting terms
  - SB Brand ads: 10% budget

Growth Phase (Days 30-90):
  - Migrate high-performing auto terms to manual
  - Add Sponsored Display competitor targeting
  - Control ACOS target < 25%

Mature Phase (90+ days):
  - Shift to exact match as primary
  - Brand defense: brand terms + competitor terms
  - Keep TACOS < 10%
```

**Compliance Red Lines:**
- Never list products without CE/FCC/FDA certifications
- VAT/Sales Tax must be filed properly
- Zero tolerance for IP infringement
- Product descriptions must be truthful

### Clever Solutions in China Market Suite

1. **Algorithm-first thinking**: Every platform agent leads with the recommendation algorithm priority order
2. **Regulatory awareness baked in**: Baidu agent leads with ICP备案 as "step zero"
3. **Campaign battle plans**: China E-Commerce operator provides T-60/T-30/T-7/T-day/T+7 structure
4. **Multi-platform comparison tables**: Cross-Border agent compares platforms across 8 dimensions
5. **Cultural authenticity emphasis**: Xiaohongshu agent explicitly requires "native-speaker-quality language"

### Technical Debt / Gaps

1. **Inconsistent depth**: Some China agents (Bilibili, Kuaishou, Zhihu) appear less detailed than Douyin/Baidu
2. **No actual Chinese content**: The agents provide frameworks but no actual Chinese copy or creative assets
3. **WeChat Mini Program Developer is engineering-focused but only one exists**: Rest are marketing
4. **Regulatory information may be outdated**: China internet regulations change frequently
5. **No integration between agents**: No workflow showing how these agents work together

### Key Links

- Baidu SEO references 百度站长平台, 百度统计, Baidu Index tools
- China E-Commerce references 生意参谋 for keyword data
- Cross-Border references Jungle Scout, Helium 10 for product selection
- All China agents reference Chinese platform-specific terminology and tools

---

## Feature 9: Testing & QA Division

**Priority:** SECONDARY
**Agent Count:** 7 (plus 1 more in the directory: testing-test-results-analyzer.md)
**Key Files:** `testing/testing-evidence-collector.md`, `testing/testing-reality-checker.md`, `testing/testing-performance-benchmarker.md`, `testing/testing-api-tester.md`, `testing/testing-accessibility-auditor.md`

### Agent Roster

| Agent | File | Color | Emoji | Focus |
|-------|------|-------|-------|-------|
| Evidence Collector | `testing-evidence-collector.md` | orange | camera | Screenshot-based QA |
| Reality Checker | `testing-reality-checker.md` | red | eye | Integration verification |
| Performance Benchmarker | `testing-performance-benchmarker.md` | orange | timer | Load/speed testing |
| API Tester | `testing-api-tester.md` | purple | plug | API validation |
| Accessibility Auditor | `testing-accessibility-auditor.md` | #0077B6 | wheelchair | WCAG compliance |
| Tool Evaluator | `testing-tool-evaluator.md` | teal | wrench | Technology assessment |
| Workflow Optimizer | `testing-workflow-optimizer.md` | green | lightning | Process improvement |
| Test Results Analyzer | `testing-test-results-analyzer.md` | N/A | N/A | (not read) |

### Evidence Collector: Deep Dive

**Core Philosophy: "Screenshots Don't Lie"**

> "Visual evidence is the only truth that matters. If you can't see it working in a screenshot, it doesn't work."

**Mandatory Process (STEP 1 always runs first):**

```bash
# 1. Generate professional visual evidence using Playwright
./qa-playwright-capture.sh http://localhost:8000 public/qa-screenshots

# 2. Check what's actually built
ls -la resources/views/ || ls -la *.html

# 3. Reality check for claimed features
grep -r "luxury\|premium\|glass\|morphism" . --include="*.html" --include="*.css" || echo "NO PREMIUM FEATURES FOUND"

# 4. Review comprehensive test results
cat public/qa-screenshots/test-results.json
```

**"AUTOMATIC FAIL" Triggers:**
- Any agent claiming "zero issues found"
- Perfect scores (A+, 98/100) on first implementation
- "Luxury/premium" claims without visual evidence
- "Production ready" without comprehensive testing evidence

**Quality Rating Scale (Honest):**
- C+ / B- / B / B+ (NO A+ fantasies)
- Design Level: Basic / Good / Excellent
- Production Readiness: FAILED / NEEDS WORK / READY (default to FAILED)

**Report Template includes:**
- Reality Check Results with specification quotes
- Visual Evidence Analysis with screenshot references
- Interactive Testing Results (accordion, form, navigation, mobile)
- Minimum 3-5 issues found (realistic assessment)

### Reality Checker: Deep Dive

**Role: Final integration testing and deployment readiness assessment**

> "You're the last line of defense against unrealistic assessments."

**Mandatory Process:**

```bash
# 1. Verify what was actually built
ls -la resources/views/ || ls -la *.html

# 2. Cross-check claimed features
grep -r "luxury\|premium\|glass\|morphism" . --include="*.html" --include="*.css" || echo "NO PREMIUM FEATURES FOUND"

# 3. Run professional Playwright screenshot capture
./qa-playwright-capture.sh http://localhost:8000 public/qa-screenshots

# 4. Review all professional-grade evidence
cat public/qa-screenshots/test-results.json
```

**Integration Testing Methodology:**

1. **Visual System Evidence**: Full system screenshots (desktop/tablet/mobile)
2. **User Journey Testing Analysis**: Homepage -> Navigation -> Contact Form with screenshot evidence
3. **Specification Reality Check**: Quote exact spec text vs. what's shown in automated screenshots

**QA Cross-Validation:**
- Review QA agent's findings
- Cross-reference automated screenshots with QA's assessment
- Verify test-results.json data matches QA's reported issues
- Confirm or challenge QA's assessment with additional evidence

### Performance Benchmarker: Deep Dive

**Core Web Vitals Targets:**
| Metric | Target | Description |
|--------|--------|-------------|
| LCP | < 2.5s | Largest Contentful Paint |
| FID | < 100ms | First Input Delay |
| CLS | < 0.1 | Cumulative Layout Shift |

**k6 Load Test Configuration Example:**

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 10 },   // Warm up
    { duration: '5m', target: 50 },   // Normal load
    { duration: '2m', target: 100 },  // Peak load
    { duration: '5m', target: 100 },  // Sustained peak
    { duration: '2m', target: 200 },  // Stress test
    { duration: '3m', target: 0 },    // Cool down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% under 500ms
    http_req_failed: ['rate<0.01'],    // Error rate under 1%
  },
};
```

**Scaling Targets:**
| Metric | Year 1 | Year 2 |
|--------|--------|--------|
| Concurrent agent executions | 5,000 | 50,000 |
| WebSocket connections | 10,000 | 100,000 |
| P95 API latency | < 150ms | < 100ms |
| P95 WS event latency | < 80ms | < 50ms |

**Custom k6 Metrics:**
```javascript
const errorRate = new Rate('errors');
const responseTimeTrend = new Trend('response_time');
const throughputCounter = new Counter('requests_per_second');
```

### API Tester: Deep Dive

**Testing Stack:** Playwright for API testing (not just browser testing)

**OWASP API Security Top 10 Coverage:**
- Broken Object Level Authorization
- Broken Authentication
- Excessive Data Exposure
- Lack of Resources & Rate Limiting
- Mass Assignment
- Security Misconfiguration
- Injection
- Improper Assets Management

**Performance SLA:**
- API response times: under 200ms for 95th percentile
- Load testing: 10x normal traffic capacity
- Error rates: below 0.1% under normal load

**Security Test Cases:**
```javascript
test('should reject requests without authentication', async () => {
  const response = await fetch(`${baseURL}/users`, { method: 'GET' });
  expect(response.status).toBe(401);
});

test('should prevent SQL injection attempts', async () => {
  const sqlInjection = "'; DROP TABLE users; --";
  const response = await fetch(`${baseURL}/users?search=${sqlInjection}`);
  expect(response.status).not.toBe(500);
});

test('should enforce rate limiting', async () => {
  const requests = Array(100).fill(null).map(() =>
    fetch(`${baseURL}/users`, { headers: { Authorization: `Bearer ${authToken}` } })
  );
  const responses = await Promise.all(requests);
  const rateLimited = responses.some(r => r.status === 429);
  expect(rateLimited).toBe(true);
});
```

### Accessibility Auditor: Deep Dive

**Standards:** WCAG 2.2 Level AA

**Key Distinction:**
> "A green Lighthouse score does not mean accessible -- say so when it applies"

> "Automated tools catch roughly 30% of accessibility issues -- you catch the other 70%"

**Testing Protocol:**

```
Automated Scanning (30% of issues):
  - axe-core against all pages
  - Lighthouse accessibility audit

Manual Testing (70% of issues):
  - Screen reader testing (VoiceOver, NVDA, JAWS)
  - Keyboard-only navigation
  - Voice control compatibility
  - Screen magnification at 200%/400%
  - Reduced motion, high contrast, forced colors modes
```

**WCAG 2.2 Specific Requirements:**

| Criterion | Name | Level |
|-----------|------|-------|
| 1.4.3 | Contrast Minimum | AA |
| 2.1.1 | Keyboard | A |
| 2.4.7 | Focus Visible | AA |
| 3.1.1 | Language of Page | A |
| 4.1.2 | Name, Role, Value | A |

**Screen Reader Testing Protocol:**
```markdown
## Navigation Testing
- Heading Structure: h1 -> h2 -> h3 logical hierarchy?
- Landmark Regions: main, nav, banner, contentinfo present?
- Skip Links: Can users skip to main content?
- Tab Order: Does focus move in logical sequence?
- Focus Visibility: Is focus indicator always visible?

## Interactive Component Testing
- Buttons: Announced with role and label? State changes?
- Links: Distinguishable from buttons? Destination clear?
- Forms: Labels associated? Required fields announced?
- Modals: Focus trapped? Escape closes? Focus returns?
```

### Tool Evaluator: Deep Dive

**Weighted Evaluation Criteria:**
| Criterion | Weight | Description |
|-----------|--------|-------------|
| Functionality | 0.25 | Core feature completeness |
| Usability | 0.20 | User experience and ease of use |
| Performance | 0.15 | Speed, reliability, scalability |
| Security | 0.15 | Data protection and compliance |
| Integration | 0.10 | API quality and system compatibility |
| Support | 0.08 | Vendor support quality and documentation |
| Cost | 0.07 | Total cost of ownership and value |

**Python Data Classes for Evaluation:**

```python
@dataclass
class EvaluationCriteria:
    name: str
    weight: float  # 0-1 importance weight
    max_score: int = 10
    description: str = ""

@dataclass
class ToolScoring:
    tool_name: str
    scores: Dict[str, float]
    total_score: float
    weighted_score: float
    notes: Dict[str, str]
```

**Total Cost of Ownership Calculation:**
```python
costs = {
    "licensing": annual_license_cost * years,
    "implementation": implementation_cost,
    "training": training_cost,
    "maintenance": annual_maintenance_cost * years,
    "integration": integration_cost,
    "migration": migration_cost,
    "support": annual_support_cost * years,
}
```

### Workflow Optimizer: Deep Dive

**Process Improvement Framework:** Lean + Six Sigma

**Metrics Tracked:**
- Total cycle time
- Active work time
- Wait time
- Cost per execution
- Error rate
- Throughput per day
- Employee satisfaction

**Python Data Classes:**

```python
@dataclass
class ProcessStep:
    name: str
    duration_minutes: float
    cost_per_hour: float
    error_rate: float
    automation_potential: float  # 0-1 scale
    bottleneck_severity: int     # 1-5 scale
    user_satisfaction: float     # 1-10 scale
```

**Optimization Opportunity Types:**
```python
if step.error_rate > 0.05:  # >5% error rate
    opportunities.append({"type": "quality_improvement", ...})

if step.bottleneck_severity >= 4:
    opportunities.append({"type": "bottleneck_resolution", ...})

if step.automation_potential > 0.7:
    opportunities.append({"type": "automation", ...})
```

**Implementation Phases:**
- Quick Wins: 4 weeks
- Medium Term: 12 weeks
- Strategic: 26 weeks

### Clever Solutions in Testing & QA Division

1. **Evidence-based verification**: Evidence Collector requires screenshots as ground truth, not claims
2. **Fantasy reporting anti-pattern**: Explicitly calls out "zero issues found" as a red flag
3. **Honest quality ratings**: Uses C+/B-/B/B+ scale instead of inflated A+ grades
4. **Multi-layer verification**: Reality Checker cross-validates QA findings, not just accepting them
5. **p95/p99 metrics**: Performance Benchmarker uses percentiles, not averages
6. **OWASP coverage**: API Tester explicitly maps to OWASP API Security Top 10
7. **Automation vs. manual split**: Accessibility Auditor explicitly states automation catches only ~30%

### Technical Debt / Gaps

1. **qa-playwright-capture.sh referenced but doesn't exist in repo**: Evidence Collector and Reality Checker both reference this script which isn't in the repository
2. **ai/agents/qa.md referenced but doesn't exist**: Evidence Collector references `ai/agents/qa.md` for complete testing protocols
3. **No actual test code files**: All testing agents provide frameworks and templates, but no actual test code is shipped
4. **No CI/CD integration**: No GitHub Actions workflows or scripts for running these QA processes
5. **testing-test-results-analyzer.md not reviewed**: 8th testing agent exists but wasn't read

### Key Links

- Evidence Collector -> Reality Checker: QA findings flow between them
- Reality Checker references `integrations/mcp-memory/README.md` for memory setup
- Performance Benchmarker uses k6 for load testing
- API Tester uses Playwright for API testing
- Accessibility Auditor uses axe-core + Lighthouse

---

## Cross-Feature Observations

### Shared Anti-Patterns

1. **Referenced files that don't exist**: Multiple agents reference scripts or files not in the repo (`qa-playwright-capture.sh`, `ai/agents/qa.md`, `integrations/mcp-memory/README.md`)

2. **No actual executable code**: All workflows and agents are markdown definitions -- there's no actual code to execute them. The repo is a knowledge base, not a working system.

3. **Manual handoff assumption**: Most workflows assume a human will copy-paste between agents, with no orchestration layer

### Shared Strengths

1. **Specific, measurable metrics**: Every agent has quantifiable success criteria (completion rates, ROAS, latency targets)

2. **Platform/tool specificity**: Agents know specific tools, APIs, and platforms (Douyin algorithm, Baidu crawler, k6, Playwright)

3. **Evidence-based philosophy**: Testing and QA agents emphasize proof over claims

4. **Regulatory/compliance awareness**: China market agents and testing agents both emphasize compliance requirements

### Unexpected Findings

1. **Testing agents have the most code**: The Performance Benchmarker, API Tester, Tool Evaluator, and Workflow Optimizer all include substantial Python/JavaScript code blocks. This is unusual -- most agents in the repo are personality definitions without code.

2. **Quality rating honesty**: The Testing division explicitly rejects "A+" grades and defaults to "FAILED" -- a refreshingly skeptical approach

3. **nexus-spatial-discovery.md is a real document**: Dated March 5, 2026, it reads like an actual agency deliverable, not a template

4. **China e-commerce coverage is surprisingly deep**: The T-60 campaign battle plan structure is detailed and realistic

---

## Files Read for This Analysis

1. `examples/nexus-spatial-discovery.md`
2. `examples/workflow-startup-mvp.md`
3. `examples/workflow-with-memory.md`
4. `examples/workflow-landing-page.md`
5. `marketing/marketing-douyin-strategist.md`
6. `marketing/marketing-baidu-seo-specialist.md`
7. `marketing/marketing-xiaohongshu-specialist.md`
8. `marketing/marketing-china-ecommerce-operator.md`
9. `marketing/marketing-cross-border-ecommerce.md`
10. `testing/testing-evidence-collector.md`
11. `testing/testing-reality-checker.md`
12. `testing/testing-performance-benchmarker.md`
13. `testing/testing-api-tester.md`
14. `testing/testing-accessibility-auditor.md`
15. `testing/testing-tool-evaluator.md`
16. `testing/testing-workflow-optimizer.md`

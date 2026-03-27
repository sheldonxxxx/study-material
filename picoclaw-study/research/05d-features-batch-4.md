# PicoClaw Feature Deep Dive - Batch 4

**Date:** 2026-03-26
**Features:** Skills System, Cron Scheduling, Web Search Tools
**Repo:** `/Users/sheldon/Documents/claw/reference/picoclaw`

---

## Feature 10: Skills System

### Overview

The Skills System provides modular capability extensions loaded from `SKILL.md` files. Skills can be installed from registries (primarily ClawHub) or created locally, and are loaded into the agent's context at runtime.

### Architecture

```
pkg/skills/
  service.go      - SkillsLoader: loads skills from filesystem
  installer.go     - SkillInstaller: GitHub-based skill installation
  registry.go     - RegistryManager: multi-registry search with concurrency control
  clawhub_registry.go - ClawHub registry implementation
  search_cache.go - Trigram-based search result caching

pkg/tools/
  skills_install.go - InstallSkillTool (LLM tool interface)
  skills_search.go  - FindSkillsTool (LLM tool interface)
```

### Core Data Structures

**pkg/skills/service.go (lines 28-59)**
```go
type SkillMetadata struct {
    Name        string `json:"name"`
    Description string `json:"description"`
}

type SkillInfo struct {
    Name        string `json:"name"`
    Path        string `json:"path"`
    Source      string `json:"source"`  // "workspace", "global", "builtin"
    Description string `json:"description"`
}
```

**pkg/skills/registry.go (lines 16-58)**
```go
type SearchResult struct {
    Score        float64 `json:"score"`
    Slug         string  `json:"slug"`
    DisplayName  string  `json:"display_name"`
    Summary      string  `json:"summary"`
    Version      string  `json:"version"`
    RegistryName string  `json:"registry_name"`
}

type SkillMeta struct {
    Slug             string `json:"slug"`
    DisplayName      string `json:"display_name"`
    Summary          string `json:"summary"`
    LatestVersion    string `json:"latest_version"`
    IsMalwareBlocked bool   `json:"is_malware_blocked"`
    IsSuspicious     bool   `json:"is_suspicious"`
    RegistryName     string `json:"registry_name"`
}

type SkillRegistry interface {
    Name() string
    Search(ctx context.Context, query string, limit int) ([]SearchResult, error)
    GetSkillMeta(ctx context.Context, slug string) (*SkillMeta, error)
    DownloadAndInstall(ctx context.Context, slug, version, targetDir string) (*InstallResult, error)
}
```

### SkillsLoader: Multi-Source Skill Loading

**pkg/skills/service.go (lines 61-192)**

The `SkillsLoader` implements a priority system for loading skills:

```go
func (sl *SkillsLoader) SkillRoots() []string {
    roots := []string{sl.workspaceSkills, sl.globalSkills, sl.builtinSkills}
    // Deduplication and cleanup
}

func (sl *SkillsLoader) LoadSkill(name string) (string, bool) {
    // Priority: workspace > global > builtin
    if sl.workspaceSkills != "" {
        skillFile := filepath.Join(sl.workspaceSkills, name, "SKILL.md")
        if content, err := os.ReadFile(skillFile); err == nil {
            return sl.stripFrontmatter(string(content)), true
        }
    }
    // ... similar for global, then builtin
}
```

**Skill loading priority (lines 142-145):**
1. `workspace/skills/{name}/SKILL.md` - Project-level skills
2. `~/.picoclaw/skills/{name}/SKILL.md` - Global skills
3. Builtin skills from embedded assets

**Metadata extraction (lines 219-270):**
- Supports YAML frontmatter: `{"name": "...", "description": "..."}`
- Falls back to JSON frontmatter for backward compatibility
- Falls back to parsing H1 title and first paragraph from markdown body

```go
func (sl *SkillsLoader) getSkillMetadata(skillPath string) *SkillMetadata {
    frontmatter, bodyContent := splitFrontmatter(string(content))
    title, bodyDescription := extractMarkdownMetadata(bodyContent)

    // Try JSON first, then simple YAML
    var jsonMeta struct {
        Name        string `json:"name"`
        Description string `json:"description"`
    }
    if err := json.Unmarshal([]byte(frontmatter), &jsonMeta); err == nil {
        // use JSON
    }
    // Fall back to YAML
}
```

### Registry Manager: Concurrent Multi-Registry Search

**pkg/skills/registry.go (lines 80-210)**

```go
type RegistryManager struct {
    registries    []SkillRegistry
    maxConcurrent int    // Default: 2
    mu            sync.RWMutex
}

func (rm *RegistryManager) SearchAll(ctx context.Context, query string, limit int) ([]SearchResult, error) {
    // Fan-out to all registries with semaphore-limited concurrency
    sem := make(chan struct{}, rm.maxConcurrent)
    resultsCh := make(chan regResult, len(regs))

    // Each registry search runs with 1-minute timeout
    searchCtx, cancel := context.WithTimeout(ctx, 1*time.Minute)

    // Results merged and sorted by score descending (insertion sort)
    sortByScoreDesc(merged)

    // Clamp to limit
    if limit > 0 && len(merged) > limit {
        merged = merged[:limit]
    }
}
```

**Key design decisions:**
- Concurrent search with configurable max parallelism (default 2)
- 1-minute timeout per registry search
- Graceful degradation: if ALL registries fail, returns last error
- Results sorted by relevance score

### Search Cache: Trigram-Based Similarity

**pkg/skills/search_cache.go**

Implements a smart cache that recognizes similar queries:

```go
const similarityThreshold = 0.7  // Jaccard similarity threshold

func (sc *SearchCache) Get(query string) ([]SearchResult, bool) {
    normalized := normalizeQuery(query)  // lowercase + trim

    // Exact match first
    if entry, ok := sc.entries[normalized]; ok {
        if time.Since(entry.createdAt) < sc.ttl {
            sc.moveToEndLocked(normalized)  // LRU update
            return copyResults(entry.results), true
        }
    }

    // Similarity match using trigram Jaccard
    queryTrigrams := buildTrigrams(normalized)
    sim := jaccardSimilarity(queryTrigrams, entry.trigrams)
    if sim >= similarityThreshold {
        return copyResults(entry.results), true
    }
}
```

**Trigram implementation (lines 172-196):**
```go
func buildTrigrams(s string) []uint32 {
    // "hello" → {"hel", "ell", "llo"}
    // Each trigram packed into 3 bytes as uint32
    trigrams := make([]uint32, 0, len(s)-2)
    for i := 0; i <= len(s)-3; i++ {
        trigrams[i] = uint32(s[i])<<16 | uint32(s[i+1])<<8 | uint32(s[i+2])
    }
    slices.Sort(trigrams)
    // Deduplicate
}
```

**Cache eviction:**
- LRU order with max entries (default 50)
- TTL-based expiration (default 5 minutes)
- Exact match hit moves entry to end of LRU list

### ClawHub Registry

**pkg/skills/clawhub_registry.go**

```go
type ClawHubRegistry struct {
    baseURL         string  // https://clawhub.ai
    authToken       string  // Optional auth for rate limits
    searchPath      string  // /api/v1/search
    skillsPath      string  // /api/v1/skills
    downloadPath    string  // /api/v1/download
    maxZipSize      int     // 50 MB default
    maxResponseSize int     // 2 MB default
    client          *http.Client
}
```

**Download with retry (lines 315-362):**
```go
func (c *ClawHubRegistry) downloadToTempFileWithRetry(ctx context.Context, urlStr string) (string, error) {
    resp, err := utils.DoRequestWithRetry(c.client, req)

    tmpFile, err := os.CreateTemp("", "picoclaw-dl-*")
    // Stream in ~32KB chunks with size limit
    src := io.LimitReader(resp.Body, int64(c.maxZipSize)+1)
    written, err := io.Copy(tmpFile, src)
}
```

### InstallSkillTool

**pkg/tools/skills_install.go (lines 71-176)**

```go
func (t *InstallSkillTool) Execute(ctx context.Context, args map[string]any) *ToolResult {
    t.mu.Lock()  // Workspace-level lock to prevent concurrent installs
    defer t.mu.Unlock()

    // Validate slug and registry identifiers
    slug := utils.ValidateSkillIdentifier(slug)

    // Check if already installed (unless force=true)
    if !force {
        if _, err := os.Stat(targetDir); err == nil {
            return ErrorResult("already installed. Use force=true to reinstall.")
        }
    }

    // Download and install
    result, err := registry.DownloadAndInstall(ctx, slug, version, targetDir)

    // Moderation check
    if result.IsMalwareBlocked {
        os.RemoveAll(targetDir)
        return ErrorResult("skill is flagged as malicious")
    }

    // Write origin metadata
    writeOriginMeta(targetDir, registry.Name(), slug, result.Version)
}
```

**Malware detection flow:**
1. Fetch metadata (moderation flags) before download
2. Block installation if `IsMalwareBlocked = true`
3. Warn if `IsSuspicious = true` but allow installation
4. Clean up partial installs on failure

### FindSkillsTool

**pkg/tools/skills_search.go (lines 54-88)**

```go
func (t *FindSkillsTool) Execute(ctx context.Context, args map[string]any) *ToolResult {
    query := strings.ToLower(strings.TrimSpace(query))
    limit := 5  // Default

    // Check cache first
    if cached, hit := t.cache.Get(query); hit {
        return SilentResult(formatSearchResults(query, cached, true))  // Mark as cached
    }

    // Search all registries
    results, err := t.registryMgr.SearchAll(ctx, query, limit)

    // Cache results
    t.cache.Put(query, results)

    return SilentResult(formatSearchResults(query, results, false))
}
```

### Clever Solutions

1. **Atomic file writes** (`fileutil.WriteFileAtomic`): All skill metadata and cron state use atomic writes with explicit fsync for flash storage reliability.

2. **Trigram-based similarity caching**: Avoids redundant API calls for semantically similar queries (e.g., "github integration" vs "github tools").

3. **Multi-source priority loading**: Workspace skills take precedence, allowing project-specific overrides of global/builtin skills.

4. **Graceful degradation**: If all registries fail, returns partial results rather than complete failure.

### Technical Debt / Concerns

1. **Workspace-level install lock**: The mutex in `InstallSkillTool` locks the entire workspace during install, not per-skill. Concurrent installs of different skills could conflict.

2. **No version pinning in origin metadata**: The `.skill-origin.json` stores version but doesn't enforce it on subsequent loads.

3. **GitHub token in plain text**: `SkillInstaller` accepts `githubToken` as a string parameter, which may end up in logs or memory dumps.

---

## Feature 11: Cron Scheduling

### Overview

Built-in scheduling for reminders and recurring tasks using natural language ("remind me in 10 minutes"), interval-based scheduling ("every 2 hours"), or cron expressions ("0 9 * * *" for daily at 9am).

### Architecture

```
pkg/cron/
  service.go      - CronService: core scheduling engine

pkg/tools/
  cron.go         - CronTool: LLM tool interface
```

### Core Data Structures

**pkg/cron/service.go (lines 18-70)**

```go
type CronSchedule struct {
    Kind    string `json:"kind"`  // "at", "every", "cron"
    AtMS    *int64 `json:"atMs,omitempty"`
    EveryMS *int64 `json:"everyMs,omitempty"`
    Expr    string `json:"expr,omitempty"`
    TZ      string `json:"tz,omitempty"`
}

type CronPayload struct {
    Kind    string `json:"kind"`
    Message string `json:"message"`
    Command string `json:"command,omitempty"`  // Shell command to execute
    Deliver bool   `json:"deliver"`           // Direct delivery vs agent processing
    Channel string `json:"channel,omitempty"`
    To      string `json:"to,omitempty"`
}

type CronJobState struct {
    NextRunAtMS *int64 `json:"nextRunAtMs,omitempty"`
    LastRunAtMS *int64 `json:"lastRunAtMs,omitempty"`
    LastStatus  string `json:"lastStatus,omitempty"`
    LastError   string `json:"lastError,omitempty"`
}

type CronJob struct {
    ID             string       `json:"id"`
    Name           string       `json:"name"`
    Enabled        bool         `json:"enabled"`
    Schedule       CronSchedule `json:"schedule"`
    Payload        CronPayload  `json:"payload"`
    State          CronJobState `json:"state"`
    CreatedAtMS    int64        `json:"createdAtMs"`
    UpdatedAtMS    int64        `json:"updatedAtMs"`
    DeleteAfterRun bool         `json:"deleteAfterRun"`  // True for "at" (one-time) jobs
}
```

### CronService: Event Loop Implementation

**pkg/cron/service.go (lines 84-171)**

```go
func (cs *CronService) Start() error {
    cs.loadStore()
    cs.recomputeNextRuns()
    cs.saveStoreUnsafe()
    cs.stopChan = make(chan struct{})
    cs.running = true
    go cs.runLoop(cs.stopChan)
}

func (cs *CronService) runLoop(stopChan chan struct{}) {
    timer := time.NewTimer(time.Hour)
    defer timer.Stop()

    for {
        cs.mu.RLock()
        nextWake := cs.getNextWakeMS()  // Find earliest job
        cs.mu.RUnlock()

        var delay time.Duration
        now := time.Now().UnixMilli()

        if nextWake == nil {
            delay = time.Hour  // No jobs, sleep long
        } else {
            diff := *nextWake - now
            if diff <= 0 {
                delay = 0  // Due now
            } else {
                delay = time.Duration(diff) * time.Millisecond
            }
        }

        timer.Reset(delay)

        select {
        case <-stopChan:
            return
        case <-cs.wakeChan:  // Wake on new job
            if !timer.Stop() {
                <-timer.C  // Drain
            }
            continue
        case <-timer.C:
            cs.checkJobs()
        }
    }
}
```

**Wake channel pattern (lines 339-345):**
```go
func (cs *CronService) notify() {
    select {
    case cs.wakeChan <- struct{}{}:
    default:  // Non-blocking; if full, loop will wake soon anyway
    }
}
```

### Job Execution Flow

**pkg/cron/service.go (lines 173-302)**

```go
func (cs *CronService) checkJobs() {
    cs.mu.Lock()

    now := time.Now().UnixMilli()
    var dueJobIDs []string

    // Collect due jobs
    for i := range cs.store.Jobs {
        job := &cs.store.Jobs[i]
        if job.Enabled && job.State.NextRunAtMS != nil && *job.State.NextRunAtMS <= now {
            dueJobIDs = append(dueJobIDs, job.ID)
        }
    }

    // Reset NextRunAtMS before unlocking to prevent duplicate execution
    for i := range cs.store.Jobs {
        if dueMap[cs.store.Jobs[i].ID] {
            cs.store.Jobs[i].State.NextRunAtMS = nil
        }
    }

    cs.saveStoreUnsafe()
    cs.mu.Unlock()

    // Execute outside lock
    for _, jobID := range dueJobIDs {
        cs.executeJobByID(jobID)
    }
}
```

**Pre-reset pattern prevents double execution:**
By clearing `NextRunAtMS` before releasing the lock, a new job added during execution won't be picked up by the currently-running check.

### Schedule Computation

**pkg/cron/service.go (lines 304-336)**

```go
func (cs *CronService) computeNextRun(schedule *CronSchedule, nowMS int64) *int64 {
    switch schedule.Kind {
    case "at":
        if schedule.AtMS != nil && *schedule.AtMS > nowMS {
            return schedule.AtMS
        }
        return nil  // Past time = don't run
    case "every":
        if schedule.EveryMS == nil || *schedule.EveryMS <= 0 {
            return nil
        }
        next := nowMS + *schedule.EveryMS
        return &next
    case "cron":
        // Uses gronx library for cron expression parsing
        now := time.UnixMilli(nowMS)
        nextTime, err := gronx.NextTickAfter(schedule.Expr, now, false)
        if err != nil {
            return nil
        }
        return &nextTime
    }
}
```

### CronTool: LLM Interface

**pkg/tools/cron.go**

```go
func (t *CronTool) Parameters() map[string]any {
    return map[string]any{
        "type": "object",
        "properties": map[string]any{
            "action": map[string]any{
                "enum": []string{"add", "list", "remove", "enable", "disable"},
            },
            "message": map[string]any{"type": "string"},
            "command": map[string]any{"type": "string"},  // Shell command
            "at_seconds": map[string]any{"type": "integer"},   // One-time (e.g., 600 = 10 min)
            "every_seconds": map[string]any{"type": "integer"}, // Recurring interval
            "cron_expr": map[string]any{"type": "string"},     // Cron expression
            "deliver": map[string]any{"type": "boolean"},       // Direct vs agent
        },
        "required": []string{"action"},
    }
}
```

**Type assertion fix (lines 166-169):**
```go
// Fix: type assertions return true for zero values
hasAt = hasAt && atSeconds > 0   // Prevent LLMs filling 0 for unused params
hasEvery = hasEvery && everySeconds > 0
```

**Command scheduling security (lines 200-216):**
```go
// GHSA-pv8c-p6jf-3fpp mitigation
command, _ := args["command"].(string)
commandConfirm, _ := args["command_confirm"].(bool)
if command != "" {
    if !t.execEnabled {
        return ErrorResult("command execution is disabled")
    }
    if !constants.IsInternalChannel(channel) {
        return ErrorResult("scheduling command execution is restricted to internal channels")
    }
    if !t.allowCommand && !commandConfirm {
        return ErrorResult("command_confirm=true is required when allow_command is disabled")
    }
    deliver = false  // Force agent processing
}
```

### Job Execution Methods

**pkg/tools/cron.go (lines 298-380)**

```go
func (t *CronTool) ExecuteJob(ctx context.Context, job *cron.CronJob) string {
    channel := job.Payload.Channel
    chatID := job.Payload.To

    // 1. Command execution
    if job.Payload.Command != "" {
        result := t.execTool.Execute(ctx, args)
        t.msgBus.PublishOutbound(pubCtx, bus.OutboundMessage{
            Channel: channel, ChatID: chatID, Content: output,
        })
        return "ok"
    }

    // 2. Direct delivery (no agent)
    if job.Payload.Deliver {
        t.msgBus.PublishOutbound(pubCtx, bus.OutboundMessage{
            Channel: channel, ChatID: chatID, Content: job.Payload.Message,
        })
        return "ok"
    }

    // 3. Agent processing (for complex tasks)
    sessionKey := fmt.Sprintf("cron-%s", job.ID)
    response, err := t.executor.ProcessDirectWithChannel(ctx, job.Payload.Message, sessionKey, channel, chatID)
    return "ok"
}
```

### Persistence

**pkg/cron/service.go (lines 381-406)**

```go
func (cs *CronService) loadStore() error {
    data, err := os.ReadFile(cs.storePath)
    if os.IsNotExist(err) {
        return nil  // Empty store is fine
    }
    return json.Unmarshal(data, cs.store)
}

func (cs *CronService) saveStoreUnsafe() error {
    data, err := json.MarshalIndent(cs.store, "", "  ")
    // Atomic write with fsync for flash storage
    return fileutil.WriteFileAtomic(cs.storePath, data, 0o600)
}
```

### Clever Solutions

1. **Wake channel for immediate reschedule**: Adding a new job signals `wakeChan` to immediately recalculate the next wake time instead of waiting for the timer.

2. **Pre-reset pattern**: Clearing `NextRunAtMS` before releasing the lock prevents the same job from being picked up twice if a new check starts while execution is ongoing.

3. **Non-blocking notify**: Using `default` case in `notify()` avoids blocking if the loop is already about to wake up.

4. **Atomic writes with fsync**: All state changes are persisted atomically with explicit sync for flash storage reliability.

5. **ID generation using crypto/rand**: Uses `crypto/rand` for unique job IDs with nanosecond fallback.

### Technical Debt / Concerns

1. **Goron library cron parsing**: Uses `adhocore/gronx` for cron expression parsing. If this library has bugs, cron expressions may misfire.

2. **No distributed lock**: In a multi-instance deployment, multiple instances would each run their own cron loop. No leader election or distributed locking.

3. **In-memory job list**: While the store is persisted to disk, the in-memory `[]CronJob` slice is the source of truth during runtime. A corrupted store reload could cause issues.

4. **Command output truncation**: No visible limit on command output size before publishing to message bus.

---

## Feature 12: Web Search Tools

### Overview

Integrated web search via multiple engines (DuckDuckGo built-in, Tavily, Brave, Perplexity, SearXNG, Baidu, GLM) with a unified `SearchProvider` interface and a companion `WebFetchTool` for content retrieval.

### Architecture

```
pkg/tools/
  web.go          - WebSearchTool, WebFetchTool, and all SearchProvider implementations
  search_tool.go   - Internal tool discovery (RegexSearchTool, BM25SearchTool)
```

### Search Provider Interface

**pkg/tools/web.go (lines 88-90)**

```go
type SearchProvider interface {
    Search(ctx context.Context, query string, count int) (string, error)
}
```

### Provider Priority Chain

**pkg/tools/web.go (lines 730-825) - `NewWebSearchTool`**

```go
func NewWebSearchTool(opts WebSearchToolOptions) (*WebSearchTool, error) {
    // Priority: Perplexity > Brave > SearXNG > Tavily > DuckDuckGo > Baidu > GLM
    if opts.PerplexityEnabled && len(opts.PerplexityAPIKeys) > 0 {
        provider = &PerplexitySearchProvider{...}
    } else if opts.BraveEnabled && len(opts.BraveAPIKeys) > 0 {
        provider = &BraveSearchProvider{...}
    } else if opts.SearXNGEnabled && opts.SearXNGBaseURL != "" {
        provider = &SearXNGSearchProvider{...}
    } else if opts.TavilyEnabled && len(opts.TavilyAPIKeys) > 0 {
        provider = &TavilySearchProvider{...}
    } else if opts.DuckDuckGoEnabled {
        provider = &DuckDuckGoSearchProvider{...}
    } else if opts.BaiduSearchEnabled && opts.BaiduSearchAPIKey != "" {
        provider = &BaiduSearchProvider{...}
    } else if opts.GLMSearchEnabled && opts.GLMSearchAPIKey != "" {
        provider = &GLMSearchProvider{...}
    }
}
```

**DuckDuckGo is the built-in free option** (no API key required).

### DuckDuckGo Provider

**pkg/tools/web.go (lines 287-376)**

Uses HTML scraping with regex extraction:

```go
type DuckDuckGoSearchProvider struct {
    proxy  string
    client *http.Client
}

func (p *DuckDuckGoSearchProvider) Search(ctx context.Context, query string, count int) (string, error) {
    searchURL := fmt.Sprintf("https://html.duckduckgo.com/html/?q=%s", url.QueryEscape(query))
    req.Header.Set("User-Agent", userAgent)
    resp, err := p.client.Do(req)
    return p.extractResults(string(body), count, query)
}

func (p *DuckDuckGoSearchProvider) extractResults(html string, count int, query string) (string, error) {
    // Regex patterns for result extraction
    matches := reDDGLink.FindAllStringSubmatch(html, count+5)  // Links
    snippetMatches := reDDGSnippet.FindAllStringSubmatch(html, count+5)  // Snippets
}
```

**Pre-compiled regexes (lines 45-48):**
```go
var (
    reDDGLink    = regexp.MustCompile(`<a[^>]*class="[^"]*result__a[^"]*"[^>]*href="([^"]+)"[^>]*>([\s\S]*?)</a>`)
    reDDGSnippet = regexp.MustCompile(`<a class="result__snippet[^"]*".*?>([\s\S]*?)</a>`)
)
```

### API Key Pool: Round-Robin with Failover

**pkg/tools/web.go (lines 50-86)**

```go
type APIKeyPool struct {
    keys    []string
    current uint32  // Atomic counter
}

func (p *APIKeyPool) NewIterator() *APIKeyIterator {
    idx := atomic.AddUint32(&p.current, 1) - 1
    return &APIKeyIterator{startIdx: idx}
}

func (it *APIKeyIterator) Next() (string, bool) {
    length := uint32(len(it.pool.keys))
    if length == 0 || it.attempt >= length {
        return "", false
    }
    key := it.pool.keys[(it.startIdx+it.attempt)%length]
    it.attempt++
    return key, true
}
```

**Usage in providers:**
```go
func (p *BraveSearchProvider) Search(ctx context.Context, query string, count int) (string, error) {
    iter := p.keyPool.NewIterator()
    for {
        apiKey, ok := iter.Next()
        if !ok {
            break  // All keys exhausted
        }
        // Try with apiKey...
        if resp.StatusCode == http.StatusOK {
            return result, nil
        }
        // Retry on rate limit, auth failure, or server error
        if resp.StatusCode == http.StatusTooManyRequests || resp.StatusCode >= 500 {
            continue  // Try next key
        }
    }
    return "", fmt.Errorf("all api keys failed")
}
```

### WebFetchTool: SSRF Protection

**pkg/tools/web.go (lines 878-1162)**

Comprehensive Server-Side Request Forgery protection:

**1. Lightweight pre-flight (lines 1003-1008):**
```go
hostname := parsedURL.Hostname()
if isObviousPrivateHost(hostname, t.whitelist) {
    return ErrorResult("fetching private or local network hosts is not allowed")
}
```

**2. DNS rebinding mitigation (lines 1197-1250):**
```go
func newSafeDialContext(dialer *net.Dialer, whitelist *privateHostWhitelist) func(...) {
    return func(ctx context.Context, network, address string) (net.Conn, error) {
        // Resolve DNS at connect time (not pre-flight)
        ipAddrs, err := net.DefaultResolver.LookupIPAddr(ctx, host)
        for _, ipAddr := range ipAddrs {
            if shouldBlockPrivateIP(ipAddr.IP, whitelist) {
                continue  // Skip private IPs
            }
            conn, err := dialer.DialContext(ctx, network, ...)
            if err == nil {
                return conn, nil
            }
        }
    }
}
```

**3. Private IP blocking (lines 1340-1384):**
```go
func isPrivateOrRestrictedIP(ip net.IP) bool {
    if ip.IsLoopback() || ip.IsLinkLocalUnicast() || ip.IsMulticast() || ip.IsUnspecified() {
        return true
    }
    if ip4 := ip.To4(); ip4 != nil {
        // RFC 1918: 10.x.x.x, 172.16-31.x.x, 192.168.x.x
        // Link-local: 169.254.x.x
        // Carrier-grade NAT: 100.64-127.x.x
    }
    // IPv6: unique-local (fc00::/7), 6to4 (2002::/16), Teredo (2001:0000::/32)
}
```

**4. Redirect following with SSRF check (lines 936-944):**
```go
client.CheckRedirect = func(req *http.Request, via []*http.Request) error {
    if len(via) >= maxRedirects {
        return fmt.Errorf("stopped after %d redirects", maxRedirects)
    }
    if isObviousPrivateHost(req.URL.Hostname(), whitelist) {
        return fmt.Errorf("redirect target is private or local network host")
    }
    return nil
}
```

### Content Extraction

**pkg/tools/web.go (lines 1096-1134)**

```go
switch {
case mediaType == "application/json":
    // Pretty-print JSON
    formatted, _ := json.MarshalIndent(jsonData, "", "  ")
    text = string(formatted)

case mediaType == "text/html" || looksLikeHTML(bodyStr):
    switch strings.ToLower(t.format) {
    case "markdown":
        text, err = utils.HtmlToMarkdown(bodyStr)
    default:
        text = t.extractText(bodyStr)  // Strip scripts, styles, tags
    }
}
```

**HTML text extraction (lines 1175-1195):**
```go
func (t *WebFetchTool) extractText(htmlContent string) string {
    result := reScript.ReplaceAllLiteralString(htmlContent, "")   // Remove JS
    result = reStyle.ReplaceAllLiteralString(result, "")           // Remove CSS
    result = reTags.ReplaceAllLiteralString(result, "")             // Strip HTML tags
    result = reWhitespace.ReplaceAllString(result, " ")            // Normalize whitespace
    result = reBlankLines.ReplaceAllString(result, "\n\n")         // Remove blank lines
    // Split, trim, filter empty lines
}
```

### Cloudflare Bypass

**pkg/tools/web.go (lines 1046-1069)**

```go
// Cloudflare (and similar WAFs) signal bot challenges with 403 + cf-mitigated: challenge.
// Retry once with an honest User-Agent that identifies picoclaw.
if resp.StatusCode == http.StatusForbidden && resp.Header.Get("Cf-Mitigated") == "challenge" {
    honestUA := fmt.Sprintf(userAgentHonest, config.Version)
    resp2, body2, err2 := doFetch(honestUA)
    if err2 == nil {
        resp, body = resp2, body2
    }
}
```

### Perplexity Provider: LLM-Backed Search

**pkg/tools/web.go (lines 378-470)**

Perplexity uses an LLM (`sonar` model) to format search results:

```go
payload := map[string]any{
    "model": "sonar",
    "messages": []map[string]string{
        {
            "role":    "system",
            "content": "You are a search assistant. Provide concise search results...",
        },
        {
            "role":    "user",
            "content": fmt.Sprintf("Search for: %s. Provide up to %d relevant results.", query, count),
        },
    },
    "max_tokens": 1000,
}
```

### SearXNG Provider: Self-Hosted Meta-Search

**pkg/tools/web.go (lines 472-532)**

SearXNG is a self-hosted meta-search engine aggregator:

```go
searchURL := fmt.Sprintf("%s/search?q=%s&format=json&categories=general",
    strings.TrimSuffix(p.baseURL, "/"),
    url.QueryEscape(query))
// Returns JSON with results array
```

### Clever Solutions

1. **Honest User-Agent retry**: When Cloudflare blocks with a bot challenge, retry with a descriptive User-Agent that legitimate bot operators might allow-list.

2. **DNS rebinding TOCTOU mitigation**: Re-resolves DNS at connection time rather than pre-flight, preventing attackers from using DNS rebinding to access private IPs.

3. **Multi-key round-robin**: API key pool with atomic counter and per-iterator start position ensures even distribution across multiple API keys.

4. **Streaming to temp file with size limit**: Downloads go directly to a temp file with `io.LimitReader` to prevent memory exhaustion from large responses.

5. **Trigram-based search caching**: The `search_cache.go` in skills uses trigram similarity; the same pattern could be applied to web search deduplication.

### Technical Debt / Concerns

1. **HTML scraping fragility**: DuckDuckGo HTML structure could change, breaking regex-based extraction without warning.

2. **No response caching**: Each web search goes directly to the provider with no caching layer, potentially burning through API quotas for repeated queries.

3. **Limited error context**: When all API keys fail, the error message doesn't indicate which specific keys were tried or their specific failure reasons.

4. **MaxBytesReader on nil response**: Line 1027 has `resp.Body = http.MaxBytesReader(nil, resp.Body, ...)` which passes nil as the limit writer - this may not work as intended.

5. **No retry with backoff**: Failed requests don't implement exponential backoff, potentially worsening rate limit issues under load.

---

## Cross-Feature Observations

1. **Atomic writes everywhere**: All three features use `fileutil.WriteFileAtomic` for persistence, showing a consistent pattern for flash storage reliability.

2. **Consistent error handling**: All features use structured error wrapping with context (`fmt.Errorf("failed to ...: %w", err)`).

3. **Context propagation**: All HTTP requests properly propagate context with timeouts.

4. **Structured logging**: Using `log/slog` and `logger` package consistently across all features.

5. **Security patterns**: Both cron (command execution) and web fetch (SSRF) implement defense-in-depth with multiple security checks.

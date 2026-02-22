# Requirements: Israel-Gaza NLP — Acquisition Layer

**Defined:** 2026-02-21
**Core Value:** Produce the largest feasible corpus of full-text articles in a resumable, structured SQLite database that the NLP processing layer can act on without data quality issues.

## v1 Requirements

### Foundation

- [ ] **FOUND-01**: Pipeline normalizes and deduplicates all URLs from the GDELT CSV before any fetch (strips UTM params, session tokens, AMP variants, fragments; normalizes to HTTPS)
- [ ] **FOUND-02**: Pipeline filters URLs to only the 5 allowlisted domains (nytimes.com, reuters.com, washingtonpost.com, apnews.com, wsj.com); all other URLs are discarded before fetching
- [ ] **FOUND-03**: Pipeline rejects URLs matching non-prose path patterns (/video/, /podcast/, /interactive/, /live/, /slideshow/, /graphic/) before any fetch attempt
- [ ] **FOUND-04**: SQLite database runs in WAL mode with a single dedicated writer thread draining a queue; worker threads never write directly to the database
- [ ] **FOUND-05**: A `urls` table tracks per-URL status (pending / processing / success / paywall / failed / skipped) with attempt count and error message
- [ ] **FOUND-06**: Pipeline resets all `status='processing'` rows to `status='pending'` on startup, enabling safe crash-resume without re-processing completed URLs

### Extraction

- [ ] **EXTR-01**: Pipeline fetches raw HTML with httpx and extracts article body using trafilatura as the primary extractor (`favor_precision=True`, no comments, no tables)
- [ ] **EXTR-02**: When trafilatura returns `None` or < 150 chars, pipeline retries extraction on the same HTML using newspaper4k as secondary fallback before attempting browser fallback
- [ ] **EXTR-03**: All requests use a rotating pool of 15-20 real browser user-agent strings; UA rotates per request (not per session)
- [ ] **EXTR-04**: Requests to each domain are rate-limited per domain (AP/Reuters: 1.5-2s, NYT/WaPo: 4s, WSJ: 6s, default: 3s); rate limiter blocks at submission, not inside workers
- [ ] **EXTR-05**: Every extracted article is validated: text < 300 chars is rejected; status is recorded as `paywall_suspected` (HTTP 200 + subscribe language) vs `error_parse` (extraction failure) vs `error_network` (HTTP 4xx/5xx)
- [ ] **EXTR-06**: Extraction runs in a ThreadPoolExecutor with configurable `max_workers` (default 20); worker count is documented with memory implications (each Chrome instance ≈ 100MB)

### Paywall Fallback

- [ ] **PWALL-01**: A Playwright-based fallback pool (3 headless Chromium instances) handles URLs from designated paywall domains (nytimes.com, washingtonpost.com, wsj.com) after trafilatura + newspaper4k both fail; `playwright-stealth` is applied to reduce automation fingerprint detection
- [ ] **PWALL-02**: Free proxy list is loaded via `fp-free-proxy`; every proxy is validated against a known endpoint before being added to the pool (timeout 5s; proxies returning non-200 or timeout are discarded)
- [ ] **PWALL-03**: Each proxy tracks consecutive failure count; proxies are banned after 3 consecutive failures; pool triggers background refresh when active proxy count drops below 10
- [ ] **PWALL-04**: Each Playwright browser context is created with its own proxy (`browser.new_context(proxy={...})`), providing isolated cookie jars and session state per proxy

### Data Quality

- [ ] **QUAL-01**: Extracted `full_text` is hashed (SHA-256 of lowercased, whitespace-normalized text) and stored in a `content_hash` column with a UNIQUE constraint; duplicate content (e.g., AP article syndicated to WaPo) is recorded as `status='duplicate'` rather than inserted
- [ ] **QUAL-02**: Publication date is extracted in cascade priority: JSON-LD `datePublished` → OpenGraph `article:published_time` → newspaper4k `publish_date` → GDELT `publish_date` from CSV (last resort, with divergence flag if GDELT date differs from extracted date by > 7 days)
- [ ] **QUAL-03**: All extracted text is encoding-normalized before SQLite insertion: encoding detected with `charset-normalizer`, text converted to UTF-8, null bytes stripped (`\x00`), HTML entities decoded via `html.unescape()`
- [ ] **QUAL-04**: All extraction outcomes are written to a rotating log file (timestamp + level + URL + extractor used + failure reason); a corpus metrics report is printed on pipeline completion showing COUNT(*) by source and status

## v2 Requirements

### Advanced Extraction

- **ADV-01**: Empirical A/B benchmark of trafilatura vs newspaper4k on a 50-100 URL sample from each source before committing to primary/secondary order
- **ADV-02**: Archive.org CDX API fallback for URLs returning 404/410 (article deleted but archived)
- **ADV-03**: `undetected-chromedriver` or `rebrowser-playwright` for advanced anti-fingerprinting on WSJ

### Operational

- **OPS-01**: Author name normalization (strip "By " prefix; map wire service bylines like "By Reuters Staff" to `wire_staff`)
- **OPS-02**: Per-domain concurrency cap via `threading.Semaphore` (max 2 simultaneous requests per domain) for finer-grained politeness control
- **OPS-03**: psutil-based memory monitoring with alerts when RSS exceeds 80% of system RAM

## Out of Scope

| Feature | Reason |
|---------|--------|
| AFP (france24.com) | Zero results in GDELT export; dropped from this milestone |
| LexisNexis / Factiva integration | No institutional access available |
| Paid proxy providers (Bright Data, Oxylabs, Smartproxy) | No budget; free proxies accepted as constraint |
| NLP processing (spaCy, AllenNLP, SRL) | Handled in Phase 3 (Processing Layer); must not be loaded during acquisition |
| Human validation interface | Handled in Phase 4 |
| Statistical analysis and reporting | Handled in Phase 5 |
| Archive.org / Google Cache fallback | Uncertain coverage for 2023-2026 range; high complexity for unverified yield improvement |
| Full browser fingerprint evasion (Canvas API, WebGL, AudioContext spoofing) | Requires commercial tools or high-maintenance patches; marginal research value |
| Scrapy framework | Overkill for a flat URL list; ThreadPoolExecutor + httpx is sufficient |
| Asyncio-first architecture | Breaks on blocking calls (trafilatura lxml, sqlite3); ThreadPoolExecutor handles I/O-bound work correctly |
| Dynamic proxy list auto-refresh daemon | High operational complexity; manual restart if pool exhausts is acceptable for research pipeline |

## Traceability

Populated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| FOUND-01 | — | Pending |
| FOUND-02 | — | Pending |
| FOUND-03 | — | Pending |
| FOUND-04 | — | Pending |
| FOUND-05 | — | Pending |
| FOUND-06 | — | Pending |
| EXTR-01 | — | Pending |
| EXTR-02 | — | Pending |
| EXTR-03 | — | Pending |
| EXTR-04 | — | Pending |
| EXTR-05 | — | Pending |
| EXTR-06 | — | Pending |
| PWALL-01 | — | Pending |
| PWALL-02 | — | Pending |
| PWALL-03 | — | Pending |
| PWALL-04 | — | Pending |
| QUAL-01 | — | Pending |
| QUAL-02 | — | Pending |
| QUAL-03 | — | Pending |
| QUAL-04 | — | Pending |

**Coverage:**
- v1 requirements: 20 total
- Mapped to phases: 0 (pending roadmap)
- Unmapped: 20 ⚠️

---
*Requirements defined: 2026-02-21*
*Last updated: 2026-02-21 after initial definition*

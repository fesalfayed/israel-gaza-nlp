# Project Research Summary

**Project:** Israel-Gaza NLP — Acquisition Layer
**Domain:** Batch news article acquisition pipeline (Python)
**Researched:** 2026-02-21
**Confidence:** HIGH

## Executive Summary

This pipeline is a Stage 2 batch acquisition system: it ingests ~12,622 GDELT-sourced URLs across five major news outlets (NYT, WaPo, WSJ, AP, Reuters) and extracts full article text into a crash-resumable SQLite store for downstream NLP processing. The standard approach for this class of problem is a tiered extraction cascade — fast HTTP extraction for open sources, browser automation for paywalled sources — coordinated by a thread pool with a single serialized DB writer. Research confirms this pattern is well-understood, tooling is stable, and the main risks are operational (paywall yield, free proxy reliability) rather than architectural.

The recommended stack departs materially from the original blueprint. newspaper3k (abandonware since 2018, broken on Python 3.10+) is replaced by trafilatura as primary extractor and newspaper4k as secondary fallback. Selenium is replaced by Playwright, which provides cleaner per-context proxy isolation, automated browser management, and native async. httpx replaces implicit requests usage. These are not marginal preferences — newspaper3k breaks silently on modern Python and trafilatura has an 8-15 point F1 advantage on US news sources in independent benchmarks. The deviations are obligatory, not optional.

The key risk is corpus yield, not pipeline correctness. Open sources (AP, Reuters, ~4,948 URLs, 39% of corpus) should achieve 90-95% yield with straightforward HTTP extraction. Paywalled sources (NYT, WaPo, WSJ, ~7,674 URLs, 61% of corpus) will yield 25-70% depending on paywall aggressiveness and proxy quality. Free proxies are fundamentally unreliable for breaking paywalls — the pipeline should be designed to maximize direct extraction quality, use the Playwright path for cache/AMP variants of paywalled URLs, and accept the yield loss rather than over-engineering proxy infrastructure. Overall expected yield: 65-75% (~8,200-9,450 articles).

---

## Key Findings

### Recommended Stack

trafilatura is the correct primary extractor. It is purpose-built for news article extraction, actively maintained (monthly releases 2023-2025), and benchmarks materially higher than all alternatives on US/European news. newspaper4k serves as secondary fallback — it fixes newspaper3k's Python 3.10+ compatibility but does not improve extraction quality. Playwright handles the paywall fallback path cleanly: it provides isolated browser contexts per proxy, automated Chromium installation, and native asyncio — Selenium requires manual chromedriver version management and blocking executor wrapping.

The concurrency model is ThreadPoolExecutor (max_workers=20) for the main fetch+extract path, a single dedicated writer thread draining a queue to SQLite (eliminating all lock contention), and a separate asyncio event loop thread for the Playwright paywall queue with a cap of 3 concurrent browser instances. SQLite is configured in WAL mode with synchronous=NORMAL — safe for a research pipeline and roughly 2x faster than FULL sync.

**Core technologies:**
- trafilatura >=1.12: primary article extractor — purpose-built for news, 8-15pt F1 advantage over newspaper3k, actively maintained
- newspaper4k >=0.9: secondary extractor — compatible newspaper3k fork, catches trafilatura misses on some CMSes
- Playwright >=1.44: paywall browser fallback — per-context proxy isolation, no chromedriver version drift, native async
- httpx >=0.27: HTTP client — async-capable, HTTP/2 support, cleaner timeout/proxy API than requests
- ThreadPoolExecutor (stdlib): concurrency — I/O-bound work releases GIL; no added dependencies; integrates with single-writer SQLite pattern
- sqlite3 WAL mode (stdlib): storage — single writer thread eliminates lock contention; WAL allows concurrent reads without blocking writes
- fp-free-proxy: proxy sourcing — functional, low confidence; free proxies are datacenter IPs and actively blocked by major publishers

**Critical version pins:**
```
trafilatura>=1.12,<2.0
lxml>=5.2,<6.0
lxml_html_clean>=0.1        # required by trafilatura >=1.8
newspaper4k>=0.9,<1.0
playwright>=1.44,<2.0
playwright-stealth>=1.0
httpx[http2]>=0.27,<1.0
tenacity>=8.3,<9.0
```

Post-install: `playwright install chromium` and `python -m nltk.downloader punkt stopwords`.

### Expected Yield by Source

| Source | URLs | Strategy | Expected Yield |
|--------|------|----------|----------------|
| reuters.com | 2,995 | httpx + trafilatura direct | 90-95% |
| apnews.com | 1,953 | httpx + trafilatura direct | 85-92% |
| nytimes.com | 4,337 | trafilatura (AMP/cached) + Playwright fallback | 55-70% |
| washingtonpost.com | 2,423 | trafilatura + Playwright fallback | 50-65% |
| wsj.com | 688 | Playwright + stealth + proxies | 25-45% |
| **Total** | **12,622** | | **~65-75% overall** |

Reuters and AP are the reliable baseline and should be processed first to validate the pipeline before tackling paywalled sources.

### Expected Features

**Must have (table stakes) — pipeline is broken without these:**
- URL deduplication before fetching — GDELT captures same article 3-5x via syndication; use normalized (netloc + path) as dedup key
- Source allowlist filter — GDELT returns false positives (kelownacapnews.com, thomsonreuters.com, etc.); exact domain match against the five targets
- URL-pattern pre-filter — block /video/, /podcast/, /interactive/, /live/, /slideshow/ before fetching
- SQLite state tracking per URL — `status` column (pending, processing, success, paywall_suspected, error, skipped); crash at URL 5,000 must not restart from zero
- Crash-resumable worker loop — reset `processing → pending` on startup; never re-process completed URLs
- Minimum text length gate (300 chars) — short extractions are paywall intercept pages or parse failures
- Per-domain rate limiting — domain-specific delays (AP/Reuters: 1.5s, NYT/WaPo: 4s, WSJ: 6s)
- Rotating user agents — pool of 15-20 real browser UA strings; fixed UA is trivially detected
- Structured logging to file — a 12K-URL job runs for hours; stdout is lost on crash
- HTTP timeout enforcement (15s) — a stalled connection blocks a worker thread indefinitely
- Source normalization — jp.reuters.com and uk.reuters.com both map to `reuters`; NLP groups by source label
- Word count pre-computation — `len(text.split())` at INSERT time, not later
- GDELT metadata passthrough — gdelt_tone and gdelt_themes carry through even when text extraction fails

**Should have (differentiators that significantly improve yield or data quality):**
- Paywall detection as distinct status — distinguish paywall from JS render failure from bot detection; use multi-signal classifier (HTTP status + body keywords + content length + response headers)
- Exponential backoff with jitter — base_delay = 2^attempt + random(0,1); max 3 retries per URL; 429/503 responses should not cause permanent failure
- Proxy validation before use — HEAD request to httpbin.org/ip with 5s timeout; 60-80% of free proxy list entries are dead
- Proxy health tracking in SQLite — proxies table with success/failure counts; ban after 3 consecutive failures
- Content hash deduplication — SHA-256 of normalized full_text; AP articles syndicated to WaPo appear with different URLs
- ThreadPool per-domain concurrency cap — flat pool of 20 workers may flood a single domain; `defaultdict(Semaphore)` keyed on netloc, max 2 concurrent
- Corpus metrics report on completion — yield by source before committing NLP compute

**Defer (evaluate after real yield numbers):**
- trafilatura A/B test vs newspaper4k on a 50-URL benchmark — validate assumption before full run
- Author extraction normalization — wire service bylines ("By Reuters Staff") need special handling
- Archive.org fallback — coverage inconsistent for 2023-2026; re-evaluate only if corpus yield is below acceptable threshold
- Per-domain concurrency cap — implement only if domain bans occur in practice

**Anti-features (do not build):**
- NLP processing during acquisition — decouples stages; loading spaCy + AllenNLP wastes RAM and couples failures
- asyncio architecture — trafilatura and sqlite3 are blocking; forcing asyncio wrapping adds complexity with no benefit
- Generic crawler / link following — pipeline has a known, finite URL list; link following is irrelevant
- Full browser fingerprint evasion — marginal research value does not justify maintenance overhead of canvas/WebGL spoofing
- Selenium for open sources — AP/Reuters get 90-95% with httpx+trafilatura; Selenium is 10-20x slower for no yield gain

### Architecture Approach

The architecture is a three-lane pipeline: a fast HTTP lane (ThreadPoolExecutor, httpx, trafilatura) handles the majority of URLs; a slow browser lane (asyncio event loop thread, Playwright, 3 concurrent contexts) handles paywall fallbacks; a single serialized writer thread drains results to SQLite via queue.Queue. The orchestrator (main.py) seeds the URL queue from CSV, manages domain rate-limit state, and handles shutdown. All components communicate through the SQLite state store rather than in-memory state, making the pipeline fully crash-resumable from any point.

**Major components:**

| Component | File | Responsibility |
|-----------|------|----------------|
| Orchestrator | main.py | CSV seeding, ThreadPoolExecutor management, domain rate-limit, SIGINT handling, progress reporting |
| StateStore | db.py | All SQLite I/O; WAL mode setup; schema creation; write/status-update/resume methods |
| ExtractorChain | extractor.py | httpx fetch → trafilatura → newspaper4k → Playwright queue; validates text length; returns structured dict |
| PlaywrightPool | browser.py | asyncio loop in daemon thread; 3 concurrent Playwright contexts; per-context proxy injection |
| ProxyPool | proxies.py | Load, validate, rotate, ban proxies; exposes get_proxy() / report_failure() |
| RateLimiter | ratelimit.py | Per-domain delay enforcer using {domain: last_request_time} dict + threading.Lock |

**Dependency direction (no circular imports):**
```
main.py  →  db.py
main.py  →  ratelimit.py
main.py  →  extractor.py  →  browser.py  →  proxies.py
                           →  proxies.py
```

**Extraction cascade per URL:**
```
1. httpx fetch (fast, non-JS)
2. trafilatura.extract(html, favor_precision=True)
3. If None or len < 150: newspaper4k on same HTML
4. If both fail AND domain is paywalled: queue for Playwright
5. If all fail: status = 'extraction_failed'
```

**URL state machine:**
```
pending → processing → success
                    → failed
                    → skipped  (paywall confirmed, no proxies left)
```
On startup: `UPDATE urls SET status='pending' WHERE status='processing'` — makes the pipeline idempotent across crashes.

**Suggested build order:**
1. db.py — schema, WAL mode, write/update/resume methods
2. ratelimit.py — no project dependencies
3. proxies.py — no project dependencies
4. extractor.py — trafilatura/newspaper4k path only
5. browser.py — PlaywrightPool (depends on proxies.py)
6. extractor.py — wire in Playwright fallback
7. main.py — orchestrator, integration test on 10-URL sample
8. Full corpus run: Reuters + AP first, then NYT/WaPo, then WSJ

### Critical Pitfalls

**Top 5 pitfalls that cause data corruption, silent loss, or full rewrites:**

1. **SQLite lock contention with ThreadPoolExecutor** — multiple worker threads sharing one connection without explicit serialization cause silent write failures. Non-deterministic row counts (pipeline logs 12,000 processed, DB has 9,400). Prevention: single writer thread via queue.Queue, WAL mode, PRAGMA busy_timeout=10000. Never use check_same_thread=False as the fix.

2. **Extraction returning empty text without raising exceptions** — trafilatura and newspaper4k both return None or short text silently on JS-rendered pages and soft paywalls. Pipeline can commit blank rows and log them as successes. Prevention: enforce `len(text.strip()) >= 300` after every extraction attempt before INSERT; log HTML length separately from extracted text length.

3. **Playwright browser processes not cleaned up on exception** — each Playwright context left unclosed exhausts RAM and file descriptors over a multi-hour run. Prevention: always wrap browser use in try/finally; cap concurrent Playwright workers at 3 via asyncio semaphore; kill stale Chrome processes at startup.

4. **Free proxy lists are ~70-95% dead and blocked by major publishers** — free proxies are datacenter IPs; NYT/WaPo/WSJ actively blocklist datacenter ranges. Using proxies can produce lower yield than direct requests. Prevention: validate each proxy with a HEAD request before use; accept that paywalled sources will have structural yield limits; do not route AP/Reuters through proxies.

5. **Resume logic bugs — stuck `processing` rows and duplicate inserts** — on crash, URLs mid-processing are stuck as `processing` and silently skipped on resume. Concurrent workers can race to process the same URL. Prevention: reset all `processing → pending` on startup; use `UNIQUE(normalized_url)` + `INSERT OR IGNORE` as DB-enforced dedup; only set `status='success'` after full row is committed.

**Additional important pitfalls:**
- URL normalization gaps — strip UTM params, session tokens, AMP variants, fragments before using URL as dedup key; extract canonical URL from HTML after fetch
- Date source confusion — GDELT `date` is crawl date, not publication date; extract from JSON-LD → OpenGraph → HTML meta → newspaper4k → GDELT (fallback only); flag divergence > 7 days
- Encoding corruption — use charset-normalizer; strip null bytes before INSERT; run html.unescape() on extracted text
- Paywall vs bot detection confusion — HTTP 403 + `cf-ray` header is bot detection, not paywall; store `block_reason` column distinguishing paywall, bot_detection, rate_limited, deleted, js_required

---

## Implications for Roadmap

### Phase 1: Foundation — Schema, State, URL Normalization

**Rationale:** State tracking is a prerequisite for all other features. Without it, no crash recovery is possible and no other feature provides value. Build this first, independently, before writing any fetch logic.

**Delivers:** SQLite database with WAL mode, full schema (urls + articles + proxies tables), URL normalization utility, crash-resume logic, and corpus metrics query.

**Addresses features:** URL deduplication, source allowlist filter, URL-pattern pre-filter, SQLite state tracking, crash-resumable loop, GDELT metadata passthrough, source normalization.

**Avoids pitfall:** Resume logic bugs (Pitfall 5) — UNIQUE constraint and INSERT OR IGNORE from day 1.

**Research flag:** Standard pattern, well-documented. No additional research needed.

### Phase 2: Open Source Extraction (AP + Reuters, ~4,948 URLs)

**Rationale:** AP and Reuters are open access. They validate the core pipeline before any paywall complexity is introduced. Achieve 85-95% yield here first to prove the extraction chain is correct before debugging paywall failures.

**Delivers:** Validated extraction pipeline for ~5,000 articles; confirmed yield numbers for AP and Reuters; structured logs showing extraction quality.

**Uses:** httpx, trafilatura (primary), newspaper4k (secondary), ThreadPoolExecutor, single writer thread, per-domain rate limiting.

**Implements:** ExtractorChain (trafilatura + newspaper4k paths), RateLimiter, ProxyPool (validation logic only, no paywall routing), Orchestrator.

**Avoids pitfalls:** Empty extraction logging as success (Pitfall 2) via 300-char gate; SQLite lock contention (Pitfall 1) via writer thread; encoding corruption (Pitfall 8) via charset-normalizer.

**Research flag:** Standard pattern. No additional research needed.

### Phase 3: Paywall Path (NYT + WaPo, ~6,760 URLs)

**Rationale:** NYT and WaPo have metered/soft paywalls — more tractable than WSJ. They share similar JS-rendering characteristics. Build and validate the Playwright path on these before tackling WSJ's harder Cloudflare layer.

**Delivers:** PlaywrightPool with asyncio event loop thread, proxy validation and health tracking, paywall detection heuristics, distinct status codes for paywall vs bot-detection vs JS-render failure.

**Uses:** Playwright, playwright-stealth, asyncio, fp-free-proxy, multi-signal response classifier.

**Implements:** browser.py (PlaywrightPool), proxies.py (health tracking), paywall detection classifier.

**Avoids pitfalls:** Browser memory leaks (Pitfall 3) via try/finally + 3-instance cap; proxy list failures (Pitfall 4) via validation before use; paywall vs bot detection confusion (Pitfall 6) via block_reason column.

**Research flag:** Playwright stealth configuration for NYT and WaPo may need targeted testing. The playwright-stealth library API may have changed since training data. Recommend a 20-URL smoke test before full run.

### Phase 4: WSJ + Corpus Validation

**Rationale:** WSJ has Cloudflare Enterprise + Dow Jones paywall — the hardest target. Run last with calibrated expectations (25-45% yield). After WSJ, run full corpus metrics, identify any source showing anomalous yield, and do targeted re-queuing of failed URLs before handing off to NLP.

**Delivers:** WSJ extraction attempt, final corpus metrics by source and status, content hash deduplication pass to remove syndicated duplicates, final articles.db ready for Stage 3.

**Addresses features:** Corpus metrics report, content hash deduplication, exponential backoff for any remaining failed URLs, per-domain concurrency cap (if bans were observed in Phase 3).

**Avoids pitfalls:** WSJ yield expectations must be set correctly — Playwright + stealth is not expected to break Cloudflare Enterprise reliably.

**Research flag:** WSJ Cloudflare configuration changes frequently. Validate current bypass approach (playwright-stealth + proxy) on 5-URL test before committing to full WSJ run. Accept that 25-45% is the realistic ceiling.

### Phase Ordering Rationale

- **State first (Phase 1)** — every other component reads from and writes to the state store. Building it first also gives the team a testable artifact early.
- **Open sources before paywalled (Phase 2 before 3)** — validates the extraction chain without paywall complexity; provides a usable partial corpus if Phases 3-4 encounter significant issues.
- **NYT/WaPo before WSJ (Phase 3 before 4)** — WSJ is significantly harder; build the Playwright path on more tractable targets first.
- **Corpus validation last (Phase 4)** — deduplication and final metrics only make sense after all sources are attempted.

### Research Flags

**Needs targeted validation before full run:**
- Phase 3: Playwright stealth configuration against NYT and WaPo — run 20-URL smoke test first
- Phase 4: WSJ Cloudflare bypass — validate 5-URL test before full run; Cloudflare configuration changes without notice

**Standard patterns, no additional research needed:**
- Phase 1: SQLite WAL mode + single writer thread is a well-documented pattern
- Phase 2: httpx + trafilatura + ThreadPoolExecutor is straightforward I/O-bound batch processing

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | trafilatura and Playwright recommendations backed by multiple independent benchmarks and active maintenance records; httpx and ThreadPoolExecutor are standard choices |
| Features | HIGH | Table stakes features are standard batch scraping patterns; confirmed against project constraints and blueprint |
| Architecture | HIGH | Single-writer thread + WAL mode + tiered extraction is a well-understood, well-documented pattern for this class of problem |
| Pitfalls | HIGH | Pitfalls 1-5 are common failure modes extensively documented in Python web scraping literature; yield estimates for WSJ are LOW confidence due to Cloudflare variability |

**Overall confidence:** HIGH for architecture and implementation approach; MEDIUM for paywall yield estimates (WSJ in particular).

### Gaps to Address

- **trafilatura vs newspaper4k relative performance on these specific sources** — run a 50-URL benchmark on each source before committing to the cascade order; assumption is trafilatura-first but this should be validated empirically.
- **Free proxy quality at time of run** — proxy pool viability degrades within hours of list publication; have a plan to run open sources without proxies if proxy pool is unusably small.
- **NYT AMP / cache URL availability** — some NYT GDELT URLs may have AMP variants accessible without paywall; worth checking URL patterns in the CSV before assuming all 4,337 require Playwright.
- **WSJ yield floor** — if WSJ yield is below 20%, evaluate whether 688 articles justify the Playwright infrastructure cost; may be better to mark WSJ as `skipped` and accept the corpus gap.
- **GDELT date quality** — validate the divergence between GDELT crawl dates and extracted publication dates on a sample before NLP handoff; high divergence may affect timeline analysis.

---

## Key Deviations from Original Blueprint

| Blueprint Spec | Research Recommendation | Impact |
|---|---|---|
| newspaper3k (primary) | trafilatura (primary) + newspaper4k (secondary) | Obligatory: newspaper3k is broken on Python 3.10+; trafilatura has 8-15pt F1 advantage |
| Selenium | Playwright | Obligatory: cleaner proxy isolation, no chromedriver version mismatch, native async |
| requests (implicit) | httpx | Recommended: async-capable, HTTP/2, cleaner API |
| Newspaper3k Config for UA/timeout | trafilatura settings + httpx client | Follow-on from primary extractor change |
| WORKER_COUNT=8 (blueprint) | max_workers=20 (research) | More threads safe for I/O-bound work; Playwright capped separately at 3 |

---

## Sources

### Primary (HIGH confidence)
- trafilatura documentation and Leipzig NLP benchmarks — extraction quality, API, version requirements
- Playwright documentation (playwright.dev) — browser automation, proxy injection, async API
- Python sqlite3 documentation — WAL mode, threading, PRAGMA options
- httpx documentation — async HTTP, proxy support, timeout configuration

### Secondary (MEDIUM confidence)
- newspaper4k PyPI and GitHub — fork status, API compatibility with newspaper3k, Python 3.10+ support
- fp-free-proxy PyPI — proxy sourcing; reliability is inherently low for free proxy lists
- playwright-stealth PyPI — stealth configuration for bot detection evasion; API may drift

### Tertiary (LOW confidence)
- Free proxy success rate estimates — industry estimates ranging 10-30% uptime; actual rates vary significantly
- WSJ yield estimates (25-45%) — highly variable; Cloudflare Enterprise configuration changes frequently

---
*Research completed: 2026-02-21*
*Ready for roadmap: yes*

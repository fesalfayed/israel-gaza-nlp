# Feature Landscape: News Article Acquisition Pipeline

**Domain:** Batch news article acquisition — URL list to full-text SQLite corpus
**Researched:** 2026-02-21

---

## Table Stakes

Features the pipeline is broken without. Missing any of these means the corpus is unusable, the job cannot run at scale, or results cannot be reproduced.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| URL deduplication before fetching | GDELT captures same article 3-5x via syndication; bloats corpus and misleads downstream NLP | Low | Normalize on (netloc + path), strip query strings and fragments |
| Source allowlist filter | GDELT returns false positives: kelownacapnews.com, wnewsj.com, thomsonreuters.com | Low | Exact domain match against {nytimes.com, washingtonpost.com, wsj.com, apnews.com, reuters.com} |
| URL-pattern pre-filter | Videos, podcasts, interactives, and live blogs contain no prose text | Low | Regex block on /video/, /podcast/, /interactive/, /live/, /slideshow/, /graphic/ |
| SQLite state tracking per URL | Without `status` column, a crash at URL 5,000 restarts from zero | Low | `status` enum: pending, processing, success, paywall, error, skipped |
| Crash-resumable worker loop | Query `WHERE status = 'pending'` on startup; never re-process completed URLs | Low | Reset `processing → pending` on startup to handle mid-run crashes |
| Minimum text length gate (300 chars) | Short extractions are paywall intercept pages or parsing failures — not articles | Low | Apply after extraction, before INSERT |
| Per-domain rate limiting | Burst requests trigger IP bans; wire services actively throttle | Medium | Token bucket or sleep-based; minimum 1-2s between requests to same domain |
| Rotating user agents | Fixed UA strings are trivially detected and blocked | Low | Pool of 15-20 real browser UA strings; rotate per request |
| Structured logging to file | A 12K-URL job runs for hours; print() output is lost on crash | Low | Python logging module + file handler; timestamp + level + url + reason |
| Newspaper3k as primary extractor | Confirmed working on Reuters (test.py); handles open sources | Low | Pin version; use Config object for UA and timeout |
| HTTP timeout enforcement (15s) | Single stalled connection blocks a worker thread indefinitely | Low | 15s per blueprint; applies to both Newspaper3k and Selenium |
| Source normalization to canonical label | jp.reuters.com, uk.reuters.com are both Reuters; NLP groups by source | Low | Map at INSERT time from URL domain to: nyt, wapo, wsj, ap, reuters |
| Word count pre-computation | NLP corpus statistics need word count; compute at INSERT not later | Low | `len(text.split())` — simple and sufficient |
| GDELT metadata passthrough | publish_date and tone_scores from CSV are pre-computed fallbacks | Low | gdelt_tone and gdelt_themes columns carry through even when extractor fails |

---

## Differentiators

Features that significantly improve corpus yield or data quality without being strictly required.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Paywall detection as distinct status | `len(text) < 300` cannot distinguish paywall, genuine short article, or parse failure | Medium | HTTP 200 + short text + "Subscribe"/"Sign in" in title = `paywall_suspected`; HTTP 4xx = `error_network` |
| Exponential backoff with jitter on failures | A domain returning 429/503 should not be abandoned — retrying immediately makes it worse | Low | `base_delay = 2 ** attempt + random(0, 1)`; max 3 retries per URL |
| Proxy validation before use | 60-80% of free proxy list entries are dead; unvalidated proxies waste Selenium startup time | Medium | HEAD request to httpbin.org/ip with 5s timeout; mark invalid proxies `is_active = False` |
| Proxy health tracking in SQLite | Track per-proxy success/failure count; ban after 3 consecutive failures | Medium | `proxies` table: ip, port, protocol, last_validated, success_count, failure_count, is_active |
| Content hash deduplication | AP articles syndicated to WaPo appear twice with different URLs | Low | SHA-256 of normalized full_text; UNIQUE constraint on `content_hash` column |
| Selenium session reuse | Starting Chrome takes 2-5s; at 3,800 Selenium-fallback URLs that's 7,600-19,000s of pure startup overhead | High | `SeleniumPool` with queue.Queue; pre-warm N drivers, lease/return |
| GDELT date fallback | Newspaper3k often fails to extract publish_date from paywalled sites | Low | Parse GDELT YYYYMMDDHHMMSS format; use as fallback only, not override |
| ThreadPool per-domain concurrency cap | Flat ThreadPool of N workers may send all N requests to the same domain simultaneously | Medium | `defaultdict(threading.Semaphore)` keyed on netloc; max 2 concurrent per domain |
| Corpus metrics report on completion | Must know yield by source before committing NLP compute time | Low | `SELECT source, status, COUNT(*) FROM urls GROUP BY source, status` |
| Author extraction normalization | Wire service bylines ("By Reuters Staff") are not individual authors | Low | Join list with "; "; strip "By "; map known byline patterns to `wire_staff` |

---

## Anti-Features

Things to deliberately NOT build for this acquisition milestone.

| Anti-Feature | Why Avoid |
|---|---|
| NLP processing during acquisition | Loading spaCy (170MB) + AllenNLP SRL (500MB) in the same process wastes RAM, slows extraction, and couples independent stages. If NLP fails, extraction progress is lost. |
| LexisNexis / Factiva integration | No institutional access available. Dead code adds complexity and misleads future maintainers. |
| Residential / paid proxy integration | Budget is zero. Designing for paid proxy APIs produces an architecture that free proxy workflow cannot fit cleanly into. |
| Archive.org / Google Cache fallback | Coverage inconsistent for 2023-2026 range; implementation complexity is high for uncertain yield. Re-evaluate only if corpus yield is unacceptably low. |
| Full browser fingerprint evasion | Canvas API spoofing, WebGL, AudioContext fingerprint evasion requires commercial tools or high-maintenance custom patches. Marginal research value does not justify the complexity. |
| Async/await (asyncio) architecture | Newspaper3k and Selenium are synchronous libraries. asyncio wrapping adds complexity without concurrency benefit. ThreadPoolExecutor handles I/O-bound parallelism correctly. |
| Generic crawler (link-following) | Pipeline has a known, finite URL list from GDELT. Link following adds robots.txt complexity and revisit logic that is irrelevant. |
| Selenium for open sources (AP, Reuters) | AP (~1,953 URLs) and Reuters (~2,995 URLs) get 90-95% success with Newspaper3k. Selenium is 10-20x slower for negligible yield gain. |

---

## Feature Dependencies

```
URL deduplication
    → Source allowlist filter
        → URL-pattern pre-filter
            → SQLite state tracking
                → Crash-resumable loop
                    → Newspaper3k extraction (primary)
                        → Minimum text length gate
                            → Paywall detection heuristic
                                → Selenium fallback (NYT/WaPo/WSJ only)
                                    → Proxy validation + health tracking
                                        → Proxy rotation

Rotating user agents + Per-domain rate limiting
    → ThreadPool per-domain concurrency cap
        → Exponential backoff with jitter

Date parsing (extractor) → GDELT date fallback (if None)
Extracted full_text → Content hash deduplication + Word count
```

**Key dependency chain:** State tracking is a prerequisite for everything. Without it, crash recovery is impossible and no other feature provides value.

---

## MVP Recommendation

**Build first (open sources: AP + Reuters ~5K URLs):**
1. URL deduplication + source allowlist filter
2. SQLite state tracking with `status` column
3. Newspaper3k + rotating user agents + 15s timeout
4. Minimum text length gate (300 chars)
5. Per-domain rate limiting
6. Structured logging to file
7. Corpus metrics report on completion

**Build second (paywalled sources: NYT/WaPo/WSJ ~7K URLs):**
8. Proxy validation and health tracking
9. Selenium + proxy fallback
10. Content hash deduplication
11. Exponential backoff with jitter
12. GDELT date fallback

**Defer (evaluate after real yield numbers):**
- trafilatura A/B test vs Newspaper3k (50-URL benchmark first)
- Author normalization
- Per-domain concurrency cap (implement only if domain bans occur)

---

## Schema Additions Beyond Blueprint

The blueprint schema (section 4.1) covers NLP annotation tables but is incomplete for acquisition state tracking. The `urls` tracking table and additional columns are needed:

```sql
-- URL state tracking (acquisition-internal, not in blueprint)
CREATE TABLE IF NOT EXISTS urls (
    url           TEXT PRIMARY KEY,
    source        TEXT NOT NULL,
    publish_date  TEXT,
    themes        TEXT,
    tone_scores   TEXT,
    status        TEXT NOT NULL DEFAULT 'pending',
    extractor     TEXT,
    attempt_count INTEGER DEFAULT 0,
    last_attempt  TEXT,
    error_msg     TEXT
);

-- Proxy health tracking (not in blueprint)
CREATE TABLE IF NOT EXISTS proxies (
    proxy_id      INTEGER PRIMARY KEY AUTOINCREMENT,
    ip            TEXT NOT NULL,
    port          INTEGER NOT NULL,
    protocol      TEXT DEFAULT 'http',
    last_validated DATETIME,
    success_count INTEGER DEFAULT 0,
    failure_count INTEGER DEFAULT 0,
    is_active     BOOLEAN DEFAULT TRUE,
    UNIQUE(ip, port)
);

CREATE INDEX IF NOT EXISTS idx_urls_status ON urls(status);
CREATE INDEX IF NOT EXISTS idx_urls_source_status ON urls(source, status);
```

---

## Confidence Assessment

| Area | Confidence | Basis |
|------|------------|-------|
| Table stakes features | HIGH | Standard batch scraping patterns; confirmed against blueprint and project constraints |
| Free proxy success rates | MEDIUM | Industry estimates; actual rates vary by source and time of day |
| Paywall bypass rates via Selenium | MEDIUM | Blueprint section 3.1 provides estimates; actual depends on proxy quality |
| SQLite schema additions | HIGH | Standard resumable-job pattern; straightforward extensions of blueprint schema |
| Per-domain rate limiting | HIGH | Standard web scraping practice; well-understood pattern |
| Extractor comparison (trafilatura) | MEDIUM | Community reports from training data; not live-verified |

---

*Feature research: 2026-02-21*

# Domain Pitfalls: Python News Acquisition Pipeline

**Domain:** Web scraping / news article acquisition pipeline (Python)
**Researched:** 2026-02-21

---

## Critical Pitfalls

Mistakes that cause data corruption, silent data loss, or full pipeline rewrites.

---

### Pitfall 1: SQLite "database is locked" with ThreadPoolExecutor

**What goes wrong:**
SQLite's connection object is not safe to share across threads. When multiple workers share a single `sqlite3.connect()` connection, they race to write simultaneously. The dangerous outcome: `database is locked` exceptions caught and swallowed inside worker threads, causing silent write failures.

**Consequences:**
- Rows that appear committed but were silently rolled back
- Non-deterministic row counts — pipeline reports 12,000 processed, DB has 9,400 rows
- Resume logic reads stale count, re-fetches articles already partially stored

**Prevention:**
1. Give each worker thread its own connection: call `sqlite3.connect(db_path)` inside the worker function
2. Enable WAL mode: `PRAGMA journal_mode=WAL;`
3. Set busy timeout: `PRAGMA busy_timeout=5000;`
4. Alternative: route all writes through a single writer thread via `queue.Queue`
5. Never use `check_same_thread=False` as the fix — it removes the guardrail without solving the race

**Warning signs:**
- Any `OperationalError: database is locked` in logs, even intermittent
- Row count in DB does not match "success" log line count
- Pipeline hangs at high worker counts but works at concurrency=1

**Phase/component:** DB writer module, connection management setup.

---

### Pitfall 2: Newspaper3k Silent Content Extraction Failures

**What goes wrong:**
Newspaper3k's `.download()` + `.parse()` sequence can succeed (no exception) while returning empty or garbage content. Pipeline logs a success and commits a row with blank text.

**Specific failure cases:**
- **JS-rendered pages:** Newspaper3k uses `requests`. If page returns an HTML shell with a JS bundle (NYT, WaPo modern pages), `article.text` is `""`. No exception raised.
- **Soft paywalls:** Article object parses successfully but contains only the lede. `len(article.text) < 200` on 60%+ of paywalled URLs.
- **Date parsing returning None silently:** Fails on non-US date formats, ISO 8601 with timezone offsets, and relative dates.
- **Author field returning URL fragments:** Some CMSes produce `article.authors = ["/staff/john-doe"]`.
- **Encoding corruption:** Pages served as Windows-1252 but declared as UTF-8 produce mojibake. No exception.

**Prevention:**
1. After every `.parse()`, validate: `if len(article.text.strip()) < 300` → mark as `extraction_failed`
2. Log HTML length separately from text length — big HTML + tiny text = JS-rendered or paywall response
3. Extract dates in cascade: JSON-LD `datePublished` → OpenGraph `article:published_time` → Newspaper3k → GDELT fallback
4. Check for content fingerprint duplicates: identical text across different URLs = generic paywall wall text

**Warning signs:**
- `article.text` shorter than 300 chars on URLs that are not known short-form pieces
- `publish_date` is `None` for > 30% of a source's articles
- Duplicate `article.text` strings across different URLs

**Phase/component:** Article extraction module, post-parse validation step.

---

### Pitfall 3: Selenium ChromeDriver Memory Leaks and Zombie Processes

**What goes wrong:**
Each `webdriver.Chrome()` spawns a `chromedriver` child process and a `chrome` process. If not explicitly `.quit()`-ed — on exceptions, timeouts, or keyboard interrupts — both stay resident. Over a multi-hour run with parallel Selenium workers, zombie processes exhaust RAM and file descriptors.

**Why it happens:**
- `driver.close()` closes the browser window but leaves `chromedriver` alive
- Unhandled exceptions skip `driver.quit()` if `finally` is absent
- Uncaught `KeyboardInterrupt` during `ThreadPoolExecutor.shutdown()` skips cleanup

**Consequences:**
- System OOM kill after several hundred articles with multiple parallel workers
- Port exhaustion (DevTools port range)
- Orphaned `/tmp/.org.chromium.*` directories filling disk

**Prevention:**
1. Always wrap driver use in `try/finally: driver.quit()`
2. Cap concurrent Selenium workers at 2–4 via semaphore regardless of thread pool size (Chrome ≈ 150-300MB per instance)
3. Kill stale processes from previous interrupted runs at startup
4. Set Chrome options: `--disable-gpu`, `--no-sandbox`, `--disable-dev-shm-usage`
5. Log `psutil.virtual_memory().percent` periodically; alert above 80%

**Warning signs:**
- More `chrome` processes running than your concurrency setting
- RSS memory of Python process grows monotonically
- `OSError: [Errno 24] Too many open files` during long run

**Phase/component:** Selenium driver factory, worker lifecycle management.

---

### Pitfall 4: Free Proxy List Failure Rates and MITM Risk

**What goes wrong:**
Free proxy lists have 70–95% failure rates within hours of publication. The proxies that "work" are often too slow to avoid timeouts, already blocked by major publishers, or operated as honeypots.

**Specific failure modes:**
- **Dead proxies:** Majority offline within 24 hours of list publication
- **Transparent proxies:** Many send `X-Forwarded-For` with your real IP, defeating evasion
- **MITM risk:** Some free proxies intercept traffic; HTTP endpoints fully readable
- **Datacenter IP blocks:** NYT, WaPo, WSJ maintain blocklists of datacenter IP ranges — free proxies are datacenter IPs, getting 403s at higher rates than direct requests
- **Honeypots:** Some IPs are operated by security researchers and log all traffic

**Prevention:**
1. Do not use free proxies for paywalled targets — those sites block datacenter IPs regardless
2. For open sources (AP, Reuters), direct requests with rate limiting are more reliable than any proxy
3. Validate each proxy before use: test against a known endpoint, verify returned IP ≠ your real IP, discard if response time > 5s
4. Never route authenticated sessions or API tokens through a free proxy

**Warning signs:**
- Timeout rate > 40% in proxy-routed requests
- All proxied requests to paywalled sources return 403 immediately
- Pipeline throughput lower with proxies than without

**Phase/component:** Proxy rotation module, source routing logic.

---

## Moderate Pitfalls

---

### Pitfall 5: Paywall Detection Confused with Anti-Bot Detection

**What goes wrong:**
A 403, login redirect, or page with < 200 chars could mean paywall, bot detection, deleted article, geographic restriction, or CDN error. Treating all as "paywall_blocked" mislabels bot-detected rows.

**How to distinguish:**

| Signal | Likely Cause |
|--------|-------------|
| HTTP 403 with login redirect | Paywall / auth wall |
| HTTP 403 with `cf-ray` response header | Bot detection (Cloudflare) |
| HTTP 429 | Rate limiting (not paywall) |
| HTTP 200, text < 200 chars, contains "subscribe" | Soft paywall |
| HTTP 200, text < 200 chars, no subscribe language | JS render failure |
| HTTP 200, CAPTCHA in HTML body | Bot detection |
| HTTP 404 or 410 | Article deleted / dead URL |

**Prevention:**
1. Multi-signal classifier: HTTP status + response body keywords + content length + response headers
2. Store classification reason in `block_reason` column: `paywall`, `bot_detection`, `rate_limited`, `deleted`, `js_required`
3. Do not permanently mark as paywall-blocked after one failure — retry bot-detected URLs with different UA
4. Keep first 2KB of raw HTML for blocked articles to allow manual reclassification

**Phase/component:** Response classification module, DB schema.

---

### Pitfall 6: URL Normalization Edge Cases

**What goes wrong:**
GDELT URLs contain tracking params, session tokens, AMP variants, and CDN cache params. Without normalization, the same article appears as multiple distinct URLs, and resume logic re-fetches it on every run.

**Specific cases:**
- `?utm_source=twitter` — tracking params that change per referrer
- `?ref=nyt_tw` / `?s=09` — session tokens that change per session
- `/amp/` path prefix or `?amp=1` — AMP variant of the same article
- `#section-anchor` — fragments that do not affect content
- `http://` vs `https://` of the same URL

**Prevention:**
1. Normalize before DB insert: strip `utm_*`, strip tracking params (`ref=`, `s=`, `ncid=`), convert to HTTPS, remove fragments, normalize trailing slash, collapse AMP variants
2. Store both original GDELT URL and normalized URL
3. Use normalized URL as deduplication key
4. After fetch, extract `<link rel="canonical">` from HTML — use as authoritative dedup key

**Warning signs:**
- Articles with same title and publish_date appearing with different URLs in DB
- Same article fetched 3-4x with different UTM variants

**Phase/component:** URL normalization utility (run before any fetch or DB insert).

---

### Pitfall 7: Date Parsing Failures and GDELT Date vs Article Date Divergence

**What goes wrong:**
GDELT's `date` field is the crawl discovery date, not the publication date. For re-crawled articles it can be days late — assigning articles to the wrong time window in a conflict timeline analysis.

**Additional failures:**
- Newspaper3k returns `None` for ISO 8601 dates with timezone offsets on some CMSes
- "Last updated" dates in metadata instead of original publication dates
- JSON-LD `datePublished` vs `dateModified` conflation

**Prevention:**
1. Extract dates in priority order: JSON-LD `datePublished` → OpenGraph `article:published_time` → HTML meta pubdate → Newspaper3k → GDELT date (last resort, flag divergence)
2. Store all date sources separately; record which source was used
3. Flag articles where GDELT date differs from extracted date by more than 7 days
4. Parse all dates with `python-dateutil`; normalize to UTC
5. Never use `datetime.strptime` with a hardcoded format on external strings

**Warning signs:**
- `publish_date` is `None` for > 20% of a given source's articles
- Cluster of article dates falling on GDELT crawl dates rather than spread across the news cycle

**Phase/component:** Metadata extraction module, date normalization utility.

---

### Pitfall 8: Encoding Corruption in Article Text

**What goes wrong:**
News sites serve mixed encodings. Without explicit detection and normalization, stored text contains mojibake that corrupts NLP downstream.

**Specific failure modes:**
- Pages declaring `charset=utf-8` but serving Windows-1252: `â€œ` instead of `"`
- HTML entities not decoded: `&amp;`, `&nbsp;`, `&mdash;` left as literals
- `\x00` null bytes causing `sqlite3.InterfaceError` on insertion

**Prevention:**
1. Use `charset-normalizer` to detect encoding before passing HTML to Newspaper3k
2. Normalize to UTF-8 before insert: `text.encode('utf-8', errors='replace').decode('utf-8')`
3. Strip null bytes: `text.replace('\x00', '')`
4. Run `html.unescape()` on extracted text as final pass

**Warning signs:**
- `â€` or `Ã©` appearing in stored article text
- `sqlite3.InterfaceError: Error binding parameter` on text insertion

**Phase/component:** Text normalization utility, pre-insert sanitization.

---

### Pitfall 9: Resume Logic Bugs — Duplicates and Half-Written Rows

**What goes wrong:**
After interruption, resume logic must correctly identify what was processed.

**Common bugs:**
- **Status set at start, not end:** Worker sets `status = 'processing'` when it starts; on crash, rows stuck in `'processing'` forever and skipped on resume
- **Half-written rows:** Worker inserts with `status = 'done'` but crashes before commit
- **Duplicate rows from concurrent workers:** Two workers see same URL as unprocessed, both insert

**Prevention:**
1. Add `UNIQUE(normalized_url)` constraint and use `INSERT OR IGNORE` — DB enforces dedup regardless of race conditions
2. Only set `status = 'success'` after full row (text + metadata) is committed; never before
3. On startup, reset all `status = 'processing'` rows to `status = 'pending'`
4. Load all successfully fetched `normalized_url` values into a Python `set` at startup for O(1) pre-fetch dedup check

**Warning signs:**
- `SELECT COUNT(*) FROM articles WHERE status = 'success'` diverges from expected
- On resume, pipeline re-fetches URLs that appear in logs as previously successful

**Phase/component:** Resume manager, DB schema, worker coordinator.

---

## Minor Pitfalls

---

### Pitfall 10: Memory Accumulation Across 12K Articles

**What goes wrong:**
Unreleased HTML strings and BeautifulSoup parse trees (held internally by Newspaper3k) cause RSS memory to grow monotonically over a 12K-URL run.

**Prevention:**
1. After each article is processed, explicitly `del` the `Article` object; call `gc.collect()` every 500 articles
2. Use `concurrent.futures.as_completed()` with a generator — do not store all futures in a list
3. Use a `threading.Semaphore` to limit simultaneously live futures to 2x worker count

**Phase/component:** Worker lifecycle, future management.

---

### Pitfall 11: GDELT URL Quality — Dead Links and Redirect Chains

**What goes wrong:**
GDELT indexes URLs at crawl time. By pipeline run time, 5-15% are dead (404/410) or have permanent redirect chains from CMS migrations.

**Prevention:**
1. Follow redirects; store the final resolved URL
2. Treat 404/410 as permanent failures: `status = 'dead'`, do not retry
3. Treat 5xx as transient: retry with exponential backoff (max 3 attempts)

**Phase/component:** HTTP response handler, retry scheduler.

---

### Pitfall 12: File Handle Exhaustion in Long-Running Processes

**What goes wrong:**
Long-running processes hit the OS file descriptor limit (256 on macOS default) via unclosed log files, DB connections, and temp files.

**Prevention:**
1. Always use `with open(...)` for file operations
2. Set `ulimit -n 4096` before starting the pipeline
3. Use `logging.handlers.RotatingFileHandler` instead of plain file handler

**Phase/component:** Process initialization, logging setup.

---

## Phase-Specific Warning Summary

| Component | Likely Pitfall | Mitigation |
|-----------|---------------|------------|
| DB Schema | No UNIQUE constraint → resume duplicates | `UNIQUE(normalized_url)`, `INSERT OR IGNORE` |
| DB Schema | No `block_reason` column → paywall/bot confusion | Distinguish failure types from day 1 |
| URL Normalization | Raw GDELT URL used as dedup key | Normalize before any insert or resume check |
| DB Connection | Single shared SQLite connection | Per-thread connections + WAL mode + busy timeout |
| Selenium | Uncaught exceptions skip `driver.quit()` | `try/finally` for every driver use |
| Extraction | No post-parse content validation | Length + keyword check after every `.parse()` |
| Date Extraction | Single date source | Multi-source cascade with GDELT date as last resort |
| Text Storage | Encoding not normalized before insert | `charset-normalizer` + null byte strip |
| Resume Logic | `status='processing'` stuck rows | Reset on startup; only set done after commit |
| Long-Run Memory | All futures created upfront | Semaphore-bounded submit + periodic `gc.collect()` |
| Proxy Routing | Free proxies used for paywalled sources | Accept unavailability; use direct for open sources |
| Response Classification | All non-200 treated as paywall | Multi-signal classifier with `block_reason` storage |

---

*Pitfalls research: 2026-02-21*

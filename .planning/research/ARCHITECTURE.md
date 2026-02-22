# Architecture Patterns

**Domain:** Python news article acquisition pipeline (Stage 2 of a 5-stage computational media analysis system)
**Researched:** 2026-02-21

---

## Context

This document covers the Acquisition Layer only. Stage 1 (Discovery) is complete — the GDELT BigQuery export produced `bq-results-*.csv` (~12,622 candidate URLs). Stage 2 reads that CSV, extracts full article text, and writes to SQLite (`articles.db`). Stage 3 (spaCy + AllenNLP NLP processing) consumes `articles.db` directly.

---

## Recommended Architecture

### System Diagram

```
CSV (GDELT export)
        │
        ▼
┌───────────────────────────────────────────┐
│           Orchestrator (main.py)          │
│  - Seeds URL queue from CSV               │
│  - Initialises DB in WAL mode             │
│  - Spawns ThreadPoolExecutor              │
│  - Manages domain rate-limit state        │
│  - Reports progress, handles shutdown     │
└────────────┬──────────────────────────────┘
             │ submits tasks
             ▼
┌───────────────────────────────────────────┐
│        Worker Thread (per URL)            │
│  1. Mark URL → processing                 │
│  2. Call ExtractorChain                   │
│  3. Persist result or failure             │
│  4. Release domain rate-limit slot        │
└──────┬─────────────────┬──────────────────┘
       │                 │
       ▼                 ▼
┌─────────────┐   ┌──────────────────────────┐
│ Newspaper3k │   │    SeleniumPool           │
│  Extractor  │   │  (session-reuse)          │
│  (primary)  │   │  (fallback)               │
└──────┬──────┘   └──────────┬───────────────┘
       │                     │
       │ failed/short text   │ still fails
       └──────────┬──────────┘
                  │
                  ▼
          mark status = failed | skipped
                  │
                  ▼
┌───────────────────────────────────────────┐
│        StateStore (db.py)                 │
│  - urls table   (state machine)           │
│  - articles table (NLP handoff surface)   │
│  WAL mode + serialised writes             │
└───────────────────────────────────────────┘
```

---

## Component Boundaries

| Component | File | Responsibility | Communicates With |
|-----------|------|----------------|-------------------|
| Orchestrator | `main.py` | Entry point: reads CSV, seeds DB, spawns workers, throttles domain slots, handles SIGINT/resume | StateStore, WorkerPool, RateLimiter |
| StateStore | `db.py` | All SQLite I/O; WAL mode; thread-safe write serialisation; schema creation | Orchestrator, Worker threads |
| ExtractorChain | `extractor.py` | Tries Newspaper3k then Selenium; validates text length; returns structured dict or raises | SeleniumPool, ProxyPool |
| Newspaper3kExtractor | `extractor.py` (inner) | Configures Newspaper3k with rotated User-Agent; 15s timeout; returns text or None | ProxyPool (for requests session) |
| SeleniumPool | `browser.py` | Manages N pre-warmed WebDriver instances; leases and returns drivers; retries on stale | ProxyPool |
| ProxyPool | `proxies.py` | Loads, validates, rotates, bans proxies; exposes `get_proxy()` / `report_failure()` | ExtractorChain, SeleniumPool |
| RateLimiter | `ratelimit.py` | Per-domain delay enforcer using a `{domain: last_request_time}` dict + threading.Lock | Orchestrator, Worker threads |

Dependency direction (no circular imports):

```
main.py  →  db.py
main.py  →  ratelimit.py
main.py  →  extractor.py  →  browser.py  →  proxies.py
                           →  proxies.py
```

---

## Data Flow

### Inbound (Discovery → Acquisition)

```
bq-results-*.csv
  columns: url, publish_date, source, themes, tone_scores
        │
        ▼ (on first run, or --resume skips already-done rows)
  urls table: status = 'pending'
```

### Per-URL Processing Flow

```
urls.status = 'pending'
        │
        ▼  (worker picks up)
urls.status = 'processing'
        │
        ├── Newspaper3k attempt
        │     ├── SUCCESS (text ≥ 300 chars)  ──► write to articles table
        │     │                                     urls.status = 'success'
        │     └── FAIL / text < 300 chars
        │               │
        │               ▼
        │           Selenium attempt (with proxy)
        │               ├── SUCCESS            ──► write to articles table
        │               │                           urls.status = 'success'
        │               └── FAIL
        │                         │
        │                         ▼
        │               urls.status = 'failed'
        │               (or 'skipped' for known-paywalled domains
        │                when proxy pool exhausted)
        │
        ▼
   RateLimiter.release(domain)   [always, even on failure]
```

### Outbound (Acquisition → NLP)

Stage 3 reads `articles` where `status = 'success'` (via join on `urls`). The table contract is fixed — Stage 3 must not need to touch `urls`.

---

## State Machine: URL Status Transitions

```
           ┌──────────┐
  (seed)   │  pending │
           └────┬─────┘
                │ worker picks up
                ▼
          ┌────────────┐
          │ processing │  ← reset to 'pending' on unclean shutdown
          └────┬───────┘
               │
      ┌────────┴──────────┐
      ▼                   ▼
 ┌─────────┐        ┌─────────┐
 │ success │        │ failed  │
 └─────────┘        └─────────┘
                          │  (manual re-queue if needed)
                    ┌─────────┐
                    │ skipped │  ← paywall confirmed, no proxies left
                    └─────────┘
```

**Resume logic:** On startup, `UPDATE urls SET status='pending' WHERE status='processing'` — any interrupted run's in-flight rows are safely re-queued. This makes the pipeline idempotent across crashes.

---

## SQLite Concurrency: WAL Mode Strategy

### Problem

`ThreadPoolExecutor` with N workers all writing to the same SQLite file will hit `database is locked` errors under the default journal mode.

### Solution: WAL Mode + Serialised Writes

```python
# db.py — open once in the main thread, share connection via lock

import sqlite3
import threading

class StateStore:
    def __init__(self, db_path: str):
        self._conn = sqlite3.connect(db_path, check_same_thread=False)
        self._conn.execute("PRAGMA journal_mode=WAL")
        self._conn.execute("PRAGMA synchronous=NORMAL")   # WAL-safe tradeoff
        self._conn.execute("PRAGMA busy_timeout=10000")   # 10s retry on lock
        self._lock = threading.Lock()                     # serialise writes

    def write_article(self, data: dict):
        with self._lock:
            self._conn.execute(
                "INSERT OR IGNORE INTO articles (...) VALUES (...)", data
            )
            self._conn.commit()

    def update_status(self, url: str, status: str):
        with self._lock:
            self._conn.execute(
                "UPDATE urls SET status=? WHERE url=?", (status, url)
            )
            self._conn.commit()
```

Key decisions:
- **One connection, shared** — not one per thread. `check_same_thread=False` + explicit `threading.Lock`.
- **WAL mode** — set once at DB creation; persists across sessions.
- **`PRAGMA busy_timeout=10000`** — 10s retry so no thread silently drops a write under contention.
- **`PRAGMA synchronous=NORMAL`** — safe under WAL; halves fsync overhead vs FULL.
- **`INSERT OR IGNORE`** on articles — idempotent re-insertion guard (URL is UNIQUE).

---

## Tiered Extraction: Wiring Newspaper3k → Selenium

```python
# extractor.py

MIN_TEXT_LENGTH = 300

def extract(url: str, proxy: str | None = None) -> dict | None:
    result = _try_newspaper(url, proxy)
    if result and len(result["full_text"]) >= MIN_TEXT_LENGTH:
        return result

    result = _try_selenium(url, proxy)
    if result and len(result["full_text"]) >= MIN_TEXT_LENGTH:
        return result

    return None   # caller marks url as failed
```

The chain is linear. Each extractor returns None on any failure; the wrapper tries the next tier.

---

## Parallel Workers: ThreadPoolExecutor Configuration

```python
WORKER_COUNT = 8   # 5 Newspaper3k + 3 Selenium (SeleniumPool size)
                   # Each Chrome headless ≈ 150MB → 8 workers ≈ 1.2GB total

with ThreadPoolExecutor(max_workers=WORKER_COUNT) as pool:
    futures = {}
    for url_row in db.iter_pending():
        domain = extract_domain(url_row["url"])
        rate_limiter.wait(domain)              # blocks submission, not worker
        fut = pool.submit(process_url, url_row, db, proxy_pool)
        futures[fut] = url_row["url"]

    for fut in as_completed(futures):
        url = futures[fut]
        try:
            fut.result()
        except Exception:
            db.update_status(url, "failed")
```

---

## Domain-Aware Rate Limiting

```python
DOMAIN_DELAYS = {
    "apnews.com":          1.5,
    "reuters.com":         2.0,
    "washingtonpost.com":  4.0,
    "nytimes.com":         5.0,
    "wsj.com":             6.0,
    "_default":            3.0,
}
```

Rate limiter is called in the orchestrator before task submission — serialises requests per domain at submission level, not inside workers.

---

## SQLite Schema for NLP Handoff

```sql
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

CREATE TABLE IF NOT EXISTS articles (
    article_id     INTEGER PRIMARY KEY AUTOINCREMENT,
    url            TEXT UNIQUE NOT NULL REFERENCES urls(url),
    source         TEXT NOT NULL,
    headline       TEXT,
    authors        TEXT,
    publish_date   TEXT,
    full_text      TEXT NOT NULL,
    word_count     INTEGER NOT NULL,
    extraction_ts  TEXT DEFAULT (datetime('now')),
    gdelt_themes   TEXT,
    gdelt_tone     TEXT
);

CREATE INDEX IF NOT EXISTS idx_urls_status     ON urls(status);
CREATE INDEX IF NOT EXISTS idx_urls_source     ON urls(source);
CREATE INDEX IF NOT EXISTS idx_articles_source ON articles(source);
CREATE INDEX IF NOT EXISTS idx_articles_date   ON articles(publish_date);
```

**NLP handoff guarantees:** non-null `full_text`, pre-computed `word_count`, normalised `publish_date` (ISO 8601), stable `article_id` as FK for downstream annotation tables.

---

## Suggested Build Order

```
1. db.py         — schema, WAL mode, write/update/resume methods
2. ratelimit.py  — no project dependencies
3. proxies.py    — no project dependencies
4. extractor.py  — Newspaper3k path only (depends on proxies.py)
5. browser.py    — SeleniumPool (depends on proxies.py)
6. extractor.py  — wire in Selenium fallback (depends on browser.py)
7. main.py       — Orchestrator, integration test on 10-URL sample
8. Full corpus run
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why Bad | Instead |
|---|---|---|
| One SQLite connection per thread | `database is locked` under WAL with concurrent writers | One shared connection + `threading.Lock` |
| `time.sleep()` for page load in Selenium | Wastes time on fast connections; times out on slow proxies | `WebDriverWait` + `ExpectedConditions` |
| Re-creating Selenium drivers per URL | 2-5s startup per URL → 7,600-19,000s overhead on 3,800 Selenium-fallback URLs | `SeleniumPool` — pre-warm N drivers, lease/return |
| Global rate limit (not per-domain) | Too fast for NYT/WSJ, too slow for AP/Reuters | Per-domain delay map in `RateLimiter` |
| Skipping the `urls` state table | On crash, in-progress URLs have no record; re-run can duplicate or gap | Explicit `urls.status` column, reset `processing→pending` on resume |

---

*Architecture analysis: 2026-02-21*

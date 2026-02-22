# Stack Research — Acquisition Pipeline

> Scope: ~12,622 URLs from GDELT export across NYT, Reuters, WaPo, AP News, WSJ.
> Goal: maximize full-text yield into a crash-resumable SQLite store using only free tools.
> Knowledge baseline: Python ecosystem state as of mid-2025.

---

## Recommendation Summary

| Component | Recommended Tool | Version | Confidence |
|---|---|---|---|
| Primary extractor | trafilatura | >=1.12 | HIGH |
| Secondary extractor | newspaper4k | >=0.9 | MEDIUM |
| Paywall fallback | Playwright | >=1.44 | HIGH |
| Browser management | `playwright install` (built-in) | — | HIGH |
| HTTP client | httpx | >=0.27 | HIGH |
| Parallelism | ThreadPoolExecutor | stdlib | HIGH |
| SQLite concurrency | WAL mode + single writer thread | stdlib sqlite3 | HIGH |
| Proxy sourcing | fp-free-proxy | latest | LOW |
| Rate limiting | Custom per-domain token bucket | — | MEDIUM |

---

## Primary Extraction

### The Three Candidates

**newspaper3k — DO NOT USE**
- Last meaningful commit: 2020. PyPI version 0.2.8 released in 2018.
- Python 3.10+ installs fail or produce DeprecationWarnings (uses deprecated `cgi`, `distutils`).
- No active maintainer. Hundreds of open issues, no triage since 2022.
- **Verdict: abandonware. Do not use.**

**newspaper4k — Secondary only**
- Maintained community fork of newspaper3k; fixes Python 3.10/3.11/3.12 compatibility.
- Preserves the newspaper3k API (`Article(url).download(); .parse()`).
- Weakness: extraction quality has not materially improved. Still fails on JS-heavy sites. Body contamination (nav links, footer text) is a known recurring problem.
- **Best role: secondary fallback where trafilatura underperforms.**

**trafilatura — Primary (recommended)**
- Actively maintained by Adrien Barbaresi (Leipzig University). Consistent monthly releases 2023-2025.
- Purpose-built for news and web article extraction. Precision-tuned readability + DOM heuristics.
- Benchmarks (Leipzig NLP group 2022, replicated independently): outperforms newspaper3k, goose3, boilerpy3, and readability-lxml on precision and recall. F1 advantage over newspaper3k: 8-15 points on US/European news sources.
- Supports direct HTML input, URL fetch, metadata extraction (title, author, date, sitename).
- **Pin: `trafilatura>=1.12,<2.0`**

### Extraction Cascade Strategy

```
For each URL:
  1. Fetch raw HTML with httpx (non-JS, fast, ~2-5s per URL)
  2. Run trafilatura.extract(html, include_comments=False, include_tables=False,
                             favor_precision=True)
  3. If trafilatura returns None or len(text) < 150 chars:
       Try newspaper4k as secondary pass on same HTML
  4. If both fail AND domain is in paywall_domains:
       Queue for Playwright fallback (slow path)
  5. If all fail: record status='extraction_failed', skip
```

---

## Paywall Fallback

### Target domains requiring browser automation

- `nytimes.com` — metered paywall, JS-rendered, requires cookies/session
- `washingtonpost.com` — hard paywall after login threshold, JS-rendered
- `wsj.com` — hard subscriber paywall with aggressive Cloudflare bot detection

### Playwright (recommended) vs Selenium

| Factor | Playwright | Selenium |
|--------|-----------|---------|
| Browser management | `playwright install chromium` — automated, zero version mismatch | Manual chromedriver matching (selenium-manager helps but imperfect) |
| Protocol | WebSocket/CDP — lower latency | HTTP WebDriver protocol |
| Async support | Native asyncio API | Blocking; needs executor wrapping |
| Proxy injection | `browser.new_context(proxy={...})` — context-level isolation | Session-level; less clean |
| Stealth | `playwright-stealth` — adequate for NYT/WaPo | `undetected-chromedriver` — roughly equivalent |
| Maintenance velocity | Higher; Microsoft-backed | Established but slower cadence |

**Verdict: Playwright.** Cleaner context isolation per proxy, no chromedriver version mismatch, native async for the paywall queue.

**WSJ caveat:** wsj.com (688 URLs) has Cloudflare Enterprise + Dow Jones proprietary paywall. Expect 25-45% yield even with stealth + proxies.

**Playwright pool pattern:**

```python
import asyncio
from playwright.async_api import async_playwright

PAYWALL_WORKERS = 3  # ~100MB RAM per browser instance

async def fetch_with_playwright(url: str, proxy: dict | None) -> str | None:
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True, proxy=proxy)
        context = await browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
            viewport={"width": 1280, "height": 800},
            locale="en-US",
        )
        page = await context.new_page()
        try:
            await page.goto(url, timeout=30000, wait_until="domcontentloaded")
            return await page.content()
        except Exception:
            return None
        finally:
            await browser.close()
```

Run from a dedicated asyncio event loop in a single daemon thread. The paywall fallback queue feeds this loop.

---

## Parallelism

### Why ThreadPoolExecutor wins

| Model | Verdict | Reason |
|-------|---------|--------|
| asyncio | Wrong tool | trafilatura's lxml parsing and sqlite3 are both blocking; forces awkward executor wrapping |
| multiprocessing | Over-engineered | IPC serialization overhead; complicates single-writer SQLite pattern; I/O wait dominates CPU |
| ThreadPoolExecutor | **Recommended** | I/O-bound work releases GIL; stdlib only; integrates cleanly with single writer thread |

### Recommended concurrency model

```
Main thread
  ├─ ThreadPoolExecutor(max_workers=20)     # httpx fetch + trafilatura extraction
  │    └─ Workers: fetch → extract → put result on writer_queue
  ├─ Writer thread (single, dedicated)
  │    └─ Drains writer_queue → batch INSERT to SQLite
  └─ Playwright asyncio loop thread (single daemon)
       └─ Handles paywall queue, max 3 concurrent Playwright tasks via asyncio.gather
```

**max_workers:** Start at 20. With free proxies and polite rate limits, expect ~200-400 URLs/hour depending on proxy latency.

---

## SQLite Concurrency

### WAL mode pragmas

```python
conn = sqlite3.connect("articles.db", check_same_thread=False)
conn.execute("PRAGMA journal_mode=WAL")
conn.execute("PRAGMA synchronous=NORMAL")     # safe with WAL; much faster than FULL
conn.execute("PRAGMA wal_autocheckpoint=1000")
conn.execute("PRAGMA cache_size=-64000")      # 64 MB page cache
conn.execute("PRAGMA temp_store=MEMORY")
```

`synchronous=NORMAL` under WAL is safe for research pipeline use.

### Single-writer thread pattern

```python
import queue, threading, sqlite3

write_queue: queue.Queue = queue.Queue(maxsize=500)

def writer_thread(db_path: str, stop_event: threading.Event) -> None:
    conn = sqlite3.connect(db_path)
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA synchronous=NORMAL")
    while not stop_event.is_set() or not write_queue.empty():
        batch = []
        try:
            batch.append(write_queue.get(timeout=0.5))
            while len(batch) < 100:
                try: batch.append(write_queue.get_nowait())
                except queue.Empty: break
        except queue.Empty:
            continue
        with conn:
            conn.executemany("INSERT OR REPLACE INTO articles VALUES (?,?,?,?,...)", batch)
    conn.close()
```

Worker threads read from the status table using their own **read-only** connections:
```python
conn = sqlite3.connect(f"file:{db_path}?mode=ro", uri=True)
```

WAL ensures readers never block the writer and vice versa. Write contention is eliminated entirely.

---

## Proxy Management

### Reality check on free proxies

- Typical uptime: 10-30% of listed proxies alive at any given moment
- Major news sites actively block known datacenter IP ranges — free proxies are datacenter IPs
- For NYT/WaPo paywall: proxies help marginally; fingerprinting checks are the real barrier
- For AP/Reuters (open sources): direct requests with rate limiting are more reliable than proxies

### Recommended library

```python
from fp.fp import FreeProxy
# PyPI: fp-free-proxy
# Scrapes free-proxy-list.net and proxyscrape.com
```

Always validate before use — unvalidated proxies waste Playwright startup time.

```python
class ProxyPool:
    def _validate(self, proxy: str) -> bool:
        try:
            r = httpx.head("http://httpbin.org/ip", proxy=proxy, timeout=5)
            return r.status_code == 200
        except Exception:
            return False

    def mark_bad(self, proxy: str) -> None:
        # remove from pool; trigger background refresh if pool < 10
        ...
```

### Per-domain rate limiting

```python
DOMAIN_DELAYS = {
    "nytimes.com":        4.0,
    "washingtonpost.com": 4.0,
    "wsj.com":            6.0,
    "reuters.com":        1.5,
    "apnews.com":         1.5,
}
```

---

## Dependency List

```
# requirements.txt — Acquisition Pipeline
# Python 3.10+

# HTTP
httpx[http2]>=0.27,<1.0
certifi>=2024.8.30

# Extraction (primary)
trafilatura>=1.12,<2.0
lxml>=5.2,<6.0
lxml_html_clean>=0.1        # required by trafilatura >=1.8

# Extraction (secondary)
newspaper4k>=0.9,<1.0
nltk>=3.8,<4.0

# Paywall fallback
playwright>=1.44,<2.0
playwright-stealth>=1.0

# Proxy management
fp-free-proxy>=1.2,<2.0

# Utilities
tqdm>=4.66
python-dateutil>=2.9
tenacity>=8.3,<9.0
charset-normalizer>=3.3
```

Post-install:
```bash
playwright install chromium
python -m nltk.downloader punkt stopwords
```

---

## What NOT to Use

| Tool | Reason |
|------|--------|
| newspaper3k | Abandonware; broken on Python 3.10+; use trafilatura or newspaper4k |
| Scrapy | Full crawl framework overkill for a flat URL list; ThreadPoolExecutor + httpx is sufficient |
| Requests-HTML | Abandoned ~2020; wraps old pyppeteer; Playwright is its successor |
| aiohttp | Forces pure async architecture; httpx covers both sync and async cleanly |
| multiprocessing.Pool | No throughput gain over threads for I/O-bound work; complicates SQLite writer pattern |
| Unvalidated free proxy lists | 60-80% dead; always validate before pooling |

---

## Realistic Yield Estimate

| Source | URLs | Strategy | Expected Yield |
|--------|------|----------|----------------|
| reuters.com | 2,995 | httpx + trafilatura | 90-95% |
| apnews.com | 1,953 | httpx + trafilatura | 85-92% |
| nytimes.com | 4,337 | trafilatura (cached/AMP) + Playwright fallback | 55-70% |
| washingtonpost.com | 2,423 | trafilatura + Playwright fallback | 50-65% |
| wsj.com | 688 | Playwright + stealth + proxies | 25-45% |
| **Total** | **12,622** | | **~65-75% overall** |

Reuters and AP (~4,948 URLs, 39% of corpus) are the reliable baseline. Extract them first.

---

## Key Deviations from Blueprint

The TWAIL_Pipeline_Blueprint.md specifies newspaper3k + Selenium. Research reveals better options:

| Blueprint Spec | Research Recommendation | Reason |
|---|---|---|
| newspaper3k | trafilatura (primary) + newspaper4k (secondary) | newspaper3k is abandonware (2018); trafilatura has 8-15pt F1 advantage |
| Selenium | Playwright | Cleaner proxy isolation, automated browser management, native async |
| requests (implicit) | httpx | Async-capable, HTTP/2 support, cleaner API |

---

## Confidence Assessment

| Recommendation | Confidence |
|---|---|
| trafilatura as primary | HIGH — multiple independent benchmarks, active maintenance |
| newspaper4k as secondary | MEDIUM — working fork, modest quality improvement |
| Playwright over Selenium | HIGH — better DX, faster protocol, cleaner isolation |
| ThreadPoolExecutor | HIGH — standard pattern for I/O-bound work |
| WAL + single writer thread | HIGH — well-documented, eliminates all lock contention |
| fp-free-proxy | LOW — free proxies fundamentally unreliable; library adequate |
| WSJ yield estimate | LOW — hard paywall + Cloudflare; highly variable |

---

*Stack research: 2026-02-21*

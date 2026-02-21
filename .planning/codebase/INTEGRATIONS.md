# External Integrations

**Analysis Date:** 2026-02-21

## APIs & External Services

**GDELT (Global Database of Events, Language, and Tone):**
- Service: Event monitoring and media analysis via Google's public dataset
- What it's used for: Discovery phase - harvesting URLs of relevant articles about Israel-Gaza conflict
  - SDK/Client: Google Cloud BigQuery
  - Auth: Google Cloud credentials (service account or user authentication)
  - Query endpoint: `gdelt-bq.gdeltv2.gkg_partitioned` table in BigQuery

**News Source APIs:**
- Reuters (reuters.com) - Wire service with high article availability (~90% success rate with Newspaper3k)
- AP News (apnews.com) - Wire service with very high availability (~95% success rate)
- AFP/France24 (france24.com/afp) - Wire service with good availability (~85% success rate)
- New York Times (nytimes.com) - Legacy print; low success without advanced techniques (~25% naive, 70% with proxies)
- Washington Post (washingtonpost.com) - Legacy print; moderate difficulty (~40% naive, 80% with proxies)
- Wall Street Journal (wsj.com) - Legacy print; most challenging (~15% naive, 55% with proxies)

**Proxy Service (Optional):**
- Residential proxy rotation for bypassing IP-based rate limiting and paywalls
- Configuration: Rotating user agents + proxy pool for tertiary extraction fallback chain

## Data Storage

**Databases:**

**SQLite3 (Local):**
- `articles.db` - Stores full-text articles with metadata
  - Tables: articles, sentences, agency_annotations, causal_annotations, terminology_annotations, validation_sample
  - Connection: File-based, no network required
  - Client: Python sqlite3 (stdlib) + manual SQL execution

**Google BigQuery (Cloud):**
- `gdelt-bq.gdeltv2.gkg_partitioned` - Read-only access to GDELT Global Knowledge Graph
- Connection: Google Cloud credential configuration
- Purpose: Querying event metadata and URLs for the discovery phase

**File Storage:**
- Local filesystem CSV exports: `gdelt_export.csv`, `urls.csv`, `validation_set.csv`, `metrics.json`
- Large dataset file: `bq-results-20260221-034608-1771647281081.csv` (32MB+, 12,623 rows from BigQuery)

**Caching:**
- None detected; data flows through pipeline sequentially without explicit caching layer

## Authentication & Identity

**Auth Provider:**
- Custom implementation with Google Cloud credentials for BigQuery access

**Implementation:**
- Google Cloud service account key or user authentication for GDELT dataset access
- User-Agent rotation in Newspaper3k and Selenium for HTTP request authentication (mimicking browser behavior)
- No user login/identity system; single-user research workflow

## Monitoring & Observability

**Error Tracking:**
- None detected; errors captured via Python try-catch blocks and logged to stdout

**Logs:**
- Console logging via Python print statements (e.g., "Extraction failed for {url}: {e}")
- Log output: Print to console during article extraction failures

**Metrics:**
- Computed metrics stored in `metrics.json` at pipeline completion
- Inter-rater reliability computed via Cohen's Kappa (target κ > 0.70)
- Precision/Recall against validation sample for automated labels

## CI/CD & Deployment

**Hosting:**
- Local research environment; no deployment infrastructure detected
- Designed as sequential 5-stage research pipeline (1 week per stage)
- BigQuery used for data sourcing, not continuous hosting

**CI Pipeline:**
- None detected; manual Python script execution model

## Environment Configuration

**Required env vars:**
- Google Cloud credentials (project ID, service account JSON path, or credential token for BigQuery access)
- Proxy configuration (proxy server URL, port, authentication if using rotating residential proxies)
- No other environment variables detected in blueprint

**Secrets location:**
- Google Cloud credentials configuration (via `GOOGLE_APPLICATION_CREDENTIALS` environment variable or ADC)
- Proxy credentials (if applicable)
- No .env file detected in codebase

## Webhooks & Callbacks

**Incoming:**
- None detected; uni-directional data flow from GDELT → acquisition → processing → analysis

**Outgoing:**
- None detected; outputs are local database and report files only

## Data Pipeline Flow

**Discovery → Acquisition → Processing → Validation → Analysis:**

1. **Discovery (Week 1):**
   - BigQuery SQL query to `gdelt-bq.gdeltv2.gkg_partitioned`
   - Filters by date range (2023-10-07 to 2025-02-21), sources, and themes
   - Output: `urls.csv` (~10K candidate URLs after deduplication)

2. **Acquisition (Week 2):**
   - Newspaper3k attempts extraction from each URL
   - Fallback chain: Selenium + residential proxies for paywalled sources
   - Optional: LexisNexis Academic/Factiva export as legal alternative
   - Output: `articles.db` SQLite database (2,000+ articles)

3. **Processing (Weeks 3-4):**
   - spaCy dependency parsing → agency extraction (nsubj, nsubjpass, agent relations)
   - AllenNLP SRL → causal linkage extraction (ARG0, ARG1 identification)
   - Custom lexicon matching → terminology asymmetry scoring
   - Output: `annotations.db` with three annotation tables

4. **Validation (Week 5):**
   - Manual coding of 150-sentence stratified sample by human coders
   - Cohen's Kappa inter-rater agreement calculation
   - Output: `validation_set.csv` with human gold-standard labels

5. **Analysis (Week 6):**
   - Statistical tests (Chi-square, time-series) across sources and time periods
   - Calibration adjustment based on validation error rates
   - Output: `metrics.json` + visual report (PDF)

---

*Integration audit: 2026-02-21*

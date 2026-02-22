# Roadmap: Israel-Gaza NLP — Acquisition Layer

## Overview

This pipeline ingests ~12,622 GDELT-sourced URLs across five major news outlets and extracts full article text into a crash-resumable SQLite database for downstream NLP processing. The build order follows the natural dependency chain: state store first (nothing else is safe without it), open-access sources next (AP + Reuters validate the extraction chain before paywall complexity is introduced), the Playwright paywall path after (NYT + WaPo are more tractable than WSJ), and WSJ last with corpus deduplication and final metrics as the handoff artifact for Stage 3.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - Schema, state tracking, URL normalization, and crash-resume infrastructure
- [ ] **Phase 2: Open Source Extraction** - Full extraction pipeline validated on AP + Reuters (~4,948 open-access URLs)
- [ ] **Phase 3: Paywall Path** - Playwright pool + proxy management for NYT + WaPo (~6,760 URLs)
- [ ] **Phase 4: WSJ + Corpus Validation** - WSJ attempt, content hash dedup, final metrics handoff to NLP

## Phase Details

### Phase 1: Foundation
**Goal**: A working SQLite state store with full schema, URL normalization, and crash-resume logic that all subsequent phases depend on
**Depends on**: Nothing (first phase)
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-04, FOUND-05, FOUND-06
**Success Criteria** (what must be TRUE):
  1. Running the pipeline twice on the same CSV produces the same `urls` table row count with no duplicate normalized URLs
  2. URLs matching non-allowlisted domains, non-prose path patterns, or AMP/UTM variants are absent from the `urls` table after CSV load
  3. Killing the process mid-run and restarting shows all previously `success` rows remain `success`; any `processing` rows are reset to `pending` and retried
  4. The SQLite database opens in WAL mode and a concurrent read completes without blocking while the writer thread is active
  5. The `urls` table contains `status`, `attempt_count`, and `error_message` columns and enforces a UNIQUE constraint on `normalized_url`
**Plans**: TBD

### Phase 2: Open Source Extraction
**Goal**: AP and Reuters articles extracted at 85-95% yield with validated text quality, structured error codes, and a running corpus that can survive a crash at any URL
**Depends on**: Phase 1
**Requirements**: EXTR-01, EXTR-02, EXTR-03, EXTR-04, EXTR-05, EXTR-06, QUAL-02, QUAL-03
**Success Criteria** (what must be TRUE):
  1. Running extraction on a 50-URL AP/Reuters sample produces at least 42 rows with `status='success'` and `len(full_text) >= 300`
  2. No `full_text` column contains null bytes, garbled encoding, or unresolved HTML entities after insertion
  3. Publication date is populated from JSON-LD or OpenGraph metadata when available; GDELT date is used only as last resort and flagged when it diverges from extracted date by more than 7 days
  4. Each URL in the `urls` table carries a distinct error code (`paywall_suspected`, `error_parse`, `error_network`) rather than a generic failure flag
  5. Extraction runs with 20 concurrent workers; per-domain request timing for AP and Reuters stays at or above the configured 1.5-2s floor
**Plans**: TBD

### Phase 3: Paywall Path
**Goal**: A Playwright-backed browser pool with proxy validation and health tracking that extracts NYT and WaPo articles paywalled against direct HTTP fetches
**Depends on**: Phase 2
**Requirements**: PWALL-01, PWALL-02, PWALL-03, PWALL-04
**Success Criteria** (what must be TRUE):
  1. A 20-URL NYT smoke test produces at least 10 `status='success'` rows from the Playwright path after trafilatura + newspaper4k both return fewer than 150 chars
  2. Dead proxies (those returning non-200 or timing out in 5s) are absent from the active proxy pool before any paywall URL is attempted
  3. A proxy that accumulates 3 consecutive failures is marked banned and replaced without manual intervention; the pool triggers a background refresh when active proxy count drops below 10
  4. Each Playwright browser context is isolated with its own proxy, cookie jar, and session state — two concurrent paywall fetches do not share cookies or authentication state
**Plans**: TBD

### Phase 4: WSJ + Corpus Validation
**Goal**: WSJ extraction attempted at calibrated expectations, duplicate content removed across all sources, and a final articles.db with corpus metrics ready for Stage 3 NLP handoff
**Depends on**: Phase 3
**Requirements**: QUAL-01, QUAL-04
**Success Criteria** (what must be TRUE):
  1. A 5-URL WSJ smoke test runs without error; overall WSJ yield is recorded and accepted (25-45% is the expected ceiling given Cloudflare Enterprise)
  2. AP articles syndicated to WaPo with identical content appear as a single row in `articles`; duplicate URLs carry `status='duplicate'` in the `urls` table rather than a second `full_text` insert
  3. Pipeline completion prints a metrics report showing `COUNT(*)` by source and status, total success rate, and date range coverage
  4. Rotating log file on disk contains a timestamped entry for every URL outcome with extractor used and failure reason; no outcome is silent
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/TBD | Not started | - |
| 2. Open Source Extraction | 0/TBD | Not started | - |
| 3. Paywall Path | 0/TBD | Not started | - |
| 4. WSJ + Corpus Validation | 0/TBD | Not started | - |

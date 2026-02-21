# Architecture

**Analysis Date:** 2026-02-21

## Pattern Overview

**Overall:** Five-stage data processing pipeline with NLP annotation layers and human validation feedback loops

**Key Characteristics:**
- Data flows through specialized layers (Discovery → Acquisition → Processing → Validation → Analysis)
- Each layer produces database artifacts that feed downstream stages
- NLP processing operates in parallel streams (syntactical, semantic, lexical)
- Validation stage provides error estimates to calibrate automated metrics
- Statistical analysis produces bias measurements with confidence bounds

## Layers

**Discovery Layer:**
- Purpose: Identify candidate articles using GDELT database
- Location: Query execution on Google BigQuery
- Contains: SQL filtering queries, URL harvesting logic
- Depends on: GDELT GKG dataset, Google Cloud services
- Used by: Acquisition layer (receives deduplicated URL list)
- Output: CSV file with ~10,000 candidate URLs filtered by source and theme

**Acquisition Layer:**
- Purpose: Extract full-text article content from heterogeneous sources
- Location: Implementation in `scraper.py` (planned)
- Contains: Newspaper3k extraction, Selenium fallback chains, proxy rotation, LexisNexis integration
- Depends on: Articles table in SQLite, user agents library
- Used by: Processing layer
- Output: `articles` table populated with full text, metadata, word counts

**Processing Layer:**
- Purpose: Apply three parallel NLP extraction streams
- Location: Three modules (`agency_extractor.py`, `causal_extractor.py`, `terminology_extractor.py`)
- Contains: spaCy dependency parsing, AllenNLP SRL, custom lexicon matching
- Depends on: Articles and sentences tables, spaCy en_core_web_lg model, AllenNLP SRL model
- Used by: Validation and analysis layers
- Output: Three annotation tables (`agency_annotations`, `causal_annotations`, `terminology_annotations`)

**Validation Layer:**
- Purpose: Provide ground truth labels for error estimation and calibration
- Location: Human coding interface (planned)
- Contains: Stratified sampling protocol, coding interface, inter-rater reliability calculation
- Depends on: Processing layer outputs
- Used by: Analysis layer (correction factors)
- Output: `validation_sample` table with human-coded labels

**Analysis Layer:**
- Purpose: Compute bias metrics with validation-adjusted confidence intervals
- Location: Statistical analysis module (planned)
- Contains: Chi-square tests, time-series analysis, bootstrapped confidence intervals, calibration adjustment
- Depends on: All annotation tables, validation results
- Used by: Reporting
- Output: metrics.json, visualizations, statistical report

## Data Flow

**Article Discovery and Acquisition:**

1. BigQuery SQL query filters GDELT GKG on source and theme (Oct 7, 2023 - Dec 31, 2025)
2. CSV export deduplicated on normalized URL (removes syndication duplicates)
3. Articles excluded if they match video/podcast/interactive patterns
4. Remaining URLs passed to acquisition layer (~10K candidates expected)
5. Newspaper3k attempts extraction with rotating user agents
6. Failed extractions routed to Selenium + proxy fallback
7. Final fallback to LexisNexis Academic/Factiva if available
8. Extracted articles stored in `articles` table with source, headline, authors, publish_date, full_text, word_count

**NLP Processing Pipeline:**

1. Full article text split into sentences and stored in `sentences` table
2. Dependency parsing (spaCy) extracts subject-verb-voice triples → `agency_annotations` table
3. Semantic role labeling (AllenNLP) extracts agent-patient-verb relationships → `causal_annotations` table
4. Lexicon matching (PhraseMatcher) finds emotive/sanitized terms with proximity → `terminology_annotations` table
5. Each annotation records the sentence_id for traceability

**Validation and Calibration:**

1. Stratified sample of 150 sentences drawn from processing results (50 wire services, 50 legacy print, 30 high-casualty, 20 diplomatic)
2. Human coders independently label agency, causal type, and terminology using coding interface
3. Inter-rater agreement calculated (Cohen's Kappa target > 0.70)
4. Confusion matrices identify systematic errors (e.g., SRL misses passive constructions)
5. Correction factors computed: if SRL misclassifies 15% of OBSCURED as IMPLICIT, adjust future results
6. Calibrated metrics = raw metrics ± (error_rate × raw_count)

**Analysis and Reporting:**

1. Agency Ratio = (ISR_active + PAL_passive) / (PAL_active + ISR_passive)
2. Causal Explicitness Index = EXPLICIT_ISR / (EXPLICIT_ISR + OBSCURED_ISR)
3. Terminology Asymmetry = EMOTIVE_PAL / EMOTIVE_ISR
4. Chi-square tests compare distributions across sources
5. Time-series analysis tracks metrics across conflict phases
6. Bootstrap confidence intervals adjusted for validation error rates
7. Results published to metrics.json and report.pdf

**State Management:**

- SQLite database serves as single source of truth for all processing state
- Tables indexed on foreign keys (article_id, sentence_id) for relational queries
- Each annotation record is immutable once inserted
- Validation data linked to processing results via sentence_id for error analysis

## Key Abstractions

**Entity Classification:**

- Purpose: Normalize entity references to three categories (ISR, PAL, OTHER)
- Pattern: Dictionary-based pattern matching on lemmatized text
- ISR patterns: israel, israeli, idf, netanyahu, tel aviv, jerusalem, knesset, mossad, shin bet, tsahal
- PAL patterns: palestinian, palestinians, gaza, hamas, west bank, fatah, plo, ramallah, jenin, rafah, khan younis
- Used in: agency_extractor.classify_entity(), causal_extractor.classify_entity()
- Limitation: Lexicon-based; may miss metonymy, indirect references, or context-dependent meanings

**Voice Classification:**

- Purpose: Distinguish active vs. passive construction to measure agency asymmetry
- Active voice: subject performs action (nsubj dependency)
- Passive voice: subject receives action (nsubjpass dependency)
- Agent preservation: passive with explicit "by" agent (agent dependency)
- Agentless passive: passive with implicit actor (nsubjpass without agent)

**Causal Type Classification:**

- Purpose: Quantify how explicitly causal relationships are stated
- EXPLICIT: Both agent and patient identified in SRL frame (who did what to whom)
- IMPLICIT: Agent identified but causal link inferred rather than stated
- OBSCURED: No clear agent identified; passive construction obscures actor

**Lexicon Categories:**

- Purpose: Measure word choice asymmetry in describing same events
- EMOTIVE: Humanizing language (massacre, slaughter, murder, atrocity, children, victims, innocent, brutal)
- SANITIZED: Technical/military language (strike, operation, precision, surgical, neutralize, collateral, engagement, response)
- NEUTRAL: Unbiased descriptors
- Constructed iteratively from corpus observations rather than a priori lexicon

## Entry Points

**Discovery Entry:**
- Location: BigQuery console or script that executes SQL template
- Triggers: Manual execution on /Users/fesalfayed/Desktop/israel-gaza NLP directory
- Responsibilities: Query GDELT GKG, export to CSV, deduplicate
- Configuration: SQL WHERE clause specifies sources (NYT, WaPo, WSJ, AP, Reuters, AFP) and themes (GAZA, ISRAEL, PALESTINIAN, HAMAS)
- Date range hardcoded: Oct 7, 2023 - Dec 31, 2025

**Acquisition Entry:**
- Location: `scraper.py` (planned)
- Triggers: Called with CSV URL list from Discovery stage
- Responsibilities: Iterate URLs, apply extraction pipeline, validate output, insert into SQLite
- Configuration: USER_AGENTS list, request_timeout (15 sec), minimum text length (300 chars)
- Output validation: Rejects extractions <300 chars (indicates paywall/failure)

**Processing Entry:**
- Location: Three separate scripts for each NLP stream
- Triggers: Called after articles table is populated
- Responsibilities: Iterate articles, load sentences, run extraction pipeline, insert annotations
- Models required: spacy en_core_web_lg (170 MB), AllenNLP SRL (500 MB)
- Processing time: ~2 sec/article CPU, ~0.3 sec/article GPU

**Validation Entry:**
- Location: Coding interface (planned - web form or CSV template)
- Triggers: Manual process after processing complete
- Responsibilities: Present stratified sentences to coders, collect labels, calculate agreement
- Coders needed: 2-3 for inter-rater reliability; target Cohen's Kappa > 0.70

**Analysis Entry:**
- Location: Statistical analysis module (planned)
- Triggers: Called after validation complete
- Responsibilities: Compute primary metrics, run statistical tests, apply calibration, generate report
- Dependencies: scipy, statsmodels, matplotlib, seaborn

## Error Handling

**Strategy:** Tiered fallback with logging and graceful degradation

**Patterns:**

- **Discovery:** SQL query execution on BigQuery; if connection fails, manual export from console
- **Acquisition:** Primary Newspaper3k → Selenium → Proxy rotation → LexisNexis; log each failure; skip articles that exhaust all fallbacks; minimum 2,000 articles required to proceed
- **Processing:** NLP models load with try-except; if model fails, skip article; log error with URL for manual inspection later
- **Validation:** If human disagreement (Cohen's Kappa < 0.70), request re-coding or adjudication; do not proceed with calibration until agreement > 0.70
- **Analysis:** Confidence intervals calculated with bootstrap resampling; if convergence fails after 10,000 iterations, return NaN bounds and flag result as uncertain

**Validation Constraints:**

- Minimum 150 sentences required for error estimation (7-10% of casualty-relevant corpus)
- If stratification protocol violated (e.g., <50 wire service sentences), re-sample to meet quotas
- If agreement < 0.60 on any variable, investigate coder calibration or annotation guidelines

## Cross-Cutting Concerns

**Logging:**
- Approach: Python logging module with file sink to `pipeline.log`
- Events logged: article extraction start/failure, NLP model load, annotation insertion, validation disagreement
- Level: INFO for stage progress, ERROR for failures, DEBUG for per-sentence processing

**Validation:**
- Approach: Input validation at each stage boundary
- Articles: required fields (source, url, full_text, publish_date); text length > 300 chars
- Sentences: required fields (article_id, sentence_idx, sentence_text); non-empty text
- Annotations: foreign key constraints enforced; entity classification must be ISR/PAL/OTHER/UNKNOWN

**Authentication:**
- Approach: Google Cloud IAM for BigQuery access; LexisNexis credentials in environment variables (never hardcoded)
- Configuration: GOOGLE_APPLICATION_CREDENTIALS points to service account JSON; LEXISNEXIS_USERNAME, LEXISNEXIS_PASSWORD in .env
- Rotation: Residential proxies configured via environment variables; user agent list in config file

---

*Architecture analysis: 2026-02-21*

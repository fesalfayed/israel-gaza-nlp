# Israel-Gaza NLP — Acquisition Layer

## What This Is

A Python pipeline that takes a GDELT BigQuery URL export (~12,622 candidate URLs from NYT, WaPo, WSJ, AP, and Reuters covering the Israel-Gaza conflict from October 2023 to February 2026) and extracts full-text article content into a SQLite database for downstream NLP analysis. This is the second stage (Acquisition Layer) of a five-stage computational TWAIL pipeline measuring linguistic bias in Western media framing.

## Core Value

Produce the largest feasible corpus of full-text articles in a resumable, structured SQLite database that the NLP processing layer can act on without data quality issues.

## Requirements

### Validated

- ✓ GDELT BigQuery export completed — `bq-results-20260221-034608-1771647281081.csv` (~12,622 URLs)
- ✓ Newspaper3k article download confirmed working on Reuters article (`test.py`)

### Active

- [ ] Load and deduplicate URLs from GDELT CSV export
- [ ] Filter URLs to allowlisted sources only (nytimes.com, reuters.com, washingtonpost.com, apnews.com, wsj.com)
- [ ] Initialize SQLite database with full schema (articles, sentences tables per blueprint)
- [ ] Extract article full-text via Newspaper3k with rotating user-agents (primary path)
- [ ] Fall back to Selenium + free proxy lists for paywalled sources (NYT, WaPo, WSJ)
- [ ] Track extraction status per URL in SQLite for crash-safe resumability
- [ ] Validate extracted articles (min 300 chars, required fields present)
- [ ] Run parallel extraction workers with per-domain rate limiting
- [ ] Log all extraction outcomes (success, failure reason, fallback used)
- [ ] Produce final corpus metrics report (articles by source, success rates, date range coverage)

### Out of Scope

- AFP (france24.com) — zero results in GDELT export, dropped from this milestone
- LexisNexis / Factiva integration — no institutional access available
- Paid proxy providers — using free proxy lists only
- NLP processing (spaCy, AllenNLP) — handled in Phase 3 (Processing Layer)
- Human validation interface — handled in Phase 4
- Statistical analysis and reporting — handled in Phase 5

## Context

- **Existing code**: `test.py` demonstrates basic Newspaper3k download on a Reuters URL
- **Input data**: `bq-results-20260221-034608-1771647281081.csv` — columns: url, publish_date, source, themes, tone_scores
- **Source distribution**: NYT 4,337 | Reuters 2,995 | WaPo 2,423 | AP 1,953 | WSJ 688 | noise 226 (to be filtered)
- **Paywall challenge**: NYT/WaPo/WSJ require Selenium fallback; AP/Reuters are mostly open (~90-98% success with Newspaper3k)
- **Blueprint reference**: `TWAIL_Pipeline_Blueprint.md` defines full schema, tiered extraction logic, and validation rules

## Constraints

- **Tech stack**: Python 3.10+, SQLite (stdlib), Newspaper3k, Selenium, pandas — matches blueprint
- **Proxies**: Free proxy lists only — expect lower success rates on paywalled sources vs. residential proxies
- **Minimum text length**: 300 characters per blueprint — articles below this threshold are discarded
- **Resumability**: SQLite must track processed URLs so pipeline can restart without re-processing
- **No NLP models**: AllenNLP and spaCy are NOT loaded during acquisition — pure text extraction only

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Allowlist source filtering | Removes kelownacapnews.com, wnewsj.com, thomsonreuters.com noise from GDELT false positives | — Pending |
| Selenium + free proxies for paywalls | No paid proxy budget; acceptable for research-grade pipeline | — Pending |
| Drop AFP | Zero results in current GDELT export; AP + Reuters cover wire service framing adequately | — Pending |
| Maximum feasible corpus target | Extract everything reachable rather than stopping at a floor count | — Pending |
| Parallel ThreadPool extraction | 12K URLs would take many hours sequentially; parallel workers with domain rate limits | — Pending |

---
*Last updated: 2026-02-21 after initialization*

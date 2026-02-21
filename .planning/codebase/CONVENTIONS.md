# Coding Conventions

**Analysis Date:** 2026-02-21

## Overview

This codebase follows Python conventions as specified in the TWAIL Pipeline Blueprint. The project implements a computational linguistics pipeline for media bias analysis. Code is organized around five operational stages: Discovery, Acquisition, Processing, Validation, and Analysis.

## Naming Patterns

**Files:**
- Lowercase with underscores for multi-word modules
- Pattern: `{stage}_{component}.py`
- Examples from blueprint:
  - `scraper.py` - Acquisition layer
  - `agency_extractor.py` - Processing layer
  - `causal_extractor.py` - Processing layer
  - `terminology_extractor.py` - Processing layer
  - `schema.sql` - Database initialization

**Functions:**
- Lowercase with underscores (snake_case)
- Verb-first naming for actions: `extract_article()`, `extract_agency()`, `extract_causal_roles()`, `find_proximate_entity()`
- Internal/private functions prefixed with single underscore: `_replace()`
- Pattern observed in test file: `urlparse()` for utility functions

**Variables:**
- Lowercase with underscores for multi-word: `article`, `full_text`, `word_count`, `sentence_idx`
- Constants in UPPERCASE: `USER_AGENTS`, `ISR_PATTERNS`, `PAL_PATTERNS`, `EMOTIVE_TERMS`, `SANITIZED_TERMS`
- Loop variables single letter or short: `e` (for exceptions), `p` (for patterns)
- Dictionary keys as simple identifiers: `url`, `headline`, `authors`, `publish_date`, `full_text`, `word_count`

**Types/Classes:**
- Not shown in current codebase (blueprint is pseudocode), but following Python convention: PascalCase
- Expected example: `Article` (from newspaper3k library)

## Code Style

**Formatting:**
- No explicit formatter configured yet. Follow PEP 8 standard
- 4-space indentation (observed in blueprint code examples)
- Line length target: 88 characters (implicit from blueprint structure)
- Blank lines: 2 between top-level functions, 1 between methods

**Linting:**
- Not yet configured. Recommendation: Use flake8 for PEP 8 compliance
- Configuration file location (when added): `.flake8` at project root

**Comments:**
- Inline comments for complex logic (seen in blueprint)
- Pattern: `# Comment describing the next section`
- Examples:
  - `# Validate extraction quality`
  - `# Get the full noun phrase`
  - `# Determine causal type`
  - `# Extract ARG0 (agent) and ARG1 (patient)`

**Function Documentation:**
- Triple-quoted docstrings describing purpose and return type
- Pattern observed:
  ```python
  def extract_article(url: str) -> dict | None:
      '''Attempt extraction with Newspaper3k, return structured dict'''
  ```
- Format: One-line summary of function purpose
- Type hints included for function parameters and return values

## Import Organization

**Order (observed from blueprint):**
1. Standard library imports: `warnings`, `sqlite3`, `time`, `random`
2. Third-party library imports: `newspaper`, `spacy`, `allennlp`, `pandas`, `numpy`, `scipy`, `statsmodels`, `matplotlib`, `seaborn`
3. Local application imports: (not yet present; pattern TBD)

**Path Aliases:**
- Not currently used. When introduced, follow pattern: `from {module} import {name}`
- Example from test.py: `from urllib.parse import urlparse`

**Specific imports preferred over wildcard:**
- Pattern: `from module import specific_class_or_function`
- Never: `from module import *`

## Error Handling

**Patterns:**
- Broad exception handling with logging followed by graceful return
- Pattern observed in `scraper.py`:
  ```python
  try:
      # Implementation
  except Exception as e:
      print(f'Extraction failed for {url}: {e}')
      return None
  ```
- Validation checks before processing:
  ```python
  if len(article.text) < 300:
      return None  # Likely paywall or extraction failure
  ```
- Silent failures acceptable for edge cases (e.g., article extraction failures)
- Callers expected to check for None returns

## Database Conventions

**Schema naming:**
- Tables: lowercase with underscores, descriptive plural or singular
  - `articles`, `sentences`, `agency_annotations`, `causal_annotations`, `terminology_annotations`, `validation_sample`
- Columns: lowercase with underscores
  - Primary keys: `{table_name_singular}_id` (e.g., `article_id`, `sentence_id`)
  - Foreign keys: `{referenced_table_singular}_id` (e.g., `article_id`, `sentence_id`)
  - Boolean flags: `is_{description}` (e.g., `is_casualty_relevant`)
  - Timestamps: `{action}_date` or `{action}_datetime` (e.g., `extraction_date`, `coded_date`)

**Index naming:**
- Pattern: `idx_{table}_{field}` or `idx_{table}_{fields}`
- Examples: `idx_articles_source`, `idx_articles_date`, `idx_sentences_article`

## Data Structure Patterns

**Dictionary returns from extraction functions:**
- Keys use lowercase underscores
- Structured flat dictionaries for single results
- Examples from blueprint:
  ```python
  {
      'url': url,
      'headline': article.title,
      'authors': ', '.join(article.authors),
      'publish_date': str(article.publish_date),
      'full_text': article.text,
      'word_count': len(article.text.split())
  }
  ```

**List returns for multiple extractions:**
- Each item is a dictionary following same convention
- Pattern: `List[Dict]` with typed returns
- Examples: `List[Dict]` for agency, causal, and terminology extractions

## Configuration

**Pattern:**
- Hardcoded lists for static data (user agents, entity patterns, lexicons)
- Named constants at module level
- Examples:
  - `USER_AGENTS` - List of browser user agent strings
  - `ISR_PATTERNS` - Set of Israeli entity identifiers
  - `PAL_PATTERNS` - Set of Palestinian entity identifiers
  - `EMOTIVE_TERMS` - List of emotive lexicon terms
  - `SANITIZED_TERMS` - List of sanitized lexicon terms

**Model/resource loading:**
- Singleton pattern for spaCy model: `nlp = spacy.load('en_core_web_lg')`
- AllenNLP models loaded via URL path
- Pattern: Module-level initialization for performance

## Type Hints

**Usage:**
- Function parameters and return types annotated
- Pattern: `(parameter: type) -> return_type`
- Union types for nullable returns: `dict | None` (Python 3.10+ syntax)
- Collection types: `List[Dict]` with typing imports expected

**Not enforced yet:**
- No mypy configuration detected
- Type hints present in blueprint code as best practice

## Logging

**Current approach:**
- Use `print()` for errors and status messages
- Pattern: `print(f'Message: {variable}')`
- Example: `print(f'Extraction failed for {url}: {e}')`

**Recommended upgrade path:**
- Migrate to Python `logging` module when project scales
- Use log levels: DEBUG, INFO, WARNING, ERROR, CRITICAL

## Code Organization by Stage

**Discovery Layer:**
- Handles GDELT BigQuery queries
- Output: `urls.csv`
- Deduplication logic in pandas

**Acquisition Layer:**
- Module: `scraper.py`
- `extract_article(url: str) -> dict | None` function
- Fallback chain: Newspaper3k → Selenium → Proxy rotation → LexisNexis

**Processing Layer:**
- Module: `agency_extractor.py` - Dependency parsing with spaCy
- Module: `causal_extractor.py` - SRL with AllenNLP
- Module: `terminology_extractor.py` - Lexicon matching with spaCy
- Shared utility: `classify_entity(text: str) -> str`

**Validation Layer:**
- Human coding interface (UI component, not yet implemented)
- Agreement metrics calculation

**Analysis Layer:**
- Statistical computations (scipy, statsmodels)
- Visualization (matplotlib, seaborn)

---

*Convention analysis: 2026-02-21*

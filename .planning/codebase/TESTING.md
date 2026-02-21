# Testing Patterns

**Analysis Date:** 2026-02-21

## Overview

Testing strategy for this project spans two distinct domains:

1. **Automated Testing:** Unit and integration tests for extraction pipelines
2. **Validation Testing:** Human-coded gold standard samples for NLP accuracy measurement

Currently, the project has minimal automated test infrastructure. Primary testing emphasis is on validation sampling with inter-rater reliability metrics.

## Test Framework

**Runner:**
- pytest (recommended but not yet configured)
- Configuration file location: `pytest.ini` or `pyproject.toml` (when added)

**Assertion Library:**
- pytest built-in assertions (when implemented)
- Manual assertions in current test.py

**Run Commands (recommended setup):**
```bash
pytest                          # Run all tests
pytest -v                       # Verbose output
pytest --cov=src coverage.html  # Coverage report
pytest -k test_extraction       # Run specific tests
pytest --tb=short               # Short traceback format
```

## Current Test Infrastructure

**Existing test file:**
- Location: `/Users/fesalfayed/Desktop/israel-gaza NLP/test.py`
- Current scope: Manual integration test for newspaper3k Article extraction
- Framework: Direct library testing (not a formal test suite)

**Content of test.py:**
```python
# library import
import warnings
warnings.filterwarnings('ignore', message='urllib3 v2 only supports OpenSSL')

from newspaper import Article
from urllib.parse import urlparse

# test url
url = 'https://www.reuters.com/world/middle-east/...'
# define article
article = Article(url)
#download
article.download()
```

**Current limitations:**
- No assertions
- Single hardcoded URL
- No error handling
- Manual execution required
- No cleanup or teardown

## Test Organization Strategy

**Recommended location:**
- Co-located pattern: `tests/` directory at project root
- File naming: `test_{module}.py`

**Structure:**
```
israel-gaza NLP/
├── tests/
│   ├── conftest.py              # Shared fixtures
│   ├── test_scraper.py          # Acquisition layer tests
│   ├── test_agency.py           # Processing layer tests
│   ├── test_causal.py           # Processing layer tests
│   ├── test_terminology.py       # Processing layer tests
│   ├── test_database.py         # Database schema tests
│   └── fixtures/
│       ├── sample_articles.json  # Test data
│       └── sample_sentences.txt  # Text samples
├── src/
│   ├── scraper.py
│   ├── agency_extractor.py
│   ├── causal_extractor.py
│   └── terminology_extractor.py
└── test.py                      # Current manual test (to be refactored)
```

## Test Types

### Unit Tests

**Scope:** Individual functions in isolation

**Extraction Function Tests (Acquisition Layer):**
- Test `extract_article()` with:
  - Valid, accessible URLs (mock HTTP responses)
  - Paywalled URLs (expect None return)
  - Malformed URLs (expect exceptions caught)
  - Timeout scenarios
  - Network errors

**Pattern:**
```python
import pytest
from src.scraper import extract_article

def test_extract_article_success_wire_service(mocker):
    """Should extract article from Reuters (wire service)."""
    url = 'https://www.reuters.com/world/middle-east/...'
    result = extract_article(url)

    assert result is not None
    assert 'url' in result
    assert 'full_text' in result
    assert len(result['full_text']) > 300

def test_extract_article_paywall(mocker):
    """Should return None for paywalled sources."""
    url = 'https://www.nytimes.com/...'
    result = extract_article(url)

    assert result is None

def test_extract_article_invalid_url():
    """Should return None for malformed URLs."""
    result = extract_article('not-a-url')

    assert result is None
```

**Entity Classification Tests (Processing Layer):**
- Test `classify_entity()` with:
  - Israeli entities: 'israel', 'idf', 'netanyahu', 'tel aviv'
  - Palestinian entities: 'palestinian', 'hamas', 'gaza', 'rafah'
  - Neutral entities: 'UN', 'humanitarian', 'ceasefire'
  - Mixed strings: 'Israeli forces and Palestinian civilians'

**Pattern:**
```python
from src.agency_extractor import classify_entity

@pytest.mark.parametrize('text,expected', [
    ('Israel', 'ISR'),
    ('Palestinian', 'PAL'),
    ('United Nations', 'OTHER'),
    ('IDF soldiers', 'ISR'),
    ('Hamas fighters', 'PAL'),
])
def test_classify_entity(text, expected):
    """Should correctly classify entities."""
    assert classify_entity(text) == expected
```

**Lexicon Matching Tests (Processing Layer):**
- Test `build_matcher()` and phrase matching with:
  - Exact term matches
  - Lemma variations ('massacred', 'massacre', 'massacres')
  - Case insensitivity
  - Multi-word phrases

**Pattern:**
```python
from src.terminology_extractor import build_matcher, EMOTIVE_TERMS
import spacy

def test_emotive_matcher():
    """Should find emotive terms in text."""
    nlp = spacy.load('en_core_web_lg')
    matcher = build_matcher(EMOTIVE_TERMS, 'EMOTIVE')

    text = "The massacre was horrific and brutal."
    doc = nlp(text)
    matches = matcher(doc)

    assert len(matches) > 0  # Should find emotive terms
```

### Integration Tests

**Scope:** Full extraction pipelines end-to-end

**Full Article Processing:**
```python
def test_full_extraction_pipeline(mocker):
    """Should process article through all extraction stages."""
    # Mock HTTP response to avoid network calls
    mock_response = mocker.patch('newspaper.Article.download')

    url = 'https://www.reuters.com/...'
    article_dict = extract_article(url)

    assert article_dict is not None
    assert len(article_dict['full_text']) > 300

    # Test NLP processing on extracted text
    agencies = extract_agency(article_dict['full_text'])
    causals = extract_causal_roles(article_dict['full_text'])

    assert isinstance(agencies, list)
    assert isinstance(causals, list)
```

**Database Integration:**
```python
def test_database_schema_creation(tmp_path):
    """Should create valid SQLite schema."""
    db_path = tmp_path / "test.db"

    # Load schema
    with open('schema.sql') as f:
        schema = f.read()

    # Create database
    conn = sqlite3.connect(str(db_path))
    conn.executescript(schema)

    # Verify tables exist
    cursor = conn.cursor()
    cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
    tables = {row[0] for row in cursor.fetchall()}

    assert 'articles' in tables
    assert 'sentences' in tables
    assert 'agency_annotations' in tables
```

### Validation Testing (Human Coding)

**Framework:** Manual annotation with agreement metrics

**Sample Size:**
- Target: 150 sentences total
- Stratified by source and event type
- Randomized presentation to coders

**Annotation Categories:**
1. **Agency:** Subject, entity type (ISR/PAL/OTHER), voice (active/passive/agentless)
2. **Causal Linkage:** Agent explicit? (yes/no), causal type (EXPLICIT/IMPLICIT/OBSCURED)
3. **Terminology:** Terms found, category (EMOTIVE/SANITIZED/NEUTRAL), applied to whom

**Agreement Metrics (Gold Standard):**
- Cohen's Kappa: Target κ > 0.70 (substantial agreement)
- Precision/Recall: Per-variable metrics against automated extraction
- Confusion Matrix: Identify systematic error patterns

**Validation Data Location:**
- File: `validation_set.csv` (output from validation layer)
- Schema: Matches `validation_sample` database table
- Columns: `sample_id`, `sentence_id`, `stratum`, `human_agency`, `human_causal`, `human_term`, `coder_id`, `coded_date`

**Calibration Adjustment (post-validation):**
```python
# Validation showed SRL misclassifies 15% of OBSCURED as IMPLICIT
adjusted_obscured = raw_obscured + (0.15 * raw_implicit)
adjusted_implicit = raw_implicit - (0.15 * raw_implicit)

# Applied globally to metrics before final reporting
```

## Mocking Strategy

**Framework:** pytest-mock or unittest.mock

**What to Mock:**
- HTTP requests to news sources (avoid rate limiting, network failures)
- External API calls (AllenNLP SRL model predictions)
- Database connections (use temporary in-memory SQLite for tests)
- File I/O operations (when possible)

**Pattern for HTTP mocking:**
```python
from unittest.mock import Mock, patch

def test_article_extraction_with_mock(mocker):
    """Should extract article using mocked HTTP response."""
    mock_article = mocker.patch('newspaper.Article')
    mock_article.return_value.download = Mock()
    mock_article.return_value.parse = Mock()
    mock_article.return_value.title = "Test Headline"
    mock_article.return_value.authors = ["Author Name"]
    mock_article.return_value.text = "Full article text..." * 100
    mock_article.return_value.publish_date = None

    result = extract_article('https://example.com/article')

    assert result is not None
    assert result['headline'] == "Test Headline"
```

**What NOT to Mock:**
- spaCy NLP models (load actual model for realistic behavior)
- Lexicon matching logic (test actual pattern matching)
- Entity classification rules (verify actual rule application)
- Database schema validation (use actual SQLite)

## Fixtures and Test Data

**Fixture Location:** `tests/fixtures/`

**Sample Articles Fixture:**
- File: `sample_articles.json`
- Contains: Pre-extracted article dictionaries for testing
- Format:
  ```json
  [
    {
      "url": "https://www.reuters.com/...",
      "headline": "Israeli airstrikes hit Gaza targets",
      "authors": "John Doe",
      "publish_date": "2023-10-07",
      "full_text": "Full article text here...",
      "word_count": 450
    }
  ]
  ```

**Sample Sentences Fixture:**
- File: `sample_sentences.txt` (or JSON)
- Contains: Representative sentences for NLP processing
- Examples:
  - Israeli agency sentences: "The IDF conducted operations in Gaza."
  - Palestinian agency sentences: "Hamas launched attacks on October 7."
  - Passive voice: "Civilians were killed in the strikes."
  - Emotive terminology: "The massacre killed dozens of innocent people."
  - Sanitized terminology: "The operation targeted military personnel."

**Fixture Loader (conftest.py):**
```python
import pytest
import json

@pytest.fixture
def sample_articles():
    """Load sample articles for testing."""
    with open('tests/fixtures/sample_articles.json') as f:
        return json.load(f)

@pytest.fixture
def sample_sentences():
    """Load sample sentences for NLP testing."""
    with open('tests/fixtures/sample_sentences.txt') as f:
        return [line.strip() for line in f if line.strip()]

@pytest.fixture
def temp_database(tmp_path):
    """Create temporary SQLite database for testing."""
    db_path = tmp_path / "test.db"
    conn = sqlite3.connect(str(db_path))

    with open('schema.sql') as f:
        conn.executescript(f.read())

    yield conn
    conn.close()
```

## Coverage Requirements

**Target:** 80% code coverage for processing layer

**Excluded from coverage:**
- `__pycache__` directories
- `.venv` virtual environment
- Notebook files (if any)
- Discovery layer (GDELT/BigQuery integration - external service)

**Coverage Report Command:**
```bash
pytest --cov=src --cov-report=html --cov-report=term
```

**Viewing Coverage:**
```bash
# Terminal report
pytest --cov=src --cov-report=term-missing

# HTML report
pytest --cov=src --cov-report=html
open htmlcov/index.html
```

## Test Async/Concurrency Patterns

**Not applicable to current pipeline:**
- Current implementation is synchronous
- Database operations are sequential
- HTTP requests use single-threaded pattern with timeouts

**Future consideration (when scaling):**
- If parallel article scraping added: Use pytest-asyncio
- If concurrent NLP processing: Use ThreadPoolExecutor with test isolation

## Error Testing

**Pattern for exception verification:**
```python
def test_extract_article_timeout(mocker):
    """Should return None on timeout."""
    mocker.patch('newspaper.Article.download', side_effect=Exception('timeout'))

    result = extract_article('https://example.com/article')

    assert result is None

def test_extract_article_invalid_html(mocker):
    """Should handle malformed HTML gracefully."""
    mocker.patch('newspaper.Article.parse', side_effect=Exception('parsing error'))

    result = extract_article('https://example.com/article')

    assert result is None

def test_classify_entity_empty_string():
    """Should return OTHER for empty input."""
    assert classify_entity('') == 'OTHER'
```

## Database Testing

**Schema Validation:**
- Verify all tables created successfully
- Verify all indexes created successfully
- Verify foreign key constraints defined
- Verify default values applied (e.g., `extraction_date`)

**Test Pattern:**
```python
def test_schema_tables(temp_database):
    """Should create all required tables."""
    cursor = temp_database.cursor()
    cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
    tables = {row[0] for row in cursor.fetchall()}

    expected = {
        'articles', 'sentences', 'agency_annotations',
        'causal_annotations', 'terminology_annotations',
        'validation_sample'
    }
    assert expected == tables

def test_schema_indexes(temp_database):
    """Should create all performance indexes."""
    cursor = temp_database.cursor()
    cursor.execute("SELECT name FROM sqlite_master WHERE type='index'")
    indexes = {row[0] for row in cursor.fetchall()}

    expected = {
        'idx_articles_source', 'idx_articles_date',
        'idx_sentences_article'
    }
    assert expected.issubset(indexes)
```

## Continuous Integration Recommendations

**When implemented, use:**
- GitHub Actions or GitLab CI
- Run full test suite on: Push to main, all pull requests
- Run on Python 3.10+ (as specified in blueprint)
- Upload coverage to Codecov or similar
- Fail PR if coverage drops below 80%

**Example GitHub Actions workflow:**
```yaml
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - run: pip install -r requirements.txt pytest pytest-cov
      - run: pytest --cov=src --cov-report=xml
```

---

*Testing analysis: 2026-02-21*

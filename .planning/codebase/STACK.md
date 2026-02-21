# Technology Stack

**Analysis Date:** 2026-02-21

## Languages

**Primary:**
- Python 3.10+ - Core pipeline for NLP processing, data extraction, and analysis

**Secondary:**
- SQL - BigQuery queries for GDELT discovery and SQLite schema operations

## Runtime

**Environment:**
- Python 3.10 or higher (as specified in TWAIL_Pipeline_Blueprint.md)

**Package Manager:**
- pip - Standard Python package manager
- Lockfile: Not detected

## Frameworks

**Core NLP Processing:**
- spaCy 3.7.0+ - Dependency parsing and entity recognition (en_core_web_lg or en_core_web_trf model)
- AllenNLP 2.10.0+ - Semantic Role Labeling (SRL) via BERT-based model
- AllenNLP-Models 2.10.0+ - Pre-trained SRL models and utilities

**Data Acquisition:**
- Newspaper3k 0.2.8+ - Article extraction from URLs with automatic content parsing
- Selenium 4.15.0+ - Browser automation for scraping paywalled content with JavaScript rendering
- WebDriver-Manager 4.0.0+ - Automatic WebDriver binary management for Selenium

**Data Processing:**
- pandas 2.0.0+ - DataFrame manipulation and CSV processing
- NumPy 1.24.0+ - Numerical operations and array processing

**Database:**
- SQLite3 - Included in Python stdlib; lightweight relational database for articles and annotations

**Statistical Analysis:**
- SciPy 1.11.0+ - Statistical distributions and hypothesis testing
- Statsmodels 0.14.0+ - Advanced statistical analysis and confidence intervals
- Matplotlib 3.8.0+ - Plotting and visualization
- Seaborn 0.13.0+ - Statistical data visualization

## Key Dependencies

**Critical:**
- spacy[trf] - Transformer-based models for higher-accuracy NLP (en_core_web_trf alternative to en_core_web_lg)
- newspaper3k - Core dependency for article text extraction from 2,000-5,000 candidate URLs
- allennlp and allennlp-models - SRL pipeline for extracting causal relationships (agent-patient pairs)

**Infrastructure:**
- pandas - Processing GDELT BigQuery exports and creating validation datasets
- NumPy - Numerical operations for calculating agency ratios and terminology asymmetry metrics
- Selenium + WebDriver-Manager - Fallback chain for accessing paywalled legacy print sources (NYT, WaPo, WSJ)

## Configuration

**Environment:**
- Configuration via environment variables (secrets location not detected)
- BigQuery authentication required for GDELT dataset access
- Proxy rotation configuration for residential IP cycling during web scraping

**Build:**
- No build configuration detected; Python script execution model
- Manual setup: `python -m spacy download en_core_web_lg` required post-installation

## Platform Requirements

**Development:**
- 8 GB RAM minimum (16 GB recommended for AllenNLP SRL model loading)
- 10 GB storage minimum (50 GB recommended for full corpus + pre-trained models)
- No GPU required; CUDA GPU speeds SRL processing 5-10x (0.3 sec/article vs 2 sec/article on CPU)
- macOS, Linux, or Windows with Python 3.10+

**Production:**
- Designed for research/analysis workflow, not continuous deployment
- Output: SQLite database (articles.db), annotations database, metrics.json, and analysis reports
- Runs locally; no cloud hosting detected (though BigQuery integration for discovery phase)

---

*Stack analysis: 2026-02-21*

# structured-extract

**Turn messy PDFs and text documents into trustworthy structured JSON data.**

A pure-Python library (zero runtime dependencies) that extracts structured fields from unstructured text using a declarative schema, self-correcting retry loop, per-field confidence scoring, and optional Claude LLM backend.

---

## Status

| Milestone | Description | Status |
|-----------|-------------|--------|
| M1 | Core schema, validator, mock extractor, pipeline, confidence | Done |
| M2 | Real LLM extractor (Anthropic Claude), CLI `sx extract` | Done |
| M3 | Eval harness (precision/recall/F1), sample docs + gold | Done |

---

## Installation

```bash
# Core library only (pure Python, no deps)
pip install structured-extract

# With Claude LLM backend
pip install "structured-extract[llm]"

# Development (tests + linting)
pip install "structured-extract[dev]"
```

---

## Quick start

### As a library

```python
from structured_extract import FieldSpec, Schema, MockExtractor, extract_with_retry, compute_confidence

schema = Schema(fields=[
    FieldSpec("vendor", "str", required=True, description="Supplier name"),
    FieldSpec("amount", "float", required=True, description="Total amount"),
    FieldSpec("issued_on", "date", required=True, description="Invoice date YYYY-MM-DD"),
])

# Use MockExtractor for testing; swap in LLMExtractor for production
extractor = MockExtractor([{"vendor": "Acme", "amount": 250.0, "issued_on": "2024-01-15"}])

result = extract_with_retry("Invoice from Acme, $250, Jan 15 2024", schema, extractor)
print(result.valid)      # True
print(result.data)       # {'vendor': 'Acme', 'amount': 250.0, 'issued_on': '2024-01-15'}
print(compute_confidence(result.data, schema))
# {'vendor': 1.0, 'amount': 1.0, 'issued_on': 1.0}
```

### With the real LLM backend

```python
from structured_extract.extractor import LLMExtractor
from structured_extract import extract_with_retry

extractor = LLMExtractor(api_key="sk-ant-...")
result = extract_with_retry(document_text, schema, extractor, max_retries=2)
```

### CLI

```bash
# Offline (uses empty MockExtractor — shows schema and confidence structure)
sx extract --schema examples/schema.json examples/doc1.txt

# With Claude API key
ANTHROPIC_API_KEY=sk-ant-... sx extract --schema examples/schema.json examples/doc1.txt
```

---

## How it works

### Schema

Define fields with name, type (`str` | `int` | `float` | `bool` | `date` | `list`), required flag, and a description that becomes part of the LLM prompt.

### Extraction + self-correction

`extract_with_retry(text, schema, extractor, max_retries=2)`:

1. Calls `extractor.extract(text, schema)` to get a raw dict.
2. Validates the dict against the schema.
3. If errors found and retries remain: re-calls the extractor, injecting the previous result and error list as correction guidance (for `LLMExtractor` this becomes a correction prompt; for `MockExtractor` the next scripted response is returned).
4. Returns the result with the **fewest validation errors** (tie: most recent wins), plus validation status and all intermediate attempts.

### Confidence heuristic

| Score | Meaning |
|-------|---------|
| `1.0` | Field present with the canonical Python type for its declared type |
| `0.7` | Field present but required coercion (numeric/bool string, numeric for str field) |
| `0.0` | Field absent, wrong type, or invalid format |

### Evaluation

```python
from structured_extract.eval import evaluate

result = evaluate(extractions, gold_dicts)
print(result.micro_precision, result.micro_recall, result.micro_f1)
```

---

## Running tests

```bash
pip install -e ".[dev]"
pytest          # runs all 25 tests, offline, no API key needed
ruff check src tests
```

---

## PM angle

Healthcare, fintech, and logistics teams deal with thousands of invoice PDFs, clinical notes, and shipping manifests daily. Manual data entry is slow and error-prone. `structured-extract` provides:

- **Declarative schemas** — non-engineers can define extraction targets in JSON.
- **Self-correction** — automatic retry with error feedback reduces hallucinated / malformed values.
- **Confidence scores** — downstream systems can gate on confidence thresholds before writing to databases.
- **Eval harness** — teams can benchmark extraction quality against gold-standard datasets as they tune prompts or switch models.

---

## License

MIT — Copyright (c) 2026 Girisankar G

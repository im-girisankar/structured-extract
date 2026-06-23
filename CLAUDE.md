# structured-extract — CLAUDE.md

## Project overview
`structured-extract` turns messy text documents into validated, schema-conformant JSON. It is a pure-Python library (no required runtime dependencies) with an optional Anthropic Claude backend for real LLM extraction.

## Repository layout
```
src/structured_extract/
  __init__.py      re-exports public API
  schema.py        FieldSpec + Schema (types: str/int/float/bool/date/list)
  validator.py     validate(data, schema) -> list[FieldError]
  extractor.py     Extractor protocol + MockExtractor + LLMExtractor
  pipeline.py      extract_with_retry() self-correction loop
  confidence.py    compute_confidence() per-field heuristic
  eval.py          evaluate() precision/recall/F1
  cli.py           sx extract --schema <json> <doc>

tests/             pytest suite (offline, deterministic, no network)
examples/          3 sample invoices + gold.json + schema.json
```

## Design decisions
- **No hard runtime deps**: anthropic is optional; importing LLMExtractor without it is safe.
- **Injectable extractor**: any object with `extract(text, schema) -> dict` works; tests use MockExtractor with scripted responses.
- **Self-correction**: `extract_with_retry` retries up to `max_retries` times, injecting validation errors back into the prompt (LLMExtractor) or advancing the mock queue (MockExtractor).
- **Confidence heuristic**: 1.0 canonical type, 0.7 coerced (numeric/bool strings), 0.0 missing/invalid.

## Common commands
```bash
# Install for development
pip install -e ".[dev]"

# Run tests
pytest

# Lint
ruff check src tests

# Use the CLI (offline — mock extractor)
sx extract --schema examples/schema.json examples/doc1.txt

# Use with Claude API
ANTHROPIC_API_KEY=sk-... sx extract --schema examples/schema.json examples/doc1.txt
```

## Extending
- Add a new field type: update `VALID_TYPES` in `schema.py`, add a checker in `validator.py`, add a branch in `confidence.py`, add tests.
- Add a new extractor backend: implement `extract(text, schema) -> dict`; duck-typing satisfies the `Extractor` protocol.

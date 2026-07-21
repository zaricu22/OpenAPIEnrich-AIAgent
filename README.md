# Enrich OpenAPI

A single-file Python script that automatically enriches a raw OpenAPI JSON file with descriptions, summaries, and examples — using Claude AI to fill only the fields left blank by the developer.

> **Note:** The script is tailored for Java/Spring projects — source file scanning is hardcoded to `src/main/java`, and `RELEVANT_PATTERNS` lists the file name patterns considered relevant to the API (controllers, request/response classes, etc.).

## What it does

You provide a raw `openapi.json` (however it was generated) and an optional existing enriched file. The script:

1. Reads the raw `openapi.json`
2. For each schema and operation, checks whether it has changed since the last run
3. Skips anything unchanged — no unnecessary API calls
4. For anything new or changed, calls Claude to fill **only** the missing fields (`description`, `summary`, `example`) without touching anything else
5. Writes the result to the enriched output file

Your existing text always wins — Claude only fills the gaps you left blank.

## Enrichment strategy

The enriched file stores a hash (`x-raw-hash`) alongside each schema and operation, calculated from the corresponding part of the previous raw JSON. On each run, hashes from the last enriched file are compared to the new raw JSON:

- **parts unchanged** → skipped entirely, no API call
- **parts changed** → re-enriched: Claude fills only the missing fields; any text already present is preserved

This means re-running the script is cheap — only actual changes to the raw file trigger new API calls.

## Requirements

```bash
pip install anthropic
```

Python 3.10+. An Anthropic API key is required.

## Setup

1. Place your `openapi.json` next to the script (or adjust `RAW_OPENAPI_PATH`).

2. Set your API key as an environment variable:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

3. Run:

```bash
python enrich_openapi.py
```

The enriched file is written to `ENRICHED_OPENAPI_PATH` (see configuration below).

## Configuration

At the top of `enrich_openapi.py`:

| Variable | Default | Description |
|---|---|---|
| `RAW_OPENAPI_PATH` | `openapi.json` | Path to the raw input file |
| `ENRICHED_OPENAPI_PATH` | `src/main/resources/static/openapi/enhanced-openapi.json` | Output path |
| `RELEVANT_PATTERNS` | `Controller.java`, `Request.java`, … | Source file patterns used to collect context for Claude |
| `MAX_FILE_CHARS` | `3000` | Max characters read from each source file for context |
| `MAX_TOKENS` | `4096` | Max tokens per Claude response |
| `MAX_RETRIES` | `3` | Retry attempts on JSON parse failure |

## AI/LLM concepts implemented

- **AI-augmented pipeline** — main() runs a fixed hash-check → context → prompt → generate → validate → persist loop per section; the LLM only handles generation, not control flow.
- **Generative AI / natural language generation (NLG)** — Claude generates missing `description`, `summary`, `example` text for OpenAPI schemas/operations (`enrich_with_ai()`).
- **Prompt engineering** — a strict, constrained instruction set defining what the model may and may not touch (`build_prompt()`).
- **Retrieval-augmented generation (heuristic/lexical RAG)** — relevant Java source files are retrieved by keyword/tag matching to ground the model's output in the actual codebase (`collect_schema_context()`, `collect_operation_context()`).
- **Context-augmented generation (CAG)** — `collect_schema_context()` / `collect_operation_context()` retrieve relevant Java files, which `build_prompt()` then injects as structured `repository_context` into the prompt.
- **AI output validation / guardrails** — checks that Claude didn't silently drop fields, plus JSON-fence stripping and retry-with-backoff on malformed generations (`_validate_enrichment()`, `enrich_with_ai()`).
- **Human-in-the-loop / AI-assisted authoring** — enrichment only fills fields left blank by the developer; existing text always wins (`schema_needs_enrichment()`, `operation_needs_enrichment()`).
- **Caching / cost-optimization for LLM calls** — content hashing (`x-raw-hash`) skips unchanged schemas/operations to avoid redundant API calls (`compute_hash()`, `main()`).

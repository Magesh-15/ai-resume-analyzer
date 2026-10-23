# Token Cost Estimator API

A FastAPI service that counts LLM tokens, estimates the cost of a request **before** it is sent, checks the model's context window, splits long text into token-aware chunks, and logs usage in SQLite.

## Problem

LLM APIs bill per **token**, not per word or per request. This causes three problems:

1. **Unpredictable cost.** Developers only see the cost after the API call.
2. **Model-specific token counts.** The same text produces different token counts on different tokenizers.
3. **Context-window failures.** Text that is too long makes the request fail or get truncated.

Rules of thumb such as "1 word is about 1.3 tokens" are inaccurate, especially for code and non-English text.

## Solution

This API takes text and a model name, tokenizes the text with the matching encoding, and returns the exact token count, the estimated cost, and whether the request fits in the model's context window.

## Features

- Exact token counts with `tiktoken`
- Cost estimation from a configurable price file (`app/config/models.json`)
- Context-window check
- Token-aware chunking with overlap and metadata
- Side-by-side model comparison, sorted by cost
- Usage logging in SQLite with a daily budget warning
- Request validation with Pydantic and clear error messages
- 24 automated tests with pytest

## Tech stack

Python, FastAPI, Pydantic, tiktoken, SQLite, Uvicorn, pytest, httpx

## Project structure

```
token-cost-estimator/
|-- app/
|   |-- main.py              # FastAPI app and endpoints
|   |-- database.py          # SQLite connection and table setup
|   |-- config/
|   |   `-- models.json      # model encodings, prices, context windows
|   `-- services/
|       |-- tokenizer.py     # token counting
|       |-- cost.py          # cost formula and context check
|       |-- chunker.py       # token-aware chunking
|       |-- compare.py       # model comparison
|       `-- logger.py        # usage logging and summaries
|-- tests/                   # pytest test suite
|-- docs/                    # screenshots used in this README
|-- pytest.ini
|-- requirements.txt
`-- README.md
```

## Setup

Requires Python 3.10 or higher.

```bash
git clone https://github.com/YOUR-USERNAME/token-cost-estimator.git
cd token-cost-estimator
python -m venv venv

# Windows
venv\Scripts\activate
# Mac / Linux
source venv/bin/activate

pip install -r requirements.txt
uvicorn app.main:app --reload
```

Open http://127.0.0.1:8000/docs for the interactive Swagger page, where every endpoint can be tested.

The first run needs an internet connection, because `tiktoken` downloads its encoding files once.

## Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/tokenize` | Token count, and optionally the tokens |
| POST | `/estimate-cost` | Cost estimate and context-window check |
| POST | `/compare-models` | Compare models for the same text, sorted by cost |
| POST | `/chunk` | Split text into token-aware chunks with metadata |
| GET | `/usage` | Usage totals by model and by day |
| GET | `/models` | Supported models and their settings |
| GET | `/health` | Health check |

### Example: `/estimate-cost`

Request:

```json
{
  "text": "Explain machine learning in simple words.",
  "model": "gpt-4o",
  "expected_output_tokens": 300
}
```

The response contains `request_id`, `input_tokens`, `output_tokens`, `input_cost_usd`, `output_cost_usd`, `total_cost_usd`, `total_tokens`, `context_window`, `fits_in_context`, and `remaining_tokens`.

![Estimate cost](docs/estimate-cost.png)

### Cost formula

```
input_cost  = (input_tokens  / 1000) x price_per_1k_input
output_cost = (output_tokens / 1000) x price_per_1k_output
total_cost  = input_cost + output_cost
```

The real output length is unknown before the call, so `expected_output_tokens` is an input you provide.

## Demo: the same text, different token counts

The same Tamil sentence compared on two models with `/compare-models`:

```json
{
  "cheapest_model_that_fits": "gpt-4o",
  "results": [
    { "model": "gpt-4o", "encoding": "o200k_base", "input_tokens": 32,  "total_cost_usd": 0.00308 },
    { "model": "gpt-4",  "encoding": "cl100k_base", "input_tokens": 132, "total_cost_usd": 0.02196 }
  ]
}
```

(Shortened for readability. Costs use the sample prices in `models.json` and 300 expected output tokens.)

The older encoding needs about **4 times more tokens** for the same sentence. A word-based estimate would be badly wrong for text like this.

![Compare models](docs/compare-models.png)

## Chunking

`/chunk` splits text by **token count**, so no chunk can exceed the limit. A sliding window moves forward by `chunk_size - chunk_overlap`, so neighbouring chunks share a few tokens and keep their context. Every chunk carries metadata: `chunk_id`, `source`, `chunk_index`, `token_count`, `start_pos`, and `end_pos`.

![Chunking](docs/chunk.png)

## Usage tracking

Every `/estimate-cost` and `/chunk` request is logged in SQLite (`data/usage.db`, created automatically). `/usage` returns totals by model and by day, and adds a `budget_warning` when today's estimated spend passes `DAILY_BUDGET_USD` in `app/services/logger.py`.

`/compare-models` is not logged, because a comparison is a "what if" question and would inflate the spending totals.

![Usage](docs/usage.png)

## Configuration

Prices, encodings, and context windows live in `app/config/models.json`, not in the code:

```json
{
  "gpt-4o": {
    "encoding": "o200k_base",
    "context_window": 128000,
    "price_per_1k_input": 0.0025,
    "price_per_1k_output": 0.01
  }
}
```

> **The prices in this file are sample values for practice.** Replace them with current numbers from each provider's official pricing page before relying on the estimates. To add a model, add a new entry. No code change is needed.

## Tests

```bash
python -m pytest -v
```

The suite covers token counting, the cost formula, context limits, chunk boundaries and overlap, API status codes and validation errors, and usage logging. API tests use a temporary database, so real data is never touched.

![Tests passing](docs/pytest.png)

## Limitations

- `tiktoken` gives exact counts for OpenAI-style encodings only. Counts for other providers are approximate.
- Output cost depends on the expected output length you provide.
- Prices must be updated manually.
- Chunking by tokens can split a multi-byte character (for example in Tamil, Hindi, or emoji) at a boundary. Token counts stay correct, but the decoded text at the edge of a chunk may look broken.
- This API does not call any LLM. It only estimates cost.

## Future work

- Budget alerts by email or Slack
- Dashboard with cost charts per day and per model
- Sentence-aware chunking
- Authentication and per-project API keys
- Docker image and cloud deployment
- Use the chunker and cost layer as the first stage of a RAG application

## Author

[Your Name] | CFRSI GenAI Course

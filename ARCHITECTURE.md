# Project Infrastructure — File by File

This doc explains what every file in this repo does, in plain language, so anyone new to the
project can get oriented in a few minutes. For "why we built this" and results, see `README.md`.
For the original build spec, see `prototype_plan.md`.

## The big picture

We're comparing two ways of answering questions with a local LLM:

- **Setup A (baseline)** — dumb and simple. Every question goes to the same model. No memory,
  no shortcuts.
- **Setup B (adaptive)** — smarter. It first checks "have I basically answered this before?" (the
  cache), and if not, it picks the cheapest model that can actually handle the question (the
  router).

Everything else in the repo exists to run both systems on the same questions and prove, with
numbers, that B is faster and cheaper without getting noticeably dumber.

## Core logic (the actual "product")

| File | What it does |
|---|---|
| `config.py` | One place for all the settings: which models map to which tier, the fake-cost-per-token assumptions, the cache similarity threshold. Change behavior here, not by hunting through other files. |
| `ollama_client.py` | The only place that talks to Ollama (the local model server) over HTTP. Handles retries if a call times out. Every other file goes through this instead of calling Ollama directly. |
| `baseline.py` | Setup A. One function, `ask_baseline(query)`, that always calls the same model and returns the answer + how long it took + what it "cost." |
| `router.py` | The brain of Setup B's cost savings. Looks at a question's wording and length and decides: is this "small" (trivial), "medium" (normal), or "large" (needs real reasoning)? Pure rules, no ML — e.g. "starts with 'what is' and is short" → small; "contains 'explain' or 'compare'" → large. |
| `cache.py` | The brain of Setup B's speed savings. Turns each question into a vector (a list of numbers representing its meaning) using a small embedding model, and checks if a *similar-meaning* question was already asked. If yes → return the old answer instantly, skip the model entirely. |
| `adaptive.py` | Setup B. One function, `ask_adaptive(query)`, that wires `cache.py` and `router.py` together: check cache first, if miss then route to a tier and call the model, then save the answer to the cache for next time. |

## Test data & experiment runner

| File | What it does |
|---|---|
| `test_queries.json` | The fixed set of 60 questions both systems are tested on. Deliberately includes exact repeats, reworded repeats (paraphrases), easy questions, hard questions, and one-off unique questions — so we can see cache hits and routing decisions happen in a controlled way. |
| `run_benchmark.py` | Runs all 60 questions through Setup A, then all 60 through Setup B (with an empty cache to start, so it's a fair fight), and logs every single result to `benchmark_results.csv`. |
| `quality_check.py` | A lightweight "did the cheaper model actually give a good answer?" check. Finds cases where the adaptive system used a smaller model and got a different answer than baseline, samples 10 of them, and asks the model itself to judge whether the smaller model's answer still holds up. |

## Reporting & demo

| File | What it does |
|---|---|
| `generate_report.py` | Reads `benchmark_results.csv` and crunches the numbers: average latency, total cost, cache hit rate, which tier got used how often. Outputs `report.md` plus two chart images. |
| `demo.py` | The live, interactive part. Type a question, see both systems answer it side by side with timing/cost/cache-hit info. This is what you run live in front of the professor. |

## Generated output (not hand-written — these get overwritten every run)

| File | What it is |
|---|---|
| `benchmark_results.csv` | Raw log: every question, which system answered it, how long it took, whether it was a cache hit, which tier, estimated cost, and the actual answer text. |
| `quality_check_results.csv` | The 10 sampled quality-check questions with both answers and a PASS/FAIL verdict. |
| `report.md` | The human-readable summary: tables + a plain-English paragraph of what the numbers mean. |
| `cost_comparison.png`, `latency_comparison.png` | The two bar charts referenced in `report.md`. |

## Tests

| File | What it does |
|---|---|
| `tests/test_router.py` | Checks the routing rules actually classify example questions correctly (e.g. "What is X?" → small, "Explain why..." → large). |
| `tests/test_cache.py` | Checks the cache returns a hit for a reworded duplicate, and `None` for something unrelated. |
| `tests/test_handlers.py` | Checks `ask_baseline` and `ask_adaptive` return the same shape of result, and that asking the same question twice through the adaptive system produces a cache hit the second time. |
| `conftest.py` | Housekeeping so pytest can find the project's modules when run from the `tests/` folder. |

## Everything else

| File | What it does |
|---|---|
| `requirements.txt` | The list of Python packages needed (`pip install -r requirements.txt` reads this). |
| `README.md` | Setup instructions, how to run everything, and the actual results from our benchmark. |
| `prototype_plan.md` | The original spec this prototype was built from — what's in scope, what's deliberately left out. |
| `.gitignore` | Tells git to ignore the virtual environment and Python's cache folders so they don't get committed. |

## How a single question flows through Setup B (adaptive)

1. `demo.py` (or `run_benchmark.py`) calls `ask_adaptive(query)` in `adaptive.py`.
2. `adaptive.py` asks `cache.py`: "have I seen something like this before?"
   - **Yes** → return the saved answer immediately. Done. (Near-zero cost, near-zero latency.)
   - **No** → continue to step 3.
3. `adaptive.py` asks `router.py`: "how hard is this question?" → gets back `small`/`medium`/`large`.
4. `adaptive.py` calls that tier's model via `ollama_client.py`.
5. `adaptive.py` saves the new question + answer into the cache via `cache.py`, so next time it's a hit.
6. Returns the answer, timing, tier used, and cost back to the caller.

Setup A (`baseline.py`) skips all of that and just does step 4, every time, with one fixed model.

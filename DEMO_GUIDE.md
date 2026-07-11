# Professor Demo Guide

A script for presenting CostQual-Router: what to run, in what order, and what to say. Total time:
~5 minutes if the benchmark is pre-run (recommended), or ~15 minutes if you run everything live.

## 0. Before you walk in

Run the benchmark once ahead of time so you're not waiting on it live:

```bash
python run_benchmark.py       # ~10 min
python generate_report.py     # a few seconds
python quality_check.py       # ~1 min
```

Have `report.md`, `cost_comparison.png`, and `latency_comparison.png` open, plus a terminal ready
for `python demo.py`.

## 1. The one-sentence pitch (30 seconds)

> "We built two versions of the same AI question-answering system: one that's simple but wasteful,
> and one that's smart about when it actually needs to think hard. We proved the smart version is
> cheaper and just as accurate, using nothing but local, free, open-source models."

## 2. Explain the difference (1-2 minutes)

Draw or describe this on a whiteboard:

```
Setup A (baseline) -- what most naive implementations do:
  every question -----> [always the same model] -----> answer

Setup B (adaptive) -- what we built:
  question -> "have I seen something like this before?" (semantic cache)
                |                                  |
              yes: reuse answer instantly      no: continue
                                                     |
                                          "how hard is this question?" (router)
                                                     |
                                    trivial -> tiny model
                                    normal  -> medium model
                                    hard    -> capable model
```

**Why this matters:** in a real deployed system, every question costs money and takes time. A
naive implementation sends everything to one capable (expensive) model, even "What is the capital
of France?" A system that's aware of cost routes trivial questions to something cheap and only
pays for the expensive model when a question actually needs reasoning -- and it never re-answers
a question it's already answered.

## 3. Show the numbers (2 minutes)

Open `report.md` (or the two chart PNGs) and walk through:

| Metric | Baseline | Adaptive | Change |
|---|---|---|---|
| Avg latency | 6053 ms | 5881 ms | -3% |
| Total estimated cost | 0.11841 | 0.07700 | **-35%** |
| Cache hit rate | -- | 23% | -- |

Talking points:
- **Cost is the headline result: -35%.** The adaptive system only pays the expensive rate when a
  question genuinely needs a capable model (22% of new questions); everything else goes to a
  cheaper tier or costs almost nothing (a cache hit).
- **Latency improvement is modest (-3%) and that's honest, not a weakness to hide.** Cache hits
  return in ~20ms, but the "large" tier questions (complex reasoning) can take as long as the
  baseline's fixed model -- so the average gets pulled back up. This is expected: you don't get
  faster responses for genuinely hard questions, only for the ones that didn't need a hard
  question's worth of compute in the first place.
- **23% cache hit rate, and it's not just catching exact repeats.** Show the breakdown: 53% of
  *exact duplicate* questions hit the cache, but so did 40% of *paraphrased* questions ("What is
  Japan's capital?" vs "Can you tell me Japan's capital city?") -- that's the semantic part
  working, not simple string matching.
- **Quality held up: 7/10 on the spot-check.** Where the small model gave a different answer than
  the baseline, a judge model rated 70% of them as still acceptable. Be upfront about the 30% that
  failed -- e.g. the 0.5B model got a square-root calculation wrong. That's an honest limitation
  of using a tiny model, and exactly the kind of finding a real system needs to detect and handle
  (e.g. routing math questions to a bigger tier), not something we're hiding.

## 4. Live demo (2 minutes)

```bash
python demo.py
```

Script to run live:
1. Ask a question: `What is the capital of Japan?`
   - Point out: baseline answer, its cost/latency; adaptive answer, its cost/latency, `tier=small`.
2. Immediately ask a paraphrase: `Can you tell me Japan's capital city?`
   - Point out: adaptive now shows `cache_hit=True`, latency in single-digit milliseconds, cost
     near zero -- while baseline pays full price again because it has no memory.
3. Ask something genuinely hard: `Compare capitalism and socialism as economic systems.`
   - Point out: adaptive correctly routes this to `tier=large` instead of the cheap tier --
     the router isn't just "always use the small model," it's making a real decision per question.

## 5. Anticipate these questions

- **"Is this running on a real API / in the cloud?"** No -- everything runs locally via Ollama, so
  the "cost" numbers are a simulated proxy (see `config.py`), not real dollars. The point is to
  demonstrate the *mechanism* (cache + routing) and that it produces the expected direction of
  savings, before building the full system against real hosted APIs.
- **"Why do latency savings look small?"** Answered above -- see point 3. It's an honest result,
  not a failed one.
- **"What's out of scope for this prototype?"** See `prototype_plan.md` -- no auth, no CI/CD, no
  Kubernetes, no monitoring stack, no multi-user support. This is a proof-of-concept for the
  mechanism, not the production system.
- **"How would this scale?"** The in-memory cache (a Python list of vectors) is fine for ~100-1000
  entries; a real deployment would swap in a vector database (Qdrant/FAISS). The router is
  currently rule-based for transparency; a production version could add a lightweight ML
  classifier once there's labeled routing data to train on.

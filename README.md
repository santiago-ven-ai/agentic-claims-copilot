# agentic-claims-copilot

[![CI](https://github.com/santiago-ven-ai/agentic-claims-copilot/actions/workflows/ci.yml/badge.svg)](https://github.com/santiago-ven-ai/agentic-claims-copilot/actions/workflows/ci.yml)
[![Coverage >= 25%](https://img.shields.io/badge/coverage-%E2%89%A525%25-brightgreen)](https://github.com/santiago-ven-ai/agentic-claims-copilot/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

An agentic retrieval loop for insurance/fintech claims — plan, retrieve, observe, retry, under a hard token budget — orchestrated with Step Functions.

## Pitch Card

**Problem** — Claims adjusters spend hours cross-referencing policy documents. Single-shot RAG returns confident answers with no evidence and no cost ceiling — it doesn't know when it doesn't know.

**Solution** — An agentic retrieval loop (plan → retrieve → observe → retry) orchestrated with Step Functions, with a hard token budget per claim enforced across a distributed state machine, and a dead-letter queue for cases that exhaust budget instead of hallucinating an answer.

**Impact** — Citation precision@3 of 0.167 vs. 0.133 single-shot (measured with real MiniMax M3 calls on a 10-claim golden set — see "Measured" below for the full story, including a regression that was found and fixed by actually running the eval twice), 2.7 average loop iterations per resolved claim, 100% of budget-exhausted cases captured in DLQ instead of silently failing.

**Stack** — Python 3 · PySpark · FastAPI · Pinecone (Chroma in dev) · MiniMax M3 (LLM, OpenAI-compatible) · AWS (Step Functions, Lambda, SQS, DynamoDB, S3) via MiniStack

---

## Architecture

```
  synthetic policy docs + claim intake
             |
             v
      S3 (claims-docs)
             |
             v
  src/ingestion/index_docs.py
    chunk + embed (sentence-transformers)
             |
             v
      Pinecone / Chroma (vector store)
             |
             v
  src/orchestration/statemachine.py drives the loop below, calling into
  src/models/agent_loop.py for each step (Plan/Tool/Observe run in Python —
  an embedding-model call doesn't fit a bare Lambda runtime):
             |
             v
       +-----------+
   +-->|   Plan     |  propose retrieval query from claim
   |   +-----+-----+
   |         v
   |   +-----------+
   |   |   Tool     |  query vector store, top-k
   |   +-----+-----+
   |         v
   |   +-----------+
   |   |  Observe   |  score evidence sufficiency
   |   +-----+-----+
   |         v
   |   Gate (Lambda): src/orchestration/lambdas/check_budget.py
   |     atomic DynamoDB token-spend check
   |         |
   |     +---+--------------------+
   |     v                        v
   |  sufficient              insufficient
   |     |                        |
   |     v                   budget left?
   |  emit answer               /    \
   |  + citations              yes    no
   |     |                      |      \
   |     v                      |       v
   |  DynamoDB                  |    Lambda: send_to_dlq.py --> SQS DLQ
   |  (answers + traces)        |
   +-----------------------------+

  nightly src/transformation/reindex.py (PySpark)
    re-embeds only changed docs, archives traces --> S3 Parquet
             |
             v
  src/serving/api.py :: FastAPI --> /ask  /trace/{id}  /eval/report
```

See `docs/architecture.md` for the diagram.

## Positioning note

This is the project that shows AI capability without pretending to be an LLM architect. The interview line is: *"I'm a data engineer; I built the orchestration, the cost control, and the evaluation harness — I didn't train the model."* That's a strength, not a disclaimer — it's exactly what a Senior DE + AI role needs.

**What actually happened building this, in order — the real story is a better interview answer than a clean number would be:**
1. First version merged evidence across retry iterations by raw embedding distance. Citation precision came out *worse* than not looping at all (0.067 vs. 0.133) — a real regression, caught by running the eval, not by reading the code.
2. Root cause: different phrasings of the same claim produce embeddings on different distance scales, so comparing raw distances *across* queries isn't valid — a query that happens to produce uniformly smaller distances dominates the merge regardless of relevance.
3. Fix: switched evidence fusion to [Reciprocal Rank Fusion](https://en.wikipedia.org/wiki/Reciprocal_rank_fusion) (RRF), which only uses each result's *rank* within its own query, sidestepping the scale problem. Re-measured: 0.167 vs. 0.133 (precision@3) and 5/10 vs. 4/10 (recall) — modest, but a real and consistent improvement, verified with actual MiniMax M3 API calls, not the fake provider.

See `docs/impact-model.md` and `docs/eval-single-shot.json` / `docs/eval-agentic.json` for the full numbers.

## Why no Java/Go here

Deliberate. Adding a JVM or Go worker just to "cover the language" would be the exact anti-pattern this portfolio avoids. Everything here is Python.

## Measured in this repo

| Metric | Single-shot | Agentic (RRF) | How it's measured |
|---|---|---|---|
| Citation precision@3 | 0.133 | **0.167** | `python3 scripts/eval.py --mode {single-shot,agentic}` with `LLM_PROVIDER=minimax` |
| Recall (≥1 correct citation) | 4/10 | **5/10** | same eval run |
| Avg iterations per claim | 1.0 | 2.7 | same eval run |
| Budget-exhaustion → DLQ, concurrent counter correctness | — | **verified**: 20 concurrent budget increments (300 tokens each) sum to exactly 6000, no lost updates | `pytest tests/integration/test_agent_loop.py::test_budget_counter_is_atomic_not_read_modify_write` |

> Real numbers from a 10-claim golden set with real MiniMax M3 calls, not the fake provider — see the "what actually happened" section above for how these numbers came to be this specific and why an earlier design scored *worse* than the baseline.

## Retrieval difficulty, honestly stated

Both precision numbers are modest in absolute terms (0.13–0.17, not 0.8+). That's the real result on a 120-clause synthetic corpus with a small local embedding model (`all-MiniLM-L6-v2`, 384 dims) and paraphrased claim language — a harder retrieval task than a demo tuned to look good. A production system would likely add a cross-encoder reranker or a larger embedding model; that's future work, not something this repo claims to have solved.

## Modeled business impact (synthetic data — assumptions documented)

| Assumption | Source | Modeled outcome |
|---|---|---|
| Adjuster hours saved per claim by citation-grounded first-pass answers | TODO — cite in `docs/impact-model.md` | TODO |

## Emulated vs. real

| Component | Dev (this repo) | Production would use | Fidelity |
|---|---|---|---|
| S3 / Lambda / SQS / DynamoDB | [MiniStack](https://ministack.org) (free, MIT, no account) | AWS | High |
| Step Functions | MiniStack (full ASL interpreter) | AWS | Medium-High |
| AWS CLI v2 | Real `aws` CLI against MiniStack (`AWS_ENDPOINT_URL`) — see `docs/RUNBOOK.md` §2 | AWS CLI v2 | High |
| Vector store | Chroma (`VECTOR_BACKEND=chroma`) or real Pinecone (`=pinecone`) | Pinecone | High — same retrieval interface, both exercised in tests |
| LLM | Deterministic fake (`LLM_PROVIDER=fake`) for `make demo`/CI, real [MiniMax M3](https://minimax-ai.chat/docs/api/) (`=minimax`, OpenAI-compatible) for `make eval` | MiniMax M3 or equivalent | Eval metrics in the README are generated with the real provider, not the fake one |

## Three non-tutorial challenges

1. **Token budget enforcement inside a distributed state machine** — a counter in DynamoDB, not in a single process's memory, since each loop step is its own Lambda invocation.
2. **Evaluation harness with a golden set** — 50 questions with correct clauses labeled, scored for citation precision@k and answer groundedness. This is what turns the repo into authority instead of a demo.
3. **A loop that doesn't diverge** — backoff, repeated-query detection, a hard cutoff. An agent that retries indefinitely is a cost bug, not a feature.
4. **Permanent vs. transient failure classification on the LLM call itself** — not every failure deserves a retry. A malformed/rejected request (`PermanentLLMError`) goes straight to the DLQ; a timeout-shaped error (`TransientLLMError`) gets bounded retries with exponential backoff first. `LLM_PROVIDER=fake-flaky` simulates both deterministically — see `docs/RUNBOOK.md`.

## Installation

```bash
git clone https://github.com/santiago-ven-ai/agentic-claims-copilot.git
cd agentic-claims-copilot
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt   # app deps + lint/type/security tooling
```

## Usage — Demo (3 minutes)

```bash
source env.sh
make demo          # bootstrap MiniStack, 20 policies + 10 claims, index into Chroma
make eval          # LLM_PROVIDER=fake by default (free); export LLM_PROVIDER=minimax for real numbers
cat docs/eval-agentic.json
```

## Testing

```bash
make test                     # unit + integration + BDD (pytest-bdd), against real MiniStack
make e2e                      # full pipeline, emits benchmarks/quality-report.json
.venv/bin/pre-commit run --all-files   # ruff, mypy, whitespace/EOF checks
```

CI (`.github/workflows/ci.yml`) runs the same suite on every push, plus an isolated `security` job (`pip-audit`, with two chromadb CVEs explicitly suppressed — see the job's own comment for why they don't apply to `PersistentClient` mode) and a coverage gate that fails the build under the threshold on the badge above.

## Learn by running

See [`docs/RUNBOOK.md`](docs/RUNBOOK.md). Build from scratch: [`docs/BUILD_GUIDE.md`](docs/BUILD_GUIDE.md).

## What this is NOT

Not "chat with your PDF." Without the loop, the budget, the eval harness, and the DLQ, this would be a single-shot RAG demo — the same class of project as ColLawRAG, which this repo is explicitly built to improve on.

## Build it yourself

See [`docs/RUNBOOK.md`](docs/RUNBOOK.md) to run the flow, or [`docs/BUILD_GUIDE.md`](docs/BUILD_GUIDE.md) to build from scratch.

## Contributing

Solo-maintained portfolio/demo repo — not actively seeking external contributions, but issues and questions are welcome via [GitHub Issues](https://github.com/santiago-ven-ai/agentic-claims-copilot/issues). See [`CODEOWNERS`](CODEOWNERS) and [`SECURITY.md`](SECURITY.md) for how reports are handled.

## License

[MIT](LICENSE) © santiago-ven-ai

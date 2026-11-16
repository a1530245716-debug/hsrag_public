# Enterprise RAG · FDE Portfolio (Sanitized)

An on-premise RAG system for Chinese enterprise documents, designed for **~100 concurrent internal users**.
This repository is a **sanitized portfolio**: architecture, delivery playbook, benchmark methodology, and a runnable local demo.

> No client data, credentials, internal endpoints, or production source code are included.
> All demo documents and addresses are synthetic.

## At a glance

- Knowledge base: **56,426 chunks** of Chinese project documents (anonymized).
- Benchmarked with a **mock LLM (2.3s per answer)** to isolate application-side bottlenecks.
- Baseline: **10 concurrent → p95 42.4s; 30 → 127.3s; 50 → 173.3s; 100 → 315.5s**.
- **After the first optimization round (2026-11)**: explicit 429 admission control;
  50-concurrency admitted TTFT p50 127.4s → 15.2s; BM25 75–172ms → 1.6–7.0ms;
  a 16-account sweep set `global_max=6` (p95 25.9s at a 20-way burst).
- Root causes: in-process Python BM25 (GIL), a 2-slot CPU reranker, and synchronous SSE occupying a 40-thread pool.
- Target architecture: **async BFF + external RAG core (RAGFlow/Elasticsearch) + internal LLM gateway + Redis/Postgres/MinIO**.
- Included: a **runnable async BFF + mock RAG demo**, no Docker or external model required.

## Baseline (before)

Measured on a 16 vCPU machine, using 18 real-style questions and `no_cache=true`.
The mock LLM answers in ~2.3s and never fails, so the numbers reflect the application layer.

| Concurrency | Success | TTFT p50 | TTFT p95 | Total p95 | Wall |
|---|---|---|---|---|---|
| 10 | 10/10 | 24.7s | 39.5s | 42.4s | 43s |
| 30 | 30/30 | 66.0s | 124.6s | 127.3s | 133s |
| 50 | 50/50 | 127.4s | 170.5s | 173.3s | 185s |
| 100 | 97/100 | 225.3s | 314.4s | 315.5s | 337s |

Evidence from logs:

- 29 reranker queue timeouts (120s wait, then degraded).
- 32 upstream streams hit a 90s timeout even though the mock only takes 2.3s — GIL starvation, not upstream slowness.
- At 100 concurrency, 97% of requests waited ≥120s for the first token; 57% waited ≥200s.

## After the first optimization round (2026-11)

The same app, after three rounds of changes (admission control, async SSE, and
in-process retrieval optimizations):

| Metric | Before | After |
|---|---|---|
| 50 concurrent admitted TTFT p50 | 127.4s | **15.2s** |
| 50 concurrent with per-user limit 2 | 127.4s | **8.5s** (rest 429) |
| `/api/health` idle p95 | 1985ms at 100 concurrency | **8–11ms** |
| BM25 query (56k chunks) | 75–172ms | **1.6–7.0ms** |
| Overload behavior | wait 2–5 minutes | immediate 429 + `Retry-After` |
| Eval | 5/6 | **6/6** |

Throughput is still bounded by the CPU reranker: admitted throughput saturates at
~**0.20 answers/s** from 4 concurrent onward, so the service admits 6 concurrent
requests (p95 25.9s under a 20-way burst) and rejects the rest explicitly instead
of silently degrading. The 16-account sweep and full trade-off table are in
**[docs/optimization.md](docs/optimization.md)**.

## Target architecture

```
Browser
   │ HTTPS
Nginx / company gateway  (SSE not buffered, 300s timeout, 100MB upload)
   │
FastAPI BFF (async, stateless)
   ├─ RAGFlow / Elasticsearch  (parsing, hybrid retrieval, rerank, citations)
   ├─ Redis                    (cache, rate limit, sessions)
   ├─ PostgreSQL               (users, KB mapping, audit)
   ├─ MinIO                    (documents, images)
   └─ Internal LLM gateway     (key pool, quotas, retries)
```

Design principles:

- Commodity RAG (parsing, indexing, retrieval, rerank) goes to a mature engine.
- Differentiated business logic (permissions, workflows, UI, domain rules) stays in the BFF.
- A `RetrievalBackend` interface keeps the engine swappable (RAGFlow, Elasticsearch, mock).
- Permissions are enforced in the BFF, not in the UI or the RAG engine.
- Async SSE + shared connection pool + explicit concurrency limits.

## Runnable local demo

The demo has no external dependencies beyond Python packages:

```
Browser → async BFF (8020) → mock RAG (9390)
```

```bash
# Host mode (fastest)
python tools/local_dev/run_local.py --keep
# open http://127.0.0.1:8020

# Docker mode
cd tools/local_dev
docker compose up --build
```

What it demonstrates:

- end-to-end async streaming (`async def` + `httpx.AsyncClient`, no threadpool-held SSE);
- citations (`sources` event);
- a global concurrency semaphore with HTTP 429;
- a shared connection pool;
- an adapter boundary between the BFF and the retrieval backend.

The mock RAG uses keyword scoring and templated generation; it is not a quality claim.

## Repository map

| Path | What it is |
|---|---|
| `docs/fde-playbook.md` | Full FDE delivery lifecycle: discover → define → design → prototype → validate → deploy → operate → adopt → evolve |
| `docs/case-study.md` | Requirements, constraints, current architecture, baseline, SLOs |
| `docs/selection.md` | Enterprise RAG stack comparison and decision rationale |
| `docs/architecture.md` | Target architecture, data model, permissions, incremental indexing, rollback |
| `docs/evaluation.md` | PoC steps, acceptance criteria, eval and load-test methodology |
| `docs/deployment.md` | Server sizing, Nginx/SSE, security, monitoring, backups, incident runbook |
| `docs/adoption.md` | Pilot plan, training, feedback loop, handover |
| `docs/optimization.md` | Implementation record: admission control, async SSE, inverted index, mmap, rerank trade-offs, multi-user sweep, `global_max=6` |
| `docs/technical-notes.md` | Concurrency/GIL/reranker/Nginx/Linux migration lessons |
| `docs/learning-roadmap.md` | 12-week FDE learning plan and portfolio guidance |
| `docs/poc/` | Acceptance checklist, Nginx config, env template, WSL2 + Docker guide |
| `tools/local_dev/` | Runnable async BFF + mock RAG demo |
| `tools/mock_llm.py` | Standalone Anthropic-compatible mock LLM used for benchmarking |

## What is not included

- Production source code, client documents, credentials, internal endpoints.
- The private RAGFlow integration and business-specific modules.
- Real customer data or screenshots.

The adapter contract is documented in `docs/architecture.md`; the production implementation lives in a private repository.

## Key lessons

1. **Concurrency is a resource-model problem, not a framework problem.** Replacing FastAPI with LangChain would not fix a 2-slot reranker or GIL contention.
2. **Measure p95 and failure tails, not averages.** The average hides a 5-minute user experience at 100 concurrency.
3. **Externalize CPU-heavy retrieval and long-lived streams.** Async HTTP belongs in the API process; retrieval/rerank belong in workers or a search engine.
4. **Design for rollback.** A BFF adapter lets you swap RAGFlow for Elasticsearch without rewriting permissions or UI.
5. **Sanitize before publishing.** A private repo with client data must never become a public portfolio repo directly.

## 中文简介

这是一个面向内网 100 并发场景的企业 RAG 脱敏作品集：包含需求与压测基线、企业级选型、目标架构、数据与权限设计、FDE 全流程交付方法、评测/压测方法，以及一个不依赖 Docker 的可运行异步 BFF + 假 RAG 演示。

生产代码、客户资料、密钥和内网地址均未包含在本仓库中。

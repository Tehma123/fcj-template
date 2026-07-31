---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Retrieval-Augmented Generation for Multi-Hop Reasoning on HotpotQA
## An Adaptive Hybrid-Retrieval RAG Pipeline with Hop-Aware Query Planning, Deployed on AWS

### 1. Executive Summary
This project designs and deploys a Retrieval-Augmented Generation (RAG) system to answer multi-hop reasoning questions from the HotpotQA dataset — questions whose answers require combining evidence from multiple documents rather than a single passage. The system is scoped as a full end-to-end demo: an offline indexing pipeline that chunks and embeds a HotpotQA validation slice, an online FastAPI retrieval-and-generation service exposing a public API, and a React front end for interactive testing. The target scale is a small, cost-conscious demo deployment (a 500-document HotpotQA validation subset, low query volume) rather than a production-scale service, intended for internal evaluation, workshop demonstration, and as a reference architecture that can later be scaled to larger corpora. Intended users are workshop reviewers, the engineering team evaluating retrieval quality, and future engineers who will extend the pipeline to other document collections.

### 2. Problem Statement
#### What's the Problem?
Standalone large language models (LLMs) and single-pass QA systems often struggle with multi-hop questions, since the answer cannot be found in one document and requires retrieving and reasoning across several related passages. Without a retrieval step grounded in the source corpus, answers can be inaccurate, ungrounded, or miss intermediate reasoning steps. This matters specifically for HotpotQA-style questions because the two supporting documents are connected only through a shared "bridge" entity (e.g., a person, film, or organization mentioned in both articles) — a single dense- or keyword-search pass frequently retrieves one of the two documents but not both, and a naive one-shot RAG pipeline (retrieve once, then generate) has no mechanism to notice that the retrieved evidence is incomplete and search again.

#### The Solution
The system retrieves relevant passages from the HotpotQA document corpus and uses a language model to generate the final answer conditioned on that retrieved evidence, chaining multiple retrieval/reasoning steps together to resolve multi-hop questions. Concretely, the stack combines: **BAAI/bge-m3** as the open-source dense embedding model; a **hybrid sparse + dense retriever** (BM25 via the `bm25s` library, fused with dense vector search through LangChain's `EnsembleRetriever` using weighted Reciprocal Rank Fusion); **Amazon S3 Vectors** (or a local ChromaDB instance for development) as the vector store; an **LLM-driven query decomposition and adaptive hop-planning loop** (via the Groq API, `llama-3.1-8b-instant`) that inspects evidence after each retrieval round and decides whether to stop or issue a more specific follow-up query; a **cross-encoder reranker** (`cross-encoder/ms-marco-MiniLM-L-6-v2`) to score candidate parent documents against the question; and a final **short-form answer generation** step matching HotpotQA's answer format for automated Exact Match / F1 scoring. The entire pipeline is orchestrated by a single `AdvancedRAGPipeline` class and served through a FastAPI application.

#### Benefits and Return on Investment
The primary benefit is measurably higher accuracy on multi-hop questions compared to a single-pass retrieval baseline, because the adaptive hop-planning loop specifically targets the bridge-entity failure mode described above instead of relying on one fixed retrieval pass. A secondary, longer-lived benefit is architectural reusability: because the offline indexing pipeline and the online query pipeline are strictly decoupled (all chunking/embedding/index-building happens once, offline; the online service only ever loads pre-built artifacts), the same codebase can be repointed at a different corpus by building a new artifact bundle and swapping one configuration parameter (`index_id`), with no code changes and no service redeployment. This makes the investment in the retrieval/reranking/hop-planning logic transferable to future, higher-value corpora (e.g., internal technical documentation) rather than a one-off HotpotQA demo. On cost, the demo is designed to run almost entirely within AWS Free Tier limits at low query volume (see Section 6), keeping the marginal cost of running and iterating on the workshop deployment close to zero.

### 3. Solution Architecture
The system is split into two independent pipelines. The **offline pipeline** runs once per corpus/index version: it reads `corpus.jsonl`, splits each article into a *parent* chunk (the full article, used for generation context) and several *child* chunks (250–500 characters, used for search precision), embeds the child chunks with BAAI/bge-m3, builds a BM25 sparse index over the same child chunks, and writes a versioned `index_manifest.json` before uploading everything to Amazon S3 / Amazon S3 Vectors. The **online pipeline** runs per HTTP request: it loads the pre-built artifacts (never re-chunking or re-embedding), optionally decomposes the question into sub-questions, retrieves candidate parent documents through hybrid BM25+vector search fused by Reciprocal Rank Fusion, expands winning child chunks back to their parent article ("small-to-big"), asks an LLM hop-planner whether the gathered evidence is sufficient or another targeted search is needed (up to a configurable maximum number of hops), reranks the surviving candidates with a cross-encoder, builds a filtered context window, and finally asks an LLM to produce a short-form answer. The response returned to the caller includes the answer, the supporting source documents with their rerank scores, per-stage latency timings, and LLM token usage — making every request self-describing for debugging and cost monitoring.

```text
Browser (React / Vite)
  -> HTTPS: AWS Amplify Hosting
  -> HTTPS: Amazon API Gateway (HTTP API)     [terminates TLS, avoids browser "Mixed Content" errors]
  -> HTTP:  Amazon EC2 (FastAPI under systemd)
       -> Amazon S3                (processed docs + BM25 index + manifest)
       -> Amazon S3 Vectors        (dense vector retrieval)
       -> .env.prod on the instance (Groq API key + runtime config; Parameter Store/Secrets Manager migration designed, not yet deployed)
       -> Groq API (third-party)   (query decomposition, hop planning, answer generation)
```

![AWS deployment architecture: Amplify -> API Gateway -> EC2 (VPC, public subnet) -> S3 (sparse search) and S3 Vectors (dense search), with EC2 calling the external Groq LLM API](/images/2-Proposal/AWS-RAG.drawio.png)
*Deployment architecture: the browser reaches the FastAPI backend through Amplify and API Gateway; the EC2 instance (behind an IAM role, inside a VPC public subnet) reads sparse/dense indexes from S3 and S3 Vectors and calls the external Groq API directly for LLM inference. The diagram also shows the originally planned Secrets Manager / Systems Manager / CloudWatch integration; in the deployed system the Groq API key and all runtime configuration instead live in a single `.env.prod` file on the instance, and operational visibility comes from the systemd journal and manual `/health`/`/warmup` checks rather than CloudWatch — see the "AWS Services Used" note below.*

#### AWS Services Used
- **Amazon S3** — durable storage for offline artifacts: parent/child document JSONL files, the serialized BM25 index, and the index manifest that records embedding model, chunk sizes, and checksums for each build.
- **Amazon S3 Vectors** — the managed vector-search service used for dense retrieval in production, queried per-request via `QueryVectors` and populated at build time via `PutVectors`; replaces a local ChromaDB instance used during development.
- **Groq API** — hosts the LLMs used for query decomposition, adaptive hop planning, and short-form answer generation. The API key currently lives in plain text in a `.env.prod` file on the EC2 instance rather than in AWS Secrets Manager; a migration that loads the Groq key from AWS Secrets Manager and non-secret settings from AWS Systems Manager Parameter Store is designed and coded but **not yet deployed**, and is tracked as an open limitation rather than a solved problem.
- **Amazon EC2, Amazon API Gateway, AWS Amplify Hosting, AWS Systems Manager Session Manager, AWS IAM, and an Elastic IP** — the compute, API, hosting, admin-access, access-control, and networking services that together host and expose the pipeline as a public demo endpoint (full breakdown in Section 4).
- **No Amazon CloudWatch.** Amazon CloudWatch is not part of the implemented system; operational visibility instead comes from the EC2 instance's systemd journal and manual `/health` / `/warmup` checks.

#### Component Design
- **Data Ingestion**: `scripts/build_offline_artifacts.py` reads a HotpotQA validation slice (`corpus.jsonl`), and `advanced_rag/chunking.py` parallelizes (via `multiprocessing.Pool`) the split into parent documents (full article text) and child documents (250–500 character chunks with a 20% overlap, using a recursive character splitter that prefers paragraph/sentence boundaries).
- **Retrieval**: `advanced_rag/retrieval.py` builds a hybrid retriever per hop by combining a BM25 sparse retriever (`bm25s`) and a dense vector retriever (Amazon S3 Vectors or ChromaDB, backed by BAAI/bge-m3 embeddings) through LangChain's `EnsembleRetriever`, which merges the two ranked lists with weighted Reciprocal Rank Fusion; winning child chunks are then expanded back to their parent article ("small-to-big").
- **Multi-Hop Reasoning**: `advanced_rag/query_optimizer.py` first decomposes the original question into ordered sub-questions with an LLM; `advanced_rag/hop_planner.py` then drives an adaptive loop (up to a configurable maximum number of hops) that reads the evidence retrieved so far and either declares the question answerable or proposes a new, more specific follow-up query grounded in a fact just discovered — replacing an earlier, brittle regex-based bridge-entity heuristic.
- **Answer Generation**: `advanced_rag/rerank.py` scores surviving candidates with a cross-encoder against the original question and every sub-question/hop query, keeping only the top-N; `advanced_rag/generation.py` then prompts an LLM to produce the shortest correct answer span (forcing an explicit intermediate "Reasoning:" step for comparison-type questions before the final "Answer:" line).
- **Evaluation**: `advanced_rag/qa_metrics.py` implements Exact Match and token-overlap F1 following the standard SQuAD/HotpotQA normalization rules; `evals/eval_hotpotqa.py` additionally measures *candidate coverage* at both the pre-rerank and post-rerank stages across an "easy" and a "hard" (mined bridge-question) tier, so that a retrieval failure can be attributed to the correct pipeline stage.

### 4. Technical Implementation
**Implementation Phases**
1. Dataset preparation — build a HotpotQA validation subset (`corpus.jsonl`) and confirm document/answer format.
2. Baseline retrieval — implement parent/child chunking, BM25 indexing, and dense embedding with BAAI/bge-m3 over a local ChromaDB store; validate single-pass retrieval quality.
3. Hybrid retrieval and reranking — fuse BM25 and vector search via Reciprocal Rank Fusion, tune per-retriever weights, and add cross-encoder reranking.
4. Multi-hop pipeline — add LLM-based query decomposition and adaptive hop planning, replacing the initial regex-based bridge-entity heuristic once its structural blind spots were identified.
5. Cloud migration — move the vector store to Amazon S3 Vectors, package offline artifacts with a versioned manifest, and deploy the online service to Amazon EC2 behind Amazon API Gateway, with the front end on AWS Amplify.
6. Evaluation and iteration — run `eval_hotpotqa.py` / `eval_full.py` for Exact Match/F1 and candidate-coverage diagnostics, and iterate on chunk size, top-k, and hop-planning limits based on measured results (see `docs/CHANGES_LOG.md`).
7. Latency hardening — introduce a `/warmup` endpoint and a `RAG_FAST_MODE` configuration flag to keep request latency within API Gateway's timeout on CPU-only hardware.

**Technical Requirements**
- Dataset: HotpotQA (multi-hop reasoning question-answering dataset), a 500-row validation slice for the demo deployment.
- Frameworks/libraries: LangChain (`langchain-core`, `langchain-classic`, `langchain-chroma`, `langchain-huggingface`), `sentence-transformers` (BAAI/bge-m3 embeddings, `ms-marco-MiniLM-L-6-v2` cross-encoder reranker), `bm25s`, ChromaDB, FastAPI, and the Groq Python SDK.
- AWS services/tools required to run and evaluate the pipeline: Amazon EC2 (backend compute), Amazon S3 (artifact storage), Amazon S3 Vectors (dense retrieval), Amazon API Gateway (public HTTPS API), AWS Amplify (frontend hosting), AWS Systems Manager Session Manager (admin access), and AWS IAM (instance role permissions in place of hard-coded credentials). Runtime configuration, including the Groq API key, currently lives in a `.env.prod` file on the instance; moving non-secret configuration to AWS Systems Manager Parameter Store and the Groq key to AWS Secrets Manager is designed but not yet deployed.

### 5. Timeline & Milestones
**Project Timeline**
- Internship period: 10/6/2026 – 30/7/2026
- Weeks 1–2: Dataset preparation and baseline single-pass retrieval (BM25 + dense embedding over a local ChromaDB store).
- Weeks 3–4: Hybrid retrieval via Reciprocal Rank Fusion, cross-encoder reranking, and initial retrieval-quality evaluation.
- Weeks 5–6: LLM-based query decomposition and adaptive hop planning; replacement of the early regex bridge-entity heuristic.
- Weeks 7: Migration to Amazon S3 Vectors and packaging of versioned offline artifacts (manifest + checksums).
- Week 8: Deployment to Amazon EC2 behind Amazon API Gateway, frontend deployment on AWS Amplify, centralized configuration via SSM/Secrets Manager.
- Week 9: Latency hardening (`/warmup`, `RAG_FAST_MODE`) and full EM/F1 + candidate-coverage evaluation runs.
- Week 10: Final report, documentation (`docs/STEP_*.md`, this proposal), and workshop presentation.

### 6. Budget Estimation
Most of the system is pay-per-use and close to free at this scale — Amazon S3, Amazon S3 Vectors, Amazon API Gateway, and AWS Amplify Hosting bill only for what is actually stored or requested. The exception is the compute tier: **Amazon EC2, its attached Elastic IP, and its EBS root volume all bill continuously by the hour, whether or not anyone sends a query**, and are only reduced (not eliminated) by stopping the instance when it is not being demonstrated. Figures below are **estimates for planning purposes**, not billed amounts.

| AWS Service | Billing Shape | Assumed Demo Usage | Estimated Monthly Cost |
| --- | --- | --- | --- |
| Amazon EC2 (t2/t3.micro) | Continuous while running, billed hourly | 1 instance, ~730 hrs/month | $0.00 if still Free-Tier-eligible (first 12 months); billed hourly otherwise |
| Amazon EC2 Elastic IP | Continuous, billed hourly regardless of instance state (current AWS public IPv4 pricing) | 1 address, kept associated | ~$3.60 |
| Amazon EBS (root volume) | Continuous, billed even while the instance is stopped | 1 gp3 volume, ~20-30 GB | ~$2.00 |
| Amazon S3 (Standard) | Per GB stored + requests | Offline artifacts well under 5 GB, low request volume | $0.00 (within Free Tier) |
| Amazon S3 Vectors | Per vector stored + queries, no idle cost | ~500 documents, low query volume | ~$0.50 |
| Amazon API Gateway (HTTP API) | Per request | A few hundred requests/month | $0.00 (within Free Tier) |
| AWS Amplify Hosting | Build minutes + storage + transfer | 1 small React build, low viewer traffic | $0.00 (within Free Tier) |
| AWS Systems Manager Session Manager | Always free | Occasional admin sessions; no Parameter Store parameters are created (config lives in `.env.prod`) | $0.00 |
| AWS IAM | Always free | 1 instance role, 2 inline policies | $0.00 |
| AWS Secrets Manager | Per secret per month | 1 secret exists but is **not yet wired into the application** (see Section 3) | ~$0.40 |
| Groq API (non-AWS, pass-through) | Per token | Demo-level query volume | ~$0.00–$1.00 |
| **Estimated total** | | | **~$6-8 / month while the instance runs continuously; near $0 additional beyond EC2/Elastic IP/EBS if stopped between demos** |

Amazon CloudWatch is intentionally absent from this table: it is not part of the implemented system, and monitoring instead relies on the EC2 instance's systemd journal and manual `/health`/`/warmup` checks. This estimate assumes the Free Tier clock has not already been exhausted by other workloads on the same account, and that traffic stays at demo scale; a production-scale deployment (higher query volume, larger corpus, GPU-backed inference) would require a separate, higher-traffic cost model.

### 7. Risk Assessment
#### Risk Matrix
| Risk | Likelihood | Impact |
| --- | --- | --- |
| Bridge-entity retrieval failure (correct documents never enter the candidate pool) | Medium | High — directly caps achievable EM/F1 |
| Hallucinated or ungrounded final answers | Medium | High — undermines the core RAG value proposition |
| Request latency approaching/exceeding API Gateway's ~30s timeout on CPU-only EC2 | Medium | Medium — failed requests in the demo |
| Small evaluation set (500-document validation slice) overstating real-world accuracy | High | Medium — results may not generalize to larger corpora |
| Groq API rate limits or transient errors during evaluation or demo | Medium | Low–Medium — handled by retry/fallback, but can still degrade single answers |
| No authentication on the public demo API | High (by design, for a demo) | Low for a short-lived workshop demo, higher if left running long-term |
| Free-tier resource constraints limit pipeline capability: (a) the cross-encoder reranker must stay disabled in production because it is too slow on a Free-Tier, CPU-only EC2 instance; (b) Groq's free-tier per-model token/minute cap throttles or blocks decomposition/hop-planning/generation calls under any real concurrent load; (c) the hybrid BM25+vector retrieval step can intermittently error out or time out under the limited CPU/memory of a Free-Tier instance running BM25 search and embedding inference concurrently | High | High — directly reduces answer quality (no rerank) and can cause failed `/query` requests |

#### Mitigation Strategies
- Bridge-entity failures are mitigated by hybrid BM25+vector retrieval with per-hop weighting, a per-hop candidate cap (so one noisy hop cannot crowd out a later correct one), and an adaptive LLM hop-planner that issues a targeted follow-up query instead of relying on a single retrieval pass.
- Hallucination is mitigated by strictly conditioning generation on the reranked, filtered context (`CONTEXT_MIN_RERANK_SCORE` threshold) and by constraining the generation prompt to short-form, context-only answers with an explicit "unknown" fallback when evidence is insufficient.
- Latency risk is mitigated by a `/warmup` endpoint invoked on service restart and a `RAG_FAST_MODE` configuration flag that skips decomposition and shrinks top-k/hop limits for demo traffic.
- The small-corpus generalization risk is mitigated by an explicit two-tier evaluation set (an "easy" tier and a harder, mined bridge-question tier spread across the corpus) that avoids overstating retrieval quality from an easy sample alone.
- Groq rate-limit risk is mitigated by per-model call throttling and automatic retry with backoff honoring the API's `Retry-After` header, with a safe fallback (return the original question / stop hopping) if retries are exhausted.
- The authentication gap is accepted for the duration of the workshop demo and flagged explicitly as a known limitation, with API-key or IAM-authorizer-based authentication identified as the mitigation before any longer-lived or public deployment.
- Free-tier resource constraints are mitigated pragmatically rather than eliminated, since upgrading compute/API tiers is out of scope for a zero-cost demo: the reranker is kept disabled by default in production (`RAG_USE_RERANKER=false`) and relies solely on hybrid retrieval order, trading some answer quality for a workload the Free-Tier CPU can sustain; Groq's per-model rate limit is respected proactively via a minimum call interval between requests to the same model plus automatic retry honoring the `Retry-After` header (`groq_utils.py`), so a burst of demo traffic degrades gracefully instead of failing outright; and intermittent hybrid-retrieval errors/timeouts are handled by `RAG_FAST_MODE` (smaller top-k, fewer hops, decomposition skipped) to keep the retrieval step's resource footprint within what the Free-Tier instance can reliably serve, with the known residual risk that a spike in concurrent requests can still cause failed `/query` calls.

#### Contingency Plans
If retrieval accuracy on the hard evaluation tier does not meet the target after tuning hybrid weights and hop-planning limits, the fallback plan is to reduce scope to the easy tier and a smaller, curated question set for the workshop demonstration while documenting the gap as future work. If EC2/API Gateway latency cannot be brought under the timeout even with `RAG_FAST_MODE`, the contingency is to demo the pipeline via the local CLI (`tools/query.py`) and a recorded walkthrough rather than the live public endpoint. If the Groq free/low-cost tier is exhausted mid-project, the contingency is to temporarily lower `GROQ_MIN_CALL_INTERVAL` demand by running fewer, smaller evaluation batches, or to swap in an alternative hosted LLM behind the same `groq_utils.py` retry/throttle interface.

### 8. Expected Outcomes
#### Technical Improvements
The expected outcome is a measurable improvement in multi-hop Exact Match/F1 compared to a single-pass retrieval baseline (one retrieval round, no decomposition, no reranking), attributable specifically to the hybrid retrieval + adaptive hop-planning + reranking stack. Beyond the headline metric, the two-stage candidate-coverage evaluation is expected to show materially higher document-coverage at the pre-rerank stage than a naive single-query baseline, confirming that the accuracy gain comes from finding the right evidence rather than only from better ranking of an already-limited candidate set.

#### Long-term Value
Because the offline/online split and the retrieval-decomposition-hop-planning-rerank-generation pipeline are corpus-agnostic (every corpus-specific detail lives in one versioned artifact bundle plus a small set of configuration parameters), this project's real long-term value is as a **reusable RAG product**, not a one-off HotpotQA demo. The same pipeline can be repointed — by building a new offline artifact bundle and swapping `index_id` — at higher-value, higher-reasoning-demand document collections such as internal engineering/legal/research documentation, technical specifications, or other knowledge bases where questions routinely require combining facts across multiple documents and where answer grounding and traceability (returned sources, rerank scores, and token usage) are business-critical. The adaptive hop-planning mechanism in particular is expected to generalize well to any domain characterized by cross-document "bridge" relationships, making this architecture a candidate foundation for future internal document-QA tooling rather than a disposable workshop artifact.

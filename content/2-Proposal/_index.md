---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

In this section, you need to summarize the contents of the workshop that you **plan** to conduct.

# Retrieval-Augmented Generation for Multi-Hop Reasoning on HotpotQA
## An Adaptive Hybrid-Retrieval RAG Pipeline with Hop-Aware Query Planning, Deployed on AWS

### 1. Executive Summary
This project designs and deploys a Retrieval-Augmented Generation (RAG) system to answer multi-hop reasoning questions from the HotpotQA dataset — questions whose answers require combining evidence from multiple documents rather than a single passage. The system is scoped as a full end-to-end demo: an offline indexing pipeline that chunks and embeds a HotpotQA validation slice, an online FastAPI retrieval-and-generation service exposing a public API, and a React front end for interactive testing. The target scale is a small, cost-conscious demo deployment (a 100-document HotpotQA validation subset, low query volume) rather than a production-scale service, intended for internal evaluation, workshop demonstration, and as a reference architecture that can later be scaled to larger corpora. Intended users are workshop reviewers, the engineering team evaluating retrieval quality, and future engineers who will extend the pipeline to other document collections.

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
       -> AWS Systems Manager Parameter Store (non-secret runtime config)
       -> AWS Secrets Manager      (Groq API key)
       -> Amazon CloudWatch        (logs & metrics)
       -> Groq API (third-party)   (query decomposition, hop planning, answer generation)
```

![AWS deployment architecture: Amplify -> API Gateway -> EC2 (VPC, public subnet) -> S3 (sparse search), S3 Vectors (dense search), Secrets Manager, Systems Manager, and CloudWatch, with EC2 calling the external Groq LLM API](/images/2-Proposal/AWS-RAG.drawio.png)
*Deployment architecture: the browser reaches the FastAPI backend through Amplify and API Gateway; the EC2 instance (behind an IAM role, inside a VPC public subnet) pulls its Groq API key from Secrets Manager, reads sparse/dense indexes from S3 and S3 Vectors, is configured via Systems Manager, ships logs and metrics to Amazon CloudWatch, and calls the external Groq API directly for LLM inference.*

#### AWS Services Used
- **Amazon S3** — durable storage for offline artifacts: parent/child document JSONL files, the serialized BM25 index, and the index manifest that records embedding model, chunk sizes, and checksums for each build.
- **Amazon S3 Vectors** — the managed vector-search service used for dense retrieval in production, queried per-request via `QueryVectors` and populated at build time via `PutVectors`; replaces a local ChromaDB instance used during development.
- **Groq API (via AWS Secrets Manager for key storage)** — hosts the LLMs used for query decomposition, adaptive hop planning, and short-form answer generation; the API key itself is stored in and retrieved from AWS Secrets Manager rather than hard-coded.
- **Amazon EC2, Amazon API Gateway, AWS Amplify Hosting, AWS Systems Manager (Parameter Store + Session Manager), AWS IAM, and an Elastic IP** — the compute, API, hosting, configuration, access-control, and networking services that together host and expose the pipeline as a public demo endpoint (full breakdown in Section 4).
- **Amazon CloudWatch** — collects logs and metrics from the EC2-hosted service for monitoring request volume, errors, and latency in the deployed demo.

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
- Dataset: HotpotQA (multi-hop reasoning question-answering dataset), a 100-row validation slice for the demo deployment.
- Frameworks/libraries: LangChain (`langchain-core`, `langchain-classic`, `langchain-chroma`, `langchain-huggingface`), `sentence-transformers` (BAAI/bge-m3 embeddings, `ms-marco-MiniLM-L-6-v2` cross-encoder reranker), `bm25s`, ChromaDB, FastAPI, and the Groq Python SDK.
- AWS services/tools required to run and evaluate the pipeline: Amazon EC2 (backend compute), Amazon S3 (artifact storage), Amazon S3 Vectors (dense retrieval), Amazon API Gateway (public HTTPS API), AWS Amplify (frontend hosting), AWS Systems Manager Parameter Store and Session Manager (runtime configuration and admin access), AWS Secrets Manager (Groq API key), and AWS IAM (instance role permissions in place of hard-coded credentials).

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
The demo deployment is designed to stay within AWS Free Tier limits wherever possible, assuming a new/eligible AWS account, a single small EC2 instance running continuously, and low demo-level traffic (on the order of a few hundred `/query` requests per month, well below any Free Tier request ceiling). Figures below are **estimates for planning purposes**, not billed amounts.

| AWS Service | Free Tier Allowance (typical) | Assumed Demo Usage | Estimated Monthly Cost |
| --- | --- | --- | --- |
| Amazon EC2 (t2/t3.micro) | 750 instance-hours/month for 12 months | 1 instance, ~730 hrs/month | $0.00 |
| Amazon EC2 Elastic IP | Free while associated with a running instance | 1 address, kept associated | $0.00 |
| Amazon S3 (Standard) | 5 GB storage, 20,000 GET / 2,000 PUT per month | Offline artifacts ≪ 5 GB, low request volume | $0.00 |
| Amazon S3 Vectors | No dedicated free tier at time of writing | ~100 documents, low query volume | ~$0.50 |
| Amazon API Gateway (HTTP API) | 1,000,000 requests/month for 12 months | A few hundred requests/month | $0.00 |
| AWS Amplify Hosting | 1,000 build minutes + 15 GB served/month | 1 small React build, low viewer traffic | $0.00 |
| AWS Systems Manager (Parameter Store standard, Session Manager) | Always free (standard tier) | ~18 parameters, occasional admin sessions | $0.00 |
| AWS IAM | Always free | 1 instance role, 2 inline policies | $0.00 |
| AWS Secrets Manager | 30-day free trial per secret, then ~$0.40/secret/month | 1 secret (Groq API key), beyond trial | ~$0.40 |
| Groq API (non-AWS, pass-through) | Free/low-cost tier for low request volume | Demo-level query volume | ~$0.00–$1.00 |
| **Estimated total** | | | **≈ $1–2 / month** |

This estimate assumes the Free Tier clock has not already been exhausted by other workloads on the same account, and that traffic stays at demo scale; a production-scale deployment (higher query volume, larger corpus, GPU-backed inference) would require a separate, higher-traffic cost model.

### 7. Risk Assessment
#### Risk Matrix
| Risk | Likelihood | Impact |
| --- | --- | --- |
| Bridge-entity retrieval failure (correct documents never enter the candidate pool) | Medium | High — directly caps achievable EM/F1 |
| Hallucinated or ungrounded final answers | Medium | High — undermines the core RAG value proposition |
| Request latency approaching/exceeding API Gateway's ~30s timeout on CPU-only EC2 | Medium | Medium — failed requests in the demo |
| Small evaluation set (100-document validation slice) overstating real-world accuracy | High | Medium — results may not generalize to larger corpora |
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

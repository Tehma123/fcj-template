---
title: "Evaluation"
date: 2026-07-31
weight: 11
chapter: false
pre: " <b> 5.11. </b> "
---

After validating that the deployment works correctly, the system is evaluated on retrieval quality, answer quality, and runtime performance. All final results use the corrected **HotpotQA Distractor v002** artifact.

The retrieval benchmark contains **500 validation questions**, **4,937 parent documents**, and **8,279 BGE-M3 child vectors**. Retrieval metrics are calculated at the supporting-document-title level, while final answer quality is measured with Exact Match (EM) and token-level F1.

## Retrieval Quality

The retrieval pipeline is evaluated at two stages. The **candidate pool** measures whether the required evidence was discovered at all, while **Selected Top-10** measures the quality of the final evidence ranking after selection.

| Metric | Candidate Pool | Selected Top-10 |
| --- | ---: | ---: |
| Mean supporting-title recall | **0.9920** | **0.9740** |
| All supporting titles found | **0.9840** | **0.9480** |
| Recall@5 | 0.5420 | **0.9310** |
| Recall@10 | 0.6270 | **0.9740** |
| Precision@10 | 0.1254 | **0.1948** |
| MRR | 0.7807 | **0.9446** |
| nDCG@10 | 0.5816 | **0.9162** |

The candidate stage recovered almost all required supporting documents, with complete supporting-title coverage for **492 of 500 questions**. After reducing the candidate pool to the final ten documents, complete coverage remained high at **474 of 500 questions**.

The strongest improvement appears in ranking quality. Recall@5 increased from **0.5420 to 0.9310**, while MRR increased from **0.7807 to 0.9446** and nDCG@10 from **0.5816 to 0.9162**. This shows that the final selection stage moves relevant evidence much closer to the top of the context presented to the generation pipeline.

## End-to-End Answer Quality

A fixed set of **20 questions** was used to evaluate the complete path from retrieval to answer generation. All 20 questions completed successfully.

| Metric | Result |
| --- | ---: |
| Answer EM | **0.7500** |
| Answer F1 | **0.7750** |
| Correct answers | **15 / 20** |
| Selected evidence recall | **0.9500** |
| All supporting titles found | **0.9000** |

The selected evidence contained all required supporting documents for **18 of 20 questions**. The difference between evidence recall and final answer accuracy also shows that finding the correct documents is necessary, but the final result still depends on how effectively the retrieved evidence is used during generation.

## Runtime Performance

Across the 500-question retrieval benchmark, the pipeline required an average of **25.91 seconds per question**.

| Stage | Mean latency | Share |
| --- | ---: | ---: |
| Query decomposition | 4.32 s | 16.7% |
| Retrieval + adaptive planning | 21.07 s | 81.3% |
| Cross-encoder reranking | 0.53 s | 2.0% |
| Total retrieval pipeline | **25.91 s** | 100% |

The main runtime cost comes from retrieval and adaptive planning rather than reranking. The reranker contributes only about 2% of total retrieval latency while producing a substantial improvement in ranking quality.

For the 20-question end-to-end benchmark, the complete response time was:

| Component | Mean latency |
| --- | ---: |
| Retrieval pipeline | 26.86 s |
| Final generation | 12.43 s |
| End-to-end | **39.29 s** |

These results show the main trade-off in CloudHop RAG: broader multi-hop retrieval and reranking improve evidence quality, but they also increase response time. The deployed AWS runtime therefore uses a lighter production configuration to reduce retrieval work and keep response latency practical on the EC2 environment.

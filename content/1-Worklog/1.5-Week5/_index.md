---
title: "Week 5 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.5. </b> "
---


### Week 5 Objectives:

* Understand what Retrieval-Augmented Generation (RAG) is and why it helps reduce hallucination.
* Build a naive (single-pass) RAG pipeline: chunk → embed → retrieve → generate.
* Run the naive pipeline on a sample of HotpotQA questions and measure a baseline accuracy.

### Tasks to be carried out this week:
| Day | Task                                                                                                                                       | Start Date | Completion Date | Reference Material                                              |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | --------------- | ----------------------------------------------------------------- |
| 4   | - Learn RAG fundamentals: retrieval + generation, grounding LLM answers in retrieved context                                                  | 07/08/2026 | 07/08/2026      |                                                                    |
| 5   | - Learn the naive RAG pipeline: <br>&emsp; + Document chunking strategies <br>&emsp; + Embedding the corpus <br>&emsp; + Storing vectors <br>&emsp; + Top-k similarity search | 07/09/2026 | 07/09/2026      |                                                                     |
| 6   | - **Practice:** chunk and embed a subset of HotpotQA context paragraphs, store the embeddings in a vector index                               | 07/10/2026 | 07/10/2026      |                                                                     |
| 2   | - **Practice:** implement single-pass retrieval + prompt construction, generate answers for sample questions                                  | 07/13/2026 | 07/13/2026      |                                                                     |
| 3   | - Evaluate the naive RAG pipeline on a small HotpotQA sample (Exact Match / F1) and note failure cases on multi-hop questions                 | 07/14/2026 | 07/14/2026      |                                                                     |


### Week 5 Achievements:

* Understood the core idea of RAG and why grounding generation in retrieved evidence reduces hallucination.

* Built a working naive RAG pipeline: chunking, embedding, vector similarity search, and prompt-based generation.

* Ran the pipeline end-to-end on a sample of HotpotQA questions.

* Measured a baseline accuracy (EM/F1) and observed that naive, single-pass retrieval frequently fails on multi-hop questions that need evidence from more than one document.

* Identified this gap as the motivation for exploring advanced RAG techniques next week.
* ...

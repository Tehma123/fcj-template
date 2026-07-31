---
title: "Week 6 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.6. </b> "
---


### Week 6 Objectives:

* Understand the limitations of naive RAG on multi-hop questions.
* Learn and apply advanced RAG techniques: query decomposition, iterative/multi-hop retrieval, and re-ranking.
* Improve retrieval quality and answer accuracy on HotpotQA compared to the Week 5 baseline.

### Tasks to be carried out this week:
| Day | Task                                                                                                                                                 | Start Date | Completion Date | Reference Material |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | --------------- | ------------------- |
| 4   | - Learn advanced RAG techniques: <br>&emsp; + Query rewriting / decomposition <br>&emsp; + Iterative retrieval <br>&emsp; + Re-ranking <br>&emsp; + Hybrid (keyword + vector) search | 07/15/2026 | 07/15/2026      |                      |
| 5   | - **Practice:** implement query decomposition to break a multi-hop question into sub-questions                                                     | 07/16/2026 | 07/16/2026      |                      |
| 6   | - **Practice:** implement an iterative/multi-hop retrieval loop that uses the answer to one sub-question to retrieve evidence for the next          | 07/17/2026 | 07/17/2026      |                      |
| 2   | - **Practice:** add a re-ranking step on top of retrieved passages to improve context relevance before generation                                   | 07/20/2026 | 07/20/2026      |                      |
| 3   | - Re-evaluate the advanced RAG pipeline on the same HotpotQA sample used in Week 5 and compare EM/F1 against the naive baseline                    | 07/21/2026 | 07/21/2026      |                      |


### Week 6 Achievements:

* Understood why naive RAG struggles with multi-hop questions and how advanced retrieval strategies address that gap.

* Implemented query decomposition to break multi-hop questions into retrievable sub-questions.

* Built an iterative, multi-hop retrieval loop that chains evidence across documents.

* Added a re-ranking step that improved the relevance of retrieved context.

* Measured a clear accuracy improvement (EM/F1) over the Week 5 naive RAG baseline on HotpotQA.

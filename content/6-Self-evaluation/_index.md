---
title: "Self-Assessment"
date: 2024-01-01
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

During my internship at **FCAJ** from **10/6/2026** to **30/7/2026**, I had the opportunity to learn, practice, and apply the knowledge acquired in school to a real-world working environment.  
I participated in **Deploy RAG to solve multi-hop reasoning questions with datasets taken from hotpot_qa**, through which I improved my skills in **AWS cloud services, RAG system design, technical writing, static site deployment with Hugo and GitHub Actions, and problem-solving/debugging**.  

In terms of work ethic, I always strived to complete tasks well, complied with workplace regulations, and actively engaged with colleagues to improve work efficiency.

To objectively reflect on my internship period, I would like to evaluate myself based on the following criteria:

| No. | Criteria                                                | Description                                                                                                                                                                  | Good | Fair | Average |
| --- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---- | ---- | ------- |
| 1   | **Professional knowledge & skills (AWS & RAG)**           | Solid grasp of core AWS services (EC2, S3, IAM, VPC endpoints) applied in the hybrid-connectivity workshop, plus hands-on deployment of a RAG pipeline for multi-hop reasoning on the HotpotQA dataset | ✅    | ☐    | ☐       |
| 2   | **Ability to learn (new AWS services & tools)**           | Picked up new AWS services (S3 Metadata/Annotations, Cost Categories, Glue, Amplify, Cognito) and the Hugo static-site framework within a few weeks, and could explain them clearly enough to publish two technical blogs | ✅    | ☐    | ☐       |
| 3   | **Proactiveness**                                         | Took initiative to explore retrieval strategies for the RAG/HotpotQA pipeline beyond the minimum ask, and self-diagnosed/fixed GitHub Pages deployment issues (baseURL, canonifyURLs, image assets) without being asked | ✅    | ☐    | ☐       |
| 4   | **Sense of responsibility**                                | Delivered weekly worklogs, workshop labs, and blog posts on schedule, and kept the RAG project's task list up to date | ✅    | ☐    | ☐       |
| 5   | **Discipline**                                             | Generally followed the weekly reporting cadence, but a few worklog and blog deliverables were finished close to the deadline instead of being spaced out evenly | ☐    | ✅    | ☐       |
| 6   | **Progressive mindset**                                    | Incorporated mentor feedback quickly, such as adjusting the RAG pipeline based on evaluation results and correcting Hugo build/theme errors after review | ✅    | ☐    | ☐       |
| 7   | **Communication**                                          | Explains technical work well in writing (blogs, proposal, worklogs), but still needs to be more concise and structured when reporting progress verbally to mentors | ☐    | ✅    | ☐       |
| 8   | **Teamwork**                                               | Coordinated with FCAJ mentors and fellow interns during the S3 hybrid-connectivity workshop and study-group blog reviews | ✅    | ☐    | ☐       |
| 9   | **Professional conduct**                                   | Respected internship rules and regulations, and engaged courteously with mentors and the AWS Study Group community when sharing blogs | ✅    | ☐    | ☐       |
| 10  | **Problem-solving skills**                                 | Debugged and resolved concrete technical issues independently, such as a mislabeled avatar image file, a broken GitHub Actions build, and reasoning failures in the RAG pipeline | ✅    | ☐    | ☐       |
| 11  | **Contribution to project/team (RAG project & blog content)** | Contributed a working RAG pipeline for multi-hop QA on HotpotQA, working blog content, and a workshop write-up that the team can reuse as reference material | ✅    | ☐    | ☐       |
| 12  | **Overall**                                                | Completed the internship objectives, produced tangible technical output (RAG deployment, blogs, proposal, workshop), and grew steadily across most competencies | ✅    | ☐    | ☐       |

### Needs Improvement

* Strengthen discipline by spacing out weekly worklogs, workshop labs, and blog deadlines more evenly instead of finishing them close to the due date
* Continue improving communication skills — presenting technical work more concisely, both in writing and when discussing progress verbally with mentors

### Personal Contribution

**Self-implementation level**

* Not just copying a template: I designed and wrote the entire RAG pipeline myself (chunking, indexing, retrieval, hop planning, generation) rather than reusing an existing RAG tutorial/template as-is.
* Customization:
  * **Added features:** Implemented Query Decomposition (breaking multi-hop questions into sub-questions via an LLM) and an Adaptive Hop Planner (deciding the next retrieval hop dynamically based on the evidence gathered so far) — both go beyond a "naive" retrieve-once, generate-once RAG pipeline.
  * **Added services:** Integrated Amazon S3 Vectors as the production vector store (replacing local ChromaDB), plus a FastAPI service layer with dedicated health-check/warmup endpoints.
  * **Different data handling:** Used a self-processed HotpotQA validation slice with custom parent/child chunking and self-built BM25 and vector indexes, instead of a pre-processed dataset.
  * **Different testing approach:** Built a proper Exact Match/F1 evaluation suite matching the HotpotQA scoring standard, instead of just running a demo on a handful of sample questions.

**Short reflection**

* **Difficulty encountered:** The biggest constraint was infrastructure — since only AWS Free Tier was available (no GPU), the cross-encoder reranking step, which needs a transformer model to re-score retrieved candidates, couldn't run efficiently in production, so the reranker had to be disabled in the actual deployment.
* **How it was solved:** I designed the system for graceful degradation — when the reranker is disabled, the pipeline still works correctly by keeping the ranking order from hybrid retrieval (BM25 ⊕ Vector via Reciprocal Rank Fusion) instead of crashing or hard-requiring a GPU.
* **Future direction:** With access to GPU infrastructure (e.g., a GPU-backed EC2 instance or a managed inference service), I would re-enable cross-encoder reranking to improve the accuracy of the context fed into answer generation, especially for multi-hop questions with many distracting candidates.

---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---
{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}

In this section, you need to summarize the contents of the workshop that you **plan** to conduct.

# Retrieval-Augmented Generation for Multi-Hop Reasoning on HotpotQA
## [Add a short subtitle describing your specific approach or deployment]

### 1. Executive Summary
This project designs and deploys a Retrieval-Augmented Generation (RAG) system to answer multi-hop reasoning questions from the HotpotQA dataset — questions whose answers require combining evidence from multiple documents rather than a single passage. [Add scope details: target scale, deployment environment, intended users, etc.]

### 2. Problem Statement
#### What's the Problem?
Standalone large language models (LLMs) and single-pass QA systems often struggle with multi-hop questions, since the answer cannot be found in one document and requires retrieving and reasoning across several related passages. Without a retrieval step grounded in the source corpus, answers can be inaccurate, ungrounded, or miss intermediate reasoning steps. [Add context specific to why this matters for your use case]

#### The Solution
The system retrieves relevant passages from the HotpotQA document corpus and uses a language model to generate the final answer conditioned on that retrieved evidence, chaining multiple retrieval/reasoning steps together to resolve multi-hop questions. [Add the specific retrieval and generation stack used — e.g., embedding model, vector store, LLM, orchestration framework]

#### Benefits and Return on Investment
[Add expected benefits, e.g., improved accuracy on multi-hop questions vs. a non-RAG baseline, a reusable retrieval pipeline for other QA datasets]  
[Add cost details if applicable]

### 3. Solution Architecture
[Add an architecture diagram and a description of how data flows from the HotpotQA dataset through retrieval, multi-hop reasoning, and answer generation]

#### AWS Services Used
- [Add the storage service used, e.g., for the HotpotQA corpus and embeddings]
- [Add the retrieval/vector search service used]
- [Add the LLM/embedding service used]
- [Add any compute, orchestration, or API service used]

#### Component Design
- **Data Ingestion**: [Describe how the HotpotQA dataset is loaded and preprocessed]
- **Retrieval**: [Describe how relevant passages are retrieved for each question]
- **Multi-Hop Reasoning**: [Describe how the system chains multiple retrieval/reasoning steps together]
- **Answer Generation**: [Describe how the final answer is generated from the retrieved evidence]
- **Evaluation**: [Describe how generated answers are scored against HotpotQA ground truth, e.g., EM/F1]

### 4. Technical Implementation
**Implementation Phases**
[Add the phases followed, e.g., dataset exploration → baseline retrieval → multi-hop pipeline → evaluation → iteration]

**Technical Requirements**
- Dataset: HotpotQA (multi-hop reasoning question-answering dataset)
- [Add frameworks/libraries used, e.g., LangChain, LlamaIndex, or custom retrieval code]
- [Add AWS services/tools required to run and evaluate the pipeline]

### 5. Timeline & Milestones
**Project Timeline**
- Internship period: 10/6/2026 – 30/7/2026
- [Add weekly/monthly milestones, e.g., research phase, pipeline development, evaluation, final report]

### 6. Budget Estimation
[Add budget details if applicable, e.g., AWS compute/storage costs for running and evaluating the RAG pipeline]

### 7. Risk Assessment
#### Risk Matrix
[Add risks specific to this project, e.g., retrieval accuracy, hallucination, latency, evaluation dataset size]

#### Mitigation Strategies
[Add how each risk above would be mitigated]

#### Contingency Plans
[Add fallback plans if the approach doesn't meet the target accuracy or timeline]

### 8. Expected Outcomes
#### Technical Improvements
[Add the expected improvement in multi-hop QA accuracy compared to a baseline]

#### Long-term Value
[Add how this RAG pipeline could be reused for other datasets or future projects]

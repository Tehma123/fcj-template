---
title: "Workshop Overview"
date: 2026-07-31
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

## HotpotQA and the Multi-Hop Question Answering Problem

Many question-answering systems retrieve a small set of relevant documents and use them directly to generate an answer. This works well when the required evidence is concentrated in one place, but it becomes less reliable when the answer depends on information spread across several documents.

**HotpotQA** is a Wikipedia-based question-answering dataset designed for multi-hop reasoning. Unlike questions that can be answered from a single passage, many HotpotQA questions require information from two or more supporting documents. The dataset also provides annotated supporting facts, making it possible to evaluate not only whether an answer is correct, but also whether the system has retrieved the evidence needed to reach it.

CloudHop RAG uses the **Distractor** setting of HotpotQA. In this setting, each question is accompanied by relevant supporting documents together with unrelated distractor documents. The questions include both **bridge** cases, where one piece of evidence leads to another, and **comparison** cases, where information about multiple entities must be combined before an answer can be produced.

This creates a useful retrieval challenge for RAG. Finding one highly relevant document is often not enough; the system must recover complementary evidence and use what it has already found to continue the search when necessary. For CloudHop RAG, HotpotQA therefore provides a controlled setting for studying one of the central goals of the project: **whether a retrieval system can find the right evidence across multiple documents before asking the language model to produce the final answer.**

## CloudHop RAG

CloudHop RAG combines lexical, semantic, and multi-hop retrieval to handle this type of question. **BM25** is effective when the question contains distinctive names, terms, or phrases that also appear in the supporting documents, while **BGE-M3** helps retrieve semantically related evidence when the wording is different.

Rather than treating retrieval as a single search, the pipeline can use the evidence already found to guide subsequent retrieval steps. The resulting candidates are combined and reduced to a focused context, which is then used by the language model to generate the final answer.

## From Question to Answer

![AWS CloudHop RAG architecture overview](/images/2-Proposal/AWS-RAG.drawio.png)

When a user submits a question, the web interface hosted on **AWS Amplify** sends the request through **Amazon API Gateway** to the FastAPI backend running on **Amazon EC2**. The backend performs BM25 retrieval using artifacts stored in **Amazon S3** and dense retrieval against **Amazon S3 Vectors** using BGE-M3 embeddings.

The retrieved evidence is processed by the multi-hop RAG pipeline and assembled into the context sent to the Groq LLM API. The generated answer and its supporting sources are then returned through API Gateway to the frontend.


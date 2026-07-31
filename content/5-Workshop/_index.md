---
title: "AWS CloudHop RAG Deployment Workshop"
date: 2026-07-31
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

## Overview

A language model can generate fluent answers, but its response is limited by the knowledge available to the model itself. **Retrieval-Augmented Generation (RAG)** addresses this by retrieving relevant information from an external knowledge source before generating the answer. Instead of relying only on what the model already knows, the system can ground its response in evidence retrieved for the current question.

This becomes more challenging when the answer depends on information spread across several documents. One source may identify an important entity, while another contains the fact needed to complete the answer. **AWS CloudHop RAG** was built for this type of multi-hop question answering, where the system must retrieve complementary evidence, connect it across multiple steps, and use the resulting context to generate a grounded answer.

The project combines **hybrid retrieval, multi-hop evidence processing, and LLM-based generation** in a complete AWS deployment. Amazon S3 stores the processed corpus and retrieval artifacts, Amazon S3 Vectors supports dense semantic search, Amazon EC2 runs the FastAPI RAG backend, Amazon API Gateway exposes the application through HTTPS, and AWS Amplify delivers the frontend.

The workshop follows this system from retrieval artifact preparation to cloud deployment, testing, evaluation, operation, and cleanup. Rather than treating the RAG pipeline and AWS infrastructure as separate parts, the following sections show how they work together as one end-to-end application.

## Workshop Content

1. [Workshop Overview](5.1-Workshop-Overview/)
2. [Prerequisites](5.2-Prerequisites/)
3. [Architecture](5.3-Architecture/)
4. [Offline Artifact Build](5.4-Offline-Artifact-Build/)
5. [Amazon S3 Storage](5.5-S3-Storage/)
6. [Amazon S3 Vectors](5.6-S3-Vectors/)
7. [Amazon EC2 Backend Deployment](5.7-EC2-Backend/)
8. [Amazon API Gateway](5.8-API-Gateway/)
9. [AWS Amplify Frontend](5.9-Amplify-Frontend/)
10. [Testing and Validation](5.10-Testing-Validation/)
11. [Evaluation](5.11-Evaluation/)
12. [Operations and Troubleshooting](5.12-Operations-Monitoring/)
13. [Security and Cost Considerations](5.13-Security-Cost/)
14. [Cleanup](5.14-Cleanup/)

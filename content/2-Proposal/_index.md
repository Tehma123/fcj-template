---
title: "Proposal"
date: 2026-06-08
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

## Project Overview

During the AWS First Cloud AI Journey internship, my team and I proposed **AWS CloudHop RAG**, a Retrieval-Augmented Generation system designed for questions that require information from more than one document. The project focuses on multi-hop question answering, where finding one relevant passage is often not enough to produce the correct answer.

We chose **HotpotQA** as the main dataset because its questions are specifically designed around multi-hop reasoning and include annotated supporting evidence. This gives us a controlled environment to develop the retrieval pipeline and evaluate whether the system can find the documents needed to answer each question.

The project is planned as an end-to-end application rather than only a retrieval experiment. Along with developing and evaluating the RAG pipeline, my team will deploy the main application components on AWS and provide a simple web interface for submitting questions and viewing generated answers with their supporting sources.

## Problem and Motivation

Retrieval-Augmented Generation improves question answering by retrieving relevant information from an external knowledge source before asking a language model to generate an answer. This helps the model rely on retrieved evidence instead of depending entirely on what it already knows.

However, many RAG systems perform retrieval only once. This works well when the information needed to answer a question is contained in one relevant document, but it becomes less reliable when several pieces of evidence have to be connected.

A multi-hop question may require the system to first find one document, identify an important person, place, organization, or relationship from that evidence, and then use that information to locate another document. Missing either part of this evidence can lead to an incomplete or incorrect answer.

HotpotQA provides a useful benchmark for this problem because it contains both **bridge questions**, where one piece of evidence leads to another, and **comparison questions**, where information from multiple entities must be combined.

For this project, we therefore want to explore whether combining lexical retrieval, semantic retrieval, and additional retrieval steps can provide more complete evidence for multi-hop questions while still being practical to deploy as an AWS application.

## Objectives and Scope

### Project Objectives

The main objectives of AWS CloudHop RAG are to:

1. Develop a RAG pipeline that can answer multi-hop questions using evidence retrieved from multiple documents.
2. Explore both lexical and semantic retrieval methods and combine their strengths through hybrid retrieval.
3. Support additional retrieval steps when the evidence from the initial search is not sufficient.
4. Evaluate retrieval quality separately from final answer quality so that retrieval failures can be identified clearly.
5. Deploy the completed application using AWS services and provide a simple interface for interacting with the system.
6. Build the project in a reproducible way so that retrieval artifacts, evaluation results, and deployment steps can be recreated and documented.

### Project Scope

The project will focus on the **HotpotQA Distractor** setting as the main development and evaluation environment.

The planned scope includes:

- HotpotQA data preparation;
- lexical and dense retrieval;
- hybrid retrieval;
- multi-hop evidence retrieval;
- evidence ranking and context construction;
- LLM-based answer generation;
- retrieval and answer evaluation;
- AWS storage and vector search;
- cloud backend deployment;
- API and frontend integration;
- functional testing and technical documentation.

The project is intended as an internship-scale implementation and evaluation rather than a production service for large numbers of users. Large-scale traffic, enterprise authentication, and deployment over very large document collections are outside the main scope.

## Proposed Solution and Architecture

### Proposed RAG Approach

The proposed pipeline combines several retrieval strategies instead of relying on a single search method.

The overall flow is:

**Question → Query Analysis → Lexical and Semantic Retrieval → Multi-Hop Retrieval → Evidence Ranking → Context Construction → LLM Generation → Answer and Supporting Sources**

For lexical retrieval, the project will use **BM25**, which is effective when important names, entities, or phrases in the question also appear directly in the source documents.

For semantic retrieval, the project will use **BGE-M3 embeddings** to represent text as dense vectors. This allows the system to find relevant evidence even when the wording of the question and the source document is different.

The results from lexical and semantic retrieval will be combined into a shared candidate set. For questions that require several pieces of information, the system will also explore additional retrieval steps based on evidence found earlier in the process.

After retrieval, the strongest evidence will be ranked and reduced to a focused context before it is passed to the language model. The final response will contain both the generated answer and the supporting sources used to construct the context.

### Proposed AWS Architecture

The RAG pipeline will be deployed as a web application using several AWS services with separate responsibilities.

![AWS CloudHop RAG proposed architecture](/images/2-Proposal/AWS-RAG.drawio.png)

The planned application flow is:

**User → AWS Amplify → Amazon API Gateway → Amazon EC2 → Amazon S3 / Amazon S3 Vectors → Groq API → Answer**

| Component | Planned Role |
| --- | --- |
| **AWS Amplify** | Host the web frontend used to submit questions and display answers |
| **Amazon API Gateway** | Provide an HTTPS API between the browser and backend |
| **Amazon EC2** | Run the FastAPI backend and coordinate the RAG pipeline |
| **Amazon S3** | Store the processed corpus, BM25 artifacts, mappings, and manifests |
| **Amazon S3 Vectors** | Store and search dense BGE-M3 vectors |
| **AWS IAM** | Control access between AWS resources |
| **AWS Systems Manager** | Support administration and access to the EC2 backend |
| **Groq API** | Provide language-model inference for the RAG pipeline |

The retrieval artifacts will be prepared before serving user queries. This keeps expensive preprocessing such as document preparation, indexing, and embedding generation outside the online request path.

At runtime, the EC2 backend will load the required lexical retrieval artifacts from Amazon S3 and query Amazon S3 Vectors for semantic retrieval. The retrieved evidence will then be processed by the RAG pipeline and sent to the language model to generate the final answer.

## Project Plan

The project is planned as a team effort that progresses from AWS fundamentals and RAG research to retrieval development, evaluation, and full application deployment. Different components can be developed in parallel when appropriate, but the overall sequence is designed so that the retrieval pipeline is validated before it is integrated into the final AWS application.

### Development Phases

**Phase 1 – AWS Foundation and Project Definition**

The team will first build a shared understanding of the AWS services required for the project and define the CloudHop RAG problem, objectives, architecture, and evaluation approach. This phase also includes studying RAG, text embeddings, multi-hop question answering, and the structure of HotpotQA.

**Phase 2 – Dataset Preparation and Retrieval Baselines**

HotpotQA will be inspected and transformed into a consistent format for retrieval experiments. Initial lexical and dense retrieval methods will be developed to establish a baseline and identify the main retrieval difficulties of multi-hop questions.

**Phase 3 – Advanced Multi-Hop Retrieval**

The retrieval pipeline will be extended with BM25, BGE-M3 embeddings, hybrid retrieval, parent-child document representation, query decomposition, adaptive multi-hop retrieval, and evidence reranking. The goal of this phase is to improve the system's ability to recover complementary evidence from multiple documents.

**Phase 4 – Evaluation and Artifact Preparation**

The team will evaluate retrieval quality using HotpotQA supporting evidence and measure answer quality using Exact Match and F1. Retrieval artifacts such as the processed corpus, BM25 index, document mappings, embeddings, and manifests will also be prepared in a reusable format for deployment.

**Phase 5 – AWS Backend and Retrieval Deployment**

The validated retrieval artifacts will be moved to AWS. Amazon S3 will store the corpus and retrieval artifacts, while Amazon S3 Vectors will provide dense vector search. The RAG backend will be deployed on Amazon EC2 and configured with the permissions required to access the storage and vector-search services.

**Phase 6 – API and Frontend Integration**

Amazon API Gateway will be used to expose the backend through an HTTPS API. A web frontend will be deployed through AWS Amplify and connected to the API so that users can submit questions and view generated answers together with supporting sources.

**Phase 7 – System Validation and Finalization**

The complete application will be tested from the frontend through the API, backend, retrieval services, and language model. The team will validate functionality, retrieval behavior, answer quality, and response time before finalizing the deployment workshop and project documentation.

### Project Timeline

| Week | Planned Team Activities |
| --- | --- |
| **Week 1** | **AWS foundation and project orientation.** Review AWS fundamentals, AWS Console and CLI, EC2, and the internship requirements. Discuss possible AI project directions and prepare the development environment. |
| **Week 2** | **AWS storage, security, and networking.** Study and practice with Amazon S3, IAM, VPC, security groups, and service permissions. Establish the AWS knowledge required for the later application architecture. |
| **Week 3** | **Project definition and RAG research.** Study embeddings, semantic search, RAG, and multi-hop question answering. Select HotpotQA as the benchmark and define the initial CloudHop RAG architecture, objectives, and evaluation strategy. |
| **Week 4** | **Dataset preparation and retrieval baselines.** Prepare HotpotQA data, align questions with supporting evidence, and develop initial lexical and dense retrieval methods to establish a baseline. |
| **Week 5** | **Advanced retrieval development.** Develop BM25 and BGE-M3 retrieval, hybrid search, parent-child document representation, query decomposition, adaptive multi-hop retrieval, and evidence reranking. |
| **Week 6** | **Pipeline engineering and evaluation preparation.** Organize reusable project modules and configurations, validate dataset alignment, build versioned retrieval artifacts, and prepare the benchmark and evaluation workflow. |
| **Week 7** | **Evaluation and AWS deployment preparation.** Evaluate retrieval and answer quality, analyze latency, finalize the production architecture, upload retrieval artifacts to Amazon S3, prepare Amazon S3 Vectors, and configure the Amazon EC2 backend environment. |
| **Week 8** | **Full AWS integration and project finalization.** Complete the EC2 FastAPI backend deployment, connect Amazon S3 and S3 Vectors, configure IAM and Systems Manager access, expose the backend through Amazon API Gateway, deploy the frontend with AWS Amplify, validate the complete end-to-end application, consolidate evaluation results, and finalize the workshop and technical documentation. |

## Budget Estimation

AWS CloudHop RAG is planned as a small internship and demonstration workload with relatively low storage and request volume. Most managed services used by the application are usage-based, while the backend compute instance is expected to account for the largest part of the running cost.

| Resource | Expected Usage | Cost Consideration |
| --- | --- | --- |
| **Amazon EC2** | One backend instance during development and demonstration | Main continuous compute cost |
| **Amazon S3** | Store corpus and retrieval artifacts | Low storage cost at project scale |
| **Amazon S3 Vectors** | Store and query dense vectors | Depends on stored vectors and query usage |
| **Amazon API Gateway** | Low-volume API requests | Request-based |
| **AWS Amplify** | Host a small web frontend | Build, storage, and transfer usage |
| **AWS IAM** | Control AWS resource permissions | No direct service charge |
| **AWS Systems Manager** | EC2 administration | Minimal or no direct cost for the planned usage |
| **Groq API** | LLM inference | Depends on model and token usage |

The project will keep the deployment small and avoid keeping unnecessary resources active. Compute resources can be stopped when they are not needed, while persistent retrieval artifacts remain stored separately in S3 and S3 Vectors.

The budget will therefore be managed primarily by controlling EC2 runtime, limiting unnecessary requests, and cleaning up resources after the workshop and evaluation are complete.

## Risks and Mitigation

| Risk | Possible Impact | Planned Mitigation |
| --- | --- | --- |
| Relevant supporting evidence is not retrieved | The LLM receives incomplete context and may produce an incorrect answer | Combine lexical and semantic retrieval and evaluate supporting-evidence coverage |
| Multi-hop retrieval increases response time | Queries may take too long to complete | Limit retrieval depth and candidate size where necessary |
| LLM API rate limits or temporary failures | Evaluation or generation may be interrupted | Use controlled request rates, retries, and resumable evaluation |
| Dataset or retrieval artifacts are inconsistent | Evaluation results may not accurately reflect retrieval quality | Validate dataset alignment and artifact integrity before benchmarking |
| AWS permissions or configuration are incorrect | Application components may fail to communicate | Use IAM roles, least-privilege permissions, and staged functional testing |
| Cloud resources consume more than expected | Project cost may increase | Keep the deployment small, stop unused compute, and clean up resources after use |

## Expected Outcomes

By the end of the project, my team expects to have a working multi-hop RAG application that can retrieve evidence from HotpotQA, combine information from multiple documents when necessary, and generate answers grounded in the retrieved context.

The main expected outputs are:

- a reusable HotpotQA data preparation workflow;
- lexical and semantic retrieval components;
- a hybrid and multi-hop retrieval pipeline;
- a reproducible evaluation workflow;
- retrieval artifacts that can be stored and reused independently of the application runtime;
- an AWS-hosted backend and vector-search environment;
- a web interface for submitting questions and viewing answers with supporting sources;
- quantitative evaluation of retrieval and answer quality;
- a complete AWS deployment workshop and technical documentation.

Beyond the final application itself, the project is also intended to give my team practical experience connecting retrieval and language-model experimentation with cloud infrastructure. HotpotQA provides a controlled benchmark for this internship, while the overall RAG design can later be adapted to other document collections that require evidence-based question answering.
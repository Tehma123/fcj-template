---
title: "System Architecture"
date: 2026-07-31
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

CloudHop RAG is deployed as a web-based RAG application in the **Asia Pacific (Singapore) Region (`ap-southeast-1`)**. The architecture separates the frontend, API layer, RAG backend, and retrieval storage so that each part of the system has a clear responsibility while remaining connected through a single request flow.

A user interacts with the frontend hosted on **AWS Amplify**. Requests are sent through **Amazon API Gateway** to the FastAPI backend running on **Amazon EC2**. The backend performs lexical retrieval using BM25 artifacts stored in **Amazon S3** and dense retrieval through **Amazon S3 Vectors**, then processes the retrieved evidence before sending the final context to the Groq LLM API.

This chapter explains **how the system is put together and why each AWS service was chosen**, before we start building it in the following chapters. Read this first - every later step (5.4 to 5.9) implements one box of the diagram below.

The workload is a **multi-hop RAG (Retrieval-Augmented Generation) question-answering service**. A user asks a natural-language question that cannot be answered from a single document (for example: *"Were Scott Derrickson and Ed Wood of the same nationality?"*). The system must find several documents, combine the evidence across them, and return a short factual answer together with the sources it used.

---

## 1. The core design decision: split offline from online

The single most important architectural choice in this project is that **expensive work never happens during a request**.

| | Offline pipeline | Online pipeline |
| --- | --- | --- |
| Runs | Once per corpus / index version | On every user request |
| Does | Chunk documents, run the embedding model, build the BM25 index, upload artifacts | Load pre-built artifacts, retrieve, call the LLM, answer |
| Where | A notebook / script on a workstation | FastAPI process on EC2 |
| Cost profile | Heavy, but paid once | Light, must stay under a few seconds |

The online service **never chunks, never embeds a document, and never rebuilds an index**. It only *loads* artifacts that already exist in Amazon S3.

{{% notice tip %}}
Why this matters: rebuilding a vector index costs minutes of CPU. If that work sat inside the request path, every query would time out. Separating the two pipelines is what makes the API able to answer within the API Gateway timeout window, and it is also what lets us serve a brand-new index **without changing a single line of code** (see section 8).
{{% /notice %}}

---

## 2. Overall architecture

<!-- IMAGE 1 - ARCHITECTURE DIAGRAM (draw.io / Excalidraw, NOT a screenshot).
     Must show, left to right:
       [Browser] --HTTPS-> [AWS Amplify Hosting: React/Vite]
                 --HTTPS-> [Amazon API Gateway (HTTP API)]
                 --HTTP :8000-> [Amazon EC2 (Ubuntu) - FastAPI under systemd]
     From the EC2 box, draw arrows to:
       [Amazon S3]  (processed docs, BM25 index, manifest)
       [Amazon S3 Vectors]  (dense embeddings, QueryVectors)
       [Groq API]  (external - draw it OUTSIDE the AWS boundary box)
     Also draw a separate dashed box labelled "OFFLINE (run once)":
       [corpus.jsonl] -> [chunk + embed + BM25] -> [Amazon S3] + [Amazon S3 Vectors]
     Mark the AWS region ap-southeast-1 around the AWS resources.
     Put an IAM role icon on the EC2 box labelled rag-ec2-runtime-role.
     Configuration is a file on the instance, not a service - do not draw
     Parameter Store or Secrets Manager. -->

![Architecture diagram](/images/5-Workshop/5.3-Architecture/architecture-overview.png)

The system has four tiers:

1. **Presentation** - a React/Vite single-page app hosted on **AWS Amplify Hosting**, served over HTTPS from a CDN.
2. **Edge / API** - **Amazon API Gateway** (HTTP API) terminates TLS, owns CORS, and proxies to the backend. Routes: `GET /health`, `POST /warmup`, `POST /query`.
3. **Application** - a **FastAPI** service on **Amazon EC2** (Ubuntu), managed by `systemd` as `aws-rag-api`, listening on port `8000`. This is the only compute that runs per request.
4. **Data** - **Amazon S3** (text artifacts + BM25 index) and **Amazon S3 Vectors** (dense embeddings). Runtime configuration is not a service here: it lives in a single `.env.prod` file on the instance (chapter 5.7 section 7).

The **Groq API** is an external SaaS LLM endpoint. It is deliberately drawn *outside* the AWS boundary in the diagram, because it is the only dependency that leaves the account.

---

## 3. Offline ingestion pipeline

<!-- IMAGE 2 - DIAGRAM (draw.io). A simple left-to-right flow:
     corpus.jsonl (HotpotQA subset)
       -> chunk: parent docs (whole article) + child docs (sub-chunks)
       -> embed child docs with BAAI/bge-m3
       -> build BM25 index (bm25s)
       -> write index_manifest.json
       -> upload to Amazon S3  AND  PutVectors to Amazon S3 Vectors
     Annotate the "parent/child" step, it is the part readers find confusing. -->

![Offline ingestion pipeline](/images/5-Workshop/5.3-Architecture/offline-pipeline.png)

Steps, in order:

1. **Build the corpus** - extract a subset of HotpotQA into `corpus.jsonl`.
2. **Chunk into two levels** - each article becomes one **parent document** (the full article, used as final context) and several **child documents** (small sub-chunks, used for matching). This is the *small-to-big* pattern: match on small precise chunks, but hand the LLM the larger surrounding article.
3. **Embed the child documents** with the `BAAI/bge-m3` sentence-embedding model.
4. **Build a BM25 index** (the `bm25s` library) over the same child documents, giving a keyword/lexical retriever alongside the vector retriever.
5. **Write `index_manifest.json`** - records which embedding model, chunk sizes and paths produced this index. The online loader reads the manifest so it can never accidentally pair an index with the wrong embedding model.
6. **Upload** the processed documents, BM25 index and manifest to Amazon S3, and push the embeddings into Amazon S3 Vectors via `PutVectors`.

Artifacts produced (all under one versioned id, e.g. `hotpotqa-val500-v002`):

| Artifact | Stored in | Used online for |
| --- | --- | --- |
| `parent_docs.jsonl` | Amazon S3 | Final context handed to the LLM |
| `child_docs.jsonl` | Amazon S3 | Retrieval units |
| BM25 index files | Amazon S3 | Lexical retrieval |
| `index_manifest.json` | Amazon S3 | Compatibility check at startup |
| Dense embeddings | Amazon S3 Vectors | Semantic retrieval |

---

## 4. Online query pipeline

<!-- IMAGE 3 - DIAGRAM (draw.io). Vertical or left-to-right flow of ONE request:
     question
       -> [optional] Groq: decompose into sub-questions
       -> per sub-question: BM25 retrieval  +  S3 Vectors retrieval
       -> merge with Reciprocal Rank Fusion (RRF)
       -> small-to-big: child chunk -> parent article
       -> adaptive hop planning (Groq: "sufficient?" / "next query")  [loop arrow back to retrieval]
       -> [optional] cross-encoder rerank
       -> Groq: generate short-form answer
       -> response {answer, sources, timings, token_usage}
     The loop-back arrow for the hop planner is the key visual - make it obvious. -->

![Online query flow](/images/5-Workshop/5.3-Architecture/online-query-flow.png)

What happens when a request arrives at `POST /query`:

1. **Query decomposition** *(optional)* - Groq splits a multi-hop question into sub-questions.
2. **Hybrid retrieval** - for each sub-question, run **BM25** (keyword) and **S3 Vectors** (semantic) in parallel and merge the two ranked lists with **Reciprocal Rank Fusion**. Neither retriever alone is enough: BM25 is strong on exact names and rare terms, vectors are strong on paraphrase.
3. **Small-to-big expansion** - the winning child chunks are mapped back to their parent articles.
4. **Adaptive hop planning** - Groq inspects the evidence gathered so far and either declares it *sufficient* or proposes the next search query. This is what makes the system genuinely multi-hop rather than a single lookup.
5. **Cross-encoder reranking** *(optional, disabled in the deployed demo for latency)* - rescores candidate parents against the question.
6. **Answer generation** - Groq produces a **short-form** answer (a word, a name, a number) from the selected context.
7. **Response** - JSON containing `answer`, `sources`, `timings` and `token_usage`, so the frontend can show the evidence behind the answer.

{{% notice note %}}
Retrieval weighting is **hop-dependent**, not fixed. The first hop is usually a named-entity lookup, so BM25 is weighted higher; later hops are usually relational rewrites, so the vector retriever is weighted higher. This kind of tuning lives entirely in configuration - see chapter 5.12.
{{% /notice %}}

---

## 5. AWS services used, and why

The project uses **eight AWS services**. Each row states what it does here and why it was picked over the obvious alternative.

| Service | Role in this system | Why this service |
| --- | --- | --- |
| **Amazon S3** | Stores processed documents, the BM25 index and the manifest | Durable, effectively free at this data size, and decoupled from instance lifecycle. Storing artifacts on the EC2 volume instead would tie the data to one server and lose it on instance replacement. Version-prefixed keys let several index generations coexist. |
| **Amazon S3 Vectors** | Dense vector storage and similarity search (`PutVectors` / `QueryVectors`) | A managed vector store with no server to run and no idle cost. OpenSearch Serverless bills a minimum capacity even when idle; a self-hosted FAISS/Chroma index on EC2 would be memory-bound and would die with the instance. |
| **Amazon EC2** | Runs the FastAPI application | The process loads a PyTorch embedding model and the BM25 index into memory at startup and reuses them across requests. A long-lived process amortises that load. AWS Lambda would pay it on every cold start and is awkward for large ML dependencies; ECS Fargate adds orchestration complexity that a single-container demo does not need. |
| **Amazon API Gateway** (HTTP API) | HTTPS entry point, CORS, routing to EC2 | Amplify serves the frontend over HTTPS, and a browser refuses to let an HTTPS page call a plain-HTTP API (**Mixed Content**). API Gateway terminates TLS with a managed certificate and proxies to EC2 over HTTP. An ALB would also work but costs more per hour and needs its own certificate/domain setup. |
| **AWS Amplify Hosting** | Hosts and builds the React frontend | Git-push deployment, managed TLS and CDN out of the box. S3 + CloudFront gives the same result but with several more manual steps. |
| **AWS Systems Manager** | Session Manager for administrative access to the instance | Removes the need to open port `22` or manage a key pair, and records every session. This is the only Systems Manager capability the deployment uses today - see the note below. |
| **AWS IAM** | `rag-ec2-runtime-role` instance profile | The application uses **no access keys at all** - credentials are obtained from the instance role via IMDSv2. See section 7. |

{{% notice info %}}
**Groq** (external) is used for the LLM calls - decomposition, hop planning and answer generation - rather than Amazon Bedrock, because its hosted open-weight models return short factual answers with very low latency, which is what keeps the whole request inside the API Gateway timeout. It is the one non-AWS dependency, and it is isolated behind a single module so it can be swapped.
{{% /notice %}}

---

## 6. Request path end to end

```text
Browser
  |  HTTPS
  v
AWS Amplify Hosting  (React/Vite SPA)
  |  HTTPS  POST /query
  v
Amazon API Gateway (HTTP API)          <- TLS termination + CORS
  |  HTTP  :8000
  v
Amazon EC2 - FastAPI (systemd: aws-rag-api)
  |--> Amazon S3            parent/child docs, BM25 index, manifest
  |--> Amazon S3 Vectors    QueryVectors (semantic retrieval)
  `--> Groq API             decompose - hop plan - generate
```

The frontend must be configured with the **API Gateway HTTPS URL**, never the raw EC2 address. This is the single most common mistake when reproducing this workshop, and chapter 5.8 covers it in detail.

---

## 7. Security design

Security here is about removing long-lived credentials and closing default doors, not about adding products.

- **No hard-coded credentials anywhere.** The application holds no AWS access key or secret key. The EC2 instance carries the instance profile **`rag-ec2-runtime-role`**, and the AWS SDK obtains temporary credentials through IMDSv2.
- **No AWS access keys anywhere.** Every AWS call is authorised by the instance role through IMDSv2. The one non-AWS credential, the Groq API key, currently sits in `.env.prod` on the instance - an open limitation described in chapter 5.13, not a solved problem.
- **Least privilege on the instance role.** The role is scoped to what the service actually does: read the artifact objects in the project S3 bucket, query the S3 Vectors index, read the `/prod/aws-rag/*` parameters, read that one secret, and write CloudWatch logs - plus `AmazonSSMManagedInstanceCore` for Session Manager.
- **No SSH port open.** Administrative access uses **AWS Systems Manager Session Manager** instead of inbound port `22`. This removes the "allow my current IP" rule that breaks every time you change network, and it leaves an auditable session record.
- **Stable, controlled backend address.** The instance uses an **Elastic IP** so the API Gateway integration cannot silently point at the wrong host after a stop/start.
- **Private buckets.** Both the artifact bucket and the vector bucket stay private; nothing is served publicly from S3. All access is through the instance role.
- **CORS is an allowlist, not a wildcard.** `/prod/aws-rag/cors-allow-origins` names the Amplify origin and the local dev origins explicitly.

<!-- IMAGE 4 - SCREENSHOT.
     Console: IAM -> Roles -> rag-ec2-runtime-role -> Permissions tab.
     Capture the attached policies (the inline S3 + S3 Vectors policy, and
     AmazonSSMManagedInstanceCore) so the read-only claim above is evidenced.
     Blur / crop out the 12-digit AWS account id before publishing. -->

![IAM role for the EC2 backend](/images/5-Workshop/5.3-Architecture/iam-role-permissions.png)

{{% notice warning %}}
Known limitation, stated honestly: the backend still listens on port `8000` on a public IP, restricted by security group rules. For a production deployment the instance should sit in a private subnet with only API Gateway (via a VPC Link) able to reach it. This is discussed again in chapter 5.13.
{{% /notice %}}

---

## 8. Configuration-driven operation

Every tunable is read from an environment variable at process startup, supplied by a single `.env.prod` file on the instance, so **serving a different index needs no code change and no new deployment** - you build new artifacts, edit three lines, and restart the service.

| Variable group | Examples | Controls |
| --- | --- | --- |
| Artifact location | `S3_ARTIFACT_BUCKET`, `S3_PROCESSED_ID`, `S3_VECTOR_BUCKET`, `S3_VECTOR_INDEX`, `RAG_INDEX_ID` | Which corpus/index is served |
| Retrieval behaviour | `BM25_TOP_K`, `VECTOR_TOP_K`, `HOP_CANDIDATE_CAP`, `MAX_ADAPTIVE_HOPS`, `HOP_EVIDENCE_TOP_N` | How wide and how deep the search goes |
| Latency / quality trade-off | `RAG_FAST_MODE`, `RAG_USE_RERANKER`, `RERANK_TOP_N`, `RAG_DEVICE` | Speed versus answer quality |
| Operational | `CORS_ALLOW_ORIGINS`, `RAG_WARMUP_QUESTION`, `AWS_REGION` | Access and warm-up |

This is what turns the deployment into a *platform* rather than a one-off: the same running EC2 instance can serve a completely different knowledge base after an edit and a `systemctl restart`.

{{% notice note %}}
The trade-off of file-based configuration is that changing anything requires a session on the instance. The repository already contains the code to read the same settings from **AWS Systems Manager Parameter Store**, with the Groq key coming from **AWS Secrets Manager** - which would make a change a single API call and take the secret off the disk. That migration is designed but not deployed; chapter 5.7 section 7 describes it and chapter 5.13 lists it as an open limitation.
{{% /notice %}}

---

## 9. Latency budget

API Gateway enforces a hard integration timeout of roughly **30 seconds**. The full-quality pipeline - decomposition, multiple hops, cross-encoder reranking on CPU - can exceed that. Two mechanisms keep the deployed demo inside the budget:

- **`RAG_FAST_MODE`** skips query decomposition and reduces top-k and hop counts, trading some retrieval quality for speed. The reranker is switched **off** in production for the same reason.
- **A `POST /warmup` route** triggers a full dummy query so the embedding model, BM25 index and vector client are all loaded and cached before the first real user arrives. Without it, the first request after a restart pays the entire model-loading cost and times out.

The measured latency and accuracy numbers for both modes are reported in chapter 5.11.

---

## 10. Scalability and operations

Current state and the honest path forward:

| Concern | Today | Next step |
| --- | --- | --- |
| Compute | One EC2 instance, single FastAPI process | Auto Scaling group behind an ALB, or containerise and move to ECS |
| Storage | S3 + S3 Vectors, already managed and effectively unlimited | No change needed - this tier already scales |
| Availability | Single instance, single AZ | Multi-AZ once compute is behind a load balancer |
| Monitoring | CloudWatch logs and metrics, `systemd` restart on failure | CloudWatch alarms on 5xx rate and latency, covered in chapter 5.12 |
| Config changes | Edit `.env.prod` + service restart | Move to Parameter Store so no shell session is needed |

The retrieval and storage tiers are already serverless and elastic; **the only real bottleneck is the single EC2 instance**, and it is a bottleneck that was chosen deliberately to keep the demo cheap.

---

## 11. Deployed resources

| Item | Value |
| --- | --- |
| Region | `ap-southeast-1` |
| Frontend | AWS Amplify Hosting (HTTPS) |
| API | Amazon API Gateway HTTP API → EC2 Elastic IP `:8000` |
| Backend service | `aws-rag-api` (systemd) on Ubuntu EC2 |
| Artifact bucket | `aws-rag-bucket-vanh1234` |
| Vector bucket | `rag-vectors-vanh1234` |
| Vector index | `hotpotqa-val500-bge-m3-v002` |
| Processed id | `hotpotqa-val500-v002` |
| Embedding model | `BAAI/bge-m3` |
| Config prefix | `/prod/aws-rag/*` |
| Instance role | `rag-ec2-runtime-role` |

<!-- IMAGE 5 - SCREENSHOT (end-to-end proof, put it last).
     The deployed frontend in a browser, after asking:
       "Were Scott Derrickson and Ed Wood of the same nationality?"
     Capture the whole window so that ALL of these are visible:
       - the browser address bar showing the https:// Amplify URL (proves HTTPS end to end)
       - the padlock icon
       - the returned answer
       - the source/evidence list under the answer
     This single screenshot is the evidence for the end-to-end deployment criterion in the grading rubric. -->

![Deployed system answering a multi-hop question](/images/5-Workshop/5.3-Architecture/end-to-end-demo.png)

{{% notice note %}}
Bucket names are globally unique - when you reproduce this workshop you must choose your own. Every chapter from 5.4 onward refers back to the names you pick here.
{{% /notice %}}

---

With the architecture settled, chapter 5.4 starts the build by producing the offline artifacts.

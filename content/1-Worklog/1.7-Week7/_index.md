---
title: "Week 7 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.7. </b> "
---


### Week 7 Objectives:

* Move the RAG pipeline's components onto AWS services instead of running everything locally.
* Understand how to host the corpus, embeddings, and orchestration logic on AWS.
* Add monitoring for the pipeline using CloudWatch.

### Tasks to be carried out this week:
| Day | Task                                                                                                                             | Start Date | Completion Date | Reference Material                        |
| --- | ----------------------------------------------------------------------------------------------------------------------------------- | ---------- | --------------- | ----------------------------------------- |
| 2   | - Plan which AWS services will host each pipeline component (corpus storage, vector index, orchestration, LLM access)               | 07/20/2026 | 07/20/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 3   | - **Practice:** upload the HotpotQA corpus and precomputed embeddings to S3                                                          | 07/21/2026 | 07/21/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 4   | - **Practice:** wrap the retrieval + multi-hop reasoning logic in a Lambda function / SageMaker-hosted service                       | 07/22/2026 | 07/22/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 5   | - **Practice:** expose the pipeline through an API endpoint and test it with sample HotpotQA questions                              | 07/23/2026 | 07/23/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 6   | - Add CloudWatch logging and alarms around the pipeline (errors, latency) <br> - Review progress with mentor                        | 07/24/2026 | 07/24/2026      | <https://cloudjourney.awsstudygroup.com/> |


### Week 7 Achievements:

* Migrated the HotpotQA corpus and embeddings from local storage to S3.

* Wrapped the advanced RAG retrieval/reasoning logic in an AWS-hosted service.

* Exposed the pipeline through an API endpoint and validated it with sample questions.

* Added CloudWatch logging and alarms to monitor pipeline errors and latency.

* Confirmed the AWS-hosted pipeline produces the same results as the local prototype from Week 6.
* ...

---
title: "Security and Cost Considerations"
date: 2026-07-31
weight: 13
chapter: false
pre: " <b> 5.13. </b> "
---

CloudHop RAG was deployed as a practical project environment, but security and cost still influenced several implementation choices. AWS permissions are provided to the EC2 backend through an IAM role rather than embedding AWS credentials in the application, and Systems Manager Session Manager is used to manage the instance.

API Gateway provides the HTTPS-facing API used by the Amplify frontend, while CORS limits browser access to the configured frontend origin. At the same time, the main recurring cost comes from keeping the EC2 instance running, together with storage, vector search, API requests, and frontend hosting.

---

## Security

### 1. No long-lived AWS credentials

The application holds no AWS access key. The EC2 instance carries the instance profile `rag-ec2-runtime-role`, and the AWS SDK obtains temporary credentials through IMDSv2. This can be verified on the instance itself:

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" -s)

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

```text
rag-ec2-runtime-role
```

Every AWS call the service makes - reading artifacts from S3, querying S3 Vectors - is authorised this way. Nothing in the repository, the AMI or the environment file contains an AWS key.

{{% notice warning %}}
**The Groq API key is the exception, and it is a real one.** It is stored in plain text in `/home/ubuntu/aws-rag-project/backend/.env.prod` on the instance. The file is git-ignored, owned by the `ubuntu` user, and never leaves the machine - but it is still a plaintext credential on a disk.

The mitigation is designed and coded but not deployed: `app/aws_runtime_config.py` can load the key from **AWS Secrets Manager** through the same instance role, removing it from the disk entirely. Chapter 5.7 section 7 describes the migration, and section 6 below lists it as an open limitation rather than a solved problem.
{{% /notice %}}

### 2. A read-only serving role

The inline policy on `rag-ec2-runtime-role` grants two things, and both are read-only:

| Granted | Not granted |
| --- | --- |
| `s3:GetObject`, `s3:ListBucket` on the artifact bucket | Any `s3:PutObject` or `s3:DeleteObject` |
| `s3vectors:QueryVectors`, `GetIndex`, `GetVectors`, `ListVectors`, `GetVectorBucket` | `PutVectors`, `DeleteVectors`, `CreateIndex`, `DeleteIndex` |

Plus the managed policy `AmazonSSMManagedInstanceCore`, which exists only so Session Manager works.

The running service is therefore **read-only against all of its own data**. A bug in the application cannot corrupt an index or overwrite an artifact, because IAM refuses the call - the guarantee does not depend on the code being correct. The screenshot evidencing this is in chapter 5.7 section 2.

{{% notice note %}}
Two honest qualifications. The S3 statement covers the **whole bucket** rather than only the `rag/*` prefix, and the S3 Vectors statement uses `"Resource": "*"` rather than the index ARN. Both are wider than necessary. Neither weakens the read-only property, which is the security-relevant part, but a stricter policy would scope each statement to the exact prefix and ARN. This is listed in section 6.
{{% /notice %}}

### 3. Administrative access without an open SSH port

Port `22` is not open. Administration goes through **Session Manager**, which needs no inbound port, no key pair, and records each session. This also removed a recurring operational problem: a security-group rule pinned to "My IP" breaks every time the laptop changes network.

### 4. HTTPS end to end, and a CORS allowlist

Amplify serves the frontend over HTTPS with a managed certificate, and API Gateway terminates HTTPS in front of the backend. The browser never makes a plain-HTTP request - this is what the Mixed Content work in chapters 5.8 and 5.9 achieved.

Browser access is restricted by an **explicit origin allowlist on two layers**: API Gateway CORS, and the FastAPI `CORSMiddleware` fed by the `cors-allow-origins` parameter. Neither uses a wildcard. Given that the API has no authentication, the difference between naming one origin and allowing `*` is meaningful rather than cosmetic.

### 5. Data at rest

Both storage resources are private:

- Block Public Access is fully enabled on the artifact bucket; no object is publicly readable.
- Default encryption uses SSE-S3 (`AES256`) on the artifact bucket, and S3 Vectors applies the same by default.
- Versioning is enabled, so an accidental overwrite of an index that production is serving is recoverable.
- No bucket policy grants access to anyone; all access flows through IAM identity policies.

### 6. Known limitations, stated plainly

These are real and are not presented as solved:

| Limitation | Why it exists | What production would do |
| --- | --- | --- |
| **Port `8000` reachable from the internet** | API Gateway is managed and calls from AWS-owned addresses that cannot be allowlisted by IP | EC2 in a private subnet behind an API Gateway **VPC Link** |
| **No end-user authentication** | The demo is meant to be openly testable | JWT authorizer, Lambda authorizer, or an API key on the HTTP API |
| **No rate limiting** | Not configured | Throttling on the API Gateway stage; every request also spends Groq tokens |
| **Single instance, single AZ** | Chosen to keep the demo cheap | Auto Scaling group behind an ALB across two AZs |
| **One external dependency leaves the account** | Groq performs LLM generation | Amazon Bedrock, or accept and document the data-flow boundary |
| **The Groq API key sits in plain text on the instance** | The Secrets Manager migration is coded but not deployed | Set `GROQ_SECRET_NAME` in `.env.prod` and grant `secretsmanager:GetSecretValue` to the role |
| **IAM statements wider than necessary** | S3 covers the whole bucket; S3 Vectors uses `Resource: "*"` | Scope S3 to the `rag/*` prefix and S3 Vectors to the index ARN |

Stating these is not a weakness of the report - an architecture review that lists no residual risk has usually not looked.

---

## Cost

### 7. Where the money actually goes

| Source | Billing shape | Notes |
| --- | --- | --- |
| **Amazon EC2** | **Continuous while running** | The dominant cost. Billed per hour whether or not anyone asks a question |
| **Elastic IP / public IPv4** | **Continuous** | Public IPv4 addresses are billed hourly in all cases, including while attached to a stopped instance |
| **EBS volume** | **Continuous** | Keeps being billed while the instance is stopped |
| Amazon S3 | Per GB stored + requests | A few hundred megabytes of artifacts; negligible at this scale |
| Amazon S3 Vectors | Per vector stored + queries | **No idle cost** - nothing is billed when no query runs |
| Amazon API Gateway | Per request | HTTP API, roughly a third the price of a REST API |
| AWS Amplify Hosting | Build minutes + storage + transfer | Small for a single-page app |
| AWS Secrets Manager | Per secret per month | One secret exists but is **not yet in use** - see section 1 |
| **Groq API** *(external)* | Per token | Not an AWS charge. Each question spends tokens |

The pattern worth noticing: **only the compute tier bills continuously.** Storage and retrieval are effectively pay-per-use, so an idle system costs close to nothing once the instance is stopped.

### 8. Design choices that were also cost choices

Several decisions made for other reasons happen to be the cheap ones, and that is worth making explicit:

- **Offline build runs on Google Colab, not on AWS.** The single expensive compute step - embedding the corpus with `BAAI/bge-m3` - never touches the AWS bill. No GPU instance and no SageMaker job was provisioned (chapter 5.4).
- **S3 Vectors instead of a provisioned vector database.** OpenSearch Serverless bills a minimum capacity even when idle; a self-managed vector store would need its own instance. S3 Vectors bills for storage and queries only.
- **API Gateway instead of an Application Load Balancer.** An ALB bills per hour regardless of traffic; the HTTP API bills per request.
- **The reranker is disabled in production.** Cross-encoder scoring on CPU was the reason to consider a larger instance; turning it off (chapter 5.7 section 9) kept the instance small.
- **Fast mode removes one Groq call per question.** Skipping query decomposition cuts both latency and token spend.
- **`/warmup` does not call Groq generation.** Warming the pipeline loads models and runs one retrieval, deliberately stopping before the billable generation step.

### 9. Practical controls

- **Stop the EC2 instance when it is not being demonstrated.** This is the single highest-impact action. Note that the EBS volume and the Elastic IP keep billing while stopped - stopping reduces cost substantially but does not reduce it to zero.
- **Do not accumulate index versions.** Every corpus rebuild creates a new versioned prefix in S3 and a new vector index. Keeping one previous version for rollback is prudent; keeping six is waste. Delete the ones no longer referenced by any SSM parameter.
- **Add a lifecycle rule for non-current object versions.** Versioning is enabled on the artifact bucket, so every overwritten object is retained indefinitely unless a rule expires it. Thirty days is a reasonable default once you start iterating on indexes.
- **Keep everything in one Region.** Same-Region traffic between EC2, S3 and S3 Vectors is not billed as data transfer; splitting Regions would add a charge to every artifact download.
- **Set an AWS Budget with an email alert.** A budget does not prevent spending, but it converts a surprise at the end of the month into a notification within a day.
- **Delete the deployment when the project ends.** Chapter 5.14 covers the order.

{{% notice tip %}}
The most common source of unexpected charges in a project like this is not the service anyone worries about. It is an **Elastic IP left allocated after the instance was terminated**, or an EBS volume belonging to a forgotten instance. Both keep billing quietly, and neither appears where you would look for "the RAG system". Chapter 5.14 checks for exactly these.
{{% /notice %}}

<!-- IMAGE 1 - SCREENSHOT.
     AWS Billing -> Budgets, showing a monthly budget with an alert threshold configured.
     This is direct evidence for the cost-control criterion in the grading rubric.
     If you have not created one yet, create it now - it takes two minutes and it is
     genuinely useful for the rest of the internship.
     Blur the amount if you prefer not to publish it. -->

![A monthly budget with an alert](/images/5-Workshop/5.13-Security-Cost/aws-budget.png)

<!-- IMAGE 2 - SCREENSHOT (optional but strong).
     Cost Explorer, grouped by Service, filtered to this project's Region, over the
     project period.
     The point is the SHAPE of the breakdown - EC2 dominating, everything else small -
     which is exactly what section 7 claims. Blur the dollar amounts if you prefer;
     the relative proportions are what matter. -->

![Cost breakdown by service](/images/5-Workshop/5.13-Security-Cost/cost-explorer.png)

---

## Summary

Security in this deployment rests on three things that are enforced rather than promised: no static credentials anywhere, a serving role that cannot write to its own data, and no inbound administrative port. What remains open - a publicly reachable backend port and an unauthenticated API - is documented above rather than omitted.

On cost, the architecture is shaped so that only the EC2 instance bills continuously. Everything else is pay-per-use or free at this scale, which means stopping one resource takes the running cost close to zero, and chapter 5.14 takes it to zero.

---
title: "Amazon EC2 Backend Deployment"
date: 2026-07-31
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

The FastAPI backend is the main execution layer of CloudHop RAG. It receives questions from the API layer, loads the prepared retrieval artifacts, performs BM25 and dense retrieval, coordinates the multi-hop pipeline, constructs the final context, and sends the generation request to Groq.

The backend is deployed on **Amazon EC2**, giving the application a persistent environment in which the Python RAG pipeline and its model dependencies can remain loaded between requests.

The data is in place. This chapter stands up the only compute in the system: a FastAPI service on EC2 that loads those artifacts at startup and answers questions.

{{% notice info %}}
**What you will have at the end of this chapter:** an Ubuntu EC2 instance with a stable Elastic IP, an instance role that grants read-only access to the two data services, all runtime configuration in a single `.env.prod` file, and a `systemd` service that survives reboots and warms itself up. `POST /query` will answer over plain HTTP; chapter 5.8 puts HTTPS in front of it.
{{% /notice %}}

This is the longest chapter in the workshop, and the order matters - the IAM role must exist before the instance launches, and the configuration must exist before the service starts.

---

## 1. Sizing and the shape of the workload

The service is a **single long-lived Python process**. At startup it downloads artifacts from S3, loads `parent_docs.jsonl` and `child_docs.jsonl` into memory, loads the BM25 index, and lazily loads the `BAAI/bge-m3` embedding model on first use. After that, each request is retrieval plus a few HTTP calls to Groq.

That shape drives every decision here:

- **RAM is the binding constraint, not CPU.** The embedding model dominates memory use. An instance that is too small does not run slowly - it gets its Python process killed by the OOM killer, the same failure seen in chapter 5.4.
- **The process must stay alive between requests.** Restarting means re-downloading artifacts and re-loading the model. This is why it runs under `systemd` and not as a script in a terminal.
- **CPU inference is acceptable here** because the embedding model only encodes a short query per request, not a corpus. `rag-device` is set to `cpu`; no GPU instance is needed at serving time.

---

## 2. Create the instance role first

The role must exist before you launch the instance, so you can attach it during launch rather than fixing it afterwards.

Create a role named **`rag-ec2-runtime-role`** with trusted entity **EC2**, then attach:

**a) The managed policy `AmazonSSMManagedInstanceCore`** - this is what makes Session Manager work (section 5). Without it the SSM Agent reports credential errors and you are stuck with SSH.

**b) An inline policy for the two data services the application uses:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadRegularS3Artifacts",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::aws-rag-bucket-vanh1234",
        "arn:aws:s3:::aws-rag-bucket-vanh1234/*"
      ]
    },
    {
      "Sid": "QueryS3Vectors",
      "Effect": "Allow",
      "Action": [
        "s3vectors:GetVectorBucket",
        "s3vectors:GetIndex",
        "s3vectors:ListVectors",
        "s3vectors:QueryVectors",
        "s3vectors:GetVectors"
      ],
      "Resource": "*"
    }
  ]
}
```

Both statements are read-only. The service can read artifacts and query vectors; it cannot write an object, put a vector or delete an index. **This is what makes "no hard-coded AWS credentials" true rather than aspirational** - the application never sees an AWS access key, because the SDK fetches temporary credentials from the instance metadata service.

{{% notice note %}}
This policy is shown as it actually is, not as an idealised version. Two things are broader than they need to be: S3 is granted on the whole bucket rather than only the `rag/*` prefix, and S3 Vectors uses `"Resource": "*"` rather than the index ARN. Neither grants write access, so the read-only guarantee holds - but both are listed as hardening opportunities in chapter 5.13.
{{% /notice %}}

## 3. Launch the instance

| Setting | Value |
| --- | --- |
| AMI | Ubuntu Server (LTS) |
| Region | `ap-southeast-1` |
| Instance type | Enough RAM for the embedding model - see section 1 |
| Key pair | Optional; Session Manager replaces SSH (section 5) |
| IAM instance profile | **`rag-ec2-runtime-role`** |
| Security group | See section 4 |
| Storage | Room for the venv, PyTorch, the model cache and the downloaded artifacts |

Then **allocate an Elastic IP and associate it with the instance.**

This step is not optional bookkeeping. A default public IP changes every time the instance stops and starts. API Gateway's integration in chapter 5.8 points at a literal address - when that address changes, the API silently starts proxying to nothing. An Elastic IP fixes the address for the life of the project.

```bash
aws ec2 allocate-address --domain vpc --region ap-southeast-1
aws ec2 associate-address --instance-id <instance-id> --allocation-id <allocation-id> --region ap-southeast-1
```

{{% notice note %}}
Public IPv4 addresses are billed hourly whether or not they are in use, so an Elastic IP is a small but real running cost for as long as the project exists. Chapter 5.14 releases it during cleanup - an Elastic IP left allocated after the instance is terminated keeps charging.
{{% /notice %}}

<!-- IMAGE 2 - SCREENSHOT.
     EC2 -> Instances -> your instance -> Details tab.
     Must show together: instance state "Running", the instance type, the Elastic IP
     in the "Public IPv4 address" field, and the "IAM Role" field showing
     rag-ec2-runtime-role.
     Blur the instance id and the IP if you prefer not to publish them. -->

![The backend instance with its Elastic IP and role](/images/5-Workshop/5.7-EC2-Backend/ec2-instance.png)

---

## 4. Security group

The demo configuration, stated honestly:

```text
Inbound   Custom TCP  8000   0.0.0.0/0
Outbound  All traffic        0.0.0.0/0
```

Port `8000` is open because API Gateway (chapter 5.8) is a fully managed service that calls the backend from AWS-owned addresses, not from a fixed range you can allowlist by IP. Outbound must be open for S3, S3 Vectors, Systems Manager (for Session Manager) and the Groq API.

Port `22` is **not** opened. Section 5 explains what replaces it.

{{% notice warning %}}
This is the weakest point in the deployment and the report says so plainly. Anyone who discovers the address can call the API, and there is no authentication on it. The correct production architecture is an EC2 instance in a **private subnet**, reachable only through an API Gateway **VPC Link**, so port `8000` is never exposed at all. That is documented as a known limitation in chapter 5.13 rather than quietly omitted.
{{% /notice %}}

---

## 5. Administrative access without SSH

Opening port `22` to "My IP" works until you change network - a different Wi-Fi means a new public IP and a broken rule. **AWS Systems Manager Session Manager** removes the problem entirely: no inbound port, no key pair, and an auditable record of every session.

Install and enable the agent on Ubuntu:

```bash
sudo snap install amazon-ssm-agent --classic
sudo systemctl enable snap.amazon-ssm-agent.amazon-ssm-agent.service
sudo systemctl start snap.amazon-ssm-agent.amazon-ssm-agent.service
sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service
```

Confirm the instance really is using the role, via IMDSv2:

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" -s)

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Expected output:

```text
rag-ec2-runtime-role
```

Connect from the Console: **EC2 → Instances → select instance → Connect → Session Manager**. The session opens as `ssm-user`; switch to the project user:

```bash
sudo su - ubuntu
```

{{% notice tip %}}
If the agent logs credential errors, the cause is almost always a missing `AmazonSSMManagedInstanceCore` on the role. Attach it, then restart the agent - the log should then show `Successfully connected with instance profile role credentials`.
{{% /notice %}}

<!-- IMAGE 3 - SCREENSHOT.
     A live Session Manager session in the browser, after running `sudo su - ubuntu`
     and `sudo systemctl status aws-rag-api`.
     Two things must be visible: the Session Manager interface itself (proving no SSH
     was used) and the service showing "active (running)". -->

![Connecting through Session Manager](/images/5-Workshop/5.7-EC2-Backend/session-manager.png)

---

## 6. Install the application

```bash
cd ~
git clone https://github.com/vietanh1802/aws-rag-project.git
cd ~/aws-rag-project/backend

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Only `requirements.txt` is needed. The offline dependency set from chapter 5.4 is **not** installed here - this machine never builds anything, so it does not need the dataset tooling or the build-time stack. Keeping the serving environment minimal is what lets a modest instance hold the model in memory.

The project path used throughout this workshop:

```text
/home/ubuntu/aws-rag-project/backend
```

---

## 7. Configuration through `.env.prod`

All runtime configuration lives in a single file on the instance:

```text
/home/ubuntu/aws-rag-project/backend/.env.prod
```

`systemd` passes it to the process with `EnvironmentFile=` (section 8), and the application reads plain environment variables - `advanced_rag/config.py` resolves every tunable through `os.environ.get()` with a sensible default. There is no configuration path beyond that.

```env
USE_TF=0
USE_FLAX=0
AWS_REGION=ap-southeast-1
GROQ_API_KEY=<your-groq-key>

S3_ARTIFACT_BUCKET=aws-rag-bucket-vanh1234
S3_VECTOR_BUCKET=rag-vectors-vanh1234
RAG_INDEX_ID=hotpotqa-val500-bge-m3-v002
S3_PROCESSED_ID=hotpotqa-val500-v002
S3_VECTOR_INDEX=hotpotqa-val500-bge-m3-v002
RAG_ARTIFACT_LAYOUT=s3vectors
RAG_AUTO_DOWNLOAD_ARTIFACTS=true

RAG_DEVICE=cpu
RAG_USE_RERANKER=false
RAG_FAST_MODE=true
BM25_TOP_K=15
VECTOR_TOP_K=15
RERANK_TOP_N=5
HOP_CANDIDATE_CAP=15
MAX_ADAPTIVE_HOPS=3
HOP_EVIDENCE_TOP_N=3
RAG_WARMUP_QUESTION=Were Scott Derrickson and Ed Wood of the same nationality?
```

What each group controls:

| Group | Variables | Effect |
| --- | --- | --- |
| Runtime | `USE_TF`, `USE_FLAX`, `AWS_REGION` | Keep `transformers` on PyTorch only; AWS region for every SDK client |
| Credentials | `GROQ_API_KEY` | The only secret. AWS access needs no key - the instance role provides it |
| Artifact location | `S3_ARTIFACT_BUCKET`, `S3_PROCESSED_ID`, `S3_VECTOR_BUCKET`, `S3_VECTOR_INDEX`, `RAG_INDEX_ID`, `RAG_ARTIFACT_LAYOUT`, `RAG_AUTO_DOWNLOAD_ARTIFACTS` | Which corpus and index are served, and whether to download them at boot |
| Latency / quality | `RAG_FAST_MODE`, `RAG_USE_RERANKER`, `RAG_DEVICE` | Speed versus answer quality - see section 9 |
| Retrieval budget | `BM25_TOP_K`, `VECTOR_TOP_K`, `RERANK_TOP_N`, `HOP_CANDIDATE_CAP`, `MAX_ADAPTIVE_HOPS`, `HOP_EVIDENCE_TOP_N` | How wide and how deep the search goes |
| Operations | `RAG_WARMUP_QUESTION` | The question `/warmup` runs |

{{% notice warning %}}
**`.env.prod` holds the Groq API key in plain text, and this is the weakest point of the current configuration.** The file is excluded from git, readable only by the `ubuntu` user, and never leaves the instance - but a plaintext credential on a disk is still a credential on a disk. Anyone who obtains a shell on the instance, or a copy of the volume, obtains the key.

The file must never be committed. Confirm `.env.prod` is in `.gitignore` before the first push.
{{% /notice %}}

**The intended improvement, not yet applied.** The repository already contains `app/aws_runtime_config.py`, which can load non-secret settings from **AWS Systems Manager Parameter Store** and the Groq key from **AWS Secrets Manager**, using the instance role instead of a file on disk. The loader is opt-in: it activates only when `CONFIG_PREFIX` or `GROQ_SECRET_NAME` is present in the environment file.

That migration has **not** been performed on this deployment. It is recorded here as the planned next step rather than presented as done, and chapter 5.13 lists it among the known limitations. The benefit would be twofold: the secret leaves the disk, and a configuration change becomes a parameter update instead of an SSH session followed by a file edit.

<!-- IMAGE 4 - SCREENSHOT.
     A Session Manager terminal showing the real configuration file, with the
     GROQ_API_KEY line REMOVED. The safe way to capture it:
         grep -v GROQ_API_KEY .env.prod
     Screenshot that output rather than `cat`.
     Never publish a screenshot containing the key value. -->

![The production environment file](/images/5-Workshop/5.7-EC2-Backend/env-prod.png)

## 8. Run it under systemd

Running `uvicorn` in a terminal dies with the terminal. `systemd` gives restart-on-failure, start-on-boot and a log stream.

Service file at `/etc/systemd/system/aws-rag-api.service`:

```ini
[Unit]
Description=AWS RAG API Service
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/aws-rag-project/backend
EnvironmentFile=/home/ubuntu/aws-rag-project/backend/.env.prod
ExecStart=/home/ubuntu/aws-rag-project/backend/.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000
Restart=always
TimeoutStartSec=300
ExecStartPost=/bin/bash -lc 'sleep 20; curl -fsS -X POST http://127.0.0.1:8000/warmup || true'

[Install]
WantedBy=multi-user.target
```

Three lines deserve explanation:

- **`--host 0.0.0.0`** - binds all interfaces so API Gateway can reach the port. Binding `127.0.0.1` would make the service unreachable from outside.
- **`TimeoutStartSec=300`** - the first start downloads artifacts and loads a model. The default timeout would kill it mid-load.
- **`ExecStartPost=... /warmup`** - after the service comes up, systemd calls the warm-up endpoint itself. The `|| true` means a failed warm-up does not mark the service as failed.

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable aws-rag-api
sudo systemctl restart aws-rag-api
sudo systemctl status aws-rag-api
sudo journalctl -u aws-rag-api -f
```

`enable` is what makes the service come back after an instance reboot.

---

## 9. Fast mode and warm-up: fitting inside the timeout

This is the most instructive tuning problem in the project, and it came from a measurement rather than a guess.

**The observation.** A successful frontend request was measured at roughly **26.5 seconds**. API Gateway's integration timeout is around 30 seconds. The system worked, but had almost no margin - and the first request after any restart was far slower, because artifacts and the embedding model load lazily on first use.

**Two independent fixes:**

**`RAG_FAST_MODE=true`** attacks the steady-state cost. It skips query decomposition - removing one Groq round-trip before retrieval even begins - and shrinks the retrieval budget:

```env
RAG_FAST_MODE=true
BM25_TOP_K=15
VECTOR_TOP_K=15
RERANK_TOP_N=5
HOP_CANDIDATE_CAP=15
MAX_ADAPTIVE_HOPS=1
HOP_EVIDENCE_TOP_N=3
```

The cross-encoder reranker is also disabled (`rag-use-reranker=false`), since scoring candidates on CPU is expensive.

**`POST /warmup`** attacks the first-request cost. It loads the pipeline and performs one real vector retrieval so the embedding model, the artifacts and the S3 Vectors client are all resident - but it deliberately **does not call Groq generation**, so warming up costs no answer-generation tokens:

```json
{
  "status": "ok",
  "pipeline_loaded": true,
  "vector_backend": "s3vectors",
  "elapsed_seconds": 4.2,
  "warmup_child_hits": 12
}
```

Combined with `ExecStartPost`, every restart warms itself. A user never pays the cold-start penalty unless they arrive within the first few seconds of a restart.

{{% notice note %}}
Fast mode is an explicit **quality-for-latency trade**, and it is worth naming as such. It retrieves fewer candidates and skips multi-hop planning, so some hard questions get worse answers. For a demo that must respond reliably inside a hard timeout, predictable latency is worth more than peak accuracy. Chapter 5.11 measures both modes so the size of the trade is a number rather than an opinion.
{{% /notice %}}

---

## 10. Verify

**Health** - this endpoint reports whether centralised configuration actually loaded:

```bash
curl http://127.0.0.1:8000/health
```

```json
{"status":"ok","pipeline_loaded":false}
```

Two fields, both meaningful:

| Field | Meaning |
| --- | --- |
| `status: ok` | The process is up and answering. It does **not** mean the pipeline is ready |
| `pipeline_loaded: false` | Normal before warm-up - artifacts and the embedding model load lazily on first use |
| `pipeline_loaded: true` | Artifacts, BM25 index and the embedding model are in memory; queries will be fast |

Because configuration comes from `.env.prod`, a wrong setting does not show up here - it shows up as a failure on the first query. That is a real drawback of file-based configuration, and the reason the health endpoint of a Parameter Store deployment reports more (section 7).

**Warm up, then query:**

```bash
curl -X POST http://127.0.0.1:8000/warmup

curl -X POST http://127.0.0.1:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question":"Were Scott Derrickson and Ed Wood of the same nationality?"}'
```

From a Windows laptop, against the public address:

```powershell
curl.exe "http://<elastic-ip>:8000/health"

'{"question":"Were Scott Derrickson and Ed Wood of the same nationality?"}' | Set-Content -Encoding utf8 query.json
curl.exe -s -X POST "http://<elastic-ip>:8000/query" `
  -H "Content-Type: application/json" `
  --data-binary "@query.json"
```

The response carries the answer plus everything needed to justify it - `sources`, `timings`, `num_candidates` and `token_usage_total`.

<!-- IMAGE 5 - SCREENSHOT.
     Terminal showing, in one shot if possible:
       1. `sudo systemctl status aws-rag-api` with "active (running)"
       2. the /health response
     Take it after warm-up so pipeline_loaded shows true - that is the state that
     proves the artifacts and the model actually loaded. -->

![Service running and configuration loaded](/images/5-Workshop/5.7-EC2-Backend/service-health.png)

<!-- IMAGE 6 - SCREENSHOT.
     The JSON response from POST /query, showing the answer and at least the beginning
     of the sources array and the timings object.
     This proves the backend works end to end BEFORE API Gateway exists - which is
     exactly what makes debugging chapter 5.8 tractable. -->

![A successful query against the backend](/images/5-Workshop/5.7-EC2-Backend/query-response.png)

---

## 11. Serving a different index

Switching corpus or index needs no code change, no new AMI and no frontend change - only an edit to the environment file and a restart:

```bash
cd ~/aws-rag-project/backend
cp .env.prod .env.prod.backup      # always, before editing
nano .env.prod
```

Change the three identifiers that select the index:

```env
RAG_INDEX_ID=<new-index-id>
S3_PROCESSED_ID=<new-processed-id>
S3_VECTOR_INDEX=<new-vector-index>
```

```bash
sudo systemctl restart aws-rag-api
curl http://127.0.0.1:8000/health
curl -X POST http://127.0.0.1:8000/warmup
```

Because chapter 5.4 versions every identifier, rolling back is the same edit with the old values - or simply restoring the backup file.

The cost of this approach is that it requires a shell session on the instance, which is exactly the friction the Parameter Store migration described in section 7 would remove.

## 12. Daily operation

1. Start the instance and wait for status checks to pass.
2. Connect with Session Manager.
3. `sudo systemctl status aws-rag-api` and `curl http://127.0.0.1:8000/health`.
4. If needed, `curl -X POST http://127.0.0.1:8000/warmup`.
5. Test through API Gateway (chapter 5.8), then open the frontend (chapter 5.9).

Logs live in the journal:

```bash
sudo journalctl -u aws-rag-api -n 200 --no-pager
sudo journalctl -u aws-rag-api -f
```

---

## 13. Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| Service fails to start, log ends with `Killed` | Out of RAM loading the embedding model | Larger instance, or a smaller model |
| `systemd` marks the start as timed out | Artifact download plus model load exceeded the default timeout | `TimeoutStartSec=300` |
| Groq authentication error on the first query | `GROQ_API_KEY` missing or stale in `.env.prod` | Check the file, then restart the service |
| A variable in `.env.prod` appears to be ignored | The file was written with a UTF-8 BOM, so the first variable name carries an invisible `\ufeff` prefix | Rewrite the file without a BOM |
| `FileNotFoundError` on first query | Artifacts missing or under the wrong S3 prefix | Re-check chapter 5.5 section 7 |
| Retrieval returns nonsense | Index built with a different embedding model | See chapter 5.6 section 5 |
| Works locally, unreachable from outside | `uvicorn` bound to `127.0.0.1` | Use `--host 0.0.0.0` |
| API Gateway breaks after a stop/start | Public IP changed | Elastic IP, section 3 |
| First request after restart times out | Cold start | `ExecStartPost` warm-up, section 9 |

---

## 14. Result

A backend that starts on boot, restarts on failure, warms itself, reads all of its configuration from AWS, and holds no credentials of any kind. It answers `POST /query` over HTTP on port `8000`.

That last word is the remaining problem: **HTTP**. The frontend will be served over HTTPS, and a browser will refuse to let an HTTPS page call a plain-HTTP API. Chapter 5.8 puts API Gateway in front to terminate TLS.

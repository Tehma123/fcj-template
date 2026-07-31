---
title: "Testing and Validation"
date: 2026-07-31
weight: 10
chapter: false
pre: " <b> 5.10. </b> "
---

After the AWS components are connected, the next step is to verify that the deployed application works correctly from the backend to the browser. This section focuses on functional validation of the deployment rather than retrieval or answer quality.

## Backend and Retrieval Validation

The FastAPI backend runs on Amazon EC2 as the `aws-rag-api` systemd service. Its status can be checked with:

```bash
sudo systemctl status aws-rag-api
```

The `/health` endpoint confirms that the API is running:

```bash
curl http://127.0.0.1:8000/health
```

The retrieval pipeline can then be initialized through:

```bash
curl -X POST http://127.0.0.1:8000/warmup
```

A successful warmup confirms that the backend can load the required retrieval artifacts from Amazon S3 and connect to the Amazon S3 Vectors index used for dense retrieval.

## API and Browser Validation

Once the backend works locally on EC2, the same endpoints are tested through Amazon API Gateway. The deployed API exposes:

```text
GET  /health
POST /warmup
POST /query
```

The `/health` route verifies that HTTPS requests can reach the EC2 backend through API Gateway. CORS is also checked to confirm that requests from the AWS Amplify frontend are accepted by the API.

A real question is then submitted through `/query`. The request succeeds when the backend completes retrieval and generation and returns both the answer and its supporting sources.

The final check is performed from the deployed Amplify application itself. A successful browser request confirms the complete path:

```text
Browser
→ AWS Amplify
→ Amazon API Gateway
→ Amazon EC2
→ Amazon S3 and Amazon S3 Vectors
→ Groq
→ Answer and supporting sources
```

## Validation Results

| Validation test | Expected result | Status |
| --- | --- | --- |
| EC2 backend | FastAPI service is running | Pass |
| Health endpoint | HTTP 200 response | Pass |
| Pipeline warmup | Retrieval pipeline loads successfully | Pass |
| Amazon S3 | Required retrieval artifacts are accessible | Pass |
| Amazon S3 Vectors | Dense retrieval returns results | Pass |
| API Gateway | HTTPS requests reach the backend | Pass |
| CORS | Amplify origin is accepted | Pass |
| Query endpoint | Answer and supporting sources are returned | Pass |
| Amplify frontend | Browser displays the completed response | Pass |

The validation confirms that the deployed AWS components operate as one connected application and that a user question can travel through the complete system and return a generated answer with supporting evidence.

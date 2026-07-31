---
title: "Operations and Troubleshooting"
date: 2026-07-31
weight: 12
chapter: false
pre: " <b> 5.12. </b> "
---

After deployment, most day-to-day operation of CloudHop RAG centers on the EC2 backend. The application runs as the `aws-rag-api` systemd service, while **AWS Systems Manager Session Manager** provides access to the instance when maintenance or troubleshooting is required.

The backend exposes `/health` and `/warmup` endpoints for checking application status and initializing the retrieval pipeline. Service output can also be inspected directly through the systemd journal when a request fails or the backend does not start correctly.

<!--
Continue this section with:
- starting/restarting/checking aws-rag-api with systemctl;
- viewing backend logs with journalctl;
- using /health and /warmup for runtime checks;
- accessing EC2 through Systems Manager Session Manager;
- a short troubleshooting table for the main issues encountered during deployment, such as API Gateway errors, CORS, Mixed Content, unavailable backend, or slow requests.
Do not introduce CloudWatch; it is not part of the implemented system.
-->

### Operational Guide

Daily system operation guide.

#### 1. Console Steps: Start System
After stopping the EC2 instance, when you need to use it again:
1. **Start EC2**: Start the instance in the **EC2 Console**.
2. **SSM Session**: Connect via **Session Manager** (no SSH key required).

#### 2. CLI Steps: Start Backend
After entering the EC2 terminal:
```bash
sudo systemctl restart aws-rag-api
# Warmup (required to load the model)
curl -X POST http://127.0.0.1:8000/warmup
```

#### 3. CLI Steps: Verification
```bash
curl http://127.0.0.1:8000/health
```

#### 4. Troubleshooting

| Error | Cause | Resolution |
| :--- | :--- | :--- |
| **Mixed Content** | Frontend HTTPS calling backend HTTP | Check `VITE_API_BASE_URL` in Amplify Console |
| **CORS Error** | API Gateway blocking request | Check CORS tab configuration in API Gateway Console |
| **503 Service** | Backend is down | Run `sudo systemctl status aws-rag-api` to check logs |
| **SSH Timeout** | Security Group blocking | Use Session Manager in Console instead of SSH |
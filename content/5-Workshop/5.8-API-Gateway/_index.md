---
title: "Amazon API Gateway"
date: 2026-07-31
weight: 8
chapter: false
pre: " <b> 5.8. </b> "
---

The FastAPI backend on EC2 is available over HTTP, while the frontend delivered by AWS Amplify runs over HTTPS. A browser cannot safely call the HTTP backend directly from the HTTPS frontend, so **Amazon API Gateway** is used as the public HTTPS entry point for the application.

API Gateway receives requests from the browser and forwards them to the FastAPI service on EC2. It also provides the browser-facing CORS configuration required for the Amplify frontend to call the backend successfully.

{{% notice info %}}
**What you will have at the end of this chapter:** an HTTP API with three routes proxying to EC2, CORS configured for the Amplify origin, and a verified HTTPS `POST /query` returning a real answer. This chapter is where three separate browser-level errors get solved, and each one is worth understanding rather than just copying the fix.
{{% /notice %}}

---

## 1. Why an HTTP API rather than a REST API

API Gateway offers two types. This project uses **HTTP API**:

| | HTTP API | REST API |
| --- | --- | --- |
| Purpose here | TLS termination and a simple proxy to EC2 | Full API management |
| Cost | Roughly a third of REST API pricing | Higher |
| Features | Routes, integrations, CORS, JWT auth | Adds usage plans, API keys, request validation, WAF |
| Fit | **Chosen** - everything needed, nothing more | Overkill for three proxy routes |

The backend already does its own validation with Pydantic models, so the request-validation features of a REST API would only duplicate work. An Application Load Balancer would also terminate TLS, but it bills per hour whether or not anyone uses it and needs its own certificate and domain; API Gateway provides a managed HTTPS endpoint with a certificate out of the box and bills per request.

---

## 2. Create the API and its integrations

**Console:** API Gateway → **Create API** → **HTTP API** → Build.

Name it something that says what it is - `rag-ec2-backend` in this project.

The integration target is the EC2 backend. And here is the first real trap.

{{% notice warning %}}
**Do not use a single generic integration.** Pointing one integration at the base URL:

```text
http://<elastic-ip>:8000/
```

produces this on every call:

```json
{"detail":"Not Found"}
```

The request reaches FastAPI, but the path is not forwarded the way you expect, so FastAPI is asked for a route that does not exist. The response looks like a backend bug and is not one.

**The fix is one integration per route, each pointing at the full backend path:**

```text
http://<elastic-ip>:8000/health
http://<elastic-ip>:8000/warmup
http://<elastic-ip>:8000/query
```
{{% /notice %}}

This is also the point where chapter 5.7's Elastic IP pays off. The integration stores a **literal address**. Without a fixed IP, every EC2 stop/start silently repoints the API at an address that no longer belongs to your instance - and the failure mode is a timeout, not an obvious error.

---

## 3. Routes

Three routes, matching the endpoints from chapter 5.7:

| Route | Integration target | Purpose |
| --- | --- | --- |
| `GET /health` | `http://<elastic-ip>:8000/health` | Liveness and configuration check |
| `POST /warmup` | `http://<elastic-ip>:8000/warmup` | Load the pipeline before real traffic |
| `POST /query` | `http://<elastic-ip>:8000/query` | Answer a question |

`POST /warmup` is optional at this layer - `systemd` already calls it locally on the instance after every restart (chapter 5.7 section 8). Exposing it through API Gateway is only useful if you want to warm the service from your laptop or an admin page. It costs nothing to add and it is genuinely handy when the instance has been stopped overnight.

---

## 4. CORS is two layers, not one

This is the part that costs people an afternoon. A browser calling the API from a different origin performs a **preflight** `OPTIONS` request first, and **both** API Gateway and FastAPI must be configured to satisfy it.

The error that appears if you configure only one of them:

```text
Access to fetch at 'https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/query'
from origin 'https://<your-amplify-app>.amplifyapp.com'
has been blocked by CORS policy.
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

**Layer 1 - API Gateway CORS.** In the Console, under **CORS**:

| Setting | Value |
| --- | --- |
| `Access-Control-Allow-Origin` | `https://<your-amplify-app>.amplifyapp.com` |
| `Access-Control-Allow-Methods` | `GET`, `POST`, `OPTIONS` |
| `Access-Control-Allow-Headers` | `content-type` |
| `Access-Control-Allow-Credentials` | No |

**Layer 2 - FastAPI CORS.** The backend has its own `CORSMiddleware`, and its allowed origins come from the SSM parameter `cors-allow-origins` set in chapter 5.7:

```text
http://localhost:5173,http://127.0.0.1:5173,https://<your-amplify-app>.amplifyapp.com
```

The two local origins are the Vite dev server, so the same backend serves local development without a separate configuration.

{{% notice note %}}
Note what is **not** used: a wildcard `*`. The allowed origin is named explicitly on both layers. A wildcard would let any website on the internet call this API from a user's browser, and since the API has no authentication, that is a meaningful difference rather than a formality.
{{% /notice %}}

**Verify the preflight directly** - this is much faster than debugging through the browser:

```powershell
curl.exe -i -X OPTIONS "https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/query" `
  -H "Origin: https://<your-amplify-app>.amplifyapp.com" `
  -H "Access-Control-Request-Method: POST" `
  -H "Access-Control-Request-Headers: content-type"
```

A correct configuration returns:

```text
HTTP/1.1 204 No Content
access-control-allow-origin: https://<your-amplify-app>.amplifyapp.com
access-control-allow-methods: GET,OPTIONS,POST
access-control-allow-headers: content-type
```

If those three headers are absent, the browser will refuse the real request no matter how well the backend works.

---

## 5. Deploy the API

HTTP APIs create a `$default` stage with **auto-deploy enabled**, so route and CORS changes go live without a manual deploy step. If you created a named stage instead, deploy it after every change - a surprising number of "CORS is still broken" reports are simply an undeployed stage.

Note the **Invoke URL**:

```text
https://<api-id>.execute-api.ap-southeast-1.amazonaws.com
```

This single URL is what the frontend will be configured with in chapter 5.9. Nothing else about the backend is ever exposed to a browser.

---

## 6. Test through HTTPS

Health first - it is fast and requires no pipeline:

```powershell
curl.exe "https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/health"
```

```json
{"status":"ok","pipeline_loaded":false}
```

Then a real question:

```powershell
'{"question":"Were Scott Derrickson and Ed Wood of the same nationality?"}' | Set-Content -Encoding utf8 query.json

curl.exe -s -X POST "https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/query" `
  -H "Content-Type: application/json" `
  --data-binary "@query.json"
```

```json
{ "answer": "yes", "sources": [ ... ], "timings": { ... } }
```

{{% notice tip %}}
Always test the EC2 backend **directly** first (chapter 5.7 section 10), then through API Gateway. If the direct call works and the gateway call does not, the fault is in this chapter - routing, integration path or CORS. If neither works, the fault is in chapter 5.7. Testing both halves separately turns a vague "the app is broken" into a two-minute diagnosis.
{{% /notice %}}

---

## 7. The timeout ceiling

API Gateway waits roughly **30 seconds** for the backend. When the pipeline takes longer, the caller gets:

```json
{"message":"Service Unavailable"}
```

This message is misleading - it usually does not mean the backend is down. It means API Gateway gave up waiting. A measured frontend request came in at **26.56 seconds**, which is inside the limit but with almost no margin, and the first request after a restart is far slower.

That single measurement is what drove two decisions already made in chapter 5.7:

- `RAG_FAST_MODE=true` to cut steady-state latency
- `POST /warmup` called by `systemd` after every restart, so nobody pays the cold-start cost

If the pipeline ever needs to exceed 30 seconds, API Gateway is the wrong shape for that route and the alternatives are Nginx or an ALB terminating TLS on EC2 itself, or an asynchronous job pattern where `/query` returns a job id immediately and the frontend polls for the result.

---

## 8. Security posture

Two things to state plainly:

- **The API has no authentication.** Anyone with the Invoke URL can submit questions, and each question spends Groq tokens. For a demo this is accepted; for anything real, an HTTP API supports JWT authorizers, or a Lambda authorizer, or an API key in front.
- **API Gateway does not hide the backend.** Port `8000` on the instance is still reachable directly (chapter 5.7 section 4). API Gateway solves *HTTPS*, not *exposure*. Making the backend genuinely private requires a **VPC Link** to an instance in a private subnet - noted as a known limitation in chapter 5.13.

---

## 9. Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Mixed Content ... blocked` in the browser console | The frontend is calling the EC2 HTTP address, not the gateway | Set `VITE_API_BASE_URL` to the HTTPS Invoke URL (chapter 5.9) |
| `blocked by CORS policy` | Origin missing on API Gateway, or on FastAPI, or both | Configure both layers, then re-run the preflight test in section 4 |
| `{"detail":"Not Found"}` | Generic integration URL; path not forwarded | One integration per route with the full backend path (section 2) |
| `{"message":"Service Unavailable"}` | Backend unreachable, security group closed, or query exceeded ~30s | Test EC2 directly, then warm up, then retry |
| Health works but query times out | Cold start | `POST /warmup` first |
| Everything worked yesterday, nothing works today | EC2 public IP changed | Elastic IP (chapter 5.7 section 3) |
| CORS changes appear to have no effect | Named stage not redeployed | Deploy the stage, or use `$default` with auto-deploy |

---

## 10. Result

The backend is now reachable over HTTPS at a stable managed URL, with CORS accepting exactly one browser origin. The Mixed Content problem that motivated this chapter is solved, and the last remaining piece is the interface itself.

Chapter 5.9 deploys the React frontend to Amplify and points it at this Invoke URL.

---
title: "AWS Amplify Frontend Deployment"
date: 2026-07-31
weight: 9
chapter: false
pre: " <b> 5.9. </b> "
---

The final user-facing component of CloudHop RAG is a **React/Vite** web application deployed with **AWS Amplify**. The interface allows users to submit a question, receive the generated answer, and inspect the supporting sources returned by the RAG backend.

The frontend communicates with the backend through the HTTPS endpoint created in Amazon API Gateway. This keeps the browser-facing application separate from the EC2 service while providing a single public interface for users.

{{% notice info %}}
**What you will have at the end of this chapter:** the complete system working from a browser - a question typed into a hosted page returning an answer with its evidence, over HTTPS end to end. This is the last deployment step.
{{% /notice %}}

---

## 1. What is being deployed

A single-page React 19 application built with Vite. The whole interface is one component that sends a question and renders the response:

| Field returned by `/query` | Shown as |
| --- | --- |
| `answer` | The answer, displayed prominently |
| `sources` | The supporting documents behind the answer |
| `timings` | Retrieval and total response time |
| `num_candidates` | How many documents were considered |

Displaying `sources` is not decoration. A RAG system that shows only an answer is indistinguishable from a chatbot guessing; showing the retrieved evidence is what lets a reader check the answer against its source.

Amplify Hosting was chosen over S3 + CloudFront because it provides git-push deployment, a managed TLS certificate and a CDN with no manual configuration. The equivalent S3 + CloudFront setup means creating a bucket, a distribution, a certificate and an origin access policy by hand - more moving parts for the same result.

---

## 2. Connect the repository

**Console:** Amplify → **Create new app** → **Deploy from Git** → GitHub → authorise → pick the repository and branch.

| Setting | Value in this project |
| --- | --- |
| App name | `aws-rag-project` |
| Branch | the branch you deploy from |
| **Monorepo root directory** | **`frontend`** |

The monorepo setting matters. This repository holds both `backend/` and `frontend/`, so Amplify must be told that the application lives in the `frontend/` subdirectory. Without it, the build runs at the repository root, finds no `package.json`, and fails immediately.

---

## 3. Build settings

Vite outputs a static site into `dist/`, and that is the directory Amplify must publish:

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm install
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: dist
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

Two lines to get right:

- **`baseDirectory: dist`** - Vite's output directory. Amplify defaults to `build` (the Create React App convention). Leaving the default gives a successful build followed by a blank page or a 404, because Amplify publishes a directory that does not exist.
- **`cache: node_modules/**/*`** - caches dependencies between builds, cutting build time substantially after the first run.

---

## 4. `VITE_API_BASE_URL` - the critical setting

**App settings → Environment variables:**

| Key | Value |
| --- | --- |
| `VITE_API_BASE_URL` | `https://<api-id>.execute-api.ap-southeast-1.amazonaws.com` |

This must be the **API Gateway HTTPS Invoke URL** from chapter 5.8 - never the EC2 address. Using the EC2 address produces exactly this in the browser console:

```text
Mixed Content: The page at 'https://<your-amplify-app>.amplifyapp.com/' was loaded
over HTTPS, but requested an insecure resource 'http://<elastic-ip>:8000/query'.
This request has been blocked; the content must be served over HTTPS.
```

The page loads perfectly and every question fails. This is the error that motivated chapter 5.8 existing at all.

{{% notice warning %}}
**Vite environment variables are baked in at build time, not read at runtime.** The frontend reads:

```javascript
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;
```

Vite replaces that expression with the literal string during `npm run build`. The compiled JavaScript contains the URL as text - there is no runtime lookup.

The practical consequence: **changing this variable in the Amplify Console does nothing until you redeploy.** Editing the value and refreshing the page will show the old URL still being called, which is deeply confusing if you expect environment variables to behave the way they do on a server. After any change, trigger **Redeploy this version**.
{{% /notice %}}

There is a matching setting on the other side: the Amplify origin must appear in the backend's `cors-allow-origins` SSM parameter (chapter 5.7) and in the API Gateway CORS configuration (chapter 5.8). The frontend URL is not known until the app is first created, so the usual order is: deploy once, copy the Amplify URL, then update CORS on both layers.

---

## 5. Deploy

Amplify builds automatically on every push to the connected branch. To deploy manually after a settings change, use **Redeploy this version**.

The build runs four stages - Provision, Build, Deploy, Verify - and the log is worth reading on the first run: it shows `npm install`, then `npm run build`, then the artifact upload. A failure here is almost always the monorepo root or `baseDirectory`.

On success the app is served at:

```text
https://<branch>.<app-id>.amplifyapp.com
```

---

## 6. Verify from the browser

Open the Amplify URL. The interface itself reports whether it was configured correctly - the header shows one of two states:

| Displayed | Meaning |
| --- | --- |
| `Connected through AWS API Gateway` | `VITE_API_BASE_URL` was present at build time |
| `API URL is not configured` | The variable was missing when the build ran |

The second message means the variable was added *after* the build - redeploy.

Then submit a real multi-hop question:

```text
Were Scott Derrickson and Ed Wood of the same nationality?
```

A successful response shows the answer, the supporting sources, and the elapsed time. That single interaction exercises the entire system:

```text
Browser
  → AWS Amplify (HTTPS)
  → Amazon API Gateway (HTTPS)
  → Amazon EC2 - FastAPI
  → Amazon S3 + Amazon S3 Vectors
  → Groq
  → answer + supporting sources
```

{{% notice tip %}}
If the first question after an idle period times out, that is the cold start described in chapter 5.7, not a frontend fault. Call `POST /warmup` through API Gateway, or simply ask again - the second question is fast. Keep the browser developer console open on the first test: **Mixed Content** and **CORS** failures appear there with an exact explanation, while the page itself only shows a generic error.
{{% /notice %}}

<!-- IMAGE 4 - SCREENSHOT.
     The deployed application in a browser after asking the test question.
     Must show together: the address bar with the https:// Amplify URL and padlock,
     the answer, and the supporting sources list.
     This is the single most important screenshot in the whole report - it is the
     evidence for the end-to-end deployment criterion in the grading rubric. Chapter 5.3 references
     the same shot, so take it once and reuse it. -->

![The deployed application answering a question](/images/5-Workshop/5.9-Amplify-Frontend/deployed-app.png)

---

## 7. Local development against the same backend

The same frontend runs locally without any change to the deployment. Create `frontend/.env.local`:

```env
VITE_API_BASE_URL=http://127.0.0.1:8000
```

```powershell
cd frontend
npm install
npm run dev
```

This works because `http://localhost:5173` and `http://127.0.0.1:5173` are already in the backend's allowed CORS origins (chapter 5.7). Both pages are HTTP locally, so Mixed Content does not apply - the problem only exists once the page is served over HTTPS.

`.env.local` is git-ignored and must stay that way; the deployed value belongs in the Amplify Console, not in the repository.

---

## 8. Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| Build fails immediately, no `package.json` found | Monorepo root not set | Set the app root to `frontend` |
| Build succeeds, page is blank or 404 | Wrong `baseDirectory` | Use `dist`, not `build` |
| `API URL is not configured` in the header | Variable added after the build | Redeploy |
| `Mixed Content ... blocked` | `VITE_API_BASE_URL` points at the EC2 HTTP address | Use the API Gateway HTTPS URL |
| `blocked by CORS policy` | Amplify origin not allowed | Add it to API Gateway CORS **and** the `cors-allow-origins` parameter |
| Changing the variable has no effect | Vite inlines it at build time | Redeploy this version |
| First question times out, later ones work | Backend cold start | `POST /warmup`, or ask again |
| `{"message":"Service Unavailable"}` | Query exceeded the API Gateway timeout | Chapter 5.8 section 7 |

---

## 9. Result

The deployment is complete. A user opens an HTTPS page, types a multi-hop question, and receives a short factual answer with the documents it came from - while every AWS resource behind it stays private except the two managed HTTPS endpoints.

Chapter 5.10 validates the whole path systematically, and chapter 5.11 measures how well it answers.

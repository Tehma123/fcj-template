---
title: "Prerequisites"
date: 2026-07-31
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

Before starting the deployment, prepare the AWS account, development tools, and project access required by CloudHop RAG.

## AWS Account and Access

CloudHop RAG is deployed in the **Asia Pacific (Singapore) Region (`ap-southeast-1`)**. Use this Region throughout the workshop for all AWS resources.

You need an AWS account with access to the AWS Management Console and sufficient permissions to create and manage:

- Amazon S3;
- Amazon S3 Vectors;
- Amazon EC2;
- AWS Identity and Access Management (IAM);
- Amazon API Gateway;
- AWS Amplify;
- AWS Systems Manager.

AWS Systems Manager Session Manager is used to access and manage the EC2 backend during deployment.

## Development Environment

The project uses a small set of standard development tools.

| Tool | Used for |
| --- | --- |
| **Git** | Cloning and updating the project source code |
| **Python + pip** | Backend dependencies and project scripts |
| **AWS CLI** | Creating, checking, and updating AWS resources from the terminal |
| **Node.js + npm** | Installing and building the React/Vite frontend |

You can confirm that the required tools are available with:

```bash
aws --version
python --version
node --version
npm --version
git --version
```

On some systems, Python may be available as `python3` instead of `python`.

## AWS CLI Access

Configure the AWS CLI for the account used in this workshop:

```bash
aws configure
```

Set the default Region to:

```text
ap-southeast-1
```

Then verify the active AWS identity:

```bash
aws sts get-caller-identity
```

A successful response confirms that the CLI can authenticate to the AWS account.

## Project Requirements

Before starting, make sure you have:

- the **AWS CloudHop RAG source code**;
- a valid **Groq API key** for answer generation;
- the prepared retrieval artifacts used by the deployment.

If the retrieval artifacts will be rebuilt from source, access to the **HotpotQA** dataset is also required.
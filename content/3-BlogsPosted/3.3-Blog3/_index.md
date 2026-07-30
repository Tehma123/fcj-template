---
title: "Blog 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.3. </b> "
---

# Detailed Guide: Extending Amazon CloudWatch with Cribl Stream for Any Data Source

In hybrid or multi-cloud management environments, organizations frequently need to collect telemetry data from complex sources such as proprietary network devices, on-premises APM tools, or Apache Kafka data streams.

While Amazon CloudWatch already provides native log ingestion for more than 60 AWS services and 20 third-party tools, data sources outside this ecosystem still require complex normalization work. This is where Cribl Stream — an AWS partner solution — shines, letting you bring hundreds of diverse data sources into CloudWatch seamlessly.

## 1. How Cribl Stream Works

Cribl Stream acts as an intermediary pipeline sitting between source systems and CloudWatch Logs. Here are the core technical benefits:

- **Broad source support:** Cribl Stream can ingest logs from syslog, Apache Kafka, APM tools, security agents, as well as sources based on HTTP, TCP, and UDP protocols.
- **No intermediary resources needed:** Data is written directly into a CloudWatch log group via the `PutLogEvents` API. This architecture completely removes the need for intermediate storage, SQS queues, or processing Lambda functions.
- **Data integrity guarantee:** A persistent queue inside Cribl Stream protects data from being lost during network errors or temporary CloudWatch Logs outages.
- **Replay capability:** Administrators can replay raw data for auditing, security investigation, or incident handling purposes.

## 2. The Standard 3-Tier Architecture

This connection model is designed around three lean architectural tiers:

| Architecture Tier | Technical Characteristics |
|---|---|
| **Source (Upstream)** | Each logical source system should write to its own separate log group (e.g., `/apps/any-company-firewall`) to make access control and retention policies easier to manage. |
| **Cribl Stream** | Can run on an Amazon EC2 instance from AWS Marketplace or as a container inside a VPC. Here, logs are automatically normalized into OCSF or OTel format. |
| **CloudWatch Logs** | Normalized data is stored here, letting engineers run powerful queries with Logs Insights QL, OpenSearch PPL, or SQL. |

## 3. Optimal Configuration Steps

Integration doesn't require writing any custom code. Instead, you just need to complete the following configuration:

### Deployment
- You can use the managed SaaS version, Cribl.Cloud Suite, or self-host it via AWS Marketplace.
- For closed environments, place Cribl Stream inside a private subnet and communicate through the CloudWatch Logs VPC endpoint (`com.amazonaws.logs`).

### Log Group & IAM Configuration
- **Cost savings:** For infrequently accessed data (archival, compliance), choose the "Infrequent Access" log class when creating the CloudWatch log group.
- **Tagging:** Use the `cw:datasource:name` and `cw:datasource:type` tags so CloudWatch can easily identify the log source.
- **IAM security:** Attach an IAM role to the Cribl Stream instance with the `logs:CreateLogStream`, `logs:PutLogEvents`, and `logs:DescribeLogStreams` permissions. Restrict the Resource ARN so Cribl cannot write outside the designated log groups.

### Destination
- In Cribl, configure the destination to point to CloudWatch Logs using JSON format, IAM authentication, and enable the persistent queue.
- If log volume is very high, enable compression and tune the batch size to avoid exceeding the `PutLogEvents` API limits.

## 4. Cross-Source Analysis & Agentic AI Support

By bringing data from outside the AWS ecosystem to sit alongside AWS's own native logs, security investigation becomes far more flexible than before:

- **Cross-system queries:** Use the built-in `@log` field to identify the origin of each data row. You can easily trace a suspicious IP address by joining data from the company firewall with the Amazon VPC Flow Logs stream.
- **Cross-platform error analysis:** Combine data from an identity provider (IdP) and AWS CloudTrail to centrally detect failed login patterns. The `coalesce` function in a query helps read across differing log structures.

### The Power of Agentic AI

Notably, because the data is normalized into an open format, it can be connected to an Amazon CloudWatch MCP server. MCP (Model Context Protocol) is an open standard from the Linux Foundation that links large language models (LLMs) to real-world data.

As a result, compatible AI assistants such as Claude Code, Kiro, or GitHub Copilot can analyze log data directly. Instead of writing SQL queries, on-call engineers simply need to ask the AI: *"Summarize the failed logins from the identity provider and AWS CloudTrail over the past hour"*, and the AI will return results based on real data.

**Reference link:** <https://aws.amazon.com/blogs/mt/extend-amazon-cloudwatch-beyond-native-connectors-with-cribl-stream/>
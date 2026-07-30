---
title: "Week 3 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.3. </b> "
---
{{% notice warning %}} 
⚠️ **Note:** The following information is for reference purposes only. Please **do not copy verbatim** for your own report, including this warning.
{{% /notice %}}


### Week 3 Objectives:

* Understand how to monitor AWS resources with Amazon CloudWatch (metrics, logs, alarms).
* Understand how Amazon CloudFront works as a CDN and how it integrates with S3.
* Practice setting up monitoring and content delivery for a simple workload.

### Tasks to be carried out this week:
| Day | Task                                                                                                                                                                                  | Start Date | Completion Date | Reference Material                        |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | --------------- | ----------------------------------------- |
| 2   | - Learn CloudWatch: <br>&emsp; + Metrics <br>&emsp; + Log groups & log streams <br>&emsp; + Alarms & dashboards                                                                     | 06/22/2026 | 06/22/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 3   | - **Practice:** create a CloudWatch alarm on EC2 CPU utilization; ship EC2 logs to CloudWatch Logs using the CloudWatch agent                                                       | 06/23/2026 | 06/23/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 4   | - Learn CloudFront: <br>&emsp; + Distributions & origins (S3/EC2) <br>&emsp; + Edge locations & caching behavior <br>&emsp; + Origin Access Control (OAC)                          | 06/24/2026 | 06/24/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 5   | - **Practice:** create a CloudFront distribution in front of an S3 bucket, restrict direct S3 access with OAC, and test cache invalidation                                          | 06/25/2026 | 06/25/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 6   | - Build a CloudWatch dashboard combining EC2 and CloudFront metrics <br> - Review the week's work with mentor                                                                       | 06/26/2026 | 06/26/2026      | <https://cloudjourney.awsstudygroup.com/> |


### Week 3 Achievements:

* Set up CloudWatch alarms and log collection for an EC2 instance.

* Understood how CloudFront caches and serves content from edge locations.

* Deployed a CloudFront distribution in front of an S3 bucket with Origin Access Control, so the bucket is no longer reachable directly from the public internet.

* Built a basic CloudWatch dashboard to track resource health at a glance.

* Understood how monitoring (CloudWatch) and content delivery (CloudFront) fit into a production-ready architecture.
* ...

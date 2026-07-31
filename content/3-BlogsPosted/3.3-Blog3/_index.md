---
title: "Blog 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.3. </b> "
---

# Amazon SQS Fair Queues: Ending the "Noisy Neighbor" Problem in Multi-Tenant Systems

![Blog post published on the AWS Study Group VN Facebook group](/images/BlogsPosted/blog3.png)
*Posted to the AWS Study Group VN Facebook group.*

Message queues have long been the backbone of distributed architectures — a buffer that keeps a system from collapsing under a sudden traffic spike. Amazon SQS has always been a go-to choice thanks to its near-unlimited scalability.

But there's a problem that almost every team building a multi-tenant system on a shared queue has run into: one "noisy" tenant can slow down every other tenant sharing that queue.

## 1. What Is the "Noisy Neighbor" Problem?

In a multi-tenant system sharing a single SQS queue, if one tenant suddenly sends a huge volume of messages or processes its own messages very slowly, a regular queue still delivers older messages first based on overall arrival order. This buries other tenants' messages further back in line, increasing message dwell time for every other tenant — even ones that never caused any problem. In other words, a single noisy neighbor can drag down the quality of service for an entire shared system.

## 2. How Amazon SQS Fair Queues Solves This

Amazon SQS Fair Queues is a new feature that automatically mitigates the impact of a noisy neighbor by intelligently reordering message delivery, prioritizing "quiet" tenants whenever one tenant is found to be dominating consumer resources.

### Identifying Tenants via MessageGroupId

The system uses the existing `MessageGroupId` field on a message to identify tenant boundaries. When sending a message, an application simply attaches a tenant identifier:

```java
SendMessageRequest request = new SendMessageRequest()
    .withQueueUrl(queueUrl)
    .withMessageBody(messageBody)
    .withMessageGroupId("tenant-123");  // Tenant identifier
sqs.sendMessage(request);
```

### The Fairness Algorithm

SQS continuously monitors the distribution of in-flight messages across tenants. When it detects an imbalance, the system: (1) identifies the noisy tenant, (2) automatically reprioritizes messages from quiet tenants, and (3) still preserves the queue's overall throughput.

### No Consumer-Side Code Changes Required

Fair Queues works completely transparently to consumers — no changes to message-processing logic, no added API latency, and no throughput limitations.

## 3. Monitoring via CloudWatch

The feature ships with new CloudWatch metrics dedicated to tracking queue fairness:

- `ApproximateNumberOfNoisyGroups`
- `ApproximateNumberOfMessagesVisibleInQuietGroups`
- `ApproximateNumberOfMessagesNotVisibleInQuietGroups`
- `ApproximateNumberOfMessagesDelayedInQuietGroups`
- `ApproximateAgeOfOldestMessageInQuietGroups`

Combined with **CloudWatch Contributor Insights**, an operations team can pinpoint exactly which tenant is consuming a disproportionate share of resources, even across a system with thousands of distinct message groups.

## 4. Benefits

- Keeps message dwell time low for non-noisy tenants, even during a traffic spike from another tenant.
- Removes the need for over-provisioning or building a custom tenant-isolation solution.
- Preserves quality of service transparently, with no changes to existing architecture.
- Still supports unlimited throughput, with no hard per-tenant quota imposed.

## Conclusion

Amazon SQS Fair Queues addresses exactly the problem so many multi-tenant systems run into: one noisy tenant shouldn't be the reason every other tenant gets delayed. By leveraging the existing `MessageGroupId` field and a fairness algorithm running behind the scenes, this feature delivers tenant isolation at the queue layer itself, with no changes required on the consumer side.

**Reference link:** <https://aws.amazon.com/blogs/compute/building-resilient-multi-tenant-systems-with-amazon-sqs-fair-queues/>
---
title: "Blogs Posted"
date: 2026-07-31
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

Alongside the CloudHop RAG project, our team also prepared three technical blog posts for the **AWS Study Group VN** community. We used these posts as an opportunity to explore AWS topics beyond the services used directly in our project, so each article focuses on a different part of the AWS ecosystem.

Rather than simply summarizing documentation, we tried to understand each topic well enough to explain how it works, why it matters, and where it can be useful in practice. The three posts cover data and metadata management, cloud cost optimization, and distributed databases.

### [Blog 1 - Amazon S3 Annotations: Updatable, Queryable Metadata for Each Object](3.1-Blog1/)

The first post explores **Amazon S3 Annotations**, a feature for attaching richer and independently updatable metadata directly to S3 objects.

We focused on use cases such as transcripts, OCR results, AI-generated summaries, classification outputs, processing status, and other metadata commonly produced by data and AI pipelines. The article also compares annotations with object tags and existing S3 metadata, and looks at how annotations can later be queried through S3 Metadata.

### [Blog 2 - AWS Cost Optimization: Don't Just Look at the Bill](3.2-Blog2/)

The second post looks at **AWS cost optimization** from a broader engineering perspective.

Instead of treating optimization as simply reducing the monthly bill, we discuss the approach encouraged by the AWS Well-Architected Framework: understanding where cost comes from, relating spending to useful workload output, matching resources to actual demand, assigning clear ownership, and reviewing costs continuously.

This topic was also relevant to our internship work because the architecture of a cloud application should consider not only whether it works, but also whether its resources are being used reasonably.

### [Blog 3 - Amazon Aurora DSQL: When a Distributed Database No Longer Has to Trade Off Speed for Consistency](3.3-Blog3/)

The third post explores **Amazon Aurora DSQL** and the distributed-systems ideas behind its architecture.

We looked at how Aurora DSQL separates query processing, transaction coordination, journaling, and storage, as well as how Optimistic Concurrency Control supports strongly consistent distributed transactions. This article gave our team the chance to study an AWS service outside the direct scope of CloudHop RAG and learn more about the design of distributed database systems.

Together, the three posts helped us broaden our understanding of AWS beyond the project itself while practicing how to communicate technical topics in a clear and practical way.
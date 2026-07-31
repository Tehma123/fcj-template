---
title: "Blog 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.3. </b> "
---

# Amazon Aurora DSQL: When a Distributed Database No Longer Has to Trade Off Speed for Consistency

![Blog post published on the AWS Study Group VN Facebook group](/images/BlogsPosted/blog3.png)
*Posted to the AWS Study Group VN Facebook group.*

Building a distributed relational database across multiple Regions has traditionally forced teams into one of two paths: accept eventual consistency to get low latency, or synchronously replicate every commit across Regions to keep strong consistency — at the cost of very high write latency. Amazon Aurora DSQL, which reached General Availability in May 2025 and has kept shipping new features throughout 2026, was designed by AWS to break exactly that classic trade-off: low-latency reads from any Region, while writes are still validated for consistency globally.

## 1. A Disaggregated Architecture: Four Independent Components Instead of One Monolithic Instance

Unlike traditional database architectures where compute, storage, and transaction coordination are all bundled into a single instance, Aurora DSQL splits the whole system into independent components that each scale horizontally on their own.

- **Query Processor (QP):** Receives and executes PostgreSQL-compatible SQL statements, running inside Firecracker MicroVMs and holding no local state. Because it's stateless, a QP can be provisioned and torn down almost instantly based on real load — which is also why Aurora DSQL can scale to zero without a meaningful cold start, unlike Aurora Serverless v2, which still needs roughly 15 seconds to resume after being paused.
- **Adjudicator:** Handles conflict resolution at commit time. Each adjudicator owns a specific key range; when a transaction touches data spanning multiple key ranges, the relevant adjudicators coordinate with each other to decide whether the transaction is allowed to commit.
- **Journal:** An ordered stream where committed transactions are recorded to guarantee durability, and which is also used to confirm a successful write back to the client.
- **MVCC Storage Replicas:** This storage layer keeps data under a multi-version concurrency control model, allowing multiple versions of the data to coexist so read transactions can be served from different snapshot points in time without needing locks.

Because these four components are disaggregated and scale independently, Aurora DSQL has no notion of a "primary instance" the way traditional databases do — there's no single point of failure, and an outage in one Region or Availability Zone doesn't interrupt writes elsewhere.

## 2. Optimistic Concurrency Control: No Locking, Conflicts Are Only Checked at Commit Time

The mechanism at the core of how Aurora DSQL achieves strong consistency while keeping latency low is Optimistic Concurrency Control (OCC), which is fundamentally different from the row-level locking used by standard PostgreSQL.

Under OCC, a transaction executes in full without ever requesting a lock upfront — the system optimistically assumes that most transactions won't conflict with each other. All reads within a transaction use the timestamp from when the transaction started (Tstart), and conflict checking only happens at commit time (Tcommit): if no other transaction has committed a change to the same data between Tstart and Tcommit, the transaction is allowed to complete; if one has, the transaction is rejected with a serialization error (SQLSTATE 40001) and the application must retry it itself.

This approach completely eliminates deadlocks and the situation where one slow transaction blocks others — two classic problems of the traditional locking model. In exchange, applications built on Aurora DSQL need to be designed to handle retries as a normal part of the request flow, especially for repeated writes to the same row.

To compare Tstart and Tcommit accurately at multi-Region scale — where physical clocks across servers always drift slightly from one another — Aurora DSQL relies on the Amazon Time Sync Service combined with ClockBound, a mechanism that doesn't return a single absolute timestamp but rather a guaranteed interval containing the true time (e.g., [10:00:00.123456785, 10:00:00.123456793]). By representing time as an interval rather than a point, the system can reliably determine which event happened first, or whether two events could have occurred concurrently — the foundation that lets an adjudicator make an accurate commit decision without nodes having to continuously synchronize with each other over the network.

Aurora DSQL uses strong snapshot isolation, equivalent to PostgreSQL's repeatable read level — stricter than read committed but without the global coordination overhead that serializable isolation requires in some other distributed databases. Because read-only transactions use the snapshot at Tstart, they never need to queue and almost never hit an OCC error.

## 3. PostgreSQL-Compatible, But Not "a Distributed Version of Aurora PostgreSQL"

Aurora DSQL shares its parser, planner, optimizer, and type system with PostgreSQL 16, so most SQL syntax, data types, and arithmetic operations behave identically to standard PostgreSQL. However, this is a separate product architecturally, not just another operating mode of Aurora PostgreSQL, and there are several important differences to understand before designing a schema.

- **DDL runs asynchronously:** Statements like `CREATE TABLE` or `ALTER TABLE` execute as background tasks, allowing uninterrupted reads/writes while the schema is changing — unlike traditional PostgreSQL, where some DDL operations can lock a table. In exchange, if the schema catalog is updated by another transaction while a session is still working off a stale cached copy, that session can receive an OC001 error and needs to reload the catalog.
- **Key-ordered storage:** The primary key determines how data is physically distributed and how writes are spread across adjudicators, so the choice of primary key directly affects write performance at scale.
- **IAM-based authentication:** Instead of relying only on username/password like typical PostgreSQL, Aurora DSQL integrates authentication through AWS IAM, fitting naturally into a centralized identity-management model within the AWS ecosystem.

AWS has also been steadily narrowing the compatibility gap — for example, sequences and identity columns (`GENERATED ALWAYS AS IDENTITY`, `CREATE SEQUENCE`), one of the most frequently requested features, were added in early 2026, supporting up to 5,000 sequences per database.

## 4. Designing Schemas to Avoid Conflicts: The Most Important Lesson When Using OCC

Since OCC only detects conflicts at the row level, a key design principle on Aurora DSQL is avoiding combining independently-updated fields into a single row. For example, if an `account` table bundles a balance, a login counter, and a last-updated timestamp into the same row, then even though these three values are logically completely independent, any update to any one of them will serialize against the others because they share the same row — meaning concurrent update transactions will keep getting rejected and retried, even though there's no real business-level conflict.

In addition, the longer a transaction stays open and the more rows it touches, the higher the chance another transaction "beats it" to commit first, so applications that get the most out of Aurora DSQL tend to keep transactions short and focused. Concentrating too many writes on a single key or key range (a hot key / hot key range) also reduces the benefit of the distributed architecture, since it effectively piles load onto a single adjudicator.

## 5. When Aurora DSQL Is the Right Choice

An active-active, multi-Region architecture with strong consistency makes Aurora DSQL a good fit for real-time financial transaction systems, continuously-updated leaderboards, social networks, microservices systems, and high-availability SaaS platforms — anywhere an application needs to serve users across multiple geographic regions while guaranteeing every Region reads the latest data, with no single "primary" Region acting as a bottleneck.

On the other hand, Aurora DSQL isn't yet the optimal choice for applications that depend heavily on advanced PostgreSQL extensions or custom data types that aren't fully supported yet, nor for workloads characterized by continuous, heavy writes concentrated on the same row — in that case OCC will cause a retry rate higher than the benefit it provides.

Aurora DSQL is currently available in 14 AWS Regions worldwide, and is used by organizations such as ADP, Cintra, Caylent, DeNA, and Robinhood for systems that require strict business continuity, while AWS also offers an online sandbox experience (the DSQL Playground) to try SQL against a real database without needing an AWS account.

## Conclusion

Thank you for taking the time to read this post. I hope these notes help clarify how Aurora DSQL solves this classic distributed-systems trade-off, and give you a useful perspective to consider when evaluating it for your own systems.

**Reference links:**
- <https://aws.amazon.com/blogs/database/concurrency-control-in-amazon-aurora-dsql/>
- <https://aws.amazon.com/blogs/database/dsql-sql-dialect-how-amazon-aurora-dsql-differs-from-single-instance-postgresql/>
- <https://aws.amazon.com/blogs/database/building-scalable-applications-on-amazon-aurora-dsql/>
- <https://aws.amazon.com/blogs/database/everything-you-dont-need-to-know-about-amazon-aurora-dsql-part-2-shallow-view/>
- <https://aws.amazon.com/rds/aurora/dsql/>

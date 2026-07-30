---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Amazon S3 Annotations: Updatable, Queryable Metadata for Each Object

![Blog post published on the AWS Study Group VN Facebook group](/images/BlogsPosted/blog1.png)
*Posted to the AWS Study Group VN Facebook group.*

Amazon S3 already supports several kinds of metadata for describing and managing objects, such as size, storage class, object tags, and user-defined metadata for different management needs.

However, in data-processing and AI pipelines, an object is often accompanied by a lot of information generated after processing. A system may produce a transcript, OCR results, and a summary, while also extracting key facts such as people's names, organizations, dates, or document IDs. Models may also generate classification labels, embeddings, confidence scores, PII-detection flags, or content-moderation results. This metadata usually still has to be stored in a separate database or file.

This way of organizing things creates two problems: the application has to keep the metadata in sync with the object itself, and it also has to connect to multiple systems just to find the right data to use.

Launched by AWS in June 2026, Amazon S3 Annotations adds a new way to attach named pieces of metadata directly to each object. Each annotation can be updated independently and surfaced in S3 Metadata so it can be queried by analytics tools or AI agents.

## 1. What are S3 Annotations?

Each annotation is a piece of metadata identified by its own name and attached to a specific version of an S3 object. Its content can be plain text or structured data such as JSON, XML, and YAML.

For example, a video might come with the following annotations:

```
videos/documentary-2026.mp4
├── mediainfo
├── transcript
├── ai_summary
├── moderation_result
└── licensing
```

Here, `mediainfo` stores the codec and resolution, `transcript` stores the dialogue, `ai_summary` stores the AI-generated summary, `moderation_result` stores the moderation outcome, and `licensing` describes the distribution rights and their expiration.

Each annotation is managed independently. The pipeline that generates the transcript can update `transcript` without touching the video itself or affecting metadata created by other pipelines.

Each object version can have up to 1,000 annotations, with a maximum size of 1 MiB per annotation — up to 1 GiB of annotation metadata in total. This lets annotations hold far more detailed, structured processing output than object tags or user-defined metadata ever could.

The object holds the raw data. The annotation tells you what that data is, how it has been processed, and what state it's in.

## 2. So how are S3 Annotations different from existing tags and metadata?

S3 Annotations doesn't replace the existing metadata mechanisms. Each one is designed for a different purpose:

### System-defined metadata

Attributes that S3 manages automatically, such as size, storage class, and creation time. This type of metadata describes characteristics of the object but isn't meant for storing custom information from applications.

### User-defined metadata

Suited to a small amount of information defined at upload time. Its capacity is limited, and changing it usually requires the application to rewrite the object.

### Object tags

Suited to operational tasks such as access control, applying lifecycle rules, and cost allocation. Each object version can hold at most 10 short key-value pairs.

### S3 Annotations

Suited to detailed, structured metadata such as transcripts, OCR results, classification labels, processing status, or licensing information. Each object version can hold up to 1,000 annotations, and each one can be updated independently.

The difference can be remembered simply:

Tags mainly serve object management. Annotations give applications and AI the context to understand an object's content and state.

## 3. From per-object metadata to bucket-wide search

Attaching metadata to individual objects only solves part of the problem. When a bucket holds millions of objects, the system also has to find the right data based on its content and state.

When the annotation table is enabled in S3 Metadata, S3 automatically writes annotations into an AWS-managed Apache Iceberg table. That table can be queried with Amazon Athena and other Iceberg-compatible tools.

```
S3 object → annotations → S3 Metadata → Athena / Iceberg tools → application or AI agent
```

With this, a media company can find videos that have been moderated, have Vietnamese subtitles, and still hold distribution rights — without opening every file. Similarly, a document-processing system can find contracts that contain PII but haven't yet been approved.

An AI agent can also use this metadata layer to narrow its search before reading the underlying data. Instead of seeing only an opaque object name like `archive/file_00873142.pdf`, the agent can rely on annotations about document type, language, processing status, or sensitivity level.

A natural-language request such as:

> Find movies rated PG, with Spanish subtitles, released in 2023.

can be turned into a query over annotations, instead of having to connect and cross-reference several separate metadata systems.

## 4. When are S3 Annotations a good fit?

S3 Annotations is worth considering when the metadata has one or more of the following characteristics:

* it's tied directly to a specific object;
* it's larger than a few short key-value pairs;
* it's structured, such as JSON, XML, or YAML;
* it's updated by multiple pipelines over time;
* it needs to be searched or analyzed across many objects;
* it provides context for analytics tools or AI agents.

Some typical scenarios include:

* **AI-driven document processing:** storing OCR results, classification, entity extraction, PII detection, and approval status.
* **Digital content management:** storing transcripts, subtitles, moderation results, summaries, and licensing information.
* **Data pipeline tracking:** storing schema versions, processing status, quality-check results, and data lineage.

That said, annotations don't replace every metadata system. Object tags remain a better fit for access control, lifecycle rules, and cost allocation. A dedicated database is still needed when metadata involves complex relationships, requires strongly consistent transactions, or must be updated with very low latency.

## 5. Things to keep in mind

Before adopting this, there are four important points to consider:

* Annotations are tied to each object version. Different versions of the same object have independent annotations.
* The annotation table doesn't update instantly. S3 Metadata is better suited to data discovery and analytics than to real-time transactional tasks.
* Annotation storage is billed at S3 Standard rates. This applies even when the underlying object sits in S3 Glacier or another storage class.
* Maximum capacity isn't a usage target. Not all related data should be pushed into annotations — some content is still better stored as a separate object or in a dedicated database.

It's also worth carefully testing how annotations behave during object copies, with S3 Replication, or with versioning enabled — especially if your system requires the object and its metadata to always stay in sync.

## Conclusion

Amazon S3 Annotations lets you attach named pieces of metadata to each object, update them independently, and surface them in S3 Metadata for querying at scale. Instead of managing all of that context in an external system, an application can attach a transcript, processing result/status, or licensing information directly to the object it describes.

S3 Annotations doesn't replace object tags or databases. It fills the gap between the two: metadata that needs to be more detailed and queryable than tags, but still managed alongside the data in S3.

*Source: AWS – Amazon S3 User Guide, "Annotating your objects"*

**Reference link:** <https://docs.aws.amazon.com/AmazonS3/latest/userguide/annotations-overview.html>

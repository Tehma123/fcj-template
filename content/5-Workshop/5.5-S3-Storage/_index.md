---
title: "Amazon S3 Storage"
date: 2026-07-31
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

The retrieval artifacts created in the previous step need to remain available independently of the EC2 instance that runs the backend. CloudHop RAG therefore uses **Amazon S3** as the persistent store for the processed corpus, BM25 index, document mappings, and index manifests required by the online RAG pipeline.

Keeping these files in S3 allows the backend to download and load the required artifact version when it starts, without rebuilding the corpus or retrieval indexes on EC2.

Chapter 5.4 produced a folder of artifacts on a build machine. This chapter creates the S3 bucket that will hold them and uploads them into a layout that the backend can read without being told where anything is.

{{% notice info %}}
**What you will have at the end of this chapter:** a private, versioned, encrypted S3 bucket in `ap-southeast-1` containing the `rag/` tree, verified readable. The vectors are *not* handled here - they go to Amazon S3 Vectors in chapter 5.6.
{{% /notice %}}

---

## 1. Two buckets, two different services

This project uses two storage resources with confusingly similar names. They are not the same kind of thing:

| | `aws-rag-bucket-vanh1234` | `rag-vectors-vanh1234` |
| --- | --- | --- |
| Service | **Amazon S3** (general purpose) | **Amazon S3 Vectors** (vector store) |
| Holds | JSONL documents, BM25 index, manifests | 1024-dimension embeddings |
| Created in | **This chapter** | Chapter 5.6 |
| Accessed with | `s3` API - `GetObject`, `ListObjects` | `s3vectors` API - `PutVectors`, `QueryVectors` |

A vector bucket is created through a separate API and does not appear in the ordinary S3 bucket list. Do not try to create it here.

---

## 2. Pick the region first, and use it everywhere

Everything in this project lives in **`ap-southeast-1`**: this bucket, the vector bucket, the EC2 instance, the SSM parameters, the secret, and API Gateway.

This is not a preference. The EC2 backend reads artifacts from S3 on every cold start and queries S3 Vectors on every request. Placing storage in a different region than compute adds cross-region latency to the request path and turns every artifact download into billable inter-region data transfer. Same-region traffic between EC2 and S3 costs nothing.

{{% notice warning %}}
Choose the region now and do not change it later. The region is embedded in the SSM parameters, the API Gateway integration and the vector index - changing it midway means rebuilding several chapters' work.
{{% /notice %}}

---

## 3. Create the bucket

Bucket names are **globally unique across all AWS accounts**, lowercase, and cannot be renamed. Pick your own name - `aws-rag-bucket-vanh1234` is already taken by this project.

**Console:** S3 → Create bucket

| Field | Value |
| --- | --- |
| Bucket name | your own globally unique name |
| AWS Region | `Asia Pacific (Singapore) ap-southeast-1` |
| Object Ownership | ACLs disabled (recommended) |
| Block Public Access | **Block all public access - leave every box checked** |
| Bucket Versioning | **Enable** |
| Default encryption | **SSE-S3 (Amazon S3 managed keys)** |

**CLI:**

```bash
aws s3api create-bucket \
  --bucket aws-rag-bucket-vanh1234 \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1
```

Any region other than `us-east-1` requires the `--create-bucket-configuration` flag. Omitting it is the most common first error.

<!-- IMAGE 1 - SCREENSHOT.
     S3 Console -> Create bucket form, filled in, showing at minimum:
       - the bucket name field
       - AWS Region = Asia Pacific (Singapore) ap-southeast-1
       - the "Block all public access" checkbox TICKED
     This one screenshot evidences both the region decision and the security default. -->

![Creating the artifact bucket](/images/5-Workshop/5.5-S3-Storage/create-bucket.png)

---

## 4. Harden the bucket

Three settings, each for a specific reason. If you used the Console table above they are already applied; these are the CLI equivalents and the justification.

**Block Public Access** - nothing in this bucket is ever served to a browser. The frontend talks to API Gateway, and only the EC2 instance role reads these objects. There is no legitimate reason for a public object here, so the account-level guardrail stays fully on.

```bash
aws s3api put-public-access-block \
  --bucket aws-rag-bucket-vanh1234 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

**Versioning** - artifacts are immutable by convention (chapter 5.4 bumps the version on every corpus change), but versioning is the safety net for when convention fails. If someone re-runs a build with an identifier that production is currently serving, the previous objects still exist and can be restored.

```bash
aws s3api put-bucket-versioning \
  --bucket aws-rag-bucket-vanh1234 \
  --versioning-configuration Status=Enabled
```

**Default encryption** - encryption at rest with S3-managed keys. New buckets get SSE-S3 automatically; setting it explicitly makes the intent visible and survives a policy review.

```bash
aws s3api put-bucket-encryption \
  --bucket aws-rag-bucket-vanh1234 \
  --server-side-encryption-configuration \
    '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

{{% notice note %}}
SSE-KMS with a customer-managed key would give per-key audit trails and separate key permissions, at the cost of KMS charges per request and an extra `kms:Decrypt` grant on the EC2 role. For a public-dataset demo, SSE-S3 is the proportionate choice. For a real corpus containing internal documents, SSE-KMS is the right upgrade.
{{% /notice %}}

<!-- IMAGE 2 - SCREENSHOT.
     S3 Console -> your bucket -> Properties tab.
     Scroll so that BOTH of these are visible in one shot:
       - "Bucket Versioning: Enabled"
       - "Default encryption: Server-side encryption with Amazon S3 managed keys (SSE-S3)"
     If they do not fit in one screen, take two and put them side by side. -->

![Versioning and default encryption enabled](/images/5-Workshop/5.5-S3-Storage/bucket-properties.png)

---

## 5. The key layout is a contract

The prefix structure is not organisational tidiness - the backend builds S3 keys from it at startup. Getting a prefix wrong produces a service that starts cleanly and then fails on the first query.

```text
s3://aws-rag-bucket-vanh1234/
  rag/
    corpora/hotpotqa/validation-500/v002/
      corpus.jsonl
      eval.jsonl
      corpus_manifest.json
    processed/hotpotqa-val500-v002/
      parent_docs.jsonl
      child_docs.jsonl
      child_to_parent.json
    indexes/hotpotqa-val500-bge-m3-v002/
      bm25/
        bm25_index/
        bm25_doc_ids.json
      manifests/
        index_manifest.json
      s3vectors-import/
        put_vectors_0000.json
        ...
```

Three identifiers govern the whole tree, and they come straight from chapter 5.4:

| Segment | Comes from | Deployed value |
| --- | --- | --- |
| `rag/` | `S3_ARTIFACT_PREFIX` (default `rag`) | `rag` |
| `corpora/<corpus id>/` | `CORPUS_ID` | `hotpotqa/validation-500/v002` |
| `processed/<processed id>/` | `PROCESSED_ID` | `hotpotqa-val500-v002` |
| `indexes/<index id>/` | `INDEX_ID` | `hotpotqa-val500-bge-m3-v002` |

Note that **`processed` and `indexes` are versioned separately**. That is deliberate: the same processed documents can back several indexes built with different embedding models, without duplicating the text.

---

## 6. Upload

The notebook already arranged everything into a tree that mirrors the layout above, so the upload is one recursive copy - no path juggling, no chance of misplacing a prefix.

```bash
aws s3 cp "s3_manual_upload/hotpotqa-val500-bge-m3-v002/rag" \
  "s3://aws-rag-bucket-vanh1234/rag/" \
  --recursive \
  --region ap-southeast-1
```

`aws s3 sync` also works and is better for re-uploads, since it only transfers changed objects:

```bash
aws s3 sync "s3_manual_upload/hotpotqa-val500-bge-m3-v002/rag" \
  "s3://aws-rag-bucket-vanh1234/rag" \
  --region ap-southeast-1
```

{{% notice tip %}}
On Windows, quote the local path - the notebook output folder usually sits under a Google Drive path containing spaces. In PowerShell use backtick `` ` `` for line continuation, not backslash.
{{% /notice %}}

<!-- IMAGE 3 - SCREENSHOT.
     Your terminal after the upload finishes, showing several "upload: ... to s3://..." lines
     and the final completed state.
     Make sure at least one line for each of corpora/, processed/ and indexes/ is visible,
     so the reader can see all three prefixes were written. -->

![Uploading the artifact tree to S3](/images/5-Workshop/5.5-S3-Storage/upload-output.png)

---

## 7. Verify

List what landed:

```bash
aws s3 ls s3://aws-rag-bucket-vanh1234/rag/ --recursive --region ap-southeast-1
```

Then read the manifest back - this is the single most useful check, because it is the exact object the backend fetches first:

```bash
aws s3 cp \
  s3://aws-rag-bucket-vanh1234/rag/indexes/hotpotqa-val500-bge-m3-v002/manifests/index_manifest.json - \
  --region ap-southeast-1
```

You should see the JSON from chapter 5.4, including `params.embedding_model` and `source.processed_id`. If this command fails, nothing downstream will work.

Minimum objects that must be present before moving on:

```text
rag/processed/<processed id>/parent_docs.jsonl
rag/processed/<processed id>/child_docs.jsonl
rag/processed/<processed id>/child_to_parent.json
rag/indexes/<index id>/bm25/bm25_doc_ids.json
rag/indexes/<index id>/bm25/bm25_index/
rag/indexes/<index id>/manifests/index_manifest.json
```

<!-- IMAGE 4 - SCREENSHOT.
     S3 Console object browser, inside your bucket, with the rag/ prefix opened and the
     corpora/ processed/ indexes/ folders visible.
     Keep the breadcrumb in shot - it evidences the bucket name. -->

![The rag/ prefix in the S3 console](/images/5-Workshop/5.5-S3-Storage/s3-console-tree.png)

One level deeper, `indexes/<index id>/` holds the three prefixes the backend reads at startup:

<!-- IMAGE 5 - SCREENSHOT.
     Same object browser, drilled into rag/indexes/<index id>/, showing bm25/,
     manifests/ and s3vectors-import/.
     Keep the breadcrumb in shot - it evidences the index id. -->

![Inside the index prefix](/images/5-Workshop/5.5-S3-Storage/s3-console-index.png)

---

## 8. What the backend actually reads

Worth being precise about, because it explains why the layout matters and why one prefix is uploaded but never read.

At startup the online loader performs exactly three S3 operations for the `s3vectors` layout:

1. **Downloads the manifest first** - `rag/indexes/<index id>/manifests/index_manifest.json`.
2. **Reads `source.processed_id` out of that manifest** to discover where the documents live. The processed id is not hard-coded in the application; if the SSM parameter is unset, the manifest supplies it.
3. **Downloads two prefixes** - `rag/processed/<processed id>/` and `rag/indexes/<index id>/bm25/`.

```python
manifest_key = f"{prefix}/indexes/{index_id}/manifests/index_manifest.json"
_download_file(bucket, manifest_key, manifest_path)

manifest = _load_json(manifest_path)
processed_id = processed_id or manifest.get("source", {}).get("processed_id", "")

_download_prefix(bucket, f"{prefix}/processed/{processed_id}/", ...)
_download_prefix(bucket, f"{prefix}/indexes/{index_id}/bm25/", ...)
```

What it never downloads:

- **`rag/corpora/`** - the raw corpus is only needed for rebuilding and for evaluation, never for serving.
- **`rag/indexes/<index id>/s3vectors-import/`** - those batches are consumed once, by the ingest in chapter 5.6. They are kept in S3 anyway, because if the vector index is ever lost or has to be recreated, re-ingesting from these files takes minutes and costs nothing, whereas re-embedding the corpus means standing the GPU environment back up.

So the online service downloads a few hundred megabytes at most, once per restart, and nothing at all per request.

---

## 9. Permissions

Two different principals touch this bucket, and they need different rights.

**The build machine** (you, uploading) needs write access:

```text
s3:ListBucket
s3:GetObject
s3:PutObject
s3:AbortMultipartUpload
```

Add `s3:DeleteObject` and `s3:DeleteObjectVersion` only if you need to clean up a mistaken upload - with versioning enabled, deleting also means deleting old versions.

**The EC2 instance role** (`rag-ec2-runtime-role`, created in chapter 5.7) needs **read only**, scoped to this bucket and this prefix:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListArtifactPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::aws-rag-bucket-vanh1234",
      "Condition": { "StringLike": { "s3:prefix": ["rag/*"] } }
    },
    {
      "Sid": "ReadArtifacts",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::aws-rag-bucket-vanh1234/rag/*"
    }
  ]
}
```

The serving role has **no write permission at all**. The online service is architecturally read-only against storage - it cannot corrupt an index even if the application has a bug, and that guarantee is enforced by IAM rather than by discipline.

No bucket policy is attached. Access is granted entirely through IAM identity policies, and the bucket stays private.

---

## 10. Cost

At this scale storage cost is effectively noise - a corpus of 4,937 documents with its indexes is a few hundred megabytes, well inside single-digit cents per month. Two things are worth setting up anyway, because they scale badly if ignored:

- **Versioning keeps every overwritten object forever** unless you tell it otherwise. Add a lifecycle rule to expire non-current versions after 30 days once you start iterating on indexes.
- **Same-region access keeps transfer free.** Downloading artifacts to an EC2 instance in another region would be billed per gigabyte on every restart.

Full cost breakdown is in chapter 5.13, and removal is in 5.14.

---

## 11. Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| `IllegalLocationConstraintException` on create | Missing `--create-bucket-configuration` outside `us-east-1` | Add the flag with your region |
| `BucketAlreadyExists` | Bucket names are globally unique | Choose a different name |
| `AccessDenied` on upload | Your CLI identity lacks `s3:PutObject` | Check `aws sts get-caller-identity`, then the attached policy |
| Backend starts, first query fails with `FileNotFoundError` | Objects uploaded under the wrong prefix | Compare `aws s3 ls --recursive` against the tree in section 5 |
| `No S3 objects found under s3://.../processed/...` | `processed id` and `index id` were mixed up | They differ - `hotpotqa-val500-v002` vs `hotpotqa-val500-bge-m3-v002` |
| Upload path breaks on Windows | Spaces in the Google Drive path | Quote the whole path |

---

## 12. Result

The artifact bucket is private, versioned, encrypted, and holds the complete `rag/` tree in the exact layout the backend expects. Nothing can read it except identities you grant IAM permissions to.

Chapter 5.6 creates the vector bucket and index, then runs `ingest_s3vectors.py` against the `s3vectors-import/` batches that are now sitting in this bucket.

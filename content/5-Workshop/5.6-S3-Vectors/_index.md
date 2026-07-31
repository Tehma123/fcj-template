---
title: "Amazon S3 Vectors"
date: 2026-07-31
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

BM25 is effective when the wording of a question overlaps strongly with the supporting document, but multi-hop questions may also require evidence expressed in different terms. CloudHop RAG therefore combines lexical retrieval with dense semantic retrieval using **Amazon S3 Vectors**.

Each child chunk produced during the offline build is encoded with **BGE-M3** into a 1,024-dimensional vector. These vectors are stored in an S3 Vectors index and queried by the EC2 backend at runtime using the embedding of the incoming question or retrieval query.

Chapter 5.5 uploaded the text side of the index. This chapter creates the vector store, ingests the embeddings produced in chapter 5.4, and proves that semantic retrieval works - all before any backend exists.

{{% notice info %}}
**What you will have at the end of this chapter:** a vector bucket containing an index of 1024-dimension embeddings, populated from the `put_vectors_*.json` batches, and a passing retrieval test run from your own machine. After this, both halves of the hybrid retriever have data.
{{% /notice %}}

---

## 1. What Amazon S3 Vectors is, and what it is not

S3 Vectors is a separate service that happens to share the S3 name. It is **not** a bucket with vectors in it.

| | Amazon S3 | Amazon S3 Vectors |
| --- | --- | --- |
| API namespace | `s3` | `s3vectors` |
| Unit of storage | Object under a key | Vector under a key, inside an *index* |
| Query | `GetObject` by exact key | `QueryVectors` - approximate nearest neighbour |
| Appears in | The S3 bucket list | A separate **Vector buckets** section |
| Created here | Chapter 5.5 | This chapter |

The resource hierarchy is two levels: a **vector bucket** contains one or more **indexes**, and vectors live inside an index. The index - not the bucket - is what carries the dimension and distance metric, which is why one bucket can hold several indexes built with different embedding models.

{{% notice warning %}}
Availability is per-region and narrower than ordinary S3. Confirm that S3 Vectors exists in your chosen region before committing to it - this project uses `ap-southeast-1`. If the `s3vectors` commands are not recognised, update the AWS CLI; the API version used here is `2025-07-15`.
{{% /notice %}}

---

## 2. Create the vector bucket

**Console:** S3 → **Vector buckets** → Create vector bucket

**CLI:**

```bash
aws s3vectors create-vector-bucket \
  --vector-bucket-name rag-vectors-vanh1234 \
  --region ap-southeast-1
```

Only the name is required. Encryption defaults to **SSE-S3 (`AES256`)**, the same posture as the artifact bucket in 5.5; passing an `encryptionConfiguration` with a `kmsKeyArn` switches it to SSE-KMS if your corpus warrants it.

Bucket names follow the same rules as before - lowercase, globally unique within the service, and not renameable.

<!-- IMAGE 1 - SCREENSHOT.
     S3 Console -> "Vector buckets" section -> Create vector bucket form (or the created
     bucket in the list).
     Important: make the LEFT NAVIGATION visible, showing that "Vector buckets" is a
     separate entry from "General purpose buckets". That single detail answers the
     question most readers have at this point. -->

![Creating the vector bucket](/images/5-Workshop/5.6-S3-Vectors/create-vector-bucket.png)

---

## 3. Create the index

This is the step where a wrong value costs you a rebuild, because **an index's dimension and distance metric are fixed at creation**.

```bash
aws s3vectors create-index \
  --vector-bucket-name rag-vectors-vanh1234 \
  --index-name hotpotqa-val500-bge-m3-v002 \
  --data-type float32 \
  --dimension 1024 \
  --distance-metric cosine \
  --region ap-southeast-1
```

Every parameter, and where its value comes from:

| Parameter | Value | Why |
| --- | --- | --- |
| `--index-name` | `hotpotqa-val500-bge-m3-v002` | Exactly the `INDEX_ID` from chapter 5.4. The backend passes one identifier as both the S3 index prefix and the vector index name - they must be identical. |
| `--data-type` | `float32` | The only value the API accepts, and what the notebook wrote: `np.asarray(vectors, dtype='float32')`. |
| `--dimension` | `1024` | The output width of `BAAI/bge-m3`. The API allows 1–4096. This must match the embedding model exactly - it is the same number the notebook asserted on. |
| `--distance-metric` | `cosine` | Must match how the vectors were built. Chapter 5.4 used `normalize_embeddings=True`, and cosine is the metric that pairs with unit-normalised vectors. The alternative is `euclidean`. |

{{% notice warning %}}
**Dimension and metric cannot be changed later.** If you create the index with the wrong dimension, `PutVectors` fails with a `ValidationException` on the first batch - which is the good outcome. The bad outcome is creating it with `euclidean` while your vectors are normalised: ingest succeeds, queries succeed, and the ranking is quietly wrong. Check this value now, not after the evaluation in chapter 5.11 comes back mediocre.
{{% /notice %}}

**Optional - non-filterable metadata keys.** `create-index` accepts a `metadataConfiguration` listing up to 10 metadata keys that are stored and returned but cannot be used as query filters:

```bash
  --metadata-configuration nonFilterableMetadataKeys=title,source_doc_id
```

This project does not set it. Section 6 explains why: the retriever never reads metadata back from the vector store.

<!-- IMAGE 2 - SCREENSHOT.
     The Create index form (or the index detail page after creation), showing all four
     settings legibly: index name, data type float32, dimension 1024, distance metric cosine.
     This is the highest-value screenshot in the chapter - these four values are the
     contract between chapters 5.4 and 5.7. -->

![Creating the vector index](/images/5-Workshop/5.6-S3-Vectors/create-index.png)

---

## 4. Ingest the vectors

Chapter 5.4 generated `ingest_s3vectors.py` with your bucket, index and region already baked in. Run it from inside the upload folder:

```bash
cd s3_manual_upload/hotpotqa-val500-bge-m3-v002
python ingest_s3vectors.py --region ap-southeast-1
```

It walks the batch files in order and issues one `PutVectors` call per file:

```python
client = boto3.client("s3vectors", region_name=args.region)

for path in sorted(import_dir.glob("put_vectors_*.json")):
    payload = json.loads(path.read_text(encoding="utf-8"))
    client.put_vectors(
        vectorBucketName=args.vector_bucket_name,
        indexName=args.index_name,
        vectors=payload["vectors"],
    )
```

Expected output:

```text
uploaded 500 vectors from put_vectors_0000.json
uploaded 500 vectors from put_vectors_0001.json
...
uploaded 279 vectors from put_vectors_0016.json
done: 8279 vectors uploaded
```

{{% notice note %}}
**Why 500 per batch is not an arbitrary choice.** The `PutVectors` API accepts a maximum of **500 vectors per request** - it is a hard service limit, not a tuning knob. Chapter 5.4 batched at exactly that number to use full requests without ever exceeding the cap. The practical benefit is that ingest is resumable: if a call fails halfway through, you re-run from the batch that failed rather than re-embedding the corpus.
{{% /notice %}}

`PutVectors` is idempotent on the key - re-running a batch overwrites the same vectors rather than duplicating them, because `key` (the `child_id`) uniquely identifies a vector within an index. A failed ingest can simply be re-run from the start.

<!-- IMAGE 3 - SCREENSHOT.
     Terminal running ingest_s3vectors.py, showing several "uploaded 500 vectors from
     put_vectors_XXXX.json" lines and the final "done: N vectors uploaded" line.
     Make the final total legible - it should equal the "child docs" count printed by
     the validation cell in chapter 5.4. -->

![Ingesting vectors into the index](/images/5-Workshop/5.6-S3-Vectors/ingest-output.png)

---

## 5. Verify

**Check the index configuration** - confirm the four values you just committed to:

```bash
aws s3vectors get-index \
  --vector-bucket-name rag-vectors-vanh1234 \
  --index-name hotpotqa-val500-bge-m3-v002 \
  --region ap-southeast-1
```

**Check that vectors landed:**

```bash
aws s3vectors list-vectors \
  --vector-bucket-name rag-vectors-vanh1234 \
  --index-name hotpotqa-val500-bge-m3-v002 \
  --max-results 5 \
  --region ap-southeast-1
```

The keys returned should be `child_id` values matching the ones in `child_docs.jsonl`. `--max-results` accepts up to 1000.

**Then do the test that actually matters.** Listing vectors proves data exists; it does not prove retrieval works. The project ships a script that encodes a real question, queries the index and prints the documents that come back - with no Groq call and no backend involved:

```bash
cd backend
python scripts/check_s3vectors_retrieval.py \
  --download-artifacts \
  --device cpu \
  "Were Scott Derrickson and Ed Wood of the same nationality?"
```

This is the first end-to-end proof that chapters 5.4, 5.5 and 5.6 agree with each other: it downloads the artifacts from S3, loads `child_docs.jsonl`, encodes the query with `BAAI/bge-m3`, queries S3 Vectors, and maps the returned keys back to real documents. If the retrieved titles are topically relevant, all three chapters are correct.

{{% notice tip %}}
If this returns documents that have nothing to do with the question, the cause is almost always one of three things, in order of likelihood: the index was built with a different embedding model than the one encoding the query, the distance metric does not match how the vectors were normalised, or the `child_docs.jsonl` in S3 came from a different build than the vectors. All three are configuration mismatches, not retrieval-quality problems - do not start tuning `top-k`.
{{% /notice %}}

<!-- IMAGE 4 - SCREENSHOT.
     Terminal output of check_s3vectors_retrieval.py, showing the query and the
     retrieved documents with their titles.
     Capture enough lines that a reader can see the titles are topically relevant to
     the question - that relevance is the actual evidence, not the fact it ran. -->

![Dense retrieval working against the index](/images/5-Workshop/5.6-S3-Vectors/retrieval-check.png)

---

## 6. How the backend uses this index

Worth being precise, because the design here is less obvious than it looks. The online retriever queries S3 Vectors like this:

```python
response = self._get_client().query_vectors(
    vectorBucketName=self.vector_bucket_name,
    indexName=self.index_name,
    queryVector={"float32": self._encode_query(query)},
    topK=min(self.k, len(self.docs_by_id)),
    returnDistance=self.return_distance,
)

for item in response.get("vectors", []):
    child_id = item.get("key")
    doc = self.docs_by_id.get(child_id)
    ...
```

Note what it takes from the response: **the `key` and the `distance`, and nothing else.** The document text is not stored in the vector index - it comes from `child_docs.jsonl`, which was downloaded from ordinary S3 at startup and is already in memory, keyed by `child_id`.

That split is deliberate and has three consequences:

- **The vector index stores no text.** It holds 1024 floats and a short key per chunk. Storage stays small and cheap, and no document content is duplicated across two services.
- **`returnMetadata` is never requested.** The metadata written during ingest (`parent_id`, `title`, `source_doc_id`) is effectively insurance - useful for inspecting the index by hand or for a future filtered query, but not on the hot path. This is why `nonFilterableMetadataKeys` was left unset in section 3.
- **The vector store is purely a similarity index.** It answers "which chunk ids are closest to this query vector", and everything else is resolved locally. Swapping S3 Vectors for another vector database would mean rewriting one module - `s3vectors_retriever.py` - and nothing else.

The query vector is encoded with the same normalisation used at build time:

```python
vector = self._get_model().encode(query, normalize_embeddings=True, convert_to_numpy=True)
```

Build-time and query-time normalisation must agree, and both must agree with the index's distance metric. That three-way agreement is the single most important invariant in this chapter.

---

## 7. Permissions

Two principals, two very different levels of access.

**You, during this chapter** - creating and populating:

```text
s3vectors:CreateVectorBucket
s3vectors:CreateIndex
s3vectors:PutVectors
s3vectors:GetIndex
s3vectors:ListVectors
```

**The EC2 instance role** (`rag-ec2-runtime-role`, chapter 5.7) - querying only:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "QueryVectorIndex",
      "Effect": "Allow",
      "Action": [
        "s3vectors:QueryVectors",
        "s3vectors:GetIndex"
      ],
      "Resource": "arn:aws:s3vectors:ap-southeast-1:<account-id>:bucket/rag-vectors-vanh1234/index/hotpotqa-val500-bge-m3-v002"
    }
  ]
}
```

Index ARNs take the form `arn:aws:s3vectors:<region>:<account-id>:bucket/<bucket>/index/<index>`, so the policy can be scoped to a single index rather than the whole bucket.

The serving role gets **no `PutVectors` and no `DeleteVectors`**. Exactly as with S3 in chapter 5.5, the running service is read-only against its own data by IAM enforcement, not by convention. A bug in the application cannot corrupt the index.

---

## 8. Cost and limits

The numbers that constrain design decisions:

| Limit | Value |
| --- | --- |
| Vectors per `PutVectors` request | 500 |
| Index dimension | 1 – 4096 |
| Data type | `float32` only |
| Distance metrics | `cosine`, `euclidean` |
| Non-filterable metadata keys per index | up to 10 |
| `list-vectors` page size | up to 1000 |

Cost has two components - storage of the vectors, and requests. This corpus produces 8,279 vectors of 1,024 dimensions, which is negligible storage, and the demo issues one `QueryVectors` call per retrieval. The important property for a student project is that **there is no idle cost**: unlike a provisioned vector database or OpenSearch Serverless, nothing is billed while no queries are running. Chapter 5.13 covers the full picture.

---

## 9. Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| `ValidationException` on the first `PutVectors` | Vector length does not match the index dimension | The index is fixed - delete and recreate it with the right dimension |
| `ConflictException` on create | An index or bucket with that name already exists | Use the existing one, or pick a new `INDEX_ID` |
| `NotFoundException` on ingest | Index created in a different region, or a name typo | Check the region flag and that the index name equals `INDEX_ID` exactly |
| `AccessDeniedException` | Missing `s3vectors:*` permissions on your CLI identity | S3 permissions do not grant S3 Vectors permissions - they are separate actions |
| `aws: error: argument command: Invalid choice: 's3vectors'` | AWS CLI too old | Upgrade the CLI |
| Ingest fails partway through | Transient error mid-batch | Re-run the script; `PutVectors` overwrites by key, so it is safe |
| Retrieval returns irrelevant documents | Model / metric / artifact mismatch | See the tip in section 5 |

---

## 10. Result

Both halves of the retriever now have data: the BM25 index and the documents in ordinary S3 from chapter 5.5, and the dense embeddings in an S3 Vectors index here. Retrieval has been verified from a laptop, with no server involved.

Chapter 5.7 deploys the FastAPI backend on EC2 and gives it the instance role that ties these permissions together.

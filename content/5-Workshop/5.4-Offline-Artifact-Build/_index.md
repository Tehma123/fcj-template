---
title: "Offline Artifact Build"
date: 2026-07-31
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

Before the RAG backend can answer questions, the source data must be converted into retrieval-ready artifacts. This work is performed offline so that document processing, chunking, embedding generation, and index construction do not need to happen during a user request.

CloudHop RAG uses the **HotpotQA Distractor** dataset as the benchmark source for this process. HotpotQA is designed for multi-hop question answering, where the information needed for an answer may be distributed across several supporting documents. Its annotated questions, answers, contexts, and supporting facts make it suitable for building and evaluating the retrieval pipeline used in this project.

The final artifact build uses **500 questions from the validation split**. Their contexts are normalized into the project corpus format before the parent documents, child chunks, BM25 index, and BGE-M3 embeddings are generated.

This is the first build chapter. Here we produce **every file the deployed backend will ever need**, before any AWS resource exists. Chapter 5.3 explained *why* this work is separated from the request path; this chapter does it.

{{% notice info %}}
**What you will have at the end of this chapter:** a single local folder, ready to upload, containing the corpus, the parent/child documents, the BM25 index, the vector import batches, and a manifest that ties them together. Nothing is uploaded yet - chapter 5.5 creates the S3 bucket and uploads it, chapter 5.6 creates the vector index and ingests the vectors.
{{% /notice %}}

---

## 1. Where to run this, and why not on the target instance

Embedding a corpus is the only genuinely compute-heavy part of the whole project. It is done **once**, on a machine chosen for that job, not on the EC2 instance that will serve traffic.

| Option | Verdict |
| --- | --- |
| **Google Colab with GPU** | **Chosen.** Free T4 GPU, embedding finishes in minutes, no AWS cost at all for the build. |
| Local CPU workstation | Works, but `BAAI/bge-m3` on CPU is slow for anything beyond a small demo corpus. |
| SageMaker Studio CPU Space | **Tried and abandoned.** The process was terminated with `Killed` while loading `BAAI/bge-m3` - the Space ran out of RAM. |
| SageMaker GPU Space (`ml.g4dn.xlarge`) | **Blocked.** The account quota for JupyterLab Apps on `ml.g4dn.xlarge` was `0`, so the Space could not start. |
| Build on the serving EC2 instance | Rejected by design - it would put a multi-minute CPU job on the machine that must answer requests in seconds. |

{{% notice tip %}}
This is a real constraint worth planning around, not a footnote. A large embedding model needs several GB of RAM just to load. If your build environment gets `Killed` with no traceback, that is the OS out-of-memory killer, not a bug in your code. Either move to a GPU runtime or switch to a smaller model such as `BAAI/bge-small-en-v1.5`.
{{% /notice %}}

The canonical build is the notebook:

```text
backend/notebooks/build_s3_offline_artifacts.ipynb
```

<!-- IMAGE 1 - SCREENSHOT.
     Google Colab -> Runtime -> Change runtime type dialog, with "T4 GPU" selected.
     This evidences the environment decision explained in the table above.
     If you built on CPU instead, screenshot that dialog instead and say so honestly. -->

![Selecting a GPU runtime in Colab](/images/5-Workshop/5.4-Offline-Artifact-Build/colab-gpu-runtime.png)

---

## 2. The notebook is dataset-agnostic

HotpotQA is only a **sample source adapter** here. The notebook accepts any documents - internal policies, manuals, support tickets, wiki pages, Markdown, CSV - as long as they are normalised into the project's canonical schema first.

Corpus schema (one JSON object per line, `corpus.jsonl`):

```json
{"id":"doc_001","title":"Document title","text":"Full document or passage text","metadata":{"source":"optional"}}
```

Optional evaluation schema (`eval.jsonl`, used later in chapter 5.11):

```json
{"id":"q_001","question":"Question text","answer":"Expected answer","supporting_doc_ids":["doc_001"]}
```

The notebook has two modes, set by one variable:

| `SOURCE_MODE` | Use when |
| --- | --- |
| `hotpotqa_sample` | You want the demo corpus built automatically from HotpotQA |
| `custom_jsonl` | You already have your own `corpus.jsonl` (and optionally `eval.jsonl`) |

This matters for a real deployment: **the entire pipeline downstream of this notebook never learns what the source was**. Swapping HotpotQA for a company knowledge base changes this one variable and nothing else.

---

## 3. Configure the build

All configuration sits in a single cell near the top. These are the values that produced the deployed index:

```python
S3_BUCKET        = 'aws-rag-bucket-vanh1234'
S3_PREFIX        = 'rag'
AWS_REGION       = 'ap-southeast-1'
S3_VECTOR_BUCKET = 'rag-vectors-vanh1234'

SOURCE_MODE      = 'hotpotqa_sample'
DATASET_SUBSET   = 'distractor'  # guarantees gold-title coverage
DATASET_SIZE     = 500          # 100 / 500 / 1000, or None for the full split
HOTPOTQA_VERSION = 'v002'       # bump whenever the corpus content changes

EMBEDDING_MODEL_NAME   = 'BAAI/bge-m3'
VECTOR_DIMENSION       = 1024
VECTOR_DISTANCE_METRIC = 'cosine'

CHILD_CHUNK_SIZE       = 500
CHILD_CHUNK_OVERLAP    = 100
EMBEDDING_BATCH_SIZE   = 64
VECTOR_IMPORT_BATCH_SIZE = 500
```

The three artifact identifiers are **derived**, never typed by hand:

```python
CORPUS_ID    = f'hotpotqa/validation-{DATASET_SIZE_LABEL}/{HOTPOTQA_VERSION}'
PROCESSED_ID = f'hotpotqa-val{DATASET_SIZE_LABEL}-{HOTPOTQA_VERSION}'
INDEX_ID     = f'hotpotqa-val{DATASET_SIZE_LABEL}-bge-m3-{HOTPOTQA_VERSION}'
```

With `DATASET_SIZE = 500` and `HOTPOTQA_VERSION = 'v002'` this yields exactly the identifiers used for the final build:

```text
corpus id    : hotpotqa/validation-500/v002
processed id : hotpotqa-val500-v002
index id     : hotpotqa-val500-bge-m3-v002
```

{{% notice warning %}}
**Always bump the version when the corpus changes.** Every identifier carries a version suffix, and every S3 path is built from those identifiers. Reusing an identifier overwrites an index that production may still be serving. Bumping it means the old and new index sit side by side in S3, and switching between them is a parameter change (chapter 5.7) that can be reverted instantly if the new index is worse.
{{% /notice %}}

Note that the embedding model name is baked into `INDEX_ID`. That is deliberate: an index built with one model **cannot** be queried with another, because the vectors live in different spaces. Putting `bge-m3` in the name makes an incompatible pairing visible before it becomes a debugging session.

<!-- IMAGE 2 - SCREENSHOT.
     The output of the configuration cell in the notebook, showing the printed lines:
       source mode: / device: / corpus id: / processed id: / index id: /
       artifact root: / manual upload root:
     Capture the cell code above it too if it fits, so the reader sees input and output together.
     Crop out the Drive path if it contains a personal folder name. -->

![Notebook configuration output](/images/5-Workshop/5.4-Offline-Artifact-Build/notebook-config-output.png)

---

## 4. Step 1 - Normalise the source into `corpus.jsonl` and `eval.jsonl`

The adapter reads the HotpotQA validation split, flattens each question's supporting paragraphs into individual documents, cleans whitespace, and writes the two canonical files plus a `corpus_manifest.json`.

One detail that is easy to miss and expensive to get wrong: the notebook runs a **gold-title coverage guard**. Every evaluation question must have *all* of its supporting document titles present in the corpus. If a question's evidence is missing, the question is unanswerable no matter how good the retriever is, and the evaluation in chapter 5.11 would be measuring the corpus, not the system.

Output of this step:

```text
corpus.jsonl           one line per document
eval.jsonl             one line per evaluation question
corpus_manifest.json   row counts and checksums
```

---

## 5. Step 2 - Build parent and child documents

This implements the **small-to-big** pattern described in chapter 5.3.

```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=CHILD_CHUNK_SIZE,        # 500
    chunk_overlap=CHILD_CHUNK_OVERLAP,  # 100
)
```

- Each source document becomes **one parent document** - the whole article, kept intact. This is what the LLM eventually reads.
- Each parent is split into **several child documents** of ~500 characters with 100 characters of overlap. These are the units that get indexed and matched.
- A `child_to_parent.json` map records which parent each child came from.

Why two levels rather than one: a 500-character chunk is precise enough for retrieval to match on, but too small to answer from - it often cuts a sentence in half. A full article is good context but too diffuse to match against. Indexing the small chunks and *serving* the large ones gives both. The overlap of 100 characters prevents a fact that straddles a chunk boundary from being lost.

Output:

```text
parent_docs.jsonl      full articles
child_docs.jsonl       retrieval chunks, each carrying child_id / parent_id / title
child_to_parent.json   child_id -> parent_id map
```

---

## 6. Step 3 - Build the BM25 index

```python
import bm25s
```

BM25 is a classical lexical ranking function - it scores documents by term overlap with the query. It is built over **exactly the same child documents** that will be embedded in the next step, so the two retrievers rank the same candidate pool and their results can be merged fairly.

The index is written together with `bm25_doc_ids.json`, which maps BM25's internal integer positions back to real `child_id` values. Without that file the index is meaningless at query time.

{{% notice note %}}
BM25 is not a legacy fallback here - it is half the retrieval strategy. It handles exact names, rare terms, numbers and identifiers, which is precisely where dense embeddings are weakest. Chapter 5.3 section 4 explains how the two are merged.
{{% /notice %}}

---

## 7. Step 4 - Embed the child documents and build the vector import batches

This is the expensive step and the reason for the GPU runtime.

```python
model = SentenceTransformer(EMBEDDING_MODEL_NAME, device=DEVICE)

vectors = model.encode(
    texts,
    batch_size=EMBEDDING_BATCH_SIZE,
    normalize_embeddings=True,
    convert_to_numpy=True,
    show_progress_bar=True,
)
vectors = np.asarray(vectors, dtype='float32')
assert vectors.shape[1] == VECTOR_DIMENSION
```

Two details that matter:

- **`normalize_embeddings=True`** - the vectors are unit-normalised, which is what makes the `cosine` distance metric configured on the S3 Vectors index correct. Normalising at build time and choosing cosine at index time must agree.
- **The dimension assertion** - `BAAI/bge-m3` produces 1024-dimensional vectors. The S3 Vectors index is created with a fixed dimension (chapter 5.6). Asserting here turns a silent mismatch into an immediate, obvious failure.

The vectors are then written as **`PutVectors` request payloads**, 500 vectors per file, ready to be sent to S3 Vectors:

```python
batch_vectors.append({
    'key': doc.metadata['child_id'],
    'data': {'float32': vector.tolist()},
    'metadata': {
        'parent_id': doc.metadata['parent_id'],
        'source_doc_id': doc.metadata.get('source_doc_id', ''),
        'title': doc.metadata.get('title', ''),
    },
})
```

The `key` is the `child_id`, and `parent_id` travels in the metadata. That is what allows the online retriever to go straight from a vector hit back to the parent article without a second lookup - the small-to-big expansion costs nothing at query time.

Batching into files of 500 is not cosmetic: it keeps each `PutVectors` request within API limits and makes the ingest in chapter 5.6 **resumable**. If ingest fails halfway you re-run the remaining files instead of re-embedding the corpus.

<!-- IMAGE 3 - SCREENSHOT.
     The embedding cell while running (or just after), showing the sentence-transformers
     progress bar with batch count and elapsed time, e.g. "Batches: 100%|####| 47/47".
     This is the evidence that the heavy step actually ran, and the timing is worth
     quoting in chapter 5.11. -->

![Embedding the child documents](/images/5-Workshop/5.4-Offline-Artifact-Build/embedding-progress.png)

---

## 8. Step 5 - Write the index manifest

The manifest is the contract between the offline build and the online service. It is written as `index_manifest.json` and records what was built, from what, and with which parameters:

```json
{
  "schema_version": 3,
  "index_id": "hotpotqa-val500-bge-m3-v002",
  "created_at": "...",
  "source": {
    "source_mode": "hotpotqa_sample",
    "corpus_id": "hotpotqa/validation-500/v002",
    "processed_id": "hotpotqa-val500-v002",
    "corpus_rows": 4937,
    "corpus_sha256": "..."
  },
  "params": {
    "embedding_model": "BAAI/bge-m3",
    "child_chunk_size": 500,
    "child_chunk_overlap": 100,
    "vector_backend": "s3vectors"
  },
  "artifacts": {
    "parent_docs": { "path": "processed/.../parent_docs.jsonl", "num_docs": 0, "sha256": "..." },
    "child_docs":  { "path": "processed/.../child_docs.jsonl",  "num_docs": 0, "sha256": "..." },
    "bm25":        { "path": "indexes/.../bm25/bm25_index" }
  },
  "s3_vectors": {
    "vector_bucket_name": "rag-vectors-vanh1234",
    "index_name": "hotpotqa-val500-bge-m3-v002",
    "dimension": 1024,
    "distance_metric": "cosine",
    "num_vectors": 0
  },
  "s3_layout": { "corpus": "s3://...", "processed": "s3://...", "bm25": "s3://...", "manifest": "s3://..." }
}
```

Why this file earns its place:

- The online loader reads `params.embedding_model` and **encodes queries with the same model that built the index**. This single field removes an entire class of silent failure where retrieval quietly returns nonsense because the query encoder and the index disagree.
- The `sha256` checksums let you prove that the file in S3 is the file the notebook produced.
- `s3_layout` means the online service can find every artifact from the manifest alone, rather than reconstructing paths from string concatenation in application code.

---

## 9. Step 6 - Assemble the upload folder

The notebook copies everything into one clean tree that mirrors the exact S3 layout, so the upload in chapter 5.5 is a single recursive copy with no path juggling:

```text
s3_manual_upload/hotpotqa-val500-bge-m3-v002/
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
        put_vectors_0001.json
        ...
        s3vectors_import_manifest.json
  ingest_s3vectors.py
```

The notebook also **generates `ingest_s3vectors.py`** with your bucket, index name and region already filled in. That script is what chapter 5.6 runs to push the vectors:

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

{{% notice tip %}}
Note what is **not** in the upload tree: no Chroma database, no model weights, no Python environment. The online service needs about a hundred megabytes of JSONL and index files, and gets its vectors from a managed service. That is what keeps the EC2 instance small.
{{% /notice %}}

---

## 10. Step 7 - Validate before uploading

The last cell asserts that every required file exists and prints the counts:

```python
required_files = [
    UPLOAD_ROOT / 'rag' / 'corpora' / CORPUS_ID / 'corpus.jsonl',
    UPLOAD_ROOT / 'rag' / 'corpora' / CORPUS_ID / 'eval.jsonl',
    UPLOAD_ROOT / 'rag' / 'processed' / PROCESSED_ID / 'parent_docs.jsonl',
    UPLOAD_ROOT / 'rag' / 'processed' / PROCESSED_ID / 'child_docs.jsonl',
    UPLOAD_ROOT / 'rag' / 'indexes' / INDEX_ID / 'manifests' / 'index_manifest.json',
    UPLOAD_ROOT / 'rag' / 'indexes' / INDEX_ID / 'bm25' / 'bm25_doc_ids.json',
    UPLOAD_ROOT / 'rag' / 'indexes' / INDEX_ID / 's3vectors-import' / 's3vectors_import_manifest.json',
    UPLOAD_ROOT / 'ingest_s3vectors.py',
]

missing = [path for path in required_files if not path.exists()]
assert not missing, missing

print('OK: all required files exist')
```

Expected output:

```text
OK: all required files exist
corpus docs: 4937
eval questions: 500
parent docs: 4937
child docs: 8279
vector import files: 17
```

Do not proceed to chapter 5.5 until this cell passes. Catching a missing file here costs a re-run of one cell; catching it after the backend is deployed costs a debugging session against a service that starts up and then fails on the first query.

<!-- IMAGE 4 - SCREENSHOT.
     The output of the validation cell: the line "OK: all required files exist"
     together with all the printed counts (corpus docs, eval questions, parent docs,
     child docs, vector import files).
     These numbers are quoted again in chapter 5.11, so make them legible. -->

![Validation checklist output](/images/5-Workshop/5.4-Offline-Artifact-Build/validation-checklist.png)

---

## 11. Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| Process dies with only `Killed`, no traceback | Out of RAM while loading the embedding model | Use a GPU runtime, or switch to `BAAI/bge-small-en-v1.5` |
| `ValueError: ... Keras 3 ... install tf-keras` | `transformers` detects TensorFlow and tries to load it; this project only needs PyTorch | Set `USE_TF=0` and `USE_FLAX=0` **before** importing |
| `ModuleNotFoundError: bm25s` / `sentence_transformers` / `datasets` | Missing offline dependencies | Install from `backend/requirements/offline.txt` |
| Assertion fails on `vectors.shape[1]` | The embedding model does not produce `VECTOR_DIMENSION` values | Fix whichever is wrong - the model name or the dimension - before building the index in 5.6 |
| Retrieval later returns irrelevant results for every query | The index was built with one model and queried with another | Rebuild, and keep the model name inside `INDEX_ID` |

```python
import os
os.environ['USE_TF'] = '0'
os.environ['USE_FLAX'] = '0'
```

---

## 12. Result

<!-- IMAGE 5 - SCREENSHOT.
     The finished s3_manual_upload/<INDEX_ID>/ folder in the Colab file browser
     (or Google Drive / local Explorer), expanded to show the rag/corpora,
     rag/processed and rag/indexes subfolders and the put_vectors_*.json files.
     This shows the reader exactly what they should have before chapter 5.5. -->

![The assembled upload folder](/images/5-Workshop/5.4-Offline-Artifact-Build/upload-folder-tree.png)

| Produced | Consumed by |
| --- | --- |
| `corpus.jsonl`, `eval.jsonl`, `corpus_manifest.json` | Evaluation, chapter 5.11 |
| `parent_docs.jsonl`, `child_docs.jsonl`, `child_to_parent.json` | The online retriever, every request |
| BM25 index + `bm25_doc_ids.json` | Lexical retrieval, every request |
| `index_manifest.json` | The online loader at startup |
| `put_vectors_*.json` + `ingest_s3vectors.py` | Vector ingest, chapter 5.6 |

Nothing here touched AWS yet, and nothing here will ever run at request time again. Chapter 5.5 creates the S3 bucket and uploads the `rag/` tree; chapter 5.6 creates the S3 Vectors index and ingests the batches.

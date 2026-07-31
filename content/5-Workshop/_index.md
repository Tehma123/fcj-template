---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

![Offline ingestion pipeline: corpus.jsonl is split into parent/child chunks, the child chunks are embedded with BAAI/bge-m3 and indexed with BM25, an index_manifest.json records the embedding model/chunk sizes/paths, then embeddings are uploaded to S3 Vectors and docs/BM25 index/manifest are uploaded to S3](/images/5-Workshop/offline_ingestion_pipeline.png)

![Online query pipeline: POST /query is decomposed by an LLM, retrieved via hybrid BM25 + vector search fused with RRF, expanded from child chunk to parent document, checked by an adaptive hop planner that can loop back for another query, optionally reranked, then answered by an LLM](/images/5-Workshop/online_query_pipeline.png)

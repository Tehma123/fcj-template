---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

![Pipeline offline ingestion: corpus.jsonl được tách thành chunk parent/child, các child chunk được embed bằng BAAI/bge-m3 và lập chỉ mục BM25, một index_manifest.json ghi lại embedding model/kích thước chunk/đường dẫn, sau đó embeddings được upload lên S3 Vectors còn docs/BM25 index/manifest được upload lên S3](/images/5-Workshop/offline_ingestion_pipeline.png)

![Pipeline online query: POST /query được một LLM phân rã, truy xuất qua hybrid BM25 + vector search hợp nhất bằng RRF, mở rộng từ child chunk về parent document, được adaptive hop planner kiểm tra và có thể lặp lại để truy vấn thêm, tùy chọn rerank, rồi được LLM trả lời](/images/5-Workshop/online_query_pipeline.png)

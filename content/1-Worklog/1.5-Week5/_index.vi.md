---
title: "Worklog Tuần 5"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.5. </b> "
---


### Mục tiêu tuần 5:

* Hiểu Retrieval-Augmented Generation (RAG) là gì và vì sao nó giúp giảm hallucination.
* Xây dựng một pipeline RAG đơn giản (naive, một lượt): chunk → embed → retrieve → generate.
* Chạy pipeline naive trên một mẫu câu hỏi HotpotQA và đo độ chính xác baseline.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                    | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                                                    |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ------------------------------------------------------------------ |
| 4   | - Tìm hiểu nền tảng RAG: retrieval + generation, cách "ground" câu trả lời của LLM vào context đã truy xuất                                | 08/07/2026   | 08/07/2026      |                                                                    |
| 5   | - Tìm hiểu pipeline naive RAG: <br>&emsp; + Chiến lược chunking tài liệu <br>&emsp; + Embed corpus <br>&emsp; + Lưu vector <br>&emsp; + Top-k similarity search | 09/07/2026   | 09/07/2026      |                                                                      |
| 6   | - **Thực hành:** chunk và embed một tập nhỏ đoạn context của HotpotQA, lưu embeddings vào vector index                                     | 10/07/2026   | 10/07/2026      |                                                                      |
| 2   | - **Thực hành:** cài đặt retrieval một lượt + dựng prompt, sinh câu trả lời cho các câu hỏi mẫu                                            | 13/07/2026   | 13/07/2026      |                                                                      |
| 3   | - Đánh giá pipeline naive RAG trên một mẫu nhỏ HotpotQA (Exact Match / F1) và ghi nhận các trường hợp lỗi với câu hỏi đa bước               | 14/07/2026   | 14/07/2026      |                                                                      |


### Kết quả đạt được tuần 5:

* Hiểu ý tưởng cốt lõi của RAG và vì sao việc "ground" câu trả lời vào bằng chứng đã truy xuất giúp giảm hallucination.

* Xây dựng được một pipeline naive RAG hoạt động: chunking, embedding, vector similarity search, và sinh câu trả lời dựa trên prompt.

* Chạy pipeline end-to-end trên một mẫu câu hỏi HotpotQA.

* Đo được độ chính xác baseline (EM/F1) và nhận thấy retrieval một lượt (naive) thường thất bại với các câu hỏi đa bước cần bằng chứng từ nhiều tài liệu.

* Xác định đây chính là động lực để tìm hiểu các kỹ thuật RAG nâng cao trong tuần tới.
* ...

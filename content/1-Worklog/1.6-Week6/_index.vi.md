---
title: "Worklog Tuần 6"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.6. </b> "
---


### Mục tiêu tuần 6:

* Hiểu những giới hạn của naive RAG với câu hỏi đa bước.
* Tìm hiểu và áp dụng các kỹ thuật RAG nâng cao: query decomposition, retrieval lặp (iterative/multi-hop), và re-ranking.
* Cải thiện chất lượng retrieval và độ chính xác trả lời trên HotpotQA so với baseline ở tuần 5.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc | Ngày |
| --- | --- | --- |
| 4 | - Tìm hiểu các kỹ thuật RAG nâng cao: <br>&emsp; + Query rewriting / decomposition <br>&emsp; + Retrieval lặp (iterative) <br>&emsp; + Re-ranking <br>&emsp; + Hybrid search (keyword + vector) | 15/07/2026 |
| 5 | - **Thực hành:** cài đặt query decomposition để tách một câu hỏi đa bước thành các câu hỏi con | 16/07/2026 |
| 6 | - **Thực hành:** cài đặt vòng lặp retrieval đa bước (iterative), dùng câu trả lời của câu hỏi con này để truy xuất bằng chứng cho câu hỏi con tiếp theo | 17/07/2026 |
| 2 | - **Thực hành:** thêm bước re-ranking cho các đoạn văn bản đã truy xuất để cải thiện độ liên quan của context trước khi sinh câu trả lời | 20/07/2026 |
| 3 | - Đánh giá lại pipeline advanced RAG trên cùng mẫu HotpotQA đã dùng ở tuần 5 và so sánh EM/F1 với baseline naive RAG | 21/07/2026 |


### Kết quả đạt được tuần 6:

* Hiểu vì sao naive RAG gặp khó khăn với câu hỏi đa bước và các chiến lược retrieval nâng cao giải quyết vấn đề đó như thế nào.

* Cài đặt được query decomposition để tách câu hỏi đa bước thành các câu hỏi con có thể truy xuất riêng.

* Xây dựng được vòng lặp retrieval đa bước, nối tiếp bằng chứng qua nhiều tài liệu.

* Thêm bước re-ranking giúp cải thiện độ liên quan của context được truy xuất.

* Đo được mức cải thiện độ chính xác rõ rệt (EM/F1) so với baseline naive RAG ở tuần 5 trên HotpotQA.

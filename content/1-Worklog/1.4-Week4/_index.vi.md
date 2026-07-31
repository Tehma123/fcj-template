---
title: "Worklog Tuần 4"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.4. </b> "
---


### Mục tiêu tuần 4:

* Có cái nhìn tổng quan về Amazon SageMaker và vai trò của nó trong vòng đời machine learning.
* Hiểu về text embeddings và vector similarity — nền tảng cho bước truy xuất (retrieval) sẽ dùng trong RAG.
* Thực hành notebook và ví dụ inference đầu tiên trên SageMaker.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 4   | - Tìm hiểu tổng quan SageMaker: <br>&emsp; + Studio / notebook instance <br>&emsp; + Training job & model registry <br>&emsp; + Endpoint | 01/07/2026   | 01/07/2026      |                                            |
| 5   | - **Thực hành:** khởi tạo một SageMaker notebook instance, load một model pretrained nhỏ và chạy inference                              | 02/07/2026   | 02/07/2026      |                                            |
| 6   | - Tìm hiểu text embeddings: <br>&emsp; + Embedding là gì <br>&emsp; + Cosine similarity <br>&emsp; + Vì sao embeddings hỗ trợ semantic search | 03/07/2026   | 03/07/2026      |                                            |
| 2   | - **Thực hành:** sinh embeddings cho một tập nhỏ đoạn văn bản trong SageMaker notebook và tính độ tương đồng giữa chúng                  | 06/07/2026   | 06/07/2026      |                                            |
| 3   | - Khảo sát cấu trúc bộ dữ liệu HotpotQA (câu hỏi, supporting facts, đoạn văn bản ngữ cảnh) để chuẩn bị cho dự án RAG                     | 07/07/2026   | 07/07/2026      | <https://hotpotqa.github.io/>             |


### Kết quả đạt được tuần 4:

* Hiểu vai trò của SageMaker trong vòng đời ML (build, train, deploy).

* Triển khai và truy vấn được một model từ SageMaker notebook instance.

* Hiểu cách text embeddings biểu diễn ngữ nghĩa và cách cosine similarity dùng để so sánh chúng.

* Tính được embeddings và điểm tương đồng cho một tập văn bản nhỏ, làm bước khởi động cho retrieval.

* Khảo sát cấu trúc bộ dữ liệu HotpotQA và xác định được "multi-hop" nghĩa là gì trong thực tế.

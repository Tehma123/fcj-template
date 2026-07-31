---
title: "Worklog Tuần 7"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.7. </b> "
---


### Mục tiêu tuần 7:

* Đưa các thành phần của pipeline RAG lên chạy trên dịch vụ AWS thay vì chạy hoàn toàn ở local.
* Hiểu cách host corpus, embeddings và logic điều phối (orchestration) trên AWS.
* Thêm giám sát cho pipeline bằng CloudWatch.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                          | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 4   | - Lên kế hoạch dịch vụ AWS nào sẽ host từng thành phần của pipeline (lưu trữ corpus, vector index, orchestration, truy cập LLM) | 22/07/2026   | 22/07/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 5   | - **Thực hành:** upload corpus HotpotQA và embeddings đã tính sẵn lên S3                                                        | 23/07/2026   | 23/07/2026      |                                            |
| 6   | - **Thực hành:** đóng gói logic retrieval + suy luận đa bước vào một Lambda function / dịch vụ host trên SageMaker              | 24/07/2026   | 24/07/2026      |                                            |
| 2   | - **Thực hành:** expose pipeline qua một API endpoint và kiểm thử với các câu hỏi mẫu từ HotpotQA                               | 27/07/2026   | 27/07/2026      |                                            |
| 3   | - Thêm CloudWatch logging và alarm cho pipeline (lỗi, độ trễ) <br> - Review tiến độ cùng mentor                                 | 28/07/2026   | 28/07/2026      |                                            |


### Kết quả đạt được tuần 7:

* Di chuyển corpus HotpotQA và embeddings từ lưu trữ local lên S3.

* Đóng gói logic retrieval/suy luận của advanced RAG vào một dịch vụ chạy trên AWS.

* Expose pipeline qua một API endpoint và kiểm chứng với các câu hỏi mẫu.

* Thêm CloudWatch logging và alarm để giám sát lỗi và độ trễ của pipeline.

* Xác nhận pipeline chạy trên AWS cho kết quả giống với bản prototype chạy local ở tuần 6.
* ...

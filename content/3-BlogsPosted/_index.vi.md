---
title: "Các bài blogs đã đăng"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3. </b> "
---


Tại đây sẽ là phần liệt kê, giới thiệu các blogs mà các bạn đã đăng trên [AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj). Ví dụ:

###  [Blog 1 - Amazon S3 Annotations: Metadata Có Thể Cập Nhật Và Truy Vấn Cho Từng Object](3.1-Blog1/)
Blog này giới thiệu Amazon S3 Annotations, một cách mới để gắn các phần metadata có tên riêng, cập nhật độc lập, trực tiếp vào từng S3 object và đưa vào S3 Metadata để truy vấn ở quy mô lớn bằng công cụ phân tích hoặc AI agent.

###  [Blog 2 - Tối Ưu Chi Phí AWS: Đừng Chỉ Nhìn Vào Hóa Đơn](3.2-Blog2/)
Blog này trình bày cách tiếp cận tối ưu chi phí theo AWS Well-Architected Framework: biết chi phí đến từ đâu, đo chi phí cùng với kết quả kinh doanh, điều chỉnh tài nguyên theo nhu cầu, xác định người phụ trách minh bạch và biến việc này thành một thói quen thay vì một đợt cắt giảm nhất thời.

###  [Blog 3 - Amazon SQS Fair Queues: Chấm Dứt Vấn Đề "Noisy Neighbor" Trong Hệ Thống Multi-Tenant](3.3-Blog3/)
Blog này giới thiệu Amazon SQS Fair Queues, một tính năng mới tự động sắp xếp lại thứ tự giao tin nhắn để bảo vệ các tenant "yên tĩnh" khỏi một tenant "ồn ào" trên cùng một hàng đợi dùng chung, tận dụng trường `MessageGroupId` sẵn có cùng các metric CloudWatch mới về tính công bằng, không cần sửa đổi gì ở phía consumer.
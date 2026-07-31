---
title: "Workshop triển khai AWS CloudHop RAG"
date: 2026-07-31
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

## Tổng quan

Mô hình ngôn ngữ có thể tạo ra câu trả lời tự nhiên, nhưng nội dung trả lời vẫn phụ thuộc vào kiến thức mà mô hình đã có. **Retrieval-Augmented Generation (RAG)** bổ sung một bước truy xuất thông tin từ nguồn dữ liệu bên ngoài trước khi sinh câu trả lời. Nhờ đó, hệ thống không chỉ dựa vào kiến thức sẵn có của mô hình mà còn có thể sử dụng các bằng chứng được tìm thấy cho chính câu hỏi đang xử lý.

Bài toán trở nên khó hơn khi thông tin cần thiết nằm rải rác trong nhiều tài liệu. Một nguồn có thể giúp xác định thực thể quan trọng, trong khi một nguồn khác mới chứa dữ kiện cần thiết để hoàn thiện câu trả lời. **AWS CloudHop RAG** được xây dựng cho dạng hỏi đáp đa bước này, trong đó hệ thống phải tìm các bằng chứng bổ trợ lẫn nhau, kết nối chúng qua nhiều bước truy xuất và sử dụng context thu được để sinh câu trả lời có căn cứ.

Dự án kết hợp hybrid retrieval, xử lý bằng chứng đa bước và LLM-based generation trong một hệ thống được triển khai hoàn chỉnh trên AWS. Amazon S3 lưu corpus đã xử lý và retrieval artifact, Amazon S3 Vectors hỗ trợ dense semantic search, Amazon EC2 vận hành FastAPI RAG backend, Amazon API Gateway cung cấp API qua HTTPS, còn AWS Amplify triển khai frontend cho người dùng.

Workshop trình bày hệ thống từ giai đoạn chuẩn bị retrieval artifact đến triển khai trên cloud, kiểm thử, đánh giá, vận hành và dọn dẹp tài nguyên. Các phần tiếp theo tập trung vào cách pipeline RAG và hạ tầng AWS phối hợp với nhau trong một ứng dụng end-to-end thống nhất.

## Nội dung Workshop

1. [Tổng quan Workshop](5.1-Workshop-Overview/)
2. [Yêu cầu chuẩn bị](5.2-Prerequisites/)
3. [Kiến trúc hệ thống](5.3-Architecture/)
4. [Xây dựng Offline Artifact](5.4-Offline-Artifact-Build/)
5. [Lưu trữ trên Amazon S3](5.5-S3-Storage/)
6. [Amazon S3 Vectors](5.6-S3-Vectors/)
7. [Triển khai Backend trên Amazon EC2](5.7-EC2-Backend/)
8. [Amazon API Gateway](5.8-API-Gateway/)
9. [Triển khai Frontend với AWS Amplify](5.9-Amplify-Frontend/)
10. [Kiểm thử và xác thực](5.10-Testing-Validation/)
11. [Đánh giá hệ thống](5.11-Evaluation/)
12. [Vận hành và xử lý sự cố](5.12-Operations-Monitoring/)
13. [Bảo mật và cân nhắc chi phí](5.13-Security-Cost/)
14. [Dọn dẹp tài nguyên](5.14-Cleanup/)

---
title: "Worklog Tuần 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.3. </b> "
---


### Mục tiêu tuần 3:

* Hiểu cách giám sát tài nguyên AWS bằng Amazon CloudWatch (metrics, logs, alarms).
* Hiểu cách Amazon CloudFront hoạt động như một CDN và cách nó tích hợp với S3.
* Thực hành thiết lập giám sát và phân phối nội dung cho một workload đơn giản.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                       | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Tìm hiểu CloudWatch: <br>&emsp; + Metrics <br>&emsp; + Log group & log stream <br>&emsp; + Alarm & dashboard                                                              | 22/06/2026   | 22/06/2026      |                                            |
| 3   | - **Thực hành:** tạo CloudWatch alarm theo dõi CPU utilization của EC2; đẩy log từ EC2 lên CloudWatch Logs bằng CloudWatch agent                                            | 23/06/2026   | 23/06/2026      |                                            |
| 4   | - Tìm hiểu CloudFront: <br>&emsp; + Distribution & origin (S3/EC2) <br>&emsp; + Edge location & caching behavior <br>&emsp; + Origin Access Control (OAC)                   | 24/06/2026   | 24/06/2026      |                                            |
| 5   | - **Thực hành:** tạo CloudFront distribution đặt trước một S3 bucket, giới hạn truy cập trực tiếp vào S3 bằng OAC, và kiểm tra cache invalidation                           | 25/06/2026   | 25/06/2026      |                                            |
| 6   | - Dựng CloudWatch dashboard kết hợp metrics của EC2 và CloudFront <br> - Review công việc trong tuần cùng mentor                                                             | 26/06/2026   | 26/06/2026      |                                            |


### Kết quả đạt được tuần 3:

* Thiết lập được CloudWatch alarm và thu thập log cho một EC2 instance.

* Hiểu cách CloudFront cache và phân phối nội dung từ các edge location.

* Triển khai được một CloudFront distribution đặt trước S3 bucket kèm Origin Access Control, giúp bucket không còn bị truy cập trực tiếp từ internet công cộng.

* Dựng được một CloudWatch dashboard cơ bản để theo dõi tình trạng tài nguyên.

* Hiểu cách giám sát (CloudWatch) và phân phối nội dung (CloudFront) phối hợp trong một kiến trúc sẵn sàng cho production.
* ...

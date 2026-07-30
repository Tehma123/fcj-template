---
title: "Worklog Tuần 2"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.2. </b> "
---


### Mục tiêu tuần 2:

* Hiểu các dịch vụ lưu trữ và định danh cốt lõi của AWS: Amazon S3 và IAM.
* Hiểu các khái niệm networking cơ bản của AWS với Amazon VPC.
* Thực hành tạo và bảo mật tài nguyên cloud theo nguyên tắc least privilege.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                              | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Tìm hiểu Amazon S3: <br>&emsp; + Bucket & object <br>&emsp; + Storage classes <br>&emsp; + Versioning <br> - **Thực hành:** tạo bucket, upload/download object, cấu hình bucket policy            | 15/06/2026   | 15/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 3   | - Tìm hiểu IAM: <br>&emsp; + User, group, role <br>&emsp; + Policy & least privilege <br> - **Thực hành:** tạo IAM role với policy giới hạn quyền và kiểm tra quyền truy cập                        | 16/06/2026   | 16/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 4   | - Tìm hiểu VPC cơ bản: <br>&emsp; + VPC & subnet (public/private) <br>&emsp; + Route table & internet gateway <br>&emsp; + Security group so với NACL                                               | 17/06/2026   | 17/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 5   | - **Thực hành:** dựng một VPC với subnet public/private; khởi tạo EC2 instance bên trong và kiểm soát truy cập bằng security group                                                                  | 18/06/2026   | 18/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 6   | - Tìm hiểu S3 Gateway endpoint và các mô hình kết nối riêng tư <br> - Review công việc trong tuần cùng mentor                                                                                        | 19/06/2026   | 19/06/2026      | <https://cloudjourney.awsstudygroup.com/> |


### Kết quả đạt được tuần 2:

* Hiểu các khái niệm cốt lõi của S3 (bucket, storage classes, versioning) và đã tạo/cấu hình một S3 bucket kèm bucket policy.

* Hiểu kiến thức nền tảng của IAM và đã tạo một IAM role với quyền hạn theo nguyên tắc least privilege.

* Dựng được một VPC cơ bản với subnet public/private và kiểm soát traffic bằng security group.

* Hiểu cách S3 Gateway endpoint giúp tài nguyên trong VPC truy cập S3 riêng tư mà không cần đi qua internet công cộng.

* Có cái nhìn rõ ràng hơn về cách storage, identity và networking phối hợp với nhau để bảo mật một workload.
* ...

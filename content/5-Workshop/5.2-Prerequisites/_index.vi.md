---
title: "Yêu cầu chuẩn bị"
date: 2026-07-31
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

Trước khi bắt đầu triển khai, cần chuẩn bị tài khoản AWS, các công cụ phát triển và quyền truy cập cần thiết cho CloudHop RAG.

## Tài khoản AWS và quyền truy cập

CloudHop RAG được triển khai tại **Asia Pacific (Singapore) Region (`ap-southeast-1`)**. Sử dụng Region này xuyên suốt workshop cho các tài nguyên AWS.

Cần có tài khoản AWS với quyền truy cập AWS Management Console và đủ quyền để tạo, quản lý các dịch vụ sau:

- Amazon S3;
- Amazon S3 Vectors;
- Amazon EC2;
- AWS Identity and Access Management (IAM);
- Amazon API Gateway;
- AWS Amplify;
- AWS Systems Manager.

AWS Systems Manager Session Manager được sử dụng để truy cập và quản lý EC2 backend trong quá trình triển khai.

## Môi trường phát triển

CloudHop RAG sử dụng một số công cụ phát triển phổ biến sau:

| Công cụ | Mục đích sử dụng |
| --- | --- |
| **Git** | Clone và cập nhật mã nguồn của dự án |
| **Python + pip** | Cài đặt dependency backend và chạy các script của dự án |
| **AWS CLI** | Tạo, kiểm tra và cập nhật tài nguyên AWS từ terminal |
| **Node.js + npm** | Cài đặt dependency và build React/Vite frontend |

Có thể kiểm tra nhanh các công cụ đã được cài đặt bằng:

```bash
aws --version
python --version
node --version
npm --version
git --version
```

## Cấu hình AWS CLI

Cấu hình AWS CLI cho tài khoản được sử dụng trong workshop:

```bash
aws configure
```

Đặt Region mặc định là:

```text
ap-southeast-1
```

Sau đó kiểm tra AWS identity hiện tại:

```bash
aws sts get-caller-identity
```

Nếu lệnh trả về thông tin tài khoản và IAM identity, AWS CLI đã được xác thực thành công.

## Tài nguyên dự án

Trước khi bắt đầu, cần chuẩn bị:

- mã nguồn **AWS CloudHop RAG**;
- một **Groq API key** hợp lệ cho bước sinh câu trả lời;
- bộ retrieval artifact đã được chuẩn bị cho quá trình triển khai.

Nếu cần xây dựng lại retrieval artifact từ đầu, cần thêm quyền truy cập vào bộ dữ liệu **HotpotQA**.
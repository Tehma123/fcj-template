---
title: "Vận hành và xử lý sự cố"
date: 2026-07-31
weight: 12
chapter: false
pre: " <b> 5.12. </b> "
---

Sau khi triển khai, phần lớn công việc vận hành CloudHop RAG tập trung vào EC2 backend. Ứng dụng chạy dưới dạng `aws-rag-api` systemd service, trong khi **AWS Systems Manager Session Manager** được sử dụng để truy cập instance khi cần bảo trì hoặc xử lý sự cố.

Backend cung cấp các endpoint `/health` và `/warmup` để kiểm tra trạng thái ứng dụng và khởi tạo retrieval pipeline. Khi service không khởi động đúng hoặc request gặp lỗi, log của backend có thể được kiểm tra trực tiếp thông qua systemd journal.

### Hướng dẫn vận hành

Hướng dẫn vận hành hệ thống hàng ngày.

#### 1. Console Steps: Khởi động hệ thống
Sau khi dừng EC2, khi cần dùng lại:
1. **Start EC2**: Bật instance trong **EC2 Console**.
2. **SSM Session**: Kết nối qua **Session Manager** (không cần SSH key).

#### 2. CLI Steps: Start Backend
Sau khi đã vào terminal của EC2:
```bash
sudo systemctl restart aws-rag-api
# Warmup (cần thiết để load model)
curl -X POST http://127.0.0.1:8000/warmup
```

#### 3. CLI Steps: Kiểm tra
```bash
curl http://127.0.0.1:8000/health
```

#### 4. Xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Cách khắc phục |
| :--- | :--- | :--- |
| **Mixed Content** | Frontend HTTPS gọi backend HTTP | Kiểm tra `VITE_API_BASE_URL` trong Amplify Console |
| **CORS Error** | API Gateway chặn request | Kiểm tra lại cấu hình CORS tab trong API Gateway Console |
| **503 Service** | Backend chết | `sudo systemctl status aws-rag-api` để kiểm tra log |
| **SSH Timeout** | Security Group chặn | Dùng Session Manager trong Console thay vì SSH |
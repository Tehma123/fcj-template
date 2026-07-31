---
title: "Kiểm thử và xác thực"
date: 2026-07-31
weight: 10
chapter: false
pre: " <b> 5.10. </b> "
---

Sau khi các thành phần AWS đã được kết nối với nhau, bước tiếp theo là xác minh rằng ứng dụng đã triển khai hoạt động đúng, từ backend cho tới trình duyệt. Phần này tập trung vào việc xác thực chức năng của bản triển khai, chứ chưa xét đến chất lượng retrieval hay chất lượng câu trả lời.

## Xác thực Backend và Retrieval

FastAPI backend chạy trên Amazon EC2 dưới dạng systemd service `aws-rag-api`. Có thể kiểm tra trạng thái của nó bằng:

```bash
sudo systemctl status aws-rag-api
```

Endpoint `/health` xác nhận rằng API đang chạy:

```bash
curl http://127.0.0.1:8000/health
```

Sau đó, retrieval pipeline được khởi tạo thông qua:

```bash
curl -X POST http://127.0.0.1:8000/warmup
```

Một lần warmup thành công xác nhận rằng backend có thể tải các retrieval artifact cần thiết từ Amazon S3 và kết nối được tới Amazon S3 Vectors index dùng cho dense retrieval.

## Xác thực API và trình duyệt

Khi backend đã chạy tốt cục bộ trên EC2, chính các endpoint đó được kiểm thử lại thông qua Amazon API Gateway. API đã triển khai cung cấp:

```text
GET  /health
POST /warmup
POST /query
```

Route `/health` xác minh rằng các request HTTPS có thể đi tới backend EC2 thông qua API Gateway. CORS cũng được kiểm tra để xác nhận rằng request từ frontend trên AWS Amplify được API chấp nhận.

Tiếp đó, một câu hỏi thật được gửi qua `/query`. Request được coi là thành công khi backend hoàn tất retrieval và generation, đồng thời trả về cả câu trả lời lẫn các nguồn bằng chứng đi kèm.

Bước kiểm tra cuối cùng được thực hiện ngay từ ứng dụng Amplify đã triển khai. Một request thành công từ trình duyệt xác nhận toàn bộ đường đi:

```text
Trình duyệt
→ AWS Amplify
→ Amazon API Gateway
→ Amazon EC2
→ Amazon S3 và Amazon S3 Vectors
→ Groq
→ Câu trả lời và nguồn bằng chứng
```

## Kết quả xác thực

| Hạng mục kiểm thử | Kết quả mong đợi | Trạng thái |
| --- | --- | --- |
| EC2 backend | FastAPI service đang chạy | Đạt |
| Health endpoint | Phản hồi HTTP 200 | Đạt |
| Warmup pipeline | Retrieval pipeline nạp thành công | Đạt |
| Amazon S3 | Truy cập được các retrieval artifact cần thiết | Đạt |
| Amazon S3 Vectors | Dense retrieval trả về kết quả | Đạt |
| API Gateway | Request HTTPS đi tới được backend | Đạt |
| CORS | Origin của Amplify được chấp nhận | Đạt |
| Query endpoint | Trả về câu trả lời kèm nguồn bằng chứng | Đạt |
| Amplify frontend | Trình duyệt hiển thị phản hồi hoàn chỉnh | Đạt |

Kết quả xác thực khẳng định rằng các thành phần AWS đã triển khai vận hành như một ứng dụng thống nhất, và một câu hỏi của người dùng có thể đi trọn vẹn qua toàn hệ thống rồi quay về với câu trả lời được sinh ra kèm bằng chứng hỗ trợ.

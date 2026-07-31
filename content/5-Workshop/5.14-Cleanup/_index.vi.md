---
title: "Dọn dẹp tài nguyên"
date: 2026-07-31
weight: 14
chapter: false
pre: " <b> 5.14. </b> "
---

Sau khi hoàn thành workshop, các tài nguyên AWS được tạo cho CloudHop RAG nên được xóa nếu không còn sử dụng. Việc dọn dẹp giúp tránh duy trì các tài nguyên compute, lưu trữ, network và ứng dụng không còn cần thiết trong tài khoản AWS.

Các tài nguyên nên được xóa theo thứ tự phù hợp để tránh để lại những thành phần phụ thuộc không còn được sử dụng.

{{% notice warning %}}
Việc xóa là vĩnh viễn. Nếu bạn còn có thể phải demo lại hệ thống, hãy dùng phương án **tạm dừng** ở mục 3 - nó cắt gần hết chi phí đang chạy mà không phá hủy thứ gì.
{{% /notice %}}

---

## 1. Thứ tự xóa

Làm từ trên xuống: thứ gọi tới thứ khác thì xóa trước, thứ chứa dữ liệu thì xóa sau cùng.

| # | Tài nguyên | Console | Ghi chú |
| --- | --- | --- | --- |
| 1 | **Ứng dụng Amplify** | Amplify → App settings → General → **Delete app** | Xóa hosting, lịch sử build và URL `amplifyapp.com` |
| 2 | **API của API Gateway** | API Gateway → chọn API → **Delete** | Xóa mọi route, integration và Invoke URL |
| 3 | **Instance EC2** | EC2 → Instances → Instance state → **Terminate** | Ổ đĩa gốc EBS đi kèm mặc định bị xóa theo - hãy kiểm chứng |
| 4 | **Elastic IP** | EC2 → Elastic IP addresses → **Release** | Xem cảnh báo bên dưới |
| 5 | **S3 Vectors index** | S3 → Vector buckets → bucket của bạn → xóa index | Phải làm trước khi xóa bucket |
| 6 | **Vector bucket** | S3 → Vector buckets → **Delete** | Chỉ xóa được khi đã rỗng |
| 7 | **Object S3, rồi tới bucket** | S3 → bucket của bạn → empty, rồi delete | Xem lưu ý về versioning bên dưới |
| 8 | **IAM role** | IAM → Roles → `rag-ec2-runtime-role` → **Delete** | Chỉ sau khi instance đã bị xóa |
| 9 | **Secret trên Secrets Manager** | Secrets Manager → secret của bạn → **Delete** | Có thời gian chờ khôi phục - xem bên dưới |
| 10 | **Tham số SSM** | Systems Manager → Parameter Store | Chỉ khi bạn từng tạo. Bản triển khai này cấu hình dịch vụ bằng `.env.prod` nên không có tham số nào |

{{% notice warning %}}
**Giải phóng Elastic IP là bước hay bị quên nhất, và cũng chính là thứ tiếp tục tính tiền.** Terminate instance chỉ *gỡ liên kết* địa chỉ; phần cấp phát vẫn còn, mà địa chỉ IPv4 công khai thì bị tính theo giờ bất kể có đang gắn vào đâu hay không. Hãy vào **EC2 → Elastic IP addresses** và xác nhận danh sách đã trống.
{{% /notice %}}

**Bucket có versioning.** Bucket artifact đang bật versioning (chương 5.5), nên thao tác "Empty" phải xóa **toàn bộ các phiên bản object và delete marker** - nếu không, bucket sẽ từ chối bị xóa và báo là chưa rỗng. Trên Console, dùng **Empty** rồi xác nhận; với CLI, `aws s3 rb --force` không xóa các phiên bản cũ, nên phải làm rỗng tường minh trước.

**Xóa secret.** Secrets Manager lên lịch xóa với thời gian chờ khôi phục từ 7 đến 30 ngày chứ không xóa ngay, và vẫn tính tiền secret trong khoảng thời gian đó. Muốn xóa dứt điểm ngay:

```bash
aws secretsmanager delete-secret \
  --region ap-southeast-1 \
  --secret-id /prod/aws-rag/groq-api-key \
  --force-delete-without-recovery
```

Đồng thời hãy **thu hồi Groq API key** trong console của Groq. Xóa secret trên AWS chỉ xóa bản sao, không xóa chính cái key đó.

---

## 2. Lệnh CLI tương đương

```bash
# 3-4. EC2 và Elastic IP
aws ec2 terminate-instances --instance-ids <instance-id> --region ap-southeast-1
aws ec2 release-address --allocation-id <allocation-id> --region ap-southeast-1

# 5-6. S3 Vectors
aws s3vectors delete-index --vector-bucket-name <vector-bucket> --index-name <index-id> --region ap-southeast-1
aws s3vectors delete-vector-bucket --vector-bucket-name <vector-bucket> --region ap-southeast-1

# 7. S3 thông thường (xóa hết mọi phiên bản, rồi xóa bucket)
aws s3 rm s3://<artifact-bucket> --recursive --region ap-southeast-1
aws s3 rb s3://<artifact-bucket> --region ap-southeast-1

# 9. Secret (xoa dut diem, khong cho thoi gian khoi phuc)
aws secretsmanager delete-secret --region ap-southeast-1 \
  --secret-id <ten-secret-cua-ban> --force-delete-without-recovery
```

---

## 3. Tạm dừng thay vì xóa

Nếu dự án còn có thể phải đem ra demo, hãy dừng lại chứ đừng phá đi:

| Hành động | Tác dụng |
| --- | --- |
| **Stop** instance EC2 | Cắt bỏ khoản chi phí định kỳ lớn nhất |
| Giữ S3, S3 Vectors và Secrets Manager | Không đáng kể ở quy mô dữ liệu này |
| Giữ Amplify và API Gateway | Tính theo mức dùng; không dùng thì không phải trả |

Thứ vẫn bị tính tiền khi đã dừng: **ổ đĩa gốc EBS** và **Elastic IP**. Nếu quãng tạm dừng kéo dài hàng tuần chứ không phải vài ngày thì nên giải phóng Elastic IP - lúc quay lại bạn sẽ phải cập nhật các integration của API Gateway theo địa chỉ mới (chương 5.8 mục 2).

Việc khởi động lại chính là quy trình hằng ngày ở chương 5.7 mục 12: bật instance, kết nối bằng Session Manager, kiểm tra `/health`, làm nóng.

---

## 4. Kiểm tra không sót thứ gì

Kiểm tra từng mục sau trên Console, trong Region `ap-southeast-1`:

| Kiểm tra | Kết quả mong đợi |
| --- | --- |
| EC2 → Instances | Không còn instance nào của dự án, dù đang chạy hay đã dừng |
| EC2 → **Elastic IP addresses** | Trống |
| EC2 → **Volumes** | Không còn ổ đĩa mồ côi từ instance đã terminate |
| S3 → Buckets | Bucket của dự án đã biến mất |
| S3 → **Vector buckets** | Vector bucket đã biến mất |
| API Gateway → APIs | API của dự án đã biến mất |
| Amplify → Apps | Ứng dụng đã biến mất |
| IAM → Roles | `rag-ec2-runtime-role` đã biến mất |
| Secrets Manager | Secret đã mất hoặc đã được lên lịch xóa |
| Parameter Store | Trống, nếu bạn từng tạo tham số |

{{% notice tip %}}
Hai dòng trong bảng trên tồn tại vì chúng chính là hai thủ phạm quen thuộc. **Elastic IP addresses** và **Volumes** bị tính tiền độc lập với instance mà chúng từng thuộc về, và cả hai đều không xuất hiện trong danh sách instance của EC2 - nên một đợt dọn dẹp chỉ kiểm tra "instance mất chưa" sẽ để sót chúng vẫn đang chạy. Hãy kiểm tra cả hai một cách tường minh.
{{% /notice %}}

Cuối cùng, hãy xem **Billing → Cost Explorer** sau đó một hai ngày. Chi phí được báo cáo có độ trễ, nên một Console sạch không đồng nghĩa với hóa đơn sạch ngay lập tức; con số chi phí hằng ngày gần bằng không sau 48 giờ mới là xác nhận thật sự.

---

## 5. Những gì nên giữ lại

Xóa tài nguyên AWS không có nghĩa là xóa đi công sức. Những thứ sau vẫn còn, và chính chúng khiến bản triển khai có thể tái lập được:

- **repository mã nguồn** - backend, frontend và notebook dựng artifact offline;
- **thư mục artifact offline** nếu bạn còn giữ một bản sao ngoài AWS, nhờ đó dựng lại index mà không phải embed lại corpus;
- **chính workshop này**, vốn là bản ghi chép cách mọi tài nguyên đã được tạo ra.

Dựng lại từ đầu nghĩa là chạy lại các chương từ 5.4 đến 5.9 - khoảng một buổi chiều, và không bước nào phụ thuộc vào thứ đã bị xóa.

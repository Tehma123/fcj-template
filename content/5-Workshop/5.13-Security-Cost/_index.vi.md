---
title: "Bảo mật và cân nhắc chi phí"
date: 2026-07-31
weight: 13
chapter: false
pre: " <b> 5.13. </b> "
---

CloudHop RAG được triển khai như một môi trường dự án thực tế, trong đó các yếu tố bảo mật và chi phí vẫn được xem xét trong quá trình thiết kế. Quyền truy cập AWS của EC2 backend được cấp thông qua IAM role thay vì đưa AWS credential trực tiếp vào ứng dụng, đồng thời Systems Manager Session Manager được sử dụng để quản lý instance.

API Gateway cung cấp lớp API HTTPS cho frontend trên Amplify, còn CORS giới hạn request từ trình duyệt về đúng frontend origin đã cấu hình. Về chi phí, thành phần duy trì thường xuyên nhất là EC2 instance, cùng với chi phí lưu trữ, vector search, API request và frontend hosting.

---

## Bảo mật

### 1. Không có thông tin xác thực AWS tồn tại lâu dài

Ứng dụng không giữ access key AWS nào. Instance EC2 mang instance profile `rag-ec2-runtime-role`, và AWS SDK lấy thông tin xác thực tạm thời qua IMDSv2. Có thể kiểm chứng ngay trên instance:

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" -s)

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

```text
rag-ec2-runtime-role
```

Mọi lệnh gọi AWS mà dịch vụ thực hiện - đọc artifact từ S3, truy vấn S3 Vectors - đều được cấp quyền theo cách này. Không có gì trong repository, trong AMI hay trong file môi trường chứa một AWS key nào.

{{% notice warning %}}
**Groq API key là ngoại lệ, và là một ngoại lệ có thật.** Nó được lưu dạng văn bản thuần trong `/home/ubuntu/aws-rag-project/backend/.env.prod` trên instance. File này nằm trong `.gitignore`, thuộc sở hữu người dùng `ubuntu`, và không bao giờ rời khỏi máy - nhưng nó vẫn là một thông tin xác thực dạng plaintext nằm trên ổ đĩa.

Biện pháp khắc phục đã được thiết kế và viết code nhưng chưa triển khai: `app/aws_runtime_config.py` có thể nạp key từ **AWS Secrets Manager** thông qua chính instance role đó, loại bỏ hoàn toàn key khỏi ổ đĩa. Chương 5.7 mục 7 mô tả bước chuyển đổi này, còn mục 6 bên dưới liệt kê nó như một hạn chế còn để ngỏ chứ không phải một vấn đề đã giải quyết.
{{% /notice %}}

### 2. Một role phục vụ chỉ có quyền đọc

Inline policy trên `rag-ec2-runtime-role` cấp hai thứ, và cả hai đều chỉ đọc:

| Được cấp | Không được cấp |
| --- | --- |
| `s3:GetObject`, `s3:ListBucket` trên bucket artifact | Mọi `s3:PutObject` hay `s3:DeleteObject` |
| `s3vectors:QueryVectors`, `GetIndex`, `GetVectors`, `ListVectors`, `GetVectorBucket` | `PutVectors`, `DeleteVectors`, `CreateIndex`, `DeleteIndex` |

Cộng thêm managed policy `AmazonSSMManagedInstanceCore`, vốn chỉ tồn tại để Session Manager hoạt động.

Nhờ vậy dịch vụ đang chạy **chỉ có quyền đọc trên toàn bộ dữ liệu của chính nó**. Một lỗi trong ứng dụng không thể làm hỏng index hay ghi đè artifact, vì IAM sẽ từ chối lệnh gọi - bảo đảm này không phụ thuộc vào việc code có đúng hay không. Ảnh chứng minh nằm ở chương 5.7 mục 2.

{{% notice note %}}
Hai điểm cần nói cho sòng phẳng. Statement dành cho S3 phủ **cả bucket** thay vì chỉ prefix `rag/*`, và statement dành cho S3 Vectors dùng `"Resource": "*"` thay vì ARN của index. Cả hai đều rộng hơn mức cần thiết. Không điều nào làm suy yếu tính chất chỉ-đọc - vốn là phần liên quan tới bảo mật - nhưng một policy chặt chẽ hơn sẽ giới hạn từng statement về đúng prefix và ARN cụ thể. Điều này được liệt kê ở mục 6.
{{% /notice %}}

### 3. Truy cập quản trị mà không mở cổng SSH

Cổng `22` không được mở. Việc quản trị đi qua **Session Manager**, vốn không cần cổng inbound, không cần key pair, và ghi lại từng phiên làm việc. Cách này cũng loại bỏ một vấn đề vận hành lặp đi lặp lại: một luật security group gắn với "My IP" sẽ hỏng mỗi lần máy tính đổi mạng.

### 4. HTTPS đầu-cuối và danh sách CORS cho phép

Amplify phục vụ frontend qua HTTPS với chứng chỉ được quản lý sẵn, còn API Gateway kết thúc HTTPS ở phía trước backend. Trình duyệt không bao giờ phát ra một request HTTP thuần - đó chính là thành quả của phần xử lý Mixed Content ở chương 5.8 và 5.9.

Quyền truy cập từ trình duyệt bị giới hạn bằng một **danh sách origin tường minh ở hai tầng**: CORS của API Gateway, và `CORSMiddleware` của FastAPI lấy giá trị từ tham số `cors-allow-origins`. Không tầng nào dùng ký tự đại diện. Vì API không có xác thực, khác biệt giữa việc nêu tên đúng một origin và cho phép `*` là khác biệt thực chất chứ không phải hình thức.

### 5. Dữ liệu lúc lưu trữ

Cả hai tài nguyên lưu trữ đều ở chế độ riêng tư:

- Block Public Access được bật hoàn toàn trên bucket artifact; không object nào đọc được công khai.
- Mã hóa mặc định dùng SSE-S3 (`AES256`) trên bucket artifact, và S3 Vectors cũng áp dụng như vậy theo mặc định.
- Versioning được bật, nên nếu lỡ ghi đè một index mà production đang phục vụ thì vẫn khôi phục được.
- Không có bucket policy nào cấp quyền cho ai; mọi truy cập đều đi qua IAM identity policy.

### 6. Những hạn chế đã biết, nói thẳng

Đây là những điều có thật và không được trình bày như đã giải quyết xong:

| Hạn chế | Vì sao tồn tại | Production sẽ làm gì |
| --- | --- | --- |
| **Cổng `8000` tiếp cận được từ internet** | API Gateway là dịch vụ được quản lý, gọi từ các địa chỉ thuộc AWS nên không thể lọc theo IP | EC2 trong private subnet, sau **VPC Link** của API Gateway |
| **Không xác thực người dùng cuối** | Bản demo cố ý để ai cũng thử được | JWT authorizer, Lambda authorizer, hoặc API key trên HTTP API |
| **Không giới hạn tần suất** | Chưa cấu hình | Throttling trên stage của API Gateway; mỗi request còn tiêu tốn token Groq |
| **Một instance, một AZ** | Chọn như vậy để giữ demo rẻ | Auto Scaling group sau ALB, trải trên hai AZ |
| **Một phụ thuộc bên ngoài đi khỏi tài khoản** | Groq đảm nhiệm sinh câu trả lời | Amazon Bedrock, hoặc chấp nhận và ghi rõ ranh giới luồng dữ liệu |
| **Groq API key nằm plaintext trên instance** | Bước chuyển sang Secrets Manager đã có code nhưng chưa triển khai | Đặt `GROQ_SECRET_NAME` trong `.env.prod` và cấp `secretsmanager:GetSecretValue` cho role |
| **Statement IAM rộng hơn mức cần thiết** | S3 phủ cả bucket; S3 Vectors dùng `Resource: "*"` | Giới hạn S3 về prefix `rag/*` và S3 Vectors về ARN của index |

Nêu ra những điều này không phải là điểm yếu của báo cáo - một bản rà soát kiến trúc mà không liệt kê được rủi ro còn lại nào thì thường là chưa thực sự soi kỹ.

---

## Chi phí

### 7. Tiền thực sự đi đâu

| Nguồn chi phí | Hình thái tính tiền | Ghi chú |
| --- | --- | --- |
| **Amazon EC2** | **Liên tục khi đang chạy** | Khoản lớn nhất. Tính theo giờ bất kể có ai hỏi hay không |
| **Elastic IP / IPv4 công khai** | **Liên tục** | Địa chỉ IPv4 công khai bị tính theo giờ trong mọi trường hợp, kể cả khi instance đã dừng |
| **Ổ đĩa EBS** | **Liên tục** | Vẫn bị tính tiền trong lúc instance dừng |
| Amazon S3 | Theo GB lưu trữ + số request | Vài trăm megabyte artifact; không đáng kể ở quy mô này |
| Amazon S3 Vectors | Theo số vector lưu + số truy vấn | **Không có chi phí lúc rảnh** - không truy vấn thì không tính tiền |
| Amazon API Gateway | Theo request | HTTP API, khoảng một phần ba giá REST API |
| AWS Amplify Hosting | Phút build + lưu trữ + truyền tải | Nhỏ với một ứng dụng single-page |
| AWS Secrets Manager | Theo secret mỗi tháng | Có một secret nhưng **chưa được dùng** - xem mục 1 |
| **Groq API** *(bên ngoài)* | Theo token | Không phải chi phí AWS. Mỗi câu hỏi đều tiêu tốn token |

Điểm đáng chú ý: **chỉ tầng tính toán là bị tính tiền liên tục.** Lưu trữ và truy hồi về cơ bản là trả theo mức dùng, nên một hệ thống nhàn rỗi gần như không tốn gì sau khi instance được dừng lại.

### 8. Những lựa chọn thiết kế đồng thời cũng là lựa chọn chi phí

Nhiều quyết định được đưa ra vì lý do khác nhưng hóa ra lại là phương án rẻ, và điều đó đáng nói ra rõ ràng:

- **Bản build offline chạy trên Google Colab, không chạy trên AWS.** Bước tính toán tốn kém duy nhất - embed corpus bằng `BAAI/bge-m3` - không hề chạm vào hóa đơn AWS. Không phải cấp phát instance GPU hay job SageMaker nào (chương 5.4).
- **S3 Vectors thay vì một cơ sở dữ liệu vector được cấp phát sẵn.** OpenSearch Serverless tính phí công suất tối thiểu ngay cả khi rảnh; một kho vector tự quản lý thì cần instance riêng. S3 Vectors chỉ tính tiền lưu trữ và truy vấn.
- **API Gateway thay vì Application Load Balancer.** ALB tính theo giờ bất kể lưu lượng; HTTP API tính theo request.
- **Bộ reranker bị tắt trên production.** Chấm điểm bằng cross-encoder trên CPU chính là lý do từng phải cân nhắc một instance lớn hơn; tắt nó đi (chương 5.7 mục 9) giữ được instance ở mức nhỏ.
- **Fast mode bỏ bớt một lệnh gọi Groq cho mỗi câu hỏi.** Bỏ bước tách câu hỏi giúp giảm cả độ trễ lẫn lượng token tiêu tốn.
- **`/warmup` không gọi bước sinh câu trả lời của Groq.** Việc làm nóng nạp mô hình và chạy một lần truy hồi, cố ý dừng lại trước bước sinh câu trả lời có tính phí.

### 9. Biện pháp kiểm soát thực tế

- **Dừng instance EC2 khi không cần demo.** Đây là hành động có tác động lớn nhất. Lưu ý ổ đĩa EBS và Elastic IP vẫn tiếp tục tính tiền khi instance đã dừng - dừng máy giảm chi phí đáng kể nhưng không đưa về không.
- **Đừng để tích tụ nhiều phiên bản index.** Mỗi lần dựng lại corpus là tạo thêm một prefix có phiên bản trên S3 và một vector index mới. Giữ lại một phiên bản trước để có thể quay lui là hợp lý; giữ sáu phiên bản là lãng phí. Hãy xóa những cái không còn tham số SSM nào trỏ tới.
- **Thêm lifecycle rule cho các phiên bản object không hiện hành.** Bucket artifact đang bật versioning, nên mọi object bị ghi đè đều được giữ vô thời hạn trừ khi có luật cho hết hạn. Ba mươi ngày là mức mặc định hợp lý một khi bạn bắt đầu lặp nhiều lần trên index.
- **Giữ mọi thứ trong cùng một Region.** Lưu lượng cùng Region giữa EC2, S3 và S3 Vectors không bị tính là truyền tải dữ liệu; tách Region ra sẽ thêm phí vào mỗi lần tải artifact.
- **Đặt một AWS Budget kèm cảnh báo qua email.** Budget không ngăn được việc tiêu tiền, nhưng nó biến một bất ngờ cuối tháng thành một thông báo trong vòng một ngày.
- **Xóa bản triển khai khi dự án kết thúc.** Chương 5.14 hướng dẫn thứ tự thực hiện.

{{% notice tip %}}
Nguồn phát sinh chi phí bất ngờ phổ biến nhất trong một dự án kiểu này lại không phải dịch vụ mà ai cũng lo. Đó là một **Elastic IP còn được cấp phát sau khi instance đã bị terminate**, hoặc một ổ đĩa EBS thuộc về một instance đã quên mất. Cả hai đều âm thầm tính tiền, và cả hai đều không xuất hiện ở nơi bạn sẽ đi tìm "hệ thống RAG". Chương 5.14 kiểm tra đúng hai thứ này.
{{% /notice %}}

<!-- ẢNH 1 - SCREENSHOT.
     AWS Billing -> Budgets, hiển thị một budget theo tháng có đặt ngưỡng cảnh báo.
     Đây là bằng chứng trực tiếp cho tiêu chí tối ưu chi phí trong thang điểm.
     Nếu bạn chưa tạo thì hãy tạo ngay bây giờ - chỉ mất hai phút và thực sự hữu ích
     cho phần còn lại của kỳ thực tập.
     Làm mờ số tiền nếu bạn không muốn công khai. -->

![Budget theo tháng kèm cảnh báo](/images/5-Workshop/5.13-Security-Cost/aws-budget.png)

<!-- ẢNH 2 - SCREENSHOT (tùy chọn nhưng rất có sức thuyết phục).
     Cost Explorer, nhóm theo Service, lọc theo Region của dự án, trong khoảng thời gian
     thực hiện dự án.
     Điểm mấu chốt là HÌNH DẠNG của biểu đồ phân bổ - EC2 chiếm phần lớn, mọi thứ khác
     đều nhỏ - đúng như mục 7 khẳng định. Làm mờ số tiền nếu bạn muốn; tỉ lệ tương đối
     mới là thứ quan trọng. -->

![Phân bổ chi phí theo dịch vụ](/images/5-Workshop/5.13-Security-Cost/cost-explorer.png)

---

## Tổng kết

Bảo mật của bản triển khai này dựa trên ba điều được cưỡng chế chứ không phải hứa hẹn: không có thông tin xác thực tĩnh ở bất cứ đâu, một role phục vụ không thể ghi lên chính dữ liệu của nó, và không có cổng quản trị inbound nào. Những gì còn để ngỏ - một cổng backend tiếp cận được công khai và một API không xác thực - đã được ghi rõ ở trên chứ không bị bỏ qua.

Về chi phí, kiến trúc được định hình sao cho chỉ instance EC2 là bị tính tiền liên tục. Mọi thứ còn lại đều trả theo mức dùng hoặc miễn phí ở quy mô này, nghĩa là chỉ cần dừng một tài nguyên là chi phí vận hành gần như về không, và chương 5.14 sẽ đưa nó về đúng không.

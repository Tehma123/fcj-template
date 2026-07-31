---
title: "Triển khai Backend trên Amazon EC2"
date: 2026-07-31
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

FastAPI backend là lớp xử lý chính của CloudHop RAG. Backend nhận câu hỏi từ lớp API, tải các retrieval artifact đã chuẩn bị, thực hiện BM25 và dense retrieval, điều phối multi-hop pipeline, xây dựng context cuối cùng và gửi request sinh câu trả lời đến Groq.

Backend được triển khai trên **Amazon EC2**, cung cấp một môi trường vận hành ổn định để Python RAG pipeline và các dependency của hệ thống có thể được duy trì giữa các request.

Dữ liệu đã sẵn sàng. Chương này dựng lên thành phần tính toán duy nhất của hệ thống: một dịch vụ FastAPI trên EC2, nạp các artifact đó lúc khởi động và trả lời câu hỏi.

{{% notice info %}}
**Bạn sẽ có gì sau chương này:** một instance EC2 Ubuntu với Elastic IP cố định, một instance role cấp quyền chỉ-đọc tới hai dịch vụ dữ liệu, toàn bộ cấu hình runtime nằm trong một file `.env.prod` duy nhất, cùng một dịch vụ `systemd` sống sót qua reboot và tự làm nóng chính nó. `POST /query` sẽ trả lời được qua HTTP thuần; chương 5.8 sẽ đặt HTTPS ra phía trước.
{{% /notice %}}

Đây là chương dài nhất của workshop, và thứ tự các bước rất quan trọng - IAM role phải tồn tại trước khi khởi tạo instance, và cấu hình phải tồn tại trước khi dịch vụ khởi động.

---

## 1. Chọn kích thước máy và hình dạng của tải công việc

Dịch vụ này là **một tiến trình Python sống lâu duy nhất**. Lúc khởi động nó tải artifact từ S3, nạp `parent_docs.jsonl` và `child_docs.jsonl` vào bộ nhớ, nạp chỉ mục BM25, và nạp lười mô hình embedding `BAAI/bge-m3` ở lần dùng đầu tiên. Sau đó, mỗi request chỉ gồm truy hồi cộng vài lệnh gọi HTTP tới Groq.

Hình dạng đó chi phối mọi quyết định ở đây:

- **RAM mới là ràng buộc quyết định, không phải CPU.** Mô hình embedding chiếm phần lớn bộ nhớ. Một instance quá nhỏ không chạy chậm - nó bị OOM killer giết chết tiến trình Python, đúng kiểu lỗi đã gặp ở chương 5.4.
- **Tiến trình phải sống liên tục giữa các request.** Khởi động lại đồng nghĩa với tải lại artifact và nạp lại mô hình. Đó là lý do nó chạy dưới `systemd` chứ không phải như một script trong terminal.
- **Suy luận trên CPU là chấp nhận được ở đây** vì mô hình embedding chỉ mã hóa một câu truy vấn ngắn cho mỗi request, không phải cả corpus. `rag-device` được đặt là `cpu`; không cần instance GPU ở khâu phục vụ.

---

## 2. Tạo instance role trước tiên

Role phải tồn tại trước khi bạn khởi tạo instance, để gắn nó ngay lúc launch thay vì phải sửa chữa về sau.

Tạo một role tên **`rag-ec2-runtime-role`** với trusted entity là **EC2**, rồi gắn:

**a) Managed policy `AmazonSSMManagedInstanceCore`** - đây là thứ khiến Session Manager hoạt động (mục 5). Thiếu nó thì SSM Agent sẽ báo lỗi thông tin xác thực và bạn kẹt lại với SSH.

**b) Một inline policy cho hai dịch vụ dữ liệu mà ứng dụng sử dụng:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadRegularS3Artifacts",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::aws-rag-bucket-vanh1234",
        "arn:aws:s3:::aws-rag-bucket-vanh1234/*"
      ]
    },
    {
      "Sid": "QueryS3Vectors",
      "Effect": "Allow",
      "Action": [
        "s3vectors:GetVectorBucket",
        "s3vectors:GetIndex",
        "s3vectors:ListVectors",
        "s3vectors:QueryVectors",
        "s3vectors:GetVectors"
      ],
      "Resource": "*"
    }
  ]
}
```

Cả hai statement đều chỉ đọc. Dịch vụ đọc được artifact và truy vấn được vector; nó không thể ghi một object, đẩy một vector hay xóa một index. **Đây chính là thứ khiến câu "không hard-code thông tin xác thực AWS" thành sự thật chứ không phải khẩu hiệu** - ứng dụng không bao giờ nhìn thấy một AWS access key nào, vì SDK lấy thông tin xác thực tạm thời từ instance metadata service.

{{% notice note %}}
Policy này được trình bày đúng như nó đang là, không phải một phiên bản lý tưởng hóa. Có hai chỗ rộng hơn mức cần thiết: S3 được cấp trên cả bucket thay vì chỉ prefix `rag/*`, và S3 Vectors dùng `"Resource": "*"` thay vì ARN của index. Không chỗ nào cấp quyền ghi, nên bảo đảm chỉ-đọc vẫn đứng vững - nhưng cả hai đều được liệt kê như cơ hội làm chặt thêm ở chương 5.13.
{{% /notice %}}

## 3. Khởi tạo instance

| Thiết lập | Giá trị |
| --- | --- |
| AMI | Ubuntu Server (LTS) |
| Region | `ap-southeast-1` |
| Instance type | Đủ RAM cho mô hình embedding - xem mục 1 |
| Key pair | Tùy chọn; Session Manager thay thế SSH (mục 5) |
| IAM instance profile | **`rag-ec2-runtime-role`** |
| Security group | Xem mục 4 |
| Ổ đĩa | Đủ chỗ cho venv, PyTorch, cache mô hình và artifact tải về |

Sau đó **cấp phát một Elastic IP và gán nó cho instance.**

Bước này không phải thủ tục cho có. IP công khai mặc định sẽ đổi mỗi lần instance dừng rồi khởi động lại. Tích hợp của API Gateway ở chương 5.8 trỏ tới một địa chỉ cụ thể - khi địa chỉ đó đổi, API âm thầm chuyển tiếp tới hư không. Elastic IP cố định địa chỉ đó cho suốt vòng đời dự án.

```bash
aws ec2 allocate-address --domain vpc --region ap-southeast-1
aws ec2 associate-address --instance-id <instance-id> --allocation-id <allocation-id> --region ap-southeast-1
```

{{% notice note %}}
Địa chỉ IPv4 công khai bị tính phí theo giờ bất kể có dùng hay không, nên Elastic IP là một khoản chi phí nhỏ nhưng có thật trong suốt thời gian dự án tồn tại. Chương 5.14 sẽ giải phóng nó lúc dọn dẹp - một Elastic IP còn được cấp phát sau khi instance đã bị terminate thì vẫn tiếp tục tính tiền.
{{% /notice %}}

<!-- ẢNH 2 - SCREENSHOT.
     EC2 -> Instances -> instance của bạn -> tab Details.
     Phải thấy đồng thời: trạng thái "Running", instance type, Elastic IP nằm ở trường
     "Public IPv4 address", và trường "IAM Role" hiển thị rag-ec2-runtime-role.
     Làm mờ instance id và IP nếu bạn không muốn công khai chúng. -->

![Instance backend cùng Elastic IP và role của nó](/images/5-Workshop/5.7-EC2-Backend/ec2-instance.png)

---

## 4. Security group

Cấu hình của bản demo, nói thẳng:

```text
Inbound   Custom TCP  8000   0.0.0.0/0
Outbound  All traffic        0.0.0.0/0
```

Cổng `8000` được mở vì API Gateway (chương 5.8) là một dịch vụ được quản lý hoàn toàn, nó gọi backend từ các địa chỉ thuộc sở hữu của AWS chứ không phải từ một dải cố định mà bạn có thể đưa vào danh sách cho phép theo IP. Chiều outbound phải mở để đi tới S3, S3 Vectors, Systems Manager (cho Session Manager) và Groq API.

Cổng `22` **không** được mở. Mục 5 giải thích thứ thay thế nó.

{{% notice warning %}}
Đây là điểm yếu nhất của bản triển khai, và báo cáo nói thẳng điều đó. Bất kỳ ai tìm ra địa chỉ đều có thể gọi API, và API không có xác thực. Kiến trúc production đúng đắn là đặt instance EC2 trong **private subnet**, chỉ tiếp cận được qua **VPC Link** của API Gateway, để cổng `8000` không bao giờ bị phơi ra. Điều này được ghi lại như một hạn chế đã biết ở chương 5.13 chứ không bị lặng lẽ bỏ qua.
{{% /notice %}}

---

## 5. Truy cập quản trị mà không cần SSH

Mở cổng `22` cho "My IP" thì dùng được cho tới khi bạn đổi mạng - một mạng Wi-Fi khác nghĩa là một IP công khai mới và một luật đã hỏng. **AWS Systems Manager Session Manager** xóa bỏ vấn đề đó hoàn toàn: không cổng inbound, không key pair, và có bản ghi kiểm toán cho mọi phiên.

Cài và bật agent trên Ubuntu:

```bash
sudo snap install amazon-ssm-agent --classic
sudo systemctl enable snap.amazon-ssm-agent.amazon-ssm-agent.service
sudo systemctl start snap.amazon-ssm-agent.amazon-ssm-agent.service
sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service
```

Xác nhận instance thực sự đang dùng role, thông qua IMDSv2:

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" -s)

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Kết quả mong đợi:

```text
rag-ec2-runtime-role
```

Kết nối từ Console: **EC2 → Instances → chọn instance → Connect → Session Manager**. Phiên mở ra dưới người dùng `ssm-user`; hãy chuyển sang người dùng của dự án:

```bash
sudo su - ubuntu
```

{{% notice tip %}}
Nếu agent ghi log lỗi thông tin xác thực, nguyên nhân gần như luôn là role thiếu `AmazonSSMManagedInstanceCore`. Gắn nó vào, rồi khởi động lại agent - log khi đó phải hiện `Successfully connected with instance profile role credentials`.
{{% /notice %}}

<!-- ẢNH 3 - SCREENSHOT.
     Một phiên Session Manager đang mở trên trình duyệt, sau khi chạy `sudo su - ubuntu`
     và `sudo systemctl status aws-rag-api`.
     Phải thấy được hai thứ: chính giao diện Session Manager (chứng minh không dùng SSH)
     và dịch vụ đang ở trạng thái "active (running)". -->

![Kết nối qua Session Manager](/images/5-Workshop/5.7-EC2-Backend/session-manager.png)

---

## 6. Cài đặt ứng dụng

```bash
cd ~
git clone https://github.com/vietanh1802/aws-rag-project.git
cd ~/aws-rag-project/backend

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Chỉ cần `requirements.txt`. Bộ dependency cho khâu offline ở chương 5.4 **không** được cài ở đây - máy này không dựng gì cả, nên không cần bộ công cụ dataset hay stack lúc build. Giữ môi trường phục vụ ở mức tối thiểu chính là thứ cho phép một instance khiêm tốn vẫn giữ được mô hình trong bộ nhớ.

Đường dẫn dự án dùng xuyên suốt workshop này:

```text
/home/ubuntu/aws-rag-project/backend
```

---

## 7. Cấu hình bằng `.env.prod`

Toàn bộ cấu hình runtime nằm trong một file duy nhất trên instance:

```text
/home/ubuntu/aws-rag-project/backend/.env.prod
```

`systemd` truyền file này cho tiến trình qua `EnvironmentFile=` (mục 8), còn ứng dụng chỉ đọc biến môi trường thông thường - `advanced_rag/config.py` phân giải mọi tham số qua `os.environ.get()` kèm giá trị mặc định hợp lý. Không có đường đi nào khác cho cấu hình ngoài cách đó.

```env
USE_TF=0
USE_FLAX=0
AWS_REGION=ap-southeast-1
GROQ_API_KEY=<your-groq-key>

S3_ARTIFACT_BUCKET=aws-rag-bucket-vanh1234
S3_VECTOR_BUCKET=rag-vectors-vanh1234
RAG_INDEX_ID=hotpotqa-val500-bge-m3-v002
S3_PROCESSED_ID=hotpotqa-val500-v002
S3_VECTOR_INDEX=hotpotqa-val500-bge-m3-v002
RAG_ARTIFACT_LAYOUT=s3vectors
RAG_AUTO_DOWNLOAD_ARTIFACTS=true

RAG_DEVICE=cpu
RAG_USE_RERANKER=false
RAG_FAST_MODE=true
BM25_TOP_K=15
VECTOR_TOP_K=15
RERANK_TOP_N=5
HOP_CANDIDATE_CAP=15
MAX_ADAPTIVE_HOPS=3
HOP_EVIDENCE_TOP_N=3
RAG_WARMUP_QUESTION=Were Scott Derrickson and Ed Wood of the same nationality?
```

Từng nhóm điều khiển gì:

| Nhóm | Biến | Tác dụng |
| --- | --- | --- |
| Runtime | `USE_TF`, `USE_FLAX`, `AWS_REGION` | Giữ `transformers` chỉ chạy PyTorch; region AWS cho mọi client SDK |
| Thông tin xác thực | `GROQ_API_KEY` | Bí mật duy nhất. Truy cập AWS không cần key - instance role đã lo |
| Vị trí artifact | `S3_ARTIFACT_BUCKET`, `S3_PROCESSED_ID`, `S3_VECTOR_BUCKET`, `S3_VECTOR_INDEX`, `RAG_INDEX_ID`, `RAG_ARTIFACT_LAYOUT`, `RAG_AUTO_DOWNLOAD_ARTIFACTS` | Corpus và index nào được phục vụ, và có tải chúng lúc khởi động không |
| Độ trễ / chất lượng | `RAG_FAST_MODE`, `RAG_USE_RERANKER`, `RAG_DEVICE` | Tốc độ so với chất lượng câu trả lời - xem mục 9 |
| Ngân sách truy hồi | `BM25_TOP_K`, `VECTOR_TOP_K`, `RERANK_TOP_N`, `HOP_CANDIDATE_CAP`, `MAX_ADAPTIVE_HOPS`, `HOP_EVIDENCE_TOP_N` | Tìm kiếm rộng và sâu đến đâu |
| Vận hành | `RAG_WARMUP_QUESTION` | Câu hỏi mà `/warmup` chạy |

{{% notice warning %}}
**`.env.prod` chứa Groq API key ở dạng văn bản thuần, và đây là điểm yếu nhất của cấu hình hiện tại.** File này được loại khỏi git, chỉ người dùng `ubuntu` đọc được, và không bao giờ rời khỏi instance - nhưng một thông tin xác thực nằm dạng plaintext trên đĩa thì vẫn là một thông tin xác thực nằm trên đĩa. Bất kỳ ai có được shell trên instance, hoặc một bản sao của ổ đĩa, đều có được key.

Tuyệt đối không được commit file này. Hãy xác nhận `.env.prod` đã nằm trong `.gitignore` trước lần push đầu tiên.
{{% /notice %}}

**Hướng cải thiện đã dự định, nhưng chưa thực hiện.** Repository đã có sẵn `app/aws_runtime_config.py`, module này có thể nạp các thiết lập không bí mật từ **AWS Systems Manager Parameter Store** và nạp Groq key từ **AWS Secrets Manager**, dùng instance role thay vì một file trên đĩa. Bộ nạp hoạt động theo cơ chế opt-in: nó chỉ kích hoạt khi `CONFIG_PREFIX` hoặc `GROQ_SECRET_NAME` có mặt trong file môi trường.

Bản triển khai này **chưa** thực hiện bước chuyển đó. Nó được ghi lại ở đây như bước tiếp theo đã lên kế hoạch, chứ không trình bày như đã hoàn thành, và chương 5.13 liệt kê nó trong các hạn chế đã biết. Lợi ích sẽ là hai mặt: bí mật rời khỏi ổ đĩa, và việc đổi cấu hình trở thành một lần cập nhật tham số thay vì một phiên SSH kèm thao tác sửa file.

<!-- ẢNH 4 - SCREENSHOT.
     Terminal Session Manager hiển thị file cấu hình thật, với dòng GROQ_API_KEY
     ĐÃ BỊ LOẠI BỎ. Cách chụp an toàn:
         grep -v GROQ_API_KEY .env.prod
     Hãy chụp output của lệnh đó thay vì dùng `cat`.
     Tuyệt đối không publish ảnh có chứa giá trị key. -->

![File môi trường của production](/images/5-Workshop/5.7-EC2-Backend/env-prod.png)

## 8. Chạy dưới systemd

Chạy `uvicorn` trong terminal thì nó chết theo terminal. `systemd` cho ta khởi động lại khi lỗi, tự chạy lúc boot, và một luồng log.

File service tại `/etc/systemd/system/aws-rag-api.service`:

```ini
[Unit]
Description=AWS RAG API Service
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/aws-rag-project/backend
EnvironmentFile=/home/ubuntu/aws-rag-project/backend/.env.prod
ExecStart=/home/ubuntu/aws-rag-project/backend/.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000
Restart=always
TimeoutStartSec=300
ExecStartPost=/bin/bash -lc 'sleep 20; curl -fsS -X POST http://127.0.0.1:8000/warmup || true'

[Install]
WantedBy=multi-user.target
```

Ba dòng đáng được giải thích:

- **`--host 0.0.0.0`** - lắng nghe trên mọi giao diện mạng để API Gateway tiếp cận được cổng. Nếu buộc vào `127.0.0.1` thì dịch vụ không thể truy cập từ bên ngoài.
- **`TimeoutStartSec=300`** - lần khởi động đầu tiên phải tải artifact và nạp mô hình. Timeout mặc định sẽ giết nó giữa chừng.
- **`ExecStartPost=... /warmup`** - sau khi dịch vụ lên, chính systemd gọi endpoint làm nóng. Phần `|| true` đảm bảo một lần làm nóng thất bại không khiến dịch vụ bị đánh dấu là lỗi.

Bật và khởi động:

```bash
sudo systemctl daemon-reload
sudo systemctl enable aws-rag-api
sudo systemctl restart aws-rag-api
sudo systemctl status aws-rag-api
sudo journalctl -u aws-rag-api -f
```

Chính `enable` mới là thứ khiến dịch vụ quay lại sau khi instance khởi động lại.

---

## 9. Fast mode và warm-up: lọt vừa vào giới hạn timeout

Đây là bài toán tinh chỉnh giàu tính bài học nhất của dự án, và nó xuất phát từ một phép đo chứ không phải phỏng đoán.

**Quan sát được.** Một request thành công từ frontend đo được khoảng **26,5 giây**. Giới hạn timeout tích hợp của API Gateway vào khoảng 30 giây. Hệ thống chạy được, nhưng gần như không còn biên an toàn - và request đầu tiên sau mỗi lần khởi động lại còn chậm hơn nhiều, vì artifact và mô hình embedding được nạp lười ở lần dùng đầu tiên.

**Hai biện pháp độc lập:**

**`RAG_FAST_MODE=true`** tấn công vào chi phí ở trạng thái ổn định. Nó bỏ qua bước tách câu hỏi - loại bỏ một vòng gọi Groq ngay trước khi truy hồi bắt đầu - và thu nhỏ ngân sách truy hồi:

```env
RAG_FAST_MODE=true
BM25_TOP_K=15
VECTOR_TOP_K=15
RERANK_TOP_N=5
HOP_CANDIDATE_CAP=15
MAX_ADAPTIVE_HOPS=1
HOP_EVIDENCE_TOP_N=3
```

Bộ rerank cross-encoder cũng được tắt (`rag-use-reranker=false`), vì chấm điểm ứng viên trên CPU rất tốn kém.

**`POST /warmup`** tấn công vào chi phí của request đầu tiên. Nó nạp pipeline và thực hiện một lần truy hồi vector thật để mô hình embedding, artifact và client S3 Vectors đều nằm sẵn trong bộ nhớ - nhưng nó cố ý **không gọi sinh câu trả lời của Groq**, nên việc làm nóng không tiêu tốn token sinh câu trả lời nào:

```json
{
  "status": "ok",
  "pipeline_loaded": true,
  "vector_backend": "s3vectors",
  "elapsed_seconds": 4.2,
  "warmup_child_hits": 12
}
```

Kết hợp với `ExecStartPost`, mỗi lần khởi động lại là hệ thống tự làm nóng. Người dùng không bao giờ phải trả giá cho lần nạp nguội, trừ khi họ đến đúng vài giây đầu sau một lần restart.

{{% notice note %}}
Fast mode là một cuộc **đánh đổi chất lượng lấy độ trễ** một cách tường minh, và điều đó đáng được gọi đúng tên. Nó truy hồi ít ứng viên hơn và bỏ qua lập kế hoạch đa bước, nên một số câu hỏi khó sẽ có câu trả lời tệ hơn. Với một bản demo buộc phải phản hồi ổn định trong một giới hạn timeout cứng, độ trễ dự đoán được có giá trị hơn độ chính xác đỉnh cao. Chương 5.11 đo cả hai chế độ để mức đánh đổi trở thành một con số chứ không phải một ý kiến.
{{% /notice %}}

---

## 10. Kiểm tra

**Health** - endpoint này báo cáo xem cấu hình tập trung có thực sự nạp được không:

```bash
curl http://127.0.0.1:8000/health
```

```json
{"status":"ok","pipeline_loaded":false}
```

Hai trường, cả hai đều có ý nghĩa:

| Trường | Ý nghĩa |
| --- | --- |
| `status: ok` | Tiến trình đang sống và trả lời được. Nó **không** có nghĩa là pipeline đã sẵn sàng |
| `pipeline_loaded: false` | Bình thường khi chưa warm-up - artifact và mô hình embedding được nạp lười ở lần dùng đầu |
| `pipeline_loaded: true` | Artifact, chỉ mục BM25 và mô hình embedding đã nằm trong bộ nhớ; truy vấn sẽ nhanh |

Vì cấu hình đến từ `.env.prod`, một thiết lập sai sẽ **không** lộ ra ở đây - nó chỉ lộ ra dưới dạng lỗi ở truy vấn đầu tiên. Đó là nhược điểm thật của cách cấu hình bằng file, và cũng là lý do endpoint health của một bản triển khai dùng Parameter Store báo cáo được nhiều hơn (mục 7).

**Làm nóng, rồi truy vấn:**

```bash
curl -X POST http://127.0.0.1:8000/warmup

curl -X POST http://127.0.0.1:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question":"Were Scott Derrickson and Ed Wood of the same nationality?"}'
```

Từ máy Windows, gọi tới địa chỉ công khai:

```powershell
curl.exe "http://<elastic-ip>:8000/health"

'{"question":"Were Scott Derrickson and Ed Wood of the same nationality?"}' | Set-Content -Encoding utf8 query.json
curl.exe -s -X POST "http://<elastic-ip>:8000/query" `
  -H "Content-Type: application/json" `
  --data-binary "@query.json"
```

Phản hồi mang theo câu trả lời cùng mọi thứ cần thiết để biện minh cho nó - `sources`, `timings`, `num_candidates` và `token_usage_total`.

---

## 11. Phục vụ một index khác

Đổi corpus hoặc index không cần sửa code, không cần AMI mới, không cần đổi frontend - chỉ cần sửa file môi trường rồi khởi động lại:

```bash
cd ~/aws-rag-project/backend
cp .env.prod .env.prod.backup      # luôn làm, trước khi sửa
nano .env.prod
```

Đổi ba định danh quyết định index nào được chọn:

```env
RAG_INDEX_ID=<index-id-moi>
S3_PROCESSED_ID=<processed-id-moi>
S3_VECTOR_INDEX=<vector-index-moi>
```

```bash
sudo systemctl restart aws-rag-api
curl http://127.0.0.1:8000/health
curl -X POST http://127.0.0.1:8000/warmup
```

Vì chương 5.4 đánh phiên bản cho mọi định danh, việc quay lui cũng chính là thao tác sửa đó với giá trị cũ - hoặc đơn giản là khôi phục file backup.

Cái giá của cách làm này là phải mở một phiên shell trên instance, và đó đúng là phần ma sát mà bước chuyển sang Parameter Store mô tả ở mục 7 sẽ loại bỏ.

## 12. Vận hành hằng ngày

1. Khởi động instance và chờ status check đạt.
2. Kết nối bằng Session Manager.
3. `sudo systemctl status aws-rag-api` và `curl http://127.0.0.1:8000/health`.
4. Nếu cần, `curl -X POST http://127.0.0.1:8000/warmup`.
5. Kiểm tra qua API Gateway (chương 5.8), rồi mở frontend (chương 5.9).

Log nằm trong journal:

```bash
sudo journalctl -u aws-rag-api -n 200 --no-pager
sudo journalctl -u aws-rag-api -f
```

---

## 13. Các lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| Dịch vụ không khởi động được, log kết thúc bằng `Killed` | Hết RAM lúc nạp mô hình embedding | Instance lớn hơn, hoặc mô hình nhỏ hơn |
| `systemd` báo khởi động quá thời gian | Tải artifact cộng nạp mô hình vượt timeout mặc định | `TimeoutStartSec=300` |
| Lỗi xác thực Groq ở truy vấn đầu tiên | `GROQ_API_KEY` thiếu hoặc đã cũ trong `.env.prod` | Kiểm tra file, rồi khởi động lại dịch vụ |
| Một biến trong `.env.prod` có vẻ bị bỏ qua | File được ghi kèm BOM UTF-8, khiến tên biến đầu tiên mang tiền tố vô hình `\ufeff` | Ghi lại file không kèm BOM |
| `FileNotFoundError` ở truy vấn đầu tiên | Artifact thiếu hoặc nằm sai prefix S3 | Kiểm tra lại chương 5.5 mục 7 |
| Truy hồi trả về kết quả vô nghĩa | Index dựng bằng mô hình embedding khác | Xem chương 5.6 mục 5 |
| Chạy được cục bộ nhưng không truy cập được từ ngoài | `uvicorn` buộc vào `127.0.0.1` | Dùng `--host 0.0.0.0` |
| API Gateway hỏng sau một lần stop/start | IP công khai đã đổi | Elastic IP, mục 3 |
| Request đầu tiên sau restart bị timeout | Khởi động nguội | Warm-up bằng `ExecStartPost`, mục 9 |

---

## 14. Kết quả

Một backend tự chạy lúc boot, tự khởi động lại khi lỗi, tự làm nóng, đọc toàn bộ cấu hình từ AWS, và không giữ bất kỳ thông tin xác thực nào. Nó trả lời `POST /query` qua HTTP trên cổng `8000`.

Chính chữ cuối cùng đó là vấn đề còn lại: **HTTP**. Frontend sẽ được phục vụ qua HTTPS, và trình duyệt sẽ từ chối cho một trang HTTPS gọi tới API HTTP thuần. Chương 5.8 sẽ đặt API Gateway ra phía trước để kết thúc TLS.

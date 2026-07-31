---
title: "Amazon S3 Vectors"
date: 2026-07-31
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

BM25 hoạt động hiệu quả khi câu hỏi có nhiều từ hoặc cụm từ trùng với tài liệu hỗ trợ, nhưng các câu hỏi đa bước cũng có thể cần những bằng chứng được diễn đạt theo cách khác. Vì vậy, CloudHop RAG kết hợp lexical retrieval với dense semantic retrieval thông qua **Amazon S3 Vectors**.

Mỗi child chunk được tạo trong quá trình offline build được mã hóa bằng **BGE-M3** thành vector 1.024 chiều. Các vector này được lưu trong S3 Vectors index và được EC2 backend truy vấn tại runtime bằng embedding của câu hỏi hoặc retrieval query tương ứng.

Chương 5.5 đã upload phần văn bản của chỉ mục. Chương này tạo kho vector, nạp các embedding đã sinh ra ở chương 5.4, và chứng minh rằng truy hồi theo ngữ nghĩa hoạt động - tất cả đều trước khi có bất kỳ backend nào.

{{% notice info %}}
**Bạn sẽ có gì sau chương này:** một vector bucket chứa một index gồm các embedding 1024 chiều, đã được nạp đầy từ các batch `put_vectors_*.json`, cùng một bài kiểm tra truy hồi chạy thành công ngay trên máy của bạn. Sau chương này, cả hai nửa của bộ truy hồi lai đều đã có dữ liệu.
{{% /notice %}}

---

## 1. Amazon S3 Vectors là gì, và không phải là gì

S3 Vectors là một dịch vụ riêng biệt, chỉ tình cờ mang chữ S3 trong tên. Nó **không phải** là một bucket có chứa vector bên trong.

| | Amazon S3 | Amazon S3 Vectors |
| --- | --- | --- |
| Không gian API | `s3` | `s3vectors` |
| Đơn vị lưu trữ | Object dưới một key | Vector dưới một key, nằm trong một *index* |
| Truy vấn | `GetObject` theo key chính xác | `QueryVectors` - tìm láng giềng gần nhất xấp xỉ |
| Xuất hiện ở | Danh sách bucket S3 | Mục **Vector buckets** riêng biệt |
| Tạo ở chương | 5.5 | Chương này |

Cấu trúc tài nguyên có hai cấp: một **vector bucket** chứa một hoặc nhiều **index**, và vector nằm bên trong index. Chính **index** - chứ không phải bucket - mới mang số chiều và chỉ số khoảng cách, và đó là lý do một bucket có thể chứa nhiều index dựng bằng các mô hình embedding khác nhau.

{{% notice warning %}}
Khả năng khả dụng tùy theo region và hẹp hơn S3 thông thường. Hãy xác nhận S3 Vectors có mặt ở region bạn chọn trước khi cam kết với nó - dự án này dùng `ap-southeast-1`. Nếu các lệnh `s3vectors` không được nhận diện thì hãy cập nhật AWS CLI; phiên bản API dùng ở đây là `2025-07-15`.
{{% /notice %}}

---

## 2. Tạo vector bucket

**Console:** S3 → **Vector buckets** → Create vector bucket

**CLI:**

```bash
aws s3vectors create-vector-bucket \
  --vector-bucket-name rag-vectors-vanh1234 \
  --region ap-southeast-1
```

Chỉ có tên là bắt buộc. Mã hóa mặc định là **SSE-S3 (`AES256`)**, cùng thế trận bảo mật với bucket artifact ở chương 5.5; truyền thêm `encryptionConfiguration` kèm `kmsKeyArn` sẽ chuyển sang SSE-KMS nếu corpus của bạn đủ nhạy cảm để cần.

Tên bucket tuân theo cùng quy tắc như trước - viết thường, duy nhất trong phạm vi dịch vụ, và không đổi tên được.

<!-- ẢNH 1 - SCREENSHOT.
     S3 Console -> mục "Vector buckets" -> form Create vector bucket (hoặc bucket vừa tạo
     trong danh sách).
     Quan trọng: để lộ THANH ĐIỀU HƯỚNG BÊN TRÁI, cho thấy "Vector buckets" là một mục
     tách biệt khỏi "General purpose buckets". Riêng chi tiết đó đã trả lời được câu hỏi
     mà hầu hết người đọc đang thắc mắc ở đoạn này. -->

![Tạo vector bucket](/images/5-Workshop/5.6-S3-Vectors/create-vector-bucket.png)

---

## 3. Tạo index

Đây là bước mà một giá trị sai sẽ bắt bạn phải dựng lại từ đầu, vì **số chiều và chỉ số khoảng cách của index bị cố định ngay lúc tạo**.

```bash
aws s3vectors create-index \
  --vector-bucket-name rag-vectors-vanh1234 \
  --index-name hotpotqa-val500-bge-m3-v002 \
  --data-type float32 \
  --dimension 1024 \
  --distance-metric cosine \
  --region ap-southeast-1
```

Từng tham số, và giá trị của nó đến từ đâu:

| Tham số | Giá trị | Vì sao |
| --- | --- | --- |
| `--index-name` | `hotpotqa-val500-bge-m3-v002` | Đúng bằng `INDEX_ID` ở chương 5.4. Backend truyền một định danh duy nhất làm cả prefix S3 lẫn tên vector index - hai cái phải giống hệt nhau. |
| `--data-type` | `float32` | Giá trị duy nhất mà API chấp nhận, và cũng đúng thứ notebook đã ghi: `np.asarray(vectors, dtype='float32')`. |
| `--dimension` | `1024` | Độ rộng đầu ra của `BAAI/bge-m3`. API cho phép từ 1 đến 4096. Con số này phải khớp chính xác với mô hình embedding - nó cũng chính là con số mà notebook đã assert. |
| `--distance-metric` | `cosine` | Phải khớp với cách vector được dựng. Chương 5.4 dùng `normalize_embeddings=True`, và cosine là chỉ số đi cặp với vector đã chuẩn hóa về độ dài đơn vị. Lựa chọn còn lại là `euclidean`. |

{{% notice warning %}}
**Số chiều và chỉ số khoảng cách không sửa được về sau.** Nếu bạn tạo index sai số chiều, `PutVectors` sẽ lỗi `ValidationException` ngay ở batch đầu tiên - và đó là kịch bản may mắn. Kịch bản tệ là tạo index với `euclidean` trong khi vector của bạn đã chuẩn hóa: nạp vẫn thành công, truy vấn vẫn thành công, và thứ hạng thì sai một cách âm thầm. Hãy kiểm tra giá trị này ngay bây giờ, đừng đợi đến khi kết quả đánh giá ở chương 5.11 trả về mức tầm thường.
{{% /notice %}}

**Tùy chọn - non-filterable metadata keys.** `create-index` chấp nhận một `metadataConfiguration` liệt kê tối đa 10 khóa metadata được lưu và trả về nhưng không dùng làm bộ lọc truy vấn được:

```bash
  --metadata-configuration nonFilterableMetadataKeys=title,source_doc_id
```

Dự án này không đặt tham số đó. Mục 6 sẽ giải thích lý do: bộ truy hồi không bao giờ đọc metadata từ kho vector.

<!-- ẢNH 2 - SCREENSHOT.
     Form Create index (hoặc trang chi tiết index sau khi tạo xong), hiển thị rõ cả bốn
     thiết lập: index name, data type float32, dimension 1024, distance metric cosine.
     Đây là tấm ảnh giá trị nhất của chương - bốn giá trị này chính là bản hợp đồng
     giữa chương 5.4 và chương 5.7. -->

![Tạo vector index](/images/5-Workshop/5.6-S3-Vectors/create-index.png)

---

## 4. Nạp vector

Chương 5.4 đã sinh ra `ingest_s3vectors.py` với bucket, index và region của bạn điền sẵn bên trong. Chạy nó từ trong thư mục upload:

```bash
cd s3_manual_upload/hotpotqa-val500-bge-m3-v002
python ingest_s3vectors.py --region ap-southeast-1
```

Script duyệt các file batch theo thứ tự và gọi một lệnh `PutVectors` cho mỗi file:

```python
client = boto3.client("s3vectors", region_name=args.region)

for path in sorted(import_dir.glob("put_vectors_*.json")):
    payload = json.loads(path.read_text(encoding="utf-8"))
    client.put_vectors(
        vectorBucketName=args.vector_bucket_name,
        indexName=args.index_name,
        vectors=payload["vectors"],
    )
```

Output mong đợi:

```text
uploaded 500 vectors from put_vectors_0000.json
uploaded 500 vectors from put_vectors_0001.json
...
uploaded 279 vectors from put_vectors_0016.json
done: 8279 vectors uploaded
```

{{% notice note %}}
**Vì sao mỗi batch 500 không phải con số tùy tiện.** API `PutVectors` chấp nhận tối đa **500 vector cho mỗi request** - đây là giới hạn cứng của dịch vụ, không phải một núm điều chỉnh. Chương 5.4 chia batch đúng bằng con số đó để tận dụng trọn vẹn mỗi request mà không bao giờ vượt trần. Lợi ích thực tế là bước nạp có thể chạy tiếp được: nếu một lệnh gọi hỏng giữa chừng, bạn chạy lại từ batch bị hỏng thay vì embed lại toàn bộ corpus.
{{% /notice %}}

`PutVectors` là idempotent theo key - chạy lại một batch sẽ ghi đè đúng những vector đó chứ không nhân bản, vì `key` (chính là `child_id`) định danh duy nhất một vector trong một index. Một lần nạp thất bại có thể chạy lại từ đầu một cách an toàn.

<!-- ẢNH 3 - SCREENSHOT.
     Terminal đang chạy ingest_s3vectors.py, hiển thị vài dòng "uploaded 500 vectors from
     put_vectors_XXXX.json" và dòng cuối "done: N vectors uploaded".
     Chụp cho rõ con số tổng cuối cùng - nó phải bằng đúng số "child docs" mà cell kiểm tra
     ở chương 5.4 đã in ra. -->

![Nạp vector vào index](/images/5-Workshop/5.6-S3-Vectors/ingest-output.png)

---

## 5. Kiểm tra

**Kiểm tra cấu hình index** - xác nhận lại bốn giá trị bạn vừa cam kết:

```bash
aws s3vectors get-index \
  --vector-bucket-name rag-vectors-vanh1234 \
  --index-name hotpotqa-val500-bge-m3-v002 \
  --region ap-southeast-1
```

**Kiểm tra vector đã vào chưa:**

```bash
aws s3vectors list-vectors \
  --vector-bucket-name rag-vectors-vanh1234 \
  --index-name hotpotqa-val500-bge-m3-v002 \
  --max-results 5 \
  --region ap-southeast-1
```

Các key trả về phải là những giá trị `child_id` khớp với nội dung trong `child_docs.jsonl`. `--max-results` nhận tối đa 1000.

**Rồi hãy làm phép kiểm tra thực sự có ý nghĩa.** Liệt kê vector chỉ chứng minh dữ liệu tồn tại; nó không chứng minh truy hồi hoạt động. Dự án có sẵn một script mã hóa một câu hỏi thật, truy vấn index và in ra các tài liệu thu về - không gọi Groq, không cần backend:

```bash
cd backend
python scripts/check_s3vectors_retrieval.py \
  --download-artifacts \
  --device cpu \
  "Were Scott Derrickson and Ed Wood of the same nationality?"
```

Đây là bằng chứng đầu-cuối đầu tiên cho thấy chương 5.4, 5.5 và 5.6 khớp với nhau: nó tải artifact từ S3, nạp `child_docs.jsonl`, mã hóa câu hỏi bằng `BAAI/bge-m3`, truy vấn S3 Vectors, rồi ánh xạ các key trả về thành tài liệu thật. Nếu tiêu đề các tài liệu thu về đúng chủ đề thì cả ba chương đều chính xác.

{{% notice tip %}}
Nếu kết quả trả về là những tài liệu chẳng liên quan gì đến câu hỏi, nguyên nhân gần như luôn là một trong ba thứ sau, theo thứ tự khả năng: index được dựng bằng mô hình embedding khác với mô hình đang mã hóa truy vấn, chỉ số khoảng cách không khớp với cách vector được chuẩn hóa, hoặc `child_docs.jsonl` trên S3 đến từ một bản build khác với bản đã sinh ra vector. Cả ba đều là lỗi cấu hình lệch nhau, không phải vấn đề chất lượng truy hồi - đừng vội đi chỉnh `top-k`.
{{% /notice %}}

<!-- ẢNH 4 - SCREENSHOT.
     Output terminal của check_s3vectors_retrieval.py, hiển thị câu hỏi và các tài liệu
     thu về kèm tiêu đề của chúng.
     Chụp đủ số dòng để người đọc thấy được các tiêu đề đúng chủ đề với câu hỏi - chính
     sự liên quan đó mới là bằng chứng, không phải việc script chạy được. -->

![Truy hồi ngữ nghĩa hoạt động trên index](/images/5-Workshop/5.6-S3-Vectors/retrieval-check.png)

---

## 6. Backend dùng index này như thế nào

Chỗ này đáng nói cho chính xác, vì thiết kế ở đây không hiển nhiên như vẻ ngoài của nó. Bộ truy hồi online truy vấn S3 Vectors như sau:

```python
response = self._get_client().query_vectors(
    vectorBucketName=self.vector_bucket_name,
    indexName=self.index_name,
    queryVector={"float32": self._encode_query(query)},
    topK=min(self.k, len(self.docs_by_id)),
    returnDistance=self.return_distance,
)

for item in response.get("vectors", []):
    child_id = item.get("key")
    doc = self.docs_by_id.get(child_id)
    ...
```

Hãy để ý nó lấy gì từ phản hồi: **chỉ `key` và `distance`, ngoài ra không gì khác.** Nội dung văn bản của tài liệu không hề được lưu trong vector index - nó đến từ `child_docs.jsonl`, vốn đã được tải về từ S3 thông thường lúc khởi động và đang nằm sẵn trong bộ nhớ, đánh chỉ mục theo `child_id`.

Cách tách bạch đó là có chủ đích và kéo theo ba hệ quả:

- **Vector index không lưu văn bản.** Nó chỉ giữ 1024 số thực và một key ngắn cho mỗi đoạn. Dung lượng lưu trữ nhỏ và rẻ, và không có nội dung tài liệu nào bị nhân bản qua hai dịch vụ.
- **`returnMetadata` không bao giờ được yêu cầu.** Phần metadata ghi lúc nạp (`parent_id`, `title`, `source_doc_id`) thực chất là khoản dự phòng - hữu ích khi cần soi index bằng tay hoặc cho một truy vấn có lọc trong tương lai, nhưng không nằm trên đường đi nóng. Đó là lý do `nonFilterableMetadataKeys` được bỏ trống ở mục 3.
- **Kho vector thuần túy là một chỉ mục tương đồng.** Nó trả lời câu hỏi "những chunk id nào gần nhất với vector truy vấn này", còn mọi thứ khác được giải quyết cục bộ. Thay S3 Vectors bằng một cơ sở dữ liệu vector khác chỉ có nghĩa là viết lại đúng một module - `s3vectors_retriever.py` - và không đụng gì thêm.

Vector truy vấn được mã hóa với đúng cách chuẩn hóa đã dùng lúc build:

```python
vector = self._get_model().encode(query, normalize_embeddings=True, convert_to_numpy=True)
```

Chuẩn hóa lúc build và lúc truy vấn phải khớp nhau, và cả hai phải khớp với chỉ số khoảng cách của index. Sự đồng thuận ba chiều đó là bất biến quan trọng nhất của chương này.

---

## 7. Phân quyền

Hai chủ thể, hai mức truy cập rất khác nhau.

**Bạn, trong chương này** - tạo và nạp dữ liệu:

```text
s3vectors:CreateVectorBucket
s3vectors:CreateIndex
s3vectors:PutVectors
s3vectors:GetIndex
s3vectors:ListVectors
```

**Instance role của EC2** (`rag-ec2-runtime-role`, chương 5.7) - chỉ truy vấn:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "QueryVectorIndex",
      "Effect": "Allow",
      "Action": [
        "s3vectors:QueryVectors",
        "s3vectors:GetIndex"
      ],
      "Resource": "arn:aws:s3vectors:ap-southeast-1:<account-id>:bucket/rag-vectors-vanh1234/index/hotpotqa-val500-bge-m3-v002"
    }
  ]
}
```

ARN của index có dạng `arn:aws:s3vectors:<region>:<account-id>:bucket/<bucket>/index/<index>`, nhờ vậy policy có thể thu hẹp xuống đúng một index thay vì cả bucket.

Role phục vụ **không có `PutVectors` và không có `DeleteVectors`**. Y hệt như với S3 ở chương 5.5, dịch vụ đang chạy chỉ có quyền đọc trên chính dữ liệu của nó, và điều đó do IAM cưỡng chế chứ không dựa vào quy ước. Một lỗi trong ứng dụng không thể làm hỏng index.

---

## 8. Chi phí và giới hạn

Những con số ràng buộc các quyết định thiết kế:

| Giới hạn | Giá trị |
| --- | --- |
| Số vector mỗi request `PutVectors` | 500 |
| Số chiều của index | 1 – 4096 |
| Kiểu dữ liệu | chỉ `float32` |
| Chỉ số khoảng cách | `cosine`, `euclidean` |
| Số non-filterable metadata key mỗi index | tối đa 10 |
| Kích thước trang của `list-vectors` | tối đa 1000 |

Chi phí gồm hai thành phần - lưu trữ vector, và số lượng request. Corpus này sinh ra 8.279 vector 1024 chiều, tức dung lượng không đáng kể, và bản demo phát ra một lệnh `QueryVectors` cho mỗi lần truy hồi. Tính chất quan trọng nhất với một đồ án sinh viên là **không có chi phí lúc rảnh**: khác với một cơ sở dữ liệu vector được cấp phát sẵn hay OpenSearch Serverless, không có gì bị tính tiền khi không có truy vấn nào chạy. Chương 5.13 sẽ nói đầy đủ về bức tranh chi phí.

---

## 9. Các lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| `ValidationException` ngay ở `PutVectors` đầu tiên | Độ dài vector không khớp số chiều của index | Index đã cố định - xóa và tạo lại với đúng số chiều |
| `ConflictException` lúc tạo | Đã tồn tại index hoặc bucket trùng tên | Dùng lại cái đang có, hoặc chọn `INDEX_ID` mới |
| `NotFoundException` lúc nạp | Index được tạo ở region khác, hoặc gõ sai tên | Kiểm tra cờ region và tên index có bằng đúng `INDEX_ID` không |
| `AccessDeniedException` | Danh tính CLI của bạn thiếu quyền `s3vectors:*` | Quyền S3 không tự động cấp quyền S3 Vectors - chúng là các action tách biệt |
| `aws: error: argument command: Invalid choice: 's3vectors'` | AWS CLI quá cũ | Nâng cấp CLI |
| Nạp hỏng giữa chừng | Lỗi tạm thời giữa batch | Chạy lại script; `PutVectors` ghi đè theo key nên hoàn toàn an toàn |
| Truy hồi trả về tài liệu không liên quan | Lệch mô hình / chỉ số / artifact | Xem phần tip ở mục 5 |

---

## 10. Kết quả

Cả hai nửa của bộ truy hồi giờ đều đã có dữ liệu: chỉ mục BM25 và các tài liệu nằm trong S3 thông thường từ chương 5.5, còn dense embedding nằm trong S3 Vectors index ở chương này. Truy hồi đã được kiểm chứng ngay từ máy cá nhân, chưa cần đến máy chủ nào.

Chương 5.7 sẽ triển khai backend FastAPI trên EC2 và gắn cho nó instance role buộc các quyền này lại với nhau.

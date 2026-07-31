---
title: "Lưu trữ trên Amazon S3"
date: 2026-07-31
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

Các retrieval artifact được tạo ở bước trước cần được lưu trữ độc lập với EC2 instance đang vận hành backend. Vì vậy, CloudHop RAG sử dụng **Amazon S3** làm nơi lưu trữ lâu dài cho corpus đã xử lý, BM25 index, document mapping và các index manifest cần thiết cho RAG pipeline khi chạy online.

Việc lưu các artifact này trên S3 cho phép backend tải và sử dụng đúng phiên bản artifact khi khởi động mà không cần xây dựng lại corpus hoặc retrieval index trên EC2.

Chương 5.4 đã tạo ra một thư mục artifact trên máy build. Chương này tạo S3 bucket để chứa chúng và upload lên theo đúng một layout mà backend có thể đọc được mà không cần ai chỉ đường.

{{% notice info %}}
**Bạn sẽ có gì sau chương này:** một S3 bucket riêng tư, bật versioning, bật mã hóa, đặt tại `ap-southeast-1`, chứa cây `rag/` và đã được kiểm tra đọc được. Phần vector *không* xử lý ở đây - chúng sẽ đi vào Amazon S3 Vectors ở chương 5.6.
{{% /notice %}}

---

## 1. Hai bucket, hai dịch vụ khác nhau

Dự án này dùng hai tài nguyên lưu trữ có tên gần giống nhau đến mức dễ nhầm. Chúng không cùng một loại:

| | `aws-rag-bucket-vanh1234` | `rag-vectors-vanh1234` |
| --- | --- | --- |
| Dịch vụ | **Amazon S3** (thông thường) | **Amazon S3 Vectors** (kho vector) |
| Chứa | Tài liệu JSONL, chỉ mục BM25, manifest | Embedding 1024 chiều |
| Tạo ở | **Chương này** | Chương 5.6 |
| Truy cập bằng | API `s3` - `GetObject`, `ListObjects` | API `s3vectors` - `PutVectors`, `QueryVectors` |

Vector bucket được tạo qua một API riêng và **không** xuất hiện trong danh sách bucket S3 thông thường. Đừng cố tạo nó ở chương này.

---

## 2. Chọn region trước, và dùng thống nhất ở mọi nơi

Mọi thứ trong dự án đều nằm ở **`ap-southeast-1`**: bucket này, vector bucket, instance EC2, các tham số SSM, secret, và API Gateway.

Đây không phải sở thích. Backend EC2 đọc artifact từ S3 mỗi lần khởi động nguội và truy vấn S3 Vectors ở mọi request. Đặt lưu trữ khác region với tính toán sẽ cộng thêm độ trễ liên vùng vào đường đi của request, và biến mỗi lần tải artifact thành lưu lượng liên vùng có tính phí. Lưu lượng cùng region giữa EC2 và S3 thì miễn phí.

{{% notice warning %}}
Hãy chọn region ngay bây giờ và đừng đổi về sau. Region được nhúng trong các tham số SSM, trong tích hợp API Gateway và trong vector index - đổi giữa chừng đồng nghĩa với làm lại công sức của nhiều chương.
{{% /notice %}}

---

## 3. Tạo bucket

Tên bucket là **duy nhất trên toàn bộ AWS**, phải viết thường, và không đổi tên được. Hãy chọn tên riêng của bạn - `aws-rag-bucket-vanh1234` đã bị dự án này chiếm mất rồi.

**Console:** S3 → Create bucket

| Trường | Giá trị |
| --- | --- |
| Bucket name | tên duy nhất toàn cầu của riêng bạn |
| AWS Region | `Asia Pacific (Singapore) ap-southeast-1` |
| Object Ownership | ACLs disabled (khuyến nghị) |
| Block Public Access | **Block all public access - giữ nguyên tất cả các ô đã tick** |
| Bucket Versioning | **Enable** |
| Default encryption | **SSE-S3 (khóa do Amazon S3 quản lý)** |

**CLI:**

```bash
aws s3api create-bucket \
  --bucket aws-rag-bucket-vanh1234 \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1
```

Mọi region khác `us-east-1` đều bắt buộc phải có cờ `--create-bucket-configuration`. Quên nó là lỗi đầu tiên hay gặp nhất.

<!-- ẢNH 1 - SCREENSHOT.
     S3 Console -> form Create bucket, đã điền, thấy được ít nhất:
       - ô nhập tên bucket
       - AWS Region = Asia Pacific (Singapore) ap-southeast-1
       - ô "Block all public access" ĐANG ĐƯỢC TICK
     Chỉ một tấm này đã chứng minh được cả quyết định về region lẫn mặc định bảo mật. -->

![Tạo bucket chứa artifact](/images/5-Workshop/5.5-S3-Storage/create-bucket.png)

---

## 4. Làm cứng bucket

Ba thiết lập, mỗi cái vì một lý do cụ thể. Nếu bạn đã làm theo bảng Console ở trên thì chúng đã được áp dụng; phần dưới là lệnh CLI tương đương kèm lý do.

**Block Public Access** - không có gì trong bucket này được phục vụ cho trình duyệt. Frontend nói chuyện với API Gateway, và chỉ instance role của EC2 mới đọc các object này. Không có lý do chính đáng nào để tồn tại một object công khai ở đây, nên rào chắn ở cấp tài khoản được bật hoàn toàn.

```bash
aws s3api put-public-access-block \
  --bucket aws-rag-bucket-vanh1234 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

**Versioning** - theo quy ước thì artifact là bất biến (chương 5.4 tăng phiên bản mỗi khi corpus đổi), nhưng versioning là lưới an toàn cho lúc quy ước bị phá. Nếu ai đó chạy lại bản build với định danh mà production đang phục vụ, các object cũ vẫn còn và khôi phục được.

```bash
aws s3api put-bucket-versioning \
  --bucket aws-rag-bucket-vanh1234 \
  --versioning-configuration Status=Enabled
```

**Mã hóa mặc định** - mã hóa dữ liệu lúc lưu bằng khóa do S3 quản lý. Bucket mới tự động có SSE-S3; đặt tường minh giúp thể hiện rõ chủ đích và trụ được qua một đợt rà soát chính sách.

```bash
aws s3api put-bucket-encryption \
  --bucket aws-rag-bucket-vanh1234 \
  --server-side-encryption-configuration \
    '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

{{% notice note %}}
SSE-KMS với khóa do khách hàng quản lý sẽ cho dấu vết kiểm toán theo từng khóa và phân quyền khóa riêng biệt, đổi lại là phí KMS theo từng request và phải cấp thêm quyền `kms:Decrypt` cho EC2 role. Với một bản demo dùng dữ liệu công khai, SSE-S3 là lựa chọn tương xứng. Với corpus thật chứa tài liệu nội bộ, SSE-KMS mới là bước nâng cấp đúng.
{{% /notice %}}

<!-- ẢNH 2 - SCREENSHOT.
     S3 Console -> bucket của bạn -> tab Properties.
     Cuộn sao cho CẢ HAI mục sau cùng nằm trong một tấm:
       - "Bucket Versioning: Enabled"
       - "Default encryption: Server-side encryption with Amazon S3 managed keys (SSE-S3)"
     Nếu không vừa một màn hình thì chụp hai tấm và đặt cạnh nhau. -->

![Đã bật versioning và mã hóa mặc định](/images/5-Workshop/5.5-S3-Storage/bucket-properties.png)

---

## 5. Cấu trúc key là một bản hợp đồng

Cấu trúc prefix không phải để sắp xếp cho gọn mắt - backend tự ghép key S3 từ chính cấu trúc này lúc khởi động. Sai một prefix sẽ cho ra một dịch vụ khởi động sạch sẽ rồi hỏng ở truy vấn đầu tiên.

```text
s3://aws-rag-bucket-vanh1234/
  rag/
    corpora/hotpotqa/validation-500/v002/
      corpus.jsonl
      eval.jsonl
      corpus_manifest.json
    processed/hotpotqa-val500-v002/
      parent_docs.jsonl
      child_docs.jsonl
      child_to_parent.json
    indexes/hotpotqa-val500-bge-m3-v002/
      bm25/
        bm25_index/
        bm25_doc_ids.json
      manifests/
        index_manifest.json
      s3vectors-import/
        put_vectors_0000.json
        ...
```

Ba định danh chi phối toàn bộ cây thư mục, và chúng đến thẳng từ chương 5.4:

| Đoạn đường dẫn | Đến từ | Giá trị production |
| --- | --- | --- |
| `rag/` | `S3_ARTIFACT_PREFIX` (mặc định `rag`) | `rag` |
| `corpora/<corpus id>/` | `CORPUS_ID` | `hotpotqa/validation-500/v002` |
| `processed/<processed id>/` | `PROCESSED_ID` | `hotpotqa-val500-v002` |
| `indexes/<index id>/` | `INDEX_ID` | `hotpotqa-val500-bge-m3-v002` |

Để ý rằng **`processed` và `indexes` được đánh phiên bản riêng biệt**. Đó là chủ ý: cùng một bộ tài liệu đã xử lý có thể làm nền cho nhiều index dựng bằng các mô hình embedding khác nhau mà không phải nhân bản phần văn bản.

---

## 6. Upload

Notebook đã sắp xếp sẵn mọi thứ thành một cây phản chiếu đúng layout ở trên, nên bước upload chỉ là một lệnh copy đệ quy - không phải xoay xở đường dẫn, không có cơ hội đặt nhầm prefix.

```bash
aws s3 cp "s3_manual_upload/hotpotqa-val500-bge-m3-v002/rag" \
  "s3://aws-rag-bucket-vanh1234/rag/" \
  --recursive \
  --region ap-southeast-1
```

`aws s3 sync` cũng dùng được và tốt hơn cho những lần upload lại, vì nó chỉ truyền các object đã thay đổi:

```bash
aws s3 sync "s3_manual_upload/hotpotqa-val500-bge-m3-v002/rag" \
  "s3://aws-rag-bucket-vanh1234/rag" \
  --region ap-southeast-1
```

{{% notice tip %}}
Trên Windows, nhớ đặt đường dẫn cục bộ trong dấu nháy - thư mục output của notebook thường nằm dưới một đường dẫn Google Drive có dấu cách. Trong PowerShell dùng dấu backtick `` ` `` để xuống dòng, không dùng dấu gạch chéo ngược.
{{% /notice %}}

<!-- ẢNH 3 - SCREENSHOT.
     Terminal của bạn sau khi upload xong, hiển thị vài dòng "upload: ... to s3://..."
     và trạng thái hoàn tất.
     Đảm bảo thấy được ít nhất một dòng cho mỗi prefix corpora/, processed/ và indexes/,
     để người đọc thấy cả ba đã được ghi lên. -->

![Upload cây artifact lên S3](/images/5-Workshop/5.5-S3-Storage/upload-output.png)

---

## 7. Kiểm tra

Liệt kê những gì đã lên:

```bash
aws s3 ls s3://aws-rag-bucket-vanh1234/rag/ --recursive --region ap-southeast-1
```

Rồi đọc ngược lại manifest - đây là phép kiểm tra hữu ích nhất, vì nó chính là object mà backend tải về đầu tiên:

```bash
aws s3 cp \
  s3://aws-rag-bucket-vanh1234/rag/indexes/hotpotqa-val500-bge-m3-v002/manifests/index_manifest.json - \
  --region ap-southeast-1
```

Bạn phải thấy đúng đoạn JSON từ chương 5.4, bao gồm `params.embedding_model` và `source.processed_id`. Nếu lệnh này lỗi thì không có gì phía sau chạy được.

Các object tối thiểu phải có mặt trước khi đi tiếp:

```text
rag/processed/<processed id>/parent_docs.jsonl
rag/processed/<processed id>/child_docs.jsonl
rag/processed/<processed id>/child_to_parent.json
rag/indexes/<index id>/bm25/bm25_doc_ids.json
rag/indexes/<index id>/bm25/bm25_index/
rag/indexes/<index id>/manifests/index_manifest.json
```

<!-- ẢNH 4 - SCREENSHOT.
     Trình duyệt object của S3 Console, bên trong bucket của bạn, đã mở prefix rag/
     và thấy được các thư mục corpora/ processed/ indexes/.
     Giữ breadcrumb trong khung - nó chứng minh tên bucket. -->

![Prefix rag/ trên S3 Console](/images/5-Workshop/5.5-S3-Storage/s3-console-tree.png)

Vào sâu thêm một cấp, `indexes/<index id>/` chứa đúng ba prefix mà backend đọc lúc khởi động:

<!-- ẢNH 5 - SCREENSHOT.
     Vẫn trình duyệt object đó, vào sâu tới rag/indexes/<index id>/, thấy được bm25/,
     manifests/ và s3vectors-import/.
     Giữ breadcrumb trong khung - nó chứng minh index id. -->

![Bên trong prefix index](/images/5-Workshop/5.5-S3-Storage/s3-console-index.png)

---

## 8. Backend thật sự đọc những gì

Chỗ này đáng nói cho chính xác, vì nó giải thích vì sao layout lại quan trọng và vì sao có một prefix được upload lên nhưng không bao giờ được đọc.

Lúc khởi động, bộ nạp online thực hiện đúng ba thao tác S3 cho layout `s3vectors`:

1. **Tải manifest trước tiên** - `rag/indexes/<index id>/manifests/index_manifest.json`.
2. **Đọc `source.processed_id` từ chính manifest đó** để biết tài liệu nằm ở đâu. Processed id không hề được hard-code trong ứng dụng; nếu tham số SSM chưa đặt thì manifest sẽ cung cấp nó.
3. **Tải về hai prefix** - `rag/processed/<processed id>/` và `rag/indexes/<index id>/bm25/`.

```python
manifest_key = f"{prefix}/indexes/{index_id}/manifests/index_manifest.json"
_download_file(bucket, manifest_key, manifest_path)

manifest = _load_json(manifest_path)
processed_id = processed_id or manifest.get("source", {}).get("processed_id", "")

_download_prefix(bucket, f"{prefix}/processed/{processed_id}/", ...)
_download_prefix(bucket, f"{prefix}/indexes/{index_id}/bm25/", ...)
```

Những gì nó không bao giờ tải:

- **`rag/corpora/`** - corpus thô chỉ cần cho việc dựng lại và cho khâu đánh giá, không bao giờ cần cho việc phục vụ.
- **`rag/indexes/<index id>/s3vectors-import/`** - các batch đó chỉ được dùng một lần, bởi bước nạp ở chương 5.6. Chúng vẫn được giữ trên S3, vì nếu vector index bị mất hoặc phải tạo lại thì nạp lại từ những file này chỉ tốn vài phút và không tốn chi phí, trong khi embed lại corpus đồng nghĩa với dựng lại cả môi trường GPU.

Nói cách khác, dịch vụ online tải về nhiều nhất vài trăm megabyte, một lần cho mỗi lần khởi động lại, và không tải gì thêm cho từng request.

---

## 9. Phân quyền

Có hai chủ thể khác nhau chạm vào bucket này, và chúng cần quyền khác nhau.

**Máy build** (bạn, lúc upload) cần quyền ghi:

```text
s3:ListBucket
s3:GetObject
s3:PutObject
s3:AbortMultipartUpload
```

Chỉ thêm `s3:DeleteObject` và `s3:DeleteObjectVersion` khi bạn cần dọn một lần upload nhầm - với versioning đang bật, xóa cũng có nghĩa là phải xóa cả các phiên bản cũ.

**Instance role của EC2** (`rag-ec2-runtime-role`, tạo ở chương 5.7) chỉ cần **quyền đọc**, giới hạn đúng bucket này và prefix này:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListArtifactPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::aws-rag-bucket-vanh1234",
      "Condition": { "StringLike": { "s3:prefix": ["rag/*"] } }
    },
    {
      "Sid": "ReadArtifacts",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::aws-rag-bucket-vanh1234/rag/*"
    }
  ]
}
```

Role phục vụ **hoàn toàn không có quyền ghi**. Dịch vụ online chỉ đọc kho lưu trữ ở mức kiến trúc - nó không thể làm hỏng một index ngay cả khi ứng dụng có lỗi, và bảo đảm đó được thực thi bằng IAM chứ không bằng sự cẩn thận của con người.

Không gắn bucket policy nào. Quyền truy cập được cấp hoàn toàn qua IAM identity policy, và bucket vẫn ở trạng thái riêng tư.

---

## 10. Chi phí

Ở quy mô này, chi phí lưu trữ gần như bằng không - một corpus 4.937 tài liệu cùng các chỉ mục của nó chỉ vài trăm megabyte, nằm gọn trong vài xu mỗi tháng. Dù vậy vẫn có hai thứ nên thiết lập, vì chúng phình ra rất tệ nếu bỏ mặc:

- **Versioning giữ mọi object bị ghi đè mãi mãi** trừ khi bạn bảo nó đừng làm vậy. Hãy thêm một lifecycle rule để hết hạn các phiên bản không hiện hành sau 30 ngày, ngay khi bạn bắt đầu lặp nhiều lần trên index.
- **Truy cập cùng region thì miễn phí truyền tải.** Tải artifact về một instance EC2 ở region khác sẽ bị tính tiền theo gigabyte ở mỗi lần khởi động lại.

Phân tích chi phí đầy đủ nằm ở chương 5.13, còn phần xóa tài nguyên ở chương 5.14.

---

## 11. Các lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| `IllegalLocationConstraintException` lúc tạo bucket | Thiếu `--create-bucket-configuration` khi không ở `us-east-1` | Thêm cờ đó kèm region của bạn |
| `BucketAlreadyExists` | Tên bucket là duy nhất toàn cầu | Chọn tên khác |
| `AccessDenied` lúc upload | Danh tính CLI của bạn thiếu `s3:PutObject` | Kiểm tra `aws sts get-caller-identity`, rồi xem policy đang gắn |
| Backend khởi động được nhưng truy vấn đầu tiên lỗi `FileNotFoundError` | Object được upload nhầm prefix | Đối chiếu `aws s3 ls --recursive` với cây ở mục 5 |
| `No S3 objects found under s3://.../processed/...` | Nhầm lẫn giữa `processed id` và `index id` | Hai cái khác nhau - `hotpotqa-val500-v002` với `hotpotqa-val500-bge-m3-v002` |
| Đường dẫn upload lỗi trên Windows | Đường dẫn Google Drive có dấu cách | Đặt cả đường dẫn trong dấu nháy |

---

## 12. Kết quả

Bucket artifact đã ở trạng thái riêng tư, bật versioning, bật mã hóa, và chứa trọn cây `rag/` theo đúng layout mà backend mong đợi. Không gì đọc được nó ngoài những danh tính mà bạn cấp quyền IAM.

Chương 5.6 sẽ tạo vector bucket và index, rồi chạy `ingest_s3vectors.py` trên chính các batch `s3vectors-import/` hiện đang nằm trong bucket này.

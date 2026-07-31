---
title: "Xây dựng Offline Artifact"
date: 2026-07-31
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

Trước khi RAG backend có thể xử lý câu hỏi, dữ liệu nguồn cần được chuyển thành các artifact sẵn sàng cho retrieval. Quá trình này được thực hiện offline để các bước xử lý tài liệu, chia đoạn, tạo embedding và xây dựng index không phải lặp lại mỗi khi người dùng gửi request.

CloudHop RAG sử dụng **HotpotQA Distractor** làm bộ dữ liệu benchmark cho quá trình này. HotpotQA được thiết kế cho bài toán hỏi đáp đa bước, trong đó thông tin cần thiết cho một câu trả lời có thể nằm trên nhiều tài liệu hỗ trợ khác nhau. Các trường câu hỏi, câu trả lời, context và supporting facts đã được gán nhãn tạo điều kiện thuận lợi cho việc xây dựng và đánh giá retrieval pipeline của dự án.

Bộ artifact cuối cùng được xây dựng từ **500 câu hỏi thuộc validation split**. Context của các câu hỏi được chuẩn hóa về định dạng corpus của dự án trước khi tạo parent document, child chunk, BM25 index và BGE-M3 embedding.

Đây là chương xây dựng đầu tiên. Ở đây chúng ta tạo ra **toàn bộ file mà backend sau này sẽ cần**, trước khi có bất kỳ tài nguyên AWS nào. Chương 5.3 đã giải thích *vì sao* công việc này được tách khỏi đường đi của request; chương này sẽ thực hiện nó.

{{% notice info %}}
**Bạn sẽ có gì sau chương này:** một thư mục cục bộ duy nhất, sẵn sàng để upload, chứa corpus, các tài liệu parent/child, chỉ mục BM25, các batch vector để import, và một manifest gắn kết tất cả lại với nhau. Chưa upload gì cả - chương 5.5 sẽ tạo S3 bucket và upload thư mục này, chương 5.6 sẽ tạo vector index và nạp vector vào.
{{% /notice %}}

---

## 1. Nên chạy ở đâu, và vì sao không chạy trên chính instance phục vụ

Việc embed corpus là phần duy nhất thực sự nặng về tính toán trong cả dự án. Nó chỉ được làm **một lần**, trên một máy được chọn riêng cho việc đó, chứ không phải trên instance EC2 sẽ phục vụ người dùng.

| Phương án | Kết luận |
| --- | --- |
| **Google Colab với GPU** | **Đã chọn.** GPU T4 miễn phí, embed xong trong vài phút, hoàn toàn không tốn chi phí AWS cho khâu build. |
| Máy trạm cục bộ dùng CPU | Chạy được, nhưng `BAAI/bge-m3` trên CPU rất chậm với corpus lớn hơn mức demo nhỏ. |
| SageMaker Studio CPU Space | **Đã thử và phải bỏ.** Tiến trình bị kết thúc với thông báo `Killed` ngay lúc nạp `BAAI/bge-m3` - Space hết RAM. |
| SageMaker GPU Space (`ml.g4dn.xlarge`) | **Bị chặn.** Quota của tài khoản cho JupyterLab App trên `ml.g4dn.xlarge` là `0`, nên không khởi động được Space. |
| Build ngay trên EC2 phục vụ | Bị loại ngay từ thiết kế - như vậy là đặt một tác vụ CPU kéo dài nhiều phút lên chính máy phải trả lời request trong vài giây. |

{{% notice tip %}}
Đây là một ràng buộc thật cần tính trước, không phải chú thích cho vui. Một mô hình embedding lớn cần vài GB RAM chỉ để nạp. Nếu môi trường build của bạn bị `Killed` mà không có traceback nào, đó là cơ chế hết bộ nhớ của hệ điều hành (OOM killer), không phải lỗi trong code. Hoặc chuyển sang runtime GPU, hoặc đổi sang mô hình nhỏ hơn như `BAAI/bge-small-en-v1.5`.
{{% /notice %}}

Bản build chuẩn là notebook:

```text
backend/notebooks/build_s3_offline_artifacts.ipynb
```

<!-- ẢNH 1 - SCREENSHOT.
     Google Colab -> Runtime -> Change runtime type, với "T4 GPU" đang được chọn.
     Đây là bằng chứng cho quyết định môi trường đã giải thích ở bảng trên.
     Nếu bạn build bằng CPU thì chụp đúng hộp thoại đó và ghi rõ như vậy. -->

![Chọn runtime GPU trong Colab](/images/5-Workshop/5.4-Offline-Artifact-Build/colab-gpu-runtime.png)

---

## 2. Notebook không phụ thuộc vào bộ dữ liệu cụ thể

HotpotQA ở đây chỉ là một **adapter nguồn mẫu**. Notebook chấp nhận bất kỳ tài liệu nào - quy định nội bộ, tài liệu hướng dẫn, ticket hỗ trợ, trang wiki, Markdown, CSV - miễn là chúng được chuẩn hóa về schema chuẩn của dự án trước.

Schema corpus (mỗi dòng một object JSON, `corpus.jsonl`):

```json
{"id":"doc_001","title":"Document title","text":"Full document or passage text","metadata":{"source":"optional"}}
```

Schema đánh giá tùy chọn (`eval.jsonl`, dùng ở chương 5.11):

```json
{"id":"q_001","question":"Question text","answer":"Expected answer","supporting_doc_ids":["doc_001"]}
```

Notebook có hai chế độ, đặt bằng đúng một biến:

| `SOURCE_MODE` | Dùng khi |
| --- | --- |
| `hotpotqa_sample` | Bạn muốn corpus demo được dựng tự động từ HotpotQA |
| `custom_jsonl` | Bạn đã có sẵn `corpus.jsonl` của riêng mình (và tùy chọn `eval.jsonl`) |

Điều này rất quan trọng với một hệ thống thật: **toàn bộ pipeline phía sau notebook này không hề biết nguồn dữ liệu là gì**. Thay HotpotQA bằng cơ sở tri thức của doanh nghiệp chỉ cần đổi đúng biến này, không đổi gì khác.

---

## 3. Cấu hình bản build

Toàn bộ cấu hình nằm gọn trong một cell ở đầu notebook. Đây là các giá trị đã tạo ra index đang chạy production:

```python
S3_BUCKET        = 'aws-rag-bucket-vanh1234'
S3_PREFIX        = 'rag'
AWS_REGION       = 'ap-southeast-1'
S3_VECTOR_BUCKET = 'rag-vectors-vanh1234'

SOURCE_MODE      = 'hotpotqa_sample'
DATASET_SUBSET   = 'distractor'  # bảo đảm độ phủ gold-title
DATASET_SIZE     = 500          # 100 / 500 / 1000, hoặc None để lấy nguyên split
HOTPOTQA_VERSION = 'v002'       # tăng lên mỗi khi nội dung corpus thay đổi

EMBEDDING_MODEL_NAME   = 'BAAI/bge-m3'
VECTOR_DIMENSION       = 1024
VECTOR_DISTANCE_METRIC = 'cosine'

CHILD_CHUNK_SIZE       = 500
CHILD_CHUNK_OVERLAP    = 100
EMBEDDING_BATCH_SIZE   = 64
VECTOR_IMPORT_BATCH_SIZE = 500
```

Ba định danh artifact đều được **suy ra tự động**, không bao giờ gõ tay:

```python
CORPUS_ID    = f'hotpotqa/validation-{DATASET_SIZE_LABEL}/{HOTPOTQA_VERSION}'
PROCESSED_ID = f'hotpotqa-val{DATASET_SIZE_LABEL}-{HOTPOTQA_VERSION}'
INDEX_ID     = f'hotpotqa-val{DATASET_SIZE_LABEL}-bge-m3-{HOTPOTQA_VERSION}'
```

Với `DATASET_SIZE = 500` và `HOTPOTQA_VERSION = 'v002'`, kết quả ra đúng các định danh của bản build cuối cùng:

```text
corpus id    : hotpotqa/validation-500/v002
processed id : hotpotqa-val500-v002
index id     : hotpotqa-val500-bge-m3-v002
```

{{% notice warning %}}
**Luôn tăng phiên bản mỗi khi corpus thay đổi.** Mọi định danh đều mang hậu tố phiên bản, và mọi đường dẫn S3 đều được dựng từ các định danh đó. Dùng lại một định danh cũ sẽ ghi đè lên index mà production có thể vẫn đang phục vụ. Tăng phiên bản thì index cũ và mới nằm song song trên S3, và việc chuyển qua lại giữa chúng chỉ là đổi tham số (chương 5.7) - có thể quay lui ngay lập tức nếu index mới tệ hơn.
{{% /notice %}}

Để ý rằng tên mô hình embedding được nhúng thẳng vào `INDEX_ID`. Đó là chủ ý: một index dựng bằng mô hình này **không thể** truy vấn bằng mô hình khác, vì vector nằm ở hai không gian khác nhau. Đưa `bge-m3` vào tên khiến việc ghép sai lộ ra ngay từ đầu, trước khi nó biến thành một buổi debug.

<!-- ẢNH 2 - SCREENSHOT.
     Output của cell cấu hình trong notebook, hiển thị các dòng in ra:
       source mode: / device: / corpus id: / processed id: / index id: /
       artifact root: / manual upload root:
     Chụp kèm cả phần code phía trên nếu vừa khung, để người đọc thấy được cả đầu vào lẫn đầu ra.
     Cắt bỏ đường dẫn Drive nếu nó chứa tên thư mục cá nhân. -->

![Output cell cấu hình của notebook](/images/5-Workshop/5.4-Offline-Artifact-Build/notebook-config-output.png)

---

## 4. Bước 1 - Chuẩn hóa nguồn thành `corpus.jsonl` và `eval.jsonl`

Adapter đọc split validation của HotpotQA, trải phẳng các đoạn văn hỗ trợ của từng câu hỏi thành các tài liệu riêng lẻ, làm sạch khoảng trắng, rồi ghi ra hai file chuẩn cùng với một `corpus_manifest.json`.

Có một chi tiết dễ bỏ sót nhưng trả giá đắt: notebook chạy một **bộ kiểm tra độ phủ gold-title**. Mọi câu hỏi đánh giá đều phải có *tất cả* tiêu đề tài liệu hỗ trợ của nó hiện diện trong corpus. Nếu bằng chứng của một câu hỏi bị thiếu thì câu hỏi đó không thể trả lời được dù bộ truy hồi có tốt đến đâu, và phần đánh giá ở chương 5.11 sẽ hóa ra là đang đo chất lượng corpus chứ không phải đo hệ thống.

Kết quả của bước này:

```text
corpus.jsonl           mỗi dòng một tài liệu
eval.jsonl             mỗi dòng một câu hỏi đánh giá
corpus_manifest.json   số dòng và checksum
```

---

## 5. Bước 2 - Dựng tài liệu parent và child

Bước này hiện thực hóa mẫu **small-to-big** đã mô tả ở chương 5.3.

```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=CHILD_CHUNK_SIZE,        # 500
    chunk_overlap=CHILD_CHUNK_OVERLAP,  # 100
)
```

- Mỗi tài liệu nguồn trở thành **một parent document** - nguyên bài viết, giữ nguyên vẹn. Đây là thứ mà LLM sẽ đọc ở cuối pipeline.
- Mỗi parent được cắt thành **nhiều child document** khoảng 500 ký tự với 100 ký tự chồng lấn. Đây mới là các đơn vị được đánh chỉ mục và đem đi so khớp.
- Một map `child_to_parent.json` ghi lại mỗi child đến từ parent nào.

Vì sao phải hai cấp thay vì một: một đoạn 500 ký tự đủ chính xác để truy hồi so khớp, nhưng lại quá nhỏ để trả lời - nó thường cắt ngang một câu. Còn nguyên bài viết thì là ngữ cảnh tốt nhưng quá loãng để so khớp. Đánh chỉ mục trên đoạn nhỏ mà *phục vụ* bằng đoạn lớn thì có được cả hai. Phần chồng lấn 100 ký tự tránh việc một dữ kiện nằm vắt qua ranh giới hai đoạn bị mất.

Kết quả:

```text
parent_docs.jsonl      nguyên các bài viết
child_docs.jsonl       các đoạn truy hồi, mỗi đoạn mang child_id / parent_id / title
child_to_parent.json   map child_id -> parent_id
```

---

## 6. Bước 3 - Dựng chỉ mục BM25

```python
import bm25s
```

BM25 là một hàm xếp hạng theo từ vựng cổ điển - nó chấm điểm tài liệu dựa trên độ trùng lặp từ khóa với truy vấn. Nó được dựng trên **đúng những child document** sẽ được embed ở bước sau, nhờ đó hai bộ truy hồi cùng xếp hạng trên một tập ứng viên và kết quả của chúng có thể hợp nhất một cách công bằng.

Chỉ mục được ghi kèm `bm25_doc_ids.json`, file này ánh xạ vị trí số nguyên nội bộ của BM25 ngược về giá trị `child_id` thật. Không có file đó thì chỉ mục vô nghĩa lúc truy vấn.

{{% notice note %}}
BM25 ở đây không phải phương án dự phòng cũ kỹ - nó là một nửa chiến lược truy hồi. Nó xử lý tốt tên riêng chính xác, từ hiếm, con số và mã định danh, vốn đúng là những chỗ mà dense embedding yếu nhất. Chương 5.3 mục 4 giải thích cách hai bên được hợp nhất.
{{% /notice %}}

---

## 7. Bước 4 - Embed child document và dựng các batch import vector

Đây là bước tốn kém nhất và là lý do phải dùng runtime GPU.

```python
model = SentenceTransformer(EMBEDDING_MODEL_NAME, device=DEVICE)

vectors = model.encode(
    texts,
    batch_size=EMBEDDING_BATCH_SIZE,
    normalize_embeddings=True,
    convert_to_numpy=True,
    show_progress_bar=True,
)
vectors = np.asarray(vectors, dtype='float32')
assert vectors.shape[1] == VECTOR_DIMENSION
```

Hai chi tiết quan trọng:

- **`normalize_embeddings=True`** - các vector được chuẩn hóa về độ dài đơn vị, và đó chính là điều khiến chỉ số khoảng cách `cosine` cấu hình trên S3 Vectors index trở nên đúng đắn. Chuẩn hóa lúc build và chọn cosine lúc tạo index phải khớp nhau.
- **Câu lệnh assert về số chiều** - `BAAI/bge-m3` sinh ra vector 1024 chiều. S3 Vectors index được tạo với số chiều cố định (chương 5.6). Kiểm tra ngay tại đây biến một sai lệch âm thầm thành một lỗi hiện ra tức thì và rõ ràng.

Sau đó các vector được ghi ra dưới dạng **payload của request `PutVectors`**, mỗi file 500 vector, sẵn sàng gửi lên S3 Vectors:

```python
batch_vectors.append({
    'key': doc.metadata['child_id'],
    'data': {'float32': vector.tolist()},
    'metadata': {
        'parent_id': doc.metadata['parent_id'],
        'source_doc_id': doc.metadata.get('source_doc_id', ''),
        'title': doc.metadata.get('title', ''),
    },
})
```

`key` chính là `child_id`, còn `parent_id` đi kèm trong metadata. Đó là thứ cho phép bộ truy hồi online đi thẳng từ một kết quả vector về bài viết cha mà không cần tra cứu lần hai - bước mở rộng small-to-big gần như không tốn gì lúc truy vấn.

Việc chia thành các file 500 vector không phải để cho đẹp: nó giữ mỗi request `PutVectors` nằm trong giới hạn của API, và khiến bước nạp ở chương 5.6 **có thể chạy tiếp được**. Nếu nạp hỏng giữa chừng, bạn chạy lại các file còn lại thay vì embed lại toàn bộ corpus.

<!-- ẢNH 3 - SCREENSHOT.
     Cell embedding lúc đang chạy (hoặc vừa chạy xong), hiển thị thanh tiến trình của
     sentence-transformers kèm số batch và thời gian, ví dụ "Batches: 100%|####| 47/47".
     Đây là bằng chứng bước nặng nhất đã thực sự chạy, và con số thời gian đáng để
     trích lại ở chương 5.11. -->

![Embed các child document](/images/5-Workshop/5.4-Offline-Artifact-Build/embedding-progress.png)

---

## 8. Bước 5 - Ghi index manifest

Manifest là bản hợp đồng giữa bản build offline và dịch vụ online. Nó được ghi ra thành `index_manifest.json`, ghi lại đã dựng cái gì, từ nguồn nào, với tham số ra sao:

```json
{
  "schema_version": 3,
  "index_id": "hotpotqa-val500-bge-m3-v002",
  "created_at": "...",
  "source": {
    "source_mode": "hotpotqa_sample",
    "corpus_id": "hotpotqa/validation-500/v002",
    "processed_id": "hotpotqa-val500-v002",
    "corpus_rows": 4937,
    "corpus_sha256": "..."
  },
  "params": {
    "embedding_model": "BAAI/bge-m3",
    "child_chunk_size": 500,
    "child_chunk_overlap": 100,
    "vector_backend": "s3vectors"
  },
  "artifacts": {
    "parent_docs": { "path": "processed/.../parent_docs.jsonl", "num_docs": 0, "sha256": "..." },
    "child_docs":  { "path": "processed/.../child_docs.jsonl",  "num_docs": 0, "sha256": "..." },
    "bm25":        { "path": "indexes/.../bm25/bm25_index" }
  },
  "s3_vectors": {
    "vector_bucket_name": "rag-vectors-vanh1234",
    "index_name": "hotpotqa-val500-bge-m3-v002",
    "dimension": 1024,
    "distance_metric": "cosine",
    "num_vectors": 0
  },
  "s3_layout": { "corpus": "s3://...", "processed": "s3://...", "bm25": "s3://...", "manifest": "s3://..." }
}
```

Vì sao file này xứng đáng tồn tại:

- Bộ nạp online đọc `params.embedding_model` và **mã hóa truy vấn bằng đúng mô hình đã dựng nên index**. Chỉ riêng trường này đã loại bỏ cả một nhóm lỗi âm thầm, kiểu truy hồi vẫn chạy nhưng trả về kết quả vô nghĩa vì bộ mã hóa truy vấn và chỉ mục lệch nhau.
- Các checksum `sha256` cho phép chứng minh file nằm trên S3 đúng là file mà notebook đã tạo ra.
- `s3_layout` giúp dịch vụ online tìm được mọi artifact chỉ từ manifest, thay vì phải tự ghép chuỗi đường dẫn trong code ứng dụng.

---

## 9. Bước 6 - Lắp ráp thư mục upload

Notebook sao chép mọi thứ vào một cây thư mục sạch phản chiếu đúng layout của S3, nhờ vậy bước upload ở chương 5.5 chỉ là một lệnh copy đệ quy duy nhất, không phải xoay xở đường dẫn:

```text
s3_manual_upload/hotpotqa-val500-bge-m3-v002/
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
        put_vectors_0001.json
        ...
        s3vectors_import_manifest.json
  ingest_s3vectors.py
```

Notebook còn **tự sinh ra file `ingest_s3vectors.py`** với tên bucket, tên index và region của bạn đã điền sẵn. Đó chính là script mà chương 5.6 sẽ chạy để đẩy vector lên:

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

{{% notice tip %}}
Hãy để ý những gì **không** có trong cây upload: không có cơ sở dữ liệu Chroma, không có trọng số mô hình, không có môi trường Python. Dịch vụ online chỉ cần khoảng một trăm megabyte file JSONL và file chỉ mục, còn vector thì lấy từ một dịch vụ được quản lý. Đó là lý do instance EC2 giữ được kích thước nhỏ.
{{% /notice %}}

---

## 10. Bước 7 - Kiểm tra trước khi upload

Cell cuối cùng khẳng định mọi file bắt buộc đều tồn tại và in ra các con số:

```python
required_files = [
    UPLOAD_ROOT / 'rag' / 'corpora' / CORPUS_ID / 'corpus.jsonl',
    UPLOAD_ROOT / 'rag' / 'corpora' / CORPUS_ID / 'eval.jsonl',
    UPLOAD_ROOT / 'rag' / 'processed' / PROCESSED_ID / 'parent_docs.jsonl',
    UPLOAD_ROOT / 'rag' / 'processed' / PROCESSED_ID / 'child_docs.jsonl',
    UPLOAD_ROOT / 'rag' / 'indexes' / INDEX_ID / 'manifests' / 'index_manifest.json',
    UPLOAD_ROOT / 'rag' / 'indexes' / INDEX_ID / 'bm25' / 'bm25_doc_ids.json',
    UPLOAD_ROOT / 'rag' / 'indexes' / INDEX_ID / 's3vectors-import' / 's3vectors_import_manifest.json',
    UPLOAD_ROOT / 'ingest_s3vectors.py',
]

missing = [path for path in required_files if not path.exists()]
assert not missing, missing

print('OK: all required files exist')
```

Output mong đợi:

```text
OK: all required files exist
corpus docs: 4937
eval questions: 500
parent docs: 4937
child docs: 8279
vector import files: 17
```

Đừng chuyển sang chương 5.5 chừng nào cell này chưa chạy qua. Bắt được một file thiếu ở đây chỉ tốn công chạy lại một cell; bắt được nó sau khi backend đã triển khai thì tốn cả một buổi debug trên một dịch vụ khởi động bình thường rồi mới hỏng ở truy vấn đầu tiên.

<!-- ẢNH 4 - SCREENSHOT.
     Output của cell kiểm tra: dòng "OK: all required files exist" cùng với toàn bộ
     các con số in ra (corpus docs, eval questions, parent docs, child docs,
     vector import files).
     Những con số này sẽ được trích lại ở chương 5.11, nên chụp cho rõ chữ. -->

![Output của bước kiểm tra](/images/5-Workshop/5.4-Offline-Artifact-Build/validation-checklist.png)

---

## 11. Các lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| Tiến trình chết chỉ với chữ `Killed`, không có traceback | Hết RAM lúc nạp mô hình embedding | Dùng runtime GPU, hoặc đổi sang `BAAI/bge-small-en-v1.5` |
| `ValueError: ... Keras 3 ... install tf-keras` | `transformers` phát hiện TensorFlow và cố nạp nó; dự án này chỉ cần PyTorch | Đặt `USE_TF=0` và `USE_FLAX=0` **trước khi** import |
| `ModuleNotFoundError: bm25s` / `sentence_transformers` / `datasets` | Thiếu dependency cho khâu offline | Cài từ `backend/requirements/offline.txt` |
| Assert thất bại ở `vectors.shape[1]` | Mô hình embedding không sinh ra đúng `VECTOR_DIMENSION` giá trị | Sửa cái sai - tên mô hình hoặc số chiều - trước khi tạo index ở 5.6 |
| Về sau truy hồi trả kết quả vô nghĩa với mọi câu hỏi | Index được dựng bằng một mô hình nhưng truy vấn bằng mô hình khác | Dựng lại, và luôn giữ tên mô hình bên trong `INDEX_ID` |

```python
import os
os.environ['USE_TF'] = '0'
os.environ['USE_FLAX'] = '0'
```

---

## 12. Kết quả

<!-- ẢNH 5 - SCREENSHOT.
     Thư mục s3_manual_upload/<INDEX_ID>/ đã hoàn tất, xem trong file browser của Colab
     (hoặc Google Drive / Explorer trên máy), mở rộng ra để thấy các thư mục con
     rag/corpora, rag/processed, rag/indexes và các file put_vectors_*.json.
     Ảnh này cho người đọc thấy chính xác họ phải có gì trước khi sang chương 5.5. -->

![Thư mục upload đã lắp ráp xong](/images/5-Workshop/5.4-Offline-Artifact-Build/upload-folder-tree.png)

| Tạo ra | Được dùng bởi |
| --- | --- |
| `corpus.jsonl`, `eval.jsonl`, `corpus_manifest.json` | Phần đánh giá, chương 5.11 |
| `parent_docs.jsonl`, `child_docs.jsonl`, `child_to_parent.json` | Bộ truy hồi online, mọi request |
| Chỉ mục BM25 + `bm25_doc_ids.json` | Truy hồi theo từ khóa, mọi request |
| `index_manifest.json` | Bộ nạp online lúc khởi động |
| `put_vectors_*.json` + `ingest_s3vectors.py` | Nạp vector, chương 5.6 |

Đến đây chưa có gì chạm tới AWS, và cũng không có gì trong số này còn phải chạy lại lúc xử lý request nữa. Chương 5.5 sẽ tạo S3 bucket và upload cây `rag/`; chương 5.6 sẽ tạo S3 Vectors index và nạp các batch vào.

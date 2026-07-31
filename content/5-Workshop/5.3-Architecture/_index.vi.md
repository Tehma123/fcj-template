---
title: "Kiến trúc hệ thống"
date: 2026-07-31
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

CloudHop RAG được triển khai dưới dạng một ứng dụng RAG trên nền tảng web tại **Asia Pacific (Singapore) Region (`ap-southeast-1`)**. Kiến trúc tách riêng frontend, lớp API, RAG backend và hệ thống lưu trữ phục vụ retrieval, giúp mỗi thành phần có vai trò rõ ràng nhưng vẫn phối hợp trong một luồng xử lý thống nhất.

Người dùng tương tác với frontend được triển khai trên **AWS Amplify**. Request được gửi qua **Amazon API Gateway** đến FastAPI backend chạy trên **Amazon EC2**. Backend thực hiện lexical retrieval bằng BM25 artifact lưu trên **Amazon S3**, đồng thời thực hiện dense retrieval thông qua **Amazon S3 Vectors**, sau đó xử lý các bằng chứng thu được trước khi gửi context cuối cùng đến Groq LLM API.

Chương này giải thích **hệ thống được ghép lại như thế nào và vì sao chọn từng dịch vụ AWS**, trước khi bắt tay vào xây dựng ở các chương sau. Hãy đọc phần này trước - mỗi bước từ 5.4 đến 5.9 chính là hiện thực hóa một khối trong sơ đồ bên dưới.

Bài toán ở đây là một **dịch vụ hỏi đáp RAG (Retrieval-Augmented Generation) đa bước (multi-hop)**. Người dùng đặt một câu hỏi ngôn ngữ tự nhiên mà không thể trả lời được từ một tài liệu duy nhất (ví dụ: *"Were Scott Derrickson and Ed Wood of the same nationality?"*). Hệ thống phải tìm nhiều tài liệu, tổng hợp bằng chứng từ chúng, và trả về một câu trả lời ngắn gọn kèm theo các nguồn đã sử dụng.

---

## 1. Quyết định thiết kế cốt lõi: tách offline khỏi online

Lựa chọn kiến trúc quan trọng nhất của dự án này là **những việc tốn kém không bao giờ được thực hiện trong lúc xử lý request**.

| | Pipeline offline | Pipeline online |
| --- | --- | --- |
| Chạy khi nào | Một lần cho mỗi phiên bản corpus / index | Mỗi lần người dùng gửi request |
| Làm gì | Chia nhỏ tài liệu, chạy mô hình embedding, dựng chỉ mục BM25, upload artifact | Nạp artifact có sẵn, truy hồi, gọi LLM, trả lời |
| Chạy ở đâu | Notebook / script trên máy trạm | Tiến trình FastAPI trên EC2 |
| Đặc tính chi phí | Nặng, nhưng chỉ trả một lần | Nhẹ, phải giữ trong vài giây |

Dịch vụ online **không bao giờ chia nhỏ tài liệu, không bao giờ embed tài liệu, và không bao giờ dựng lại chỉ mục**. Nó chỉ *nạp* những artifact đã tồn tại sẵn trong Amazon S3.

{{% notice tip %}}
Vì sao điều này quan trọng: dựng lại một vector index tốn hàng phút CPU. Nếu công việc đó nằm trong đường đi của request, mọi truy vấn đều sẽ timeout. Việc tách hai pipeline chính là thứ giúp API trả lời được trong giới hạn timeout của API Gateway, và cũng là thứ cho phép phục vụ một index hoàn toàn mới mà **không phải sửa một dòng code nào** (xem mục 8).
{{% /notice %}}

---

## 2. Kiến trúc tổng thể

<!-- ẢNH 1 - SƠ ĐỒ KIẾN TRÚC (vẽ bằng draw.io / Excalidraw, KHÔNG phải screenshot).
     Cần thể hiện, từ trái sang phải:
       [Trình duyệt] --HTTPS--> [AWS Amplify Hosting: React/Vite]
                     --HTTPS--> [Amazon API Gateway (HTTP API)]
                     --HTTP :8000--> [Amazon EC2 (Ubuntu) - FastAPI chạy dưới systemd]
     Từ khối EC2, vẽ mũi tên tới:
       [Amazon S3]  (processed docs, BM25 index, manifest)
       [Amazon S3 Vectors]  (dense embeddings, QueryVectors)
       [Groq API]  (bên ngoài, nằm NGOÀI khung AWS)
     Vẽ thêm một khung nét đứt riêng ghi "OFFLINE (chạy một lần)":
       [corpus.jsonl] -> [chunk + embed + BM25] -> [Amazon S3] + [Amazon S3 Vectors]
     Đánh dấu region ap-southeast-1 bao quanh các tài nguyên AWS.
     Gắn icon IAM role lên khối EC2 với nhãn rag-ec2-runtime-role. -->

![Sơ đồ kiến trúc](/images/5-Workshop/5.3-Architecture/architecture-overview.png)

Hệ thống gồm bốn tầng:

1. **Tầng giao diện** - ứng dụng single-page React/Vite host trên **AWS Amplify Hosting**, phục vụ qua HTTPS từ CDN.
2. **Tầng biên / API** - **Amazon API Gateway** (HTTP API) đảm nhiệm kết thúc TLS, quản lý CORS và chuyển tiếp request xuống backend. Các route: `GET /health`, `POST /warmup`, `POST /query`.
3. **Tầng ứng dụng** - dịch vụ **FastAPI** trên **Amazon EC2** (Ubuntu), quản lý bởi `systemd` với tên `aws-rag-api`, lắng nghe cổng `8000`. Đây là thành phần tính toán duy nhất chạy theo từng request.
4. **Tầng dữ liệu** - **Amazon S3** (artifact văn bản + chỉ mục BM25) và **Amazon S3 Vectors** (dense embeddings). Cấu hình runtime không phải một dịch vụ ở đây: nó nằm trong một file `.env.prod` duy nhất trên instance (chương 5.7 mục 7).

**Groq API** là một endpoint LLM dạng SaaS bên ngoài. Nó được vẽ *bên ngoài* ranh giới AWS trong sơ đồ một cách có chủ đích, vì đây là phụ thuộc duy nhất đi ra khỏi tài khoản AWS.

---

## 3. Pipeline nạp dữ liệu offline

<!-- ẢNH 2 - SƠ ĐỒ (draw.io). Một luồng đơn giản từ trái sang phải:
     corpus.jsonl (tập con HotpotQA)
       -> chunk: parent docs (nguyên bài viết) + child docs (đoạn nhỏ)
       -> embed child docs bằng BAAI/bge-m3
       -> dựng chỉ mục BM25 (bm25s)
       -> ghi index_manifest.json
       -> upload lên Amazon S3  VÀ  PutVectors lên Amazon S3 Vectors
     Nhớ chú thích rõ bước "parent/child", đây là phần người đọc hay nhầm nhất. -->

![Pipeline nạp dữ liệu offline](/images/5-Workshop/5.3-Architecture/offline-pipeline.png)

Các bước, theo thứ tự:

1. **Dựng corpus** - trích một tập con của HotpotQA ra `corpus.jsonl`.
2. **Chia nhỏ theo hai cấp** - mỗi bài viết trở thành một **parent document** (nguyên bài, dùng làm ngữ cảnh cuối cùng) và nhiều **child document** (các đoạn nhỏ, dùng để so khớp). Đây là mẫu thiết kế *small-to-big*: so khớp trên đoạn nhỏ để chính xác, nhưng đưa cho LLM nguyên bài viết bao quanh.
3. **Embed các child document** bằng mô hình sentence-embedding `BAAI/bge-m3`.
4. **Dựng chỉ mục BM25** (thư viện `bm25s`) trên chính các child document đó, tạo ra một bộ truy hồi theo từ khóa song song với bộ truy hồi vector.
5. **Ghi `index_manifest.json`** - ghi lại mô hình embedding, kích thước chunk và đường dẫn đã tạo ra index này. Bộ nạp online đọc manifest để không bao giờ ghép nhầm một index với sai mô hình embedding.
6. **Upload** các tài liệu đã xử lý, chỉ mục BM25 và manifest lên Amazon S3, đồng thời đẩy embedding vào Amazon S3 Vectors qua `PutVectors`.

Các artifact được tạo ra (đều nằm dưới một id có đánh phiên bản, ví dụ `hotpotqa-val500-v002`):

| Artifact | Lưu ở | Dùng online để |
| --- | --- | --- |
| `parent_docs.jsonl` | Amazon S3 | Ngữ cảnh cuối cùng đưa cho LLM |
| `child_docs.jsonl` | Amazon S3 | Đơn vị truy hồi |
| Các file chỉ mục BM25 | Amazon S3 | Truy hồi theo từ khóa |
| `index_manifest.json` | Amazon S3 | Kiểm tra tương thích lúc khởi động |
| Dense embeddings | Amazon S3 Vectors | Truy hồi theo ngữ nghĩa |

---

## 4. Pipeline xử lý truy vấn online

<!-- ẢNH 3 - SƠ ĐỒ (draw.io). Luồng dọc hoặc ngang của MỘT request:
     câu hỏi
       -> [tùy chọn] Groq: tách thành các câu hỏi con
       -> với mỗi câu hỏi con: truy hồi BM25  +  truy hồi S3 Vectors
       -> hợp nhất bằng Reciprocal Rank Fusion (RRF)
       -> small-to-big: child chunk -> parent article
       -> lập kế hoạch hop thích ứng (Groq: "đủ chưa?" / "truy vấn tiếp theo")  [mũi tên vòng lại bước truy hồi]
       -> [tùy chọn] rerank bằng cross-encoder
       -> Groq: sinh câu trả lời ngắn
       -> phản hồi {answer, sources, timings, token_usage}
     Mũi tên vòng lại của hop planner là điểm nhấn quan trọng nhất - vẽ cho thật rõ. -->

![Luồng xử lý truy vấn online](/images/5-Workshop/5.3-Architecture/online-query-flow.png)

Điều gì xảy ra khi một request đến `POST /query`:

1. **Tách câu hỏi** *(tùy chọn)* - Groq tách một câu hỏi đa bước thành các câu hỏi con.
2. **Truy hồi lai (hybrid)** - với mỗi câu hỏi con, chạy song song **BM25** (từ khóa) và **S3 Vectors** (ngữ nghĩa), rồi hợp nhất hai danh sách xếp hạng bằng **Reciprocal Rank Fusion**. Một mình bộ nào cũng không đủ: BM25 mạnh với tên riêng và từ hiếm, còn vector mạnh với cách diễn đạt khác đi.
3. **Mở rộng small-to-big** - các child chunk thắng cuộc được ánh xạ ngược về bài viết cha của chúng.
4. **Lập kế hoạch hop thích ứng** - Groq xem xét bằng chứng đã thu thập được và hoặc tuyên bố là *đã đủ*, hoặc đề xuất truy vấn tìm kiếm tiếp theo. Đây chính là thứ khiến hệ thống thực sự đa bước chứ không phải chỉ tra cứu một lần.
5. **Rerank bằng cross-encoder** *(tùy chọn, tắt trong bản demo triển khai để giảm độ trễ)* - chấm điểm lại các bài viết ứng viên so với câu hỏi.
6. **Sinh câu trả lời** - Groq tạo ra câu trả lời **dạng ngắn** (một từ, một tên, một con số) từ ngữ cảnh đã chọn.
7. **Phản hồi** - JSON gồm `answer`, `sources`, `timings` và `token_usage`, để frontend có thể hiển thị bằng chứng đằng sau câu trả lời.

{{% notice note %}}
Trọng số truy hồi **phụ thuộc vào hop**, không cố định. Hop đầu tiên thường là tra cứu thực thể có tên nên BM25 được đánh trọng số cao hơn; các hop sau thường là câu viết lại mang tính quan hệ nên bộ truy hồi vector được ưu tiên hơn. Toàn bộ phần tinh chỉnh này nằm hoàn toàn trong cấu hình - xem chương 5.12.
{{% /notice %}}

---

## 5. Các dịch vụ AWS được sử dụng và lý do lựa chọn

Dự án sử dụng **tám dịch vụ AWS**. Mỗi dòng nêu rõ vai trò trong hệ thống và lý do chọn nó thay vì phương án thay thế hiển nhiên.

| Dịch vụ | Vai trò trong hệ thống | Vì sao chọn dịch vụ này |
| --- | --- | --- |
| **Amazon S3** | Lưu tài liệu đã xử lý, chỉ mục BM25 và manifest | Bền vững, gần như miễn phí ở quy mô dữ liệu này, và tách rời khỏi vòng đời của instance. Nếu lưu artifact trên ổ đĩa EC2 thì dữ liệu bị buộc vào một máy chủ duy nhất và sẽ mất khi thay instance. Việc đặt tiền tố theo phiên bản cho phép nhiều thế hệ index cùng tồn tại. |
| **Amazon S3 Vectors** | Lưu trữ vector và tìm kiếm tương đồng (`PutVectors` / `QueryVectors`) | Một vector store được quản lý, không phải vận hành máy chủ và không tốn chi phí lúc rảnh. OpenSearch Serverless tính phí công suất tối thiểu ngay cả khi không dùng; còn tự dựng FAISS/Chroma trên EC2 thì bị giới hạn bởi RAM và mất theo instance. |
| **Amazon EC2** | Chạy ứng dụng FastAPI | Tiến trình nạp mô hình embedding PyTorch và chỉ mục BM25 vào bộ nhớ lúc khởi động rồi dùng lại cho mọi request. Một tiến trình sống lâu sẽ phân bổ đều chi phí nạp đó. AWS Lambda sẽ phải trả chi phí này ở mỗi lần cold start và bất tiện với các thư viện ML lớn; ECS Fargate thì thêm độ phức tạp điều phối mà một demo một container không cần đến. |
| **Amazon API Gateway** (HTTP API) | Điểm vào HTTPS, xử lý CORS, định tuyến tới EC2 | Amplify phục vụ frontend qua HTTPS, và trình duyệt không cho phép một trang HTTPS gọi tới API HTTP thuần (**Mixed Content**). API Gateway kết thúc TLS bằng chứng chỉ được quản lý sẵn rồi chuyển tiếp xuống EC2 qua HTTP. ALB cũng làm được nhưng tốn phí theo giờ cao hơn và phải tự thiết lập chứng chỉ/tên miền. |
| **AWS Amplify Hosting** | Host và build frontend React | Triển khai bằng git push, TLS và CDN có sẵn. S3 + CloudFront cho kết quả tương tự nhưng phải làm thủ công nhiều bước hơn. |
| **AWS Systems Manager** | Session Manager để truy cập quản trị vào instance | Loại bỏ nhu cầu mở cổng `22` hay quản lý key pair, đồng thời ghi lại mọi phiên làm việc. Đây là năng lực Systems Manager duy nhất mà bản triển khai hiện đang dùng - xem ghi chú bên dưới. |
| **AWS IAM** | Instance profile `rag-ec2-runtime-role` | Ứng dụng **hoàn toàn không dùng access key** - thông tin xác thực được lấy từ instance role qua IMDSv2. Xem mục 7. |

{{% notice info %}}
**Groq** (bên ngoài) được dùng cho các lệnh gọi LLM - tách câu hỏi, lập kế hoạch hop và sinh câu trả lời - thay vì Amazon Bedrock, vì các mô hình open-weight mà Groq host trả về câu trả lời ngắn với độ trễ rất thấp, và đó chính là thứ giữ toàn bộ request nằm trong giới hạn timeout của API Gateway. Đây là phụ thuộc phi-AWS duy nhất, và nó được cô lập sau một module riêng nên có thể thay thế được.
{{% /notice %}}

---

## 6. Đường đi của request từ đầu đến cuối

```text
Trình duyệt
  │  HTTPS
  ▼
AWS Amplify Hosting  (React/Vite SPA)
  │  HTTPS  POST /query
  ▼
Amazon API Gateway (HTTP API)          ← kết thúc TLS + CORS
  │  HTTP  :8000
  ▼
Amazon EC2 - FastAPI (systemd: aws-rag-api)
  ├──► Amazon S3            parent/child docs, chỉ mục BM25, manifest
  ├──► Amazon S3 Vectors    QueryVectors (truy hồi ngữ nghĩa)
  └──► Groq API             tách câu hỏi · lập kế hoạch hop · sinh câu trả lời
```

Frontend bắt buộc phải được cấu hình bằng **URL HTTPS của API Gateway**, tuyệt đối không dùng địa chỉ EC2 thô. Đây là lỗi phổ biến nhất khi làm lại workshop này, và chương 5.8 sẽ nói kỹ về nó.

---

## 7. Thiết kế bảo mật

Bảo mật ở đây nằm ở việc loại bỏ thông tin xác thực tồn tại lâu dài và đóng các cánh cửa mặc định, chứ không phải thêm sản phẩm mới.

- **Không hard-code thông tin xác thực ở bất cứ đâu.** Ứng dụng không giữ access key hay secret key nào của AWS. Instance EC2 mang instance profile **`rag-ec2-runtime-role`**, và AWS SDK lấy thông tin xác thực tạm thời qua IMDSv2.
- **Không có AWS access key ở bất cứ đâu.** Mọi lệnh gọi AWS đều được cấp quyền bởi instance role thông qua IMDSv2. Thông tin xác thực phi-AWS duy nhất là Groq API key thì hiện đang nằm trong `.env.prod` trên instance - đây là một hạn chế còn để ngỏ được mô tả ở chương 5.13, không phải vấn đề đã giải quyết.
- **Least privilege trên instance role.** Role chỉ được cấp đúng những gì dịch vụ thực sự làm: đọc các object artifact trong S3 bucket của dự án, truy vấn index S3 Vectors, đọc các tham số `/prod/aws-rag/*`, đọc đúng một secret đó, và ghi log CloudWatch - cộng thêm `AmazonSSMManagedInstanceCore` cho Session Manager.
- **Không mở cổng SSH.** Truy cập quản trị dùng **AWS Systems Manager Session Manager** thay cho việc mở cổng `22`. Cách này loại bỏ luật "cho phép IP hiện tại của tôi" vốn hỏng mỗi khi đổi mạng, đồng thời để lại bản ghi phiên có thể kiểm toán.
- **Địa chỉ backend ổn định và kiểm soát được.** Instance dùng **Elastic IP** để tích hợp API Gateway không âm thầm trỏ nhầm host sau mỗi lần stop/start.
- **Bucket ở chế độ riêng tư.** Cả bucket artifact lẫn bucket vector đều để private; không có gì được phục vụ công khai từ S3. Mọi truy cập đều đi qua instance role.
- **CORS là danh sách cho phép, không dùng ký tự đại diện.** Tham số `/prod/aws-rag/cors-allow-origins` liệt kê tường minh origin của Amplify và các origin dev cục bộ.

<!-- ẢNH 4 - SCREENSHOT.
     Console: IAM -> Roles -> rag-ec2-runtime-role -> tab Permissions.
     Chụp các policy đã gắn (inline policy S3 + S3 Vectors, và
     AmazonSSMManagedInstanceCore) để chứng minh cho khẳng định chỉ-đọc ở trên.
     Nhớ làm mờ / cắt bỏ AWS account id 12 chữ số trước khi publish. -->

![IAM role của backend EC2](/images/5-Workshop/5.3-Architecture/iam-role-permissions.png)

{{% notice warning %}}
Hạn chế đã biết, xin nêu thẳng thắn: backend hiện vẫn lắng nghe cổng `8000` trên một IP công khai, chỉ được giới hạn bằng luật security group. Với một triển khai production thực thụ, instance nên nằm trong private subnet và chỉ API Gateway (thông qua VPC Link) mới tiếp cận được. Vấn đề này sẽ được bàn lại ở chương 5.13.
{{% /notice %}}

---

## 8. Vận hành điều khiển bằng cấu hình

Mọi tham số điều chỉnh đều được đọc từ một biến môi trường lúc khởi động tiến trình, do một file `.env.prod` duy nhất trên instance cung cấp, nên **phục vụ một index khác không cần sửa code và không cần deploy lại** - bạn dựng artifact mới, sửa ba dòng, rồi khởi động lại dịch vụ.

| Nhóm biến | Ví dụ | Điều khiển |
| --- | --- | --- |
| Vị trí artifact | `S3_ARTIFACT_BUCKET`, `S3_PROCESSED_ID`, `S3_VECTOR_BUCKET`, `S3_VECTOR_INDEX`, `RAG_INDEX_ID` | Corpus/index nào đang được phục vụ |
| Hành vi truy hồi | `BM25_TOP_K`, `VECTOR_TOP_K`, `HOP_CANDIDATE_CAP`, `MAX_ADAPTIVE_HOPS`, `HOP_EVIDENCE_TOP_N` | Tìm kiếm rộng và sâu đến đâu |
| Đánh đổi độ trễ / chất lượng | `RAG_FAST_MODE`, `RAG_USE_RERANKER`, `RERANK_TOP_N`, `RAG_DEVICE` | Tốc độ so với chất lượng câu trả lời |
| Vận hành | `CORS_ALLOW_ORIGINS`, `RAG_WARMUP_QUESTION`, `AWS_REGION` | Truy cập và làm nóng hệ thống |

Đây chính là thứ biến bản triển khai thành một *nền tảng* thay vì một sản phẩm dùng một lần: cùng một instance EC2 đang chạy có thể phục vụ một cơ sở tri thức hoàn toàn khác chỉ sau một lần sửa file và một lệnh `systemctl restart`.

{{% notice note %}}
Cái giá của việc cấu hình bằng file là mỗi lần đổi đều phải mở một phiên làm việc trên instance. Repository đã có sẵn code để đọc đúng những thiết lập đó từ **AWS Systems Manager Parameter Store**, còn Groq key thì lấy từ **AWS Secrets Manager** - cách đó sẽ biến một lần đổi thành một lệnh gọi API duy nhất và đưa bí mật ra khỏi ổ đĩa. Bước chuyển đó đã được thiết kế nhưng chưa triển khai; chương 5.7 mục 7 mô tả nó và chương 5.13 liệt kê nó như một hạn chế còn để ngỏ.
{{% /notice %}}

---

## 9. Ngân sách độ trễ

API Gateway áp một giới hạn timeout tích hợp cứng khoảng **30 giây**. Pipeline chất lượng đầy đủ - tách câu hỏi, nhiều hop, rerank bằng cross-encoder trên CPU - có thể vượt quá mức đó. Hai cơ chế giúp bản demo triển khai nằm trong ngân sách này:

- **`RAG_FAST_MODE`** bỏ qua bước tách câu hỏi và giảm top-k cùng số hop, đánh đổi một phần chất lượng truy hồi lấy tốc độ. Bộ reranker cũng được **tắt** trên production vì cùng lý do.
- **Route `POST /warmup`** kích hoạt một truy vấn giả hoàn chỉnh để mô hình embedding, chỉ mục BM25 và vector client đều được nạp và cache sẵn trước khi người dùng thật đầu tiên tới. Nếu không có nó, request đầu tiên sau khi khởi động lại sẽ phải gánh toàn bộ chi phí nạp mô hình và bị timeout.

Số liệu độ trễ và độ chính xác đo được cho cả hai chế độ sẽ được báo cáo ở chương 5.11.

---

## 10. Khả năng mở rộng và vận hành

Hiện trạng và hướng đi tiếp theo, nêu một cách trung thực:

| Khía cạnh | Hiện tại | Bước tiếp theo |
| --- | --- | --- |
| Tính toán | Một instance EC2, một tiến trình FastAPI | Auto Scaling group đứng sau ALB, hoặc đóng gói container và chuyển sang ECS |
| Lưu trữ | S3 + S3 Vectors, vốn đã được quản lý và gần như không giới hạn | Không cần thay đổi - tầng này đã tự mở rộng |
| Tính sẵn sàng | Một instance, một AZ | Multi-AZ sau khi đưa tầng tính toán ra sau load balancer |
| Giám sát | Log và metric CloudWatch, `systemd` tự khởi động lại khi lỗi | CloudWatch alarm cho tỉ lệ 5xx và độ trễ, trình bày ở chương 5.12 |
| Thay đổi cấu hình | Sửa `.env.prod` + khởi động lại dịch vụ | Chuyển sang Parameter Store để không cần mở phiên shell |

Tầng truy hồi và lưu trữ vốn đã serverless và co giãn được; **nút thắt cổ chai duy nhất là instance EC2 đơn lẻ**, và đó là nút thắt được chọn một cách có chủ đích để giữ chi phí demo ở mức thấp.

---

## 11. Tài nguyên đã triển khai

| Hạng mục | Giá trị |
| --- | --- |
| Region | `ap-southeast-1` |
| Frontend | AWS Amplify Hosting (HTTPS) |
| API | Amazon API Gateway HTTP API → EC2 Elastic IP `:8000` |
| Dịch vụ backend | `aws-rag-api` (systemd) trên Ubuntu EC2 |
| Bucket artifact | `aws-rag-bucket-vanh1234` |
| Bucket vector | `rag-vectors-vanh1234` |
| Vector index | `hotpotqa-val500-bge-m3-v002` |
| Processed id | `hotpotqa-val500-v002` |
| Mô hình embedding | `BAAI/bge-m3` |
| Tiền tố cấu hình | `/prod/aws-rag/*` |
| Instance role | `rag-ec2-runtime-role` |

<!-- ẢNH 5 - SCREENSHOT (bằng chứng end-to-end, để ở cuối).
     Frontend đã triển khai, mở trên trình duyệt, sau khi hỏi:
       "Were Scott Derrickson and Ed Wood of the same nationality?"
     Chụp cả cửa sổ sao cho thấy được TẤT CẢ những thứ sau:
       - thanh địa chỉ hiển thị URL https:// của Amplify (chứng minh HTTPS đầu-cuối)
       - biểu tượng ổ khóa
       - câu trả lời trả về
       - danh sách nguồn/bằng chứng bên dưới câu trả lời
     Riêng tấm này chứng minh tiêu chí "triển khai end-to-end" trong thang điểm. -->

![Hệ thống đã triển khai trả lời một câu hỏi đa bước](/images/5-Workshop/5.3-Architecture/end-to-end-demo.png)

{{% notice note %}}
Tên bucket là duy nhất trên toàn cầu - khi bạn làm lại workshop này thì phải chọn tên riêng của mình. Mọi chương từ 5.4 trở đi đều tham chiếu ngược lại những tên bạn chọn ở đây.
{{% /notice %}}

---

Kiến trúc đã rõ, chương 5.4 sẽ bắt đầu xây dựng bằng việc tạo ra các artifact offline.

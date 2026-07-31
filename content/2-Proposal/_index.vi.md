---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Retrieval-Augmented Generation Cho Suy Luận Đa Bước Trên HotpotQA
## Một Pipeline RAG Hybrid-Retrieval Thích Nghi Với Lập Kế Hoạch Truy Vấn Theo Hop, Triển Khai Trên AWS

### 1. Tóm tắt điều hành
Dự án này thiết kế và triển khai một hệ thống Retrieval-Augmented Generation (RAG) để trả lời các câu hỏi suy luận đa bước (multi-hop) từ bộ dữ liệu HotpotQA — những câu hỏi mà câu trả lời đòi hỏi kết hợp bằng chứng từ nhiều tài liệu thay vì chỉ một đoạn văn bản. Hệ thống được xác định phạm vi như một demo end-to-end đầy đủ: một pipeline lập chỉ mục offline chunk và embed một validation slice của HotpotQA, một dịch vụ truy xuất-và-sinh câu trả lời online bằng FastAPI expose một API công khai, và một front end React để kiểm thử tương tác. Quy mô mục tiêu là một demo nhỏ, tiết kiệm chi phí (subset validation HotpotQA gồm 100 tài liệu, lưu lượng truy vấn thấp) thay vì một dịch vụ quy mô production, phục vụ mục đích đánh giá nội bộ, trình diễn workshop, và làm kiến trúc tham chiếu có thể mở rộng sau này cho corpus lớn hơn. Đối tượng sử dụng dự kiến là người đánh giá workshop, đội kỹ thuật đánh giá chất lượng truy xuất, và các kỹ sư tương lai sẽ mở rộng pipeline này sang các bộ tài liệu khác.

### 2. Tuyên bố vấn đề
#### Vấn đề hiện tại
Các mô hình ngôn ngữ lớn (LLM) độc lập và các hệ thống hỏi-đáp một bước thường gặp khó khăn với câu hỏi đa bước, vì câu trả lời không thể tìm thấy trong một tài liệu duy nhất mà cần truy xuất và suy luận qua nhiều đoạn văn bản liên quan. Nếu không có bước truy xuất dựa trên nguồn dữ liệu gốc, câu trả lời có thể thiếu chính xác, không có căn cứ, hoặc bỏ sót các bước suy luận trung gian. Điều này đặc biệt quan trọng với các câu hỏi kiểu HotpotQA vì hai tài liệu hỗ trợ chỉ được kết nối với nhau qua một thực thể "cầu nối" (bridge entity) chung (ví dụ: một người, bộ phim, hoặc tổ chức được nhắc đến trong cả hai bài viết) — một lượt tìm kiếm dense hoặc keyword-search đơn lẻ thường chỉ truy xuất được một trong hai tài liệu chứ không phải cả hai, và một pipeline RAG "naive" một lượt (retrieve một lần rồi generate) không có cơ chế nào để nhận ra bằng chứng truy xuất được là chưa đầy đủ và tìm kiếm thêm lần nữa.

#### Giải pháp
Hệ thống truy xuất các đoạn văn bản liên quan từ kho tài liệu HotpotQA và dùng một mô hình ngôn ngữ để sinh câu trả lời cuối cùng dựa trên bằng chứng đã truy xuất, kết hợp nhiều bước truy xuất/suy luận nối tiếp nhau để giải quyết câu hỏi đa bước. Cụ thể, stack công nghệ kết hợp: **BAAI/bge-m3** làm mô hình embedding dense mã nguồn mở; một **retriever lai giữa sparse và dense** (BM25 qua thư viện `bm25s`, hợp nhất với dense vector search qua `EnsembleRetriever` của LangChain bằng Reciprocal Rank Fusion có trọng số); **Amazon S3 Vectors** (hoặc một instance ChromaDB cục bộ cho môi trường phát triển) làm vector store; một **vòng lặp phân rã truy vấn và lập kế hoạch hop thích nghi do LLM điều khiển** (qua Groq API, `llama-3.1-8b-instant`) kiểm tra bằng chứng sau mỗi vòng truy xuất và quyết định dừng lại hay phát hành một truy vấn follow-up cụ thể hơn; một **cross-encoder reranker** (`cross-encoder/ms-marco-MiniLM-L-6-v2`) để chấm điểm các candidate parent document dựa trên câu hỏi; và bước cuối cùng **sinh câu trả lời dạng ngắn** khớp với định dạng câu trả lời của HotpotQA để phục vụ chấm điểm Exact Match / F1 tự động. Toàn bộ pipeline được điều phối bởi một class `AdvancedRAGPipeline` duy nhất và được phục vụ qua một ứng dụng FastAPI.

#### Lợi ích và hoàn vốn đầu tư (ROI)
Lợi ích chính là độ chính xác được cải thiện có thể đo lường được trên câu hỏi đa bước so với một baseline truy xuất một lượt, vì vòng lặp lập kế hoạch hop thích nghi nhắm thẳng vào lỗi bridge-entity đã mô tả ở trên thay vì dựa vào một lượt truy xuất cố định. Một lợi ích thứ hai, mang tính dài hạn hơn, là khả năng tái sử dụng về mặt kiến trúc: vì pipeline lập chỉ mục offline và pipeline truy vấn online được tách biệt nghiêm ngặt (toàn bộ chunking/embedding/xây chỉ mục chỉ chạy một lần, offline; dịch vụ online chỉ load các artifact đã build sẵn), cùng một codebase có thể được trỏ lại sang một corpus khác bằng cách build một artifact bundle mới và đổi một tham số cấu hình duy nhất (`index_id`), không cần sửa code và không cần redeploy dịch vụ. Điều này khiến khoản đầu tư vào logic truy xuất/rerank/lập kế hoạch hop có thể chuyển giao sang các corpus có giá trị cao hơn trong tương lai (ví dụ: tài liệu kỹ thuật nội bộ) thay vì chỉ là một demo HotpotQA dùng một lần. Về chi phí, demo được thiết kế để chạy gần như hoàn toàn trong giới hạn AWS Free Tier ở mức lưu lượng truy vấn thấp (xem Mục 6), giữ chi phí biên của việc chạy và lặp lại trên bản triển khai workshop gần như bằng 0.

### 3. Kiến trúc giải pháp
Hệ thống được chia thành hai pipeline độc lập. **Pipeline offline** chạy một lần cho mỗi phiên bản corpus/index: nó đọc `corpus.jsonl`, tách mỗi bài viết thành một chunk *parent* (toàn bộ bài viết, dùng làm ngữ cảnh sinh câu trả lời) và nhiều chunk *child* (250–500 ký tự, dùng cho độ chính xác tìm kiếm), embed các child chunk bằng BAAI/bge-m3, xây một chỉ mục sparse BM25 trên cùng các child chunk đó, và ghi một `index_manifest.json` có phiên bản trước khi upload toàn bộ lên Amazon S3 / Amazon S3 Vectors. **Pipeline online** chạy cho mỗi HTTP request: nó load các artifact đã build sẵn (không bao giờ chunk hay embed lại), tùy chọn phân rã câu hỏi thành các câu hỏi con, truy xuất các candidate parent document qua hybrid BM25+vector search hợp nhất bằng Reciprocal Rank Fusion, mở rộng các child chunk trúng về lại bài viết parent của chúng ("small-to-big"), hỏi một LLM hop-planner xem bằng chứng thu thập được đã đủ hay cần một lượt tìm kiếm nhắm mục tiêu khác (tối đa một số hop có thể cấu hình), rerank các candidate còn lại bằng cross-encoder, xây một context window đã lọc, và cuối cùng hỏi một LLM để sinh câu trả lời dạng ngắn. Response trả về cho caller bao gồm câu trả lời, các tài liệu nguồn hỗ trợ kèm điểm rerank, thời gian độ trễ theo từng giai đoạn, và mức sử dụng token LLM — khiến mỗi request tự mô tả được để phục vụ debug và theo dõi chi phí.

```text
Browser (React / Vite)
  -> HTTPS: AWS Amplify Hosting
  -> HTTPS: Amazon API Gateway (HTTP API)     [terminates TLS, avoids browser "Mixed Content" errors]
  -> HTTP:  Amazon EC2 (FastAPI under systemd)
       -> Amazon S3                (processed docs + BM25 index + manifest)
       -> Amazon S3 Vectors        (dense vector retrieval)
       -> AWS Systems Manager Parameter Store (non-secret runtime config)
       -> AWS Secrets Manager      (Groq API key)
       -> Amazon CloudWatch        (logs & metrics)
       -> Groq API (third-party)   (query decomposition, hop planning, answer generation)
```

![Kiến trúc triển khai trên AWS: Amplify -> API Gateway -> EC2 (VPC, public subnet) -> S3 (sparse search), S3 Vectors (dense search), Secrets Manager, Systems Manager, và CloudWatch, với EC2 gọi trực tiếp Groq LLM API bên ngoài](/images/2-Proposal/AWS-RAG.drawio.png)
*Kiến trúc triển khai: trình duyệt tiếp cận backend FastAPI qua Amplify và API Gateway; instance EC2 (gắn IAM role, nằm trong public subnet của VPC) lấy Groq API key từ Secrets Manager, đọc chỉ mục sparse/dense từ S3 và S3 Vectors, được cấu hình qua Systems Manager, gửi log và metrics về Amazon CloudWatch, và gọi trực tiếp Groq API bên ngoài để inference LLM.*

#### Dịch vụ AWS sử dụng
- **Amazon S3** — lưu trữ bền vững cho các artifact offline: file JSONL parent/child document, chỉ mục BM25 đã serialize, và index manifest ghi lại embedding model, kích thước chunk, và checksum cho mỗi lần build.
- **Amazon S3 Vectors** — dịch vụ vector-search được quản lý dùng cho dense retrieval ở production, được truy vấn theo từng request qua `QueryVectors` và được nạp dữ liệu lúc build qua `PutVectors`; thay thế một instance ChromaDB cục bộ dùng trong giai đoạn phát triển.
- **Groq API (lưu key qua AWS Secrets Manager)** — host các LLM dùng cho phân rã truy vấn, lập kế hoạch hop thích nghi, và sinh câu trả lời dạng ngắn; bản thân API key được lưu và truy xuất từ AWS Secrets Manager thay vì hard-code.
- **Amazon EC2, Amazon API Gateway, AWS Amplify Hosting, AWS Systems Manager (Parameter Store + Session Manager), AWS IAM, và một Elastic IP** — các dịch vụ compute, API, hosting, cấu hình, kiểm soát truy cập, và networking cùng nhau host và expose pipeline như một endpoint demo công khai (chi tiết đầy đủ ở Mục 4).
- **Amazon CloudWatch** — thu thập log và metrics từ dịch vụ chạy trên EC2 để theo dõi lưu lượng request, lỗi, và độ trễ của bản demo đã triển khai.

#### Thiết kế thành phần
- **Tiếp nhận dữ liệu (Data Ingestion)**: `scripts/build_offline_artifacts.py` đọc một validation slice của HotpotQA (`corpus.jsonl`), và `advanced_rag/chunking.py` song song hóa (qua `multiprocessing.Pool`) việc tách thành parent document (toàn văn bài viết) và child document (chunk 250–500 ký tự với 20% overlap, dùng một recursive character splitter ưu tiên ranh giới đoạn văn/câu).
- **Truy xuất (Retrieval)**: `advanced_rag/retrieval.py` xây một hybrid retriever cho mỗi hop bằng cách kết hợp một BM25 sparse retriever (`bm25s`) và một dense vector retriever (Amazon S3 Vectors hoặc ChromaDB, dựa trên embedding BAAI/bge-m3) qua `EnsembleRetriever` của LangChain, hợp nhất hai danh sách xếp hạng bằng Reciprocal Rank Fusion có trọng số; các child chunk trúng sau đó được mở rộng về lại bài viết parent của chúng ("small-to-big").
- **Suy luận đa bước (Multi-Hop Reasoning)**: `advanced_rag/query_optimizer.py` trước tiên phân rã câu hỏi gốc thành các câu hỏi con theo thứ tự bằng một LLM; `advanced_rag/hop_planner.py` sau đó điều khiển một vòng lặp thích nghi (tối đa một số hop có thể cấu hình) đọc bằng chứng đã truy xuất được cho tới thời điểm đó và hoặc tuyên bố câu hỏi đã có thể trả lời được, hoặc đề xuất một truy vấn follow-up mới, cụ thể hơn, dựa trên một fact vừa phát hiện được — thay thế một heuristic bridge-entity dựa trên regex trước đó vốn dễ gãy.
- **Sinh câu trả lời (Answer Generation)**: `advanced_rag/rerank.py` chấm điểm các candidate còn lại bằng cross-encoder so với câu hỏi gốc và mọi câu hỏi con/truy vấn hop, chỉ giữ lại top-N; `advanced_rag/generation.py` sau đó prompt một LLM để sinh ra đoạn trả lời ngắn nhất đúng (bắt buộc một bước "Reasoning:" trung gian tường minh cho các câu hỏi dạng so sánh trước dòng "Answer:" cuối cùng).
- **Đánh giá (Evaluation)**: `advanced_rag/qa_metrics.py` triển khai Exact Match và F1 theo token-overlap theo đúng quy tắc chuẩn hóa SQuAD/HotpotQA; `evals/eval_hotpotqa.py` còn đo thêm *candidate coverage* ở cả giai đoạn trước và sau rerank, trên cả tier "dễ" và tier "khó" (câu hỏi bridge được mining riêng), để một lỗi truy xuất có thể được quy đúng về giai đoạn pipeline gây ra nó.

### 4. Triển khai kỹ thuật
**Các giai đoạn triển khai**
1. Chuẩn bị dữ liệu — xây một subset validation của HotpotQA (`corpus.jsonl`) và xác nhận định dạng tài liệu/câu trả lời.
2. Truy xuất baseline — triển khai chunking parent/child, lập chỉ mục BM25, và dense embedding bằng BAAI/bge-m3 trên một ChromaDB store cục bộ; xác thực chất lượng truy xuất một lượt.
3. Truy xuất hybrid và rerank — hợp nhất BM25 và vector search qua Reciprocal Rank Fusion, tinh chỉnh trọng số theo từng retriever, và thêm cross-encoder reranking.
4. Pipeline đa bước — thêm phân rã truy vấn bằng LLM và lập kế hoạch hop thích nghi, thay thế heuristic bridge-entity dựa trên regex ban đầu sau khi phát hiện các điểm mù cấu trúc của nó.
5. Di chuyển lên cloud — chuyển vector store sang Amazon S3 Vectors, đóng gói artifact offline kèm manifest có phiên bản, và deploy dịch vụ online lên Amazon EC2 phía sau Amazon API Gateway, với front end trên AWS Amplify.
6. Đánh giá và lặp lại — chạy `eval_hotpotqa.py` / `eval_full.py` để đo Exact Match/F1 và chẩn đoán candidate-coverage, và lặp lại trên kích thước chunk, top-k, và giới hạn lập kế hoạch hop dựa trên kết quả đo được (xem `docs/CHANGES_LOG.md`).
7. Tối ưu độ trễ — thêm một endpoint `/warmup` và một cờ cấu hình `RAG_FAST_MODE` để giữ độ trễ request trong giới hạn timeout của API Gateway trên phần cứng chỉ có CPU.

**Yêu cầu kỹ thuật**
- Bộ dữ liệu: HotpotQA (bộ dữ liệu hỏi-đáp suy luận đa bước), một validation slice 100 dòng cho bản triển khai demo.
- Framework/thư viện: LangChain (`langchain-core`, `langchain-classic`, `langchain-chroma`, `langchain-huggingface`), `sentence-transformers` (embedding BAAI/bge-m3, cross-encoder reranker `ms-marco-MiniLM-L-6-v2`), `bm25s`, ChromaDB, FastAPI, và Groq Python SDK.
- Dịch vụ/công cụ AWS cần thiết để chạy và đánh giá pipeline: Amazon EC2 (compute backend), Amazon S3 (lưu trữ artifact), Amazon S3 Vectors (dense retrieval), Amazon API Gateway (API HTTPS công khai), AWS Amplify (hosting frontend), AWS Systems Manager Parameter Store và Session Manager (cấu hình runtime và truy cập admin), AWS Secrets Manager (Groq API key), và AWS IAM (quyền instance role thay cho credential hard-code).

### 5. Lộ trình & Mốc triển khai
**Lộ trình dự án**
- Thời gian thực tập: 10/6/2026 – 30/7/2026
- Tuần 1–2: Chuẩn bị dữ liệu và truy xuất baseline một lượt (BM25 + dense embedding trên ChromaDB store cục bộ).
- Tuần 3–4: Truy xuất hybrid qua Reciprocal Rank Fusion, cross-encoder reranking, và đánh giá chất lượng truy xuất ban đầu.
- Tuần 5–6: Phân rã truy vấn bằng LLM và lập kế hoạch hop thích nghi; thay thế heuristic bridge-entity dựa trên regex ban đầu.
- Tuần 7: Di chuyển sang Amazon S3 Vectors và đóng gói artifact offline có phiên bản (manifest + checksum).
- Tuần 8: Deploy lên Amazon EC2 phía sau Amazon API Gateway, deploy frontend trên AWS Amplify, cấu hình tập trung qua SSM/Secrets Manager.
- Tuần 9: Tối ưu độ trễ (`/warmup`, `RAG_FAST_MODE`) và chạy đầy đủ đánh giá EM/F1 + candidate-coverage.
- Tuần 10: Báo cáo cuối kỳ, tài liệu hóa (`docs/STEP_*.md`, proposal này), và thuyết trình workshop.

### 6. Ước tính ngân sách
Bản triển khai demo được thiết kế để nằm trong giới hạn AWS Free Tier bất cứ khi nào có thể, giả định một tài khoản AWS mới/đủ điều kiện, một instance EC2 nhỏ chạy liên tục, và lưu lượng ở mức demo (khoảng vài trăm request `/query` mỗi tháng, thấp hơn nhiều so với bất kỳ giới hạn request Free Tier nào). Các con số dưới đây là **ước tính phục vụ mục đích lập kế hoạch**, không phải số tiền thực tế bị tính phí.

| Dịch vụ AWS | Hạn mức Free Tier (điển hình) | Mức sử dụng demo giả định | Chi phí ước tính hằng tháng |
| --- | --- | --- | --- |
| Amazon EC2 (t2/t3.micro) | 750 giờ instance/tháng trong 12 tháng | 1 instance, ~730 giờ/tháng | $0.00 |
| Amazon EC2 Elastic IP | Miễn phí khi gắn với instance đang chạy | 1 địa chỉ, luôn được gắn | $0.00 |
| Amazon S3 (Standard) | 5 GB lưu trữ, 20.000 GET / 2.000 PUT mỗi tháng | Artifact offline ≪ 5 GB, lượng request thấp | $0.00 |
| Amazon S3 Vectors | Chưa có free tier riêng tại thời điểm viết | ~100 tài liệu, lượng truy vấn thấp | ~$0.50 |
| Amazon API Gateway (HTTP API) | 1.000.000 request/tháng trong 12 tháng | Vài trăm request/tháng | $0.00 |
| AWS Amplify Hosting | 1.000 phút build + 15 GB served/tháng | 1 bản build React nhỏ, lượng người xem thấp | $0.00 |
| AWS Systems Manager (Parameter Store standard, Session Manager) | Luôn miễn phí (tier standard) | ~18 parameter, thi thoảng có phiên admin | $0.00 |
| AWS IAM | Luôn miễn phí | 1 instance role, 2 inline policy | $0.00 |
| Amazon CloudWatch | 5 GB log ingestion/lưu trữ, 10 custom metrics, 1.000.000 request API/tháng trong 12 tháng | Log ứng dụng + metrics cơ bản từ 1 instance EC2 nhỏ, lưu lượng thấp | $0.00 |
| AWS Secrets Manager | Dùng thử miễn phí 30 ngày mỗi secret, sau đó ~$0.40/secret/tháng | 1 secret (Groq API key), sau khi hết dùng thử | ~$0.40 |
| Groq API (ngoài AWS, pass-through) | Tier miễn phí/chi phí thấp cho lượng request thấp | Lượng truy vấn ở mức demo | ~$0.00–$1.00 |
| **Tổng ước tính** | | | **≈ $1–2 / tháng** |

Ước tính này giả định đồng hồ Free Tier chưa bị các workload khác trên cùng tài khoản dùng hết, và lưu lượng vẫn ở quy mô demo; một bản triển khai quy mô production (lưu lượng truy vấn cao hơn, corpus lớn hơn, inference dùng GPU) sẽ cần một mô hình chi phí riêng, lưu lượng cao hơn.

### 7. Đánh giá rủi ro
#### Ma trận rủi ro
| Rủi ro | Khả năng xảy ra | Mức độ ảnh hưởng |
| --- | --- | --- |
| Lỗi truy xuất bridge-entity (tài liệu đúng không bao giờ lọt vào candidate pool) | Trung bình | Cao — giới hạn trực tiếp EM/F1 có thể đạt được |
| Câu trả lời cuối bị hallucinate hoặc không có căn cứ | Trung bình | Cao — làm suy yếu giá trị cốt lõi của RAG |
| Độ trễ request tiệm cận/vượt timeout ~30s của API Gateway trên EC2 chỉ có CPU | Trung bình | Trung bình — request thất bại trong demo |
| Tập đánh giá nhỏ (validation slice 100 tài liệu) đánh giá quá cao độ chính xác thực tế | Cao | Trung bình — kết quả có thể không generalize sang corpus lớn hơn |
| Giới hạn rate limit hoặc lỗi tạm thời của Groq API trong lúc đánh giá hoặc demo | Trung bình | Thấp–Trung bình — được xử lý bằng retry/fallback, nhưng vẫn có thể làm giảm chất lượng một vài câu trả lời đơn lẻ |
| Không có authentication trên API demo công khai | Cao (chủ đích, cho một demo) | Thấp với một demo workshop ngắn hạn, cao hơn nếu để chạy lâu dài |
| Giới hạn tài nguyên free-tier hạn chế khả năng của pipeline: (a) cross-encoder reranker phải giữ ở trạng thái tắt trong production vì quá chậm trên một instance EC2 Free-Tier chỉ có CPU; (b) giới hạn token/phút theo từng model của tier miễn phí Groq làm throttle hoặc chặn các lệnh gọi decomposition/hop-planning/generation dưới bất kỳ tải đồng thời thực tế nào; (c) bước truy xuất hybrid BM25+vector đôi khi có thể lỗi hoặc timeout dưới giới hạn CPU/memory của một instance Free-Tier đang chạy đồng thời BM25 search và embedding inference | Cao | Cao — giảm trực tiếp chất lượng câu trả lời (không có rerank) và có thể gây thất bại request `/query` |

#### Chiến lược giảm thiểu
- Lỗi bridge-entity được giảm thiểu bằng truy xuất hybrid BM25+vector với trọng số theo từng hop, một cap candidate riêng cho mỗi hop (để một hop nhiễu không lấn át một hop đúng ở sau), và một LLM hop-planner thích nghi phát hành một truy vấn follow-up nhắm mục tiêu thay vì dựa vào một lượt truy xuất duy nhất.
- Hallucination được giảm thiểu bằng cách ràng buộc chặt việc sinh câu trả lời vào ngữ cảnh đã rerank, đã lọc (ngưỡng `CONTEXT_MIN_RERANK_SCORE`) và bằng cách giới hạn prompt sinh câu trả lời ở dạng ngắn, chỉ dựa vào ngữ cảnh, với một fallback "unknown" tường minh khi bằng chứng không đủ.
- Rủi ro độ trễ được giảm thiểu bằng một endpoint `/warmup` được gọi khi dịch vụ khởi động lại và một cờ cấu hình `RAG_FAST_MODE` bỏ qua decomposition và thu nhỏ giới hạn top-k/hop cho lưu lượng demo.
- Rủi ro generalize kém trên corpus nhỏ được giảm thiểu bằng một tập đánh giá hai tier tường minh (một tier "dễ" và một tier câu hỏi bridge khó hơn được mining, trải rộng trên corpus) tránh đánh giá quá cao chất lượng truy xuất chỉ từ một mẫu dễ.
- Rủi ro rate-limit của Groq được giảm thiểu bằng throttle lệnh gọi theo từng model và retry tự động có backoff, tôn trọng header `Retry-After` của API, với một fallback an toàn (trả về câu hỏi gốc / dừng hop) nếu hết số lần retry.
- Khoảng trống về authentication được chấp nhận trong suốt thời gian demo workshop và được đánh dấu tường minh là một giới hạn đã biết, với authentication dựa trên API-key hoặc IAM-authorizer được xác định là biện pháp giảm thiểu trước khi có bất kỳ bản triển khai dài hạn hoặc công khai nào.
- Giới hạn tài nguyên free-tier được giảm thiểu một cách thực dụng thay vì loại bỏ hoàn toàn, vì nâng cấp compute/API tier nằm ngoài phạm vi của một demo chi phí bằng 0: reranker được giữ tắt mặc định trong production (`RAG_USE_RERANKER=false`) và chỉ dựa vào thứ tự truy xuất hybrid, đánh đổi một phần chất lượng câu trả lời để lấy một workload mà CPU Free-Tier có thể chịu được; rate limit theo từng model của Groq được tôn trọng chủ động qua một khoảng thời gian tối thiểu giữa các lệnh gọi tới cùng một model cộng với retry tự động tôn trọng header `Retry-After` (`groq_utils.py`), để một đợt tăng lưu lượng demo suy giảm nhẹ nhàng thay vì thất bại hoàn toàn; và các lỗi/timeout truy xuất hybrid không thường xuyên được xử lý bằng `RAG_FAST_MODE` (top-k nhỏ hơn, ít hop hơn, bỏ qua decomposition) để giữ dấu chân tài nguyên của bước truy xuất trong khả năng phục vụ tin cậy của instance Free-Tier, với rủi ro còn sót lại đã biết là một đợt tăng đột biến request đồng thời vẫn có thể gây thất bại cho lệnh gọi `/query`.

#### Kế hoạch dự phòng
Nếu độ chính xác truy xuất trên tier đánh giá khó không đạt mục tiêu sau khi đã tinh chỉnh trọng số hybrid và giới hạn lập kế hoạch hop, phương án dự phòng là thu hẹp phạm vi về tier dễ và một tập câu hỏi nhỏ hơn, được chọn lọc cho buổi trình diễn workshop, đồng thời ghi lại khoảng cách đó như việc cần làm trong tương lai. Nếu độ trễ EC2/API Gateway không thể đưa về dưới timeout ngay cả với `RAG_FAST_MODE`, phương án dự phòng là demo pipeline qua CLI cục bộ (`tools/query.py`) và một bản ghi hình walkthrough thay vì endpoint công khai trực tiếp. Nếu tier miễn phí/chi phí thấp của Groq bị dùng hết giữa dự án, phương án dự phòng là tạm thời giảm nhu cầu `GROQ_MIN_CALL_INTERVAL` bằng cách chạy các batch đánh giá ít hơn, nhỏ hơn, hoặc thay thế bằng một LLM hosted khác đứng sau cùng interface retry/throttle `groq_utils.py`.

### 8. Kết quả kỳ vọng
#### Cải tiến kỹ thuật
Kết quả kỳ vọng là một cải thiện có thể đo lường được về Exact Match/F1 đa bước so với một baseline truy xuất một lượt (một vòng truy xuất, không decomposition, không rerank), có thể quy trực tiếp về stack truy xuất hybrid + lập kế hoạch hop thích nghi + rerank. Ngoài chỉ số chính, đánh giá candidate-coverage hai giai đoạn được kỳ vọng sẽ cho thấy độ phủ tài liệu (document-coverage) ở giai đoạn trước rerank cao hơn đáng kể so với một baseline truy vấn đơn giản, xác nhận rằng mức tăng độ chính xác đến từ việc tìm đúng bằng chứng chứ không chỉ từ việc xếp hạng tốt hơn trên một tập candidate vốn đã bị giới hạn.

#### Giá trị dài hạn
Vì việc tách offline/online và pipeline retrieval-decomposition-hop-planning-rerank-generation không phụ thuộc vào corpus cụ thể (mọi chi tiết riêng của corpus nằm trong một artifact bundle có phiên bản cộng với một tập nhỏ tham số cấu hình), giá trị dài hạn thực sự của dự án này là như một **sản phẩm RAG có thể tái sử dụng**, không phải một demo HotpotQA dùng một lần. Cùng một pipeline có thể được trỏ lại — bằng cách build một artifact bundle offline mới và đổi `index_id` — sang các bộ tài liệu có giá trị cao hơn, đòi hỏi suy luận cao hơn như tài liệu kỹ thuật/pháp lý/nghiên cứu nội bộ, đặc tả kỹ thuật, hoặc các knowledge base khác nơi câu hỏi thường xuyên đòi hỏi kết hợp fact từ nhiều tài liệu và nơi việc có căn cứ và truy vết được câu trả lời (nguồn trả về, điểm rerank, và mức sử dụng token) mang tính quyết định đối với business. Cơ chế lập kế hoạch hop thích nghi nói riêng được kỳ vọng sẽ generalize tốt sang bất kỳ domain nào có đặc điểm quan hệ "cầu nối" giữa các tài liệu, khiến kiến trúc này trở thành một nền tảng tiềm năng cho công cụ document-QA nội bộ trong tương lai thay vì một artifact workshop dùng rồi bỏ.

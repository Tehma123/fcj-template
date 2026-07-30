---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---
{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

Tại phần này, bạn cần tóm tắt các nội dung trong workshop mà bạn **dự tính** sẽ làm.

# Retrieval-Augmented Generation (RAG) Cho Suy Luận Đa Bước Trên HotpotQA
## [Thêm phụ đề ngắn mô tả cách tiếp cận hoặc hình thức triển khai cụ thể của bạn]

### 1. Tóm tắt điều hành
Dự án này thiết kế và triển khai một hệ thống Retrieval-Augmented Generation (RAG) để trả lời các câu hỏi suy luận đa bước (multi-hop) từ bộ dữ liệu HotpotQA — những câu hỏi mà câu trả lời đòi hỏi kết hợp thông tin từ nhiều tài liệu thay vì chỉ một đoạn văn bản. [Thêm phạm vi dự án: quy mô mục tiêu, môi trường triển khai, đối tượng sử dụng, v.v.]

### 2. Tuyên bố vấn đề
#### Vấn đề hiện tại
Các mô hình ngôn ngữ lớn (LLM) độc lập và các hệ thống hỏi-đáp một bước thường gặp khó khăn với câu hỏi đa bước, vì câu trả lời không thể tìm thấy trong một tài liệu duy nhất mà cần truy xuất và suy luận qua nhiều đoạn văn bản liên quan. Nếu không có bước truy xuất dựa trên nguồn dữ liệu gốc, câu trả lời có thể thiếu chính xác, không có căn cứ, hoặc bỏ sót các bước suy luận trung gian. [Thêm bối cảnh cụ thể vì sao vấn đề này quan trọng với use case của bạn]

#### Giải pháp
Hệ thống truy xuất các đoạn văn bản liên quan từ kho tài liệu HotpotQA, sau đó dùng mô hình ngôn ngữ để sinh câu trả lời cuối cùng dựa trên các bằng chứng đã truy xuất, kết hợp nhiều bước truy xuất/suy luận liên tiếp để giải quyết câu hỏi đa bước. [Thêm stack truy xuất và sinh câu trả lời cụ thể đã dùng — ví dụ: embedding model, vector store, LLM, framework orchestration]

#### Lợi ích và hoàn vốn đầu tư (ROI)
[Thêm lợi ích kỳ vọng, ví dụ: cải thiện độ chính xác trên câu hỏi đa bước so với baseline không dùng RAG, một pipeline truy xuất có thể tái sử dụng cho các bộ dữ liệu hỏi-đáp khác]  
[Thêm chi tiết chi phí nếu có]

### 3. Kiến trúc giải pháp
[Thêm sơ đồ kiến trúc và mô tả luồng dữ liệu từ bộ dữ liệu HotpotQA qua bước truy xuất, suy luận đa bước, đến sinh câu trả lời]

#### Dịch vụ AWS sử dụng
- [Thêm dịch vụ lưu trữ đã dùng, ví dụ cho kho dữ liệu HotpotQA và embeddings]
- [Thêm dịch vụ truy xuất/vector search đã dùng]
- [Thêm dịch vụ LLM/embedding đã dùng]
- [Thêm dịch vụ compute, orchestration hoặc API khác nếu có]

#### Thiết kế thành phần
- **Tiếp nhận dữ liệu**: [Mô tả cách bộ dữ liệu HotpotQA được nạp và tiền xử lý]
- **Truy xuất (Retrieval)**: [Mô tả cách các đoạn văn bản liên quan được truy xuất cho từng câu hỏi]
- **Suy luận đa bước (Multi-Hop Reasoning)**: [Mô tả cách hệ thống nối tiếp nhiều bước truy xuất/suy luận với nhau]
- **Sinh câu trả lời**: [Mô tả cách câu trả lời cuối cùng được sinh ra từ bằng chứng đã truy xuất]
- **Đánh giá**: [Mô tả cách câu trả lời được chấm điểm so với đáp án chuẩn của HotpotQA, ví dụ EM/F1]

### 4. Triển khai kỹ thuật
**Các giai đoạn triển khai**
[Thêm các giai đoạn đã thực hiện, ví dụ: khảo sát dữ liệu → truy xuất baseline → pipeline đa bước → đánh giá → cải tiến]

**Yêu cầu kỹ thuật**
- Bộ dữ liệu: HotpotQA (bộ dữ liệu hỏi-đáp suy luận đa bước)
- [Thêm framework/thư viện đã dùng, ví dụ: LangChain, LlamaIndex, hoặc code truy xuất tự viết]
- [Thêm các dịch vụ/công cụ AWS cần thiết để chạy và đánh giá pipeline]

### 5. Lộ trình & Mốc triển khai
**Lộ trình dự án**
- Thời gian thực tập: 10/6/2026 – 30/7/2026
- [Thêm các mốc theo tuần/tháng, ví dụ: giai đoạn nghiên cứu, xây dựng pipeline, đánh giá, báo cáo cuối kỳ]

### 6. Ước tính ngân sách
[Thêm chi tiết ngân sách nếu có, ví dụ: chi phí compute/lưu trữ AWS để chạy và đánh giá pipeline RAG]

### 7. Đánh giá rủi ro
#### Ma trận rủi ro
[Thêm các rủi ro cụ thể của dự án, ví dụ: độ chính xác truy xuất, hiện tượng hallucination, độ trễ, kích thước bộ dữ liệu đánh giá]

#### Chiến lược giảm thiểu
[Thêm cách giảm thiểu từng rủi ro ở trên]

#### Kế hoạch dự phòng
[Thêm phương án dự phòng nếu cách tiếp cận không đạt độ chính xác hoặc tiến độ mục tiêu]

### 8. Kết quả kỳ vọng
#### Cải tiến kỹ thuật
[Thêm mức cải thiện độ chính xác kỳ vọng trên câu hỏi đa bước so với baseline]

#### Giá trị dài hạn
[Thêm cách pipeline RAG này có thể được tái sử dụng cho các bộ dữ liệu khác hoặc dự án tương lai]

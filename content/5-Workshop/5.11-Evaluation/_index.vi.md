---
title: "Đánh giá hệ thống"
date: 2026-07-31
weight: 11
chapter: false
pre: " <b> 5.11. </b> "
---

Sau khi đã xác thực rằng bản triển khai hoạt động đúng, hệ thống được đánh giá trên ba khía cạnh: chất lượng retrieval, chất lượng câu trả lời, và hiệu năng lúc chạy. Toàn bộ kết quả cuối cùng đều sử dụng artifact **HotpotQA Distractor v002** đã được hiệu chỉnh.

Bộ benchmark retrieval gồm **500 câu hỏi validation**, **4.937 parent document** và **8.279 child vector BGE-M3**. Các chỉ số retrieval được tính ở mức tiêu đề tài liệu hỗ trợ (supporting-document-title), còn chất lượng câu trả lời cuối cùng được đo bằng Exact Match (EM) và F1 ở mức token.

## Chất lượng Retrieval

Retrieval pipeline được đánh giá ở hai giai đoạn. **Candidate pool** đo xem bằng chứng cần thiết có được tìm thấy hay không, trong khi **Selected Top-10** đo chất lượng xếp hạng của bằng chứng cuối cùng sau bước chọn lọc.

| Chỉ số | Candidate Pool | Selected Top-10 |
| --- | ---: | ---: |
| Recall trung bình theo supporting title | **0.9920** | **0.9740** |
| Tìm đủ toàn bộ supporting title | **0.9840** | **0.9480** |
| Recall@5 | 0.5420 | **0.9310** |
| Recall@10 | 0.6270 | **0.9740** |
| Precision@10 | 0.1254 | **0.1948** |
| MRR | 0.7807 | **0.9446** |
| nDCG@10 | 0.5816 | **0.9162** |

Giai đoạn candidate thu hồi được gần như toàn bộ tài liệu hỗ trợ cần thiết, với độ phủ supporting title đầy đủ ở **492 trên 500 câu hỏi**. Sau khi thu hẹp candidate pool xuống mười tài liệu cuối cùng, độ phủ đầy đủ vẫn giữ ở mức cao là **474 trên 500 câu hỏi**.

Cải thiện rõ rệt nhất nằm ở chất lượng xếp hạng. Recall@5 tăng từ **0.5420 lên 0.9310**, trong khi MRR tăng từ **0.7807 lên 0.9446** và nDCG@10 từ **0.5816 lên 0.9162**. Điều này cho thấy giai đoạn chọn lọc cuối cùng đã đưa bằng chứng liên quan lên gần đầu ngữ cảnh được cấp cho pipeline sinh câu trả lời hơn rất nhiều.

## Chất lượng câu trả lời đầu-cuối

Một tập cố định gồm **20 câu hỏi** được dùng để đánh giá toàn bộ đường đi từ retrieval tới sinh câu trả lời. Cả 20 câu hỏi đều chạy hoàn tất thành công.

| Chỉ số | Kết quả |
| --- | ---: |
| Answer EM | **0.7500** |
| Answer F1 | **0.7750** |
| Số câu trả lời đúng | **15 / 20** |
| Recall của bằng chứng được chọn | **0.9500** |
| Tìm đủ toàn bộ supporting title | **0.9000** |

Bằng chứng được chọn chứa đủ toàn bộ tài liệu hỗ trợ cần thiết ở **18 trên 20 câu hỏi**. Khoảng chênh giữa recall của bằng chứng và độ chính xác của câu trả lời cuối cùng cũng cho thấy: tìm đúng tài liệu là điều kiện cần, nhưng kết quả cuối cùng vẫn phụ thuộc vào việc bằng chứng thu được được sử dụng hiệu quả đến đâu trong quá trình sinh câu trả lời.

## Hiệu năng lúc chạy

Trên toàn bộ benchmark retrieval 500 câu hỏi, pipeline cần trung bình **25,91 giây cho mỗi câu hỏi**.

| Giai đoạn | Độ trễ trung bình | Tỉ trọng |
| --- | ---: | ---: |
| Tách câu hỏi (query decomposition) | 4,32 s | 16,7% |
| Retrieval + lập kế hoạch thích ứng | 21,07 s | 81,3% |
| Rerank bằng cross-encoder | 0,53 s | 2,0% |
| Tổng retrieval pipeline | **25,91 s** | 100% |

Chi phí thời gian chủ yếu đến từ retrieval và lập kế hoạch thích ứng, chứ không phải từ rerank. Bộ reranker chỉ chiếm khoảng 2% tổng độ trễ retrieval nhưng lại mang lại cải thiện đáng kể về chất lượng xếp hạng.

Với benchmark đầu-cuối trên 20 câu hỏi, tổng thời gian phản hồi là:

| Thành phần | Độ trễ trung bình |
| --- | ---: |
| Retrieval pipeline | 26,86 s |
| Sinh câu trả lời cuối | 12,43 s |
| Đầu-cuối | **39,29 s** |

Những kết quả này cho thấy đánh đổi cốt lõi của CloudHop RAG: retrieval đa bước rộng hơn cùng với rerank giúp cải thiện chất lượng bằng chứng, nhưng đồng thời làm tăng thời gian phản hồi. Vì vậy, môi trường AWS đã triển khai sử dụng một cấu hình production nhẹ hơn để giảm khối lượng retrieval và giữ độ trễ phản hồi ở mức thực tế trên nền EC2.

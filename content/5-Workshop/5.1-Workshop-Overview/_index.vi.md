---
title: "Tổng quan Workshop"
date: 2026-07-31
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

## HotpotQA và bài toán hỏi đáp đa bước

Nhiều hệ thống hỏi đáp hoạt động theo cách truy xuất một số tài liệu có liên quan rồi sử dụng trực tiếp các tài liệu đó để sinh câu trả lời. Cách tiếp cận này phù hợp khi phần lớn bằng chứng cần thiết nằm trong cùng một nguồn, nhưng trở nên kém tin cậy hơn khi thông tin phải được tổng hợp từ nhiều tài liệu khác nhau.

**HotpotQA** là bộ dữ liệu hỏi đáp dựa trên Wikipedia, được xây dựng cho các bài toán cần suy luận qua nhiều bước. Khác với những câu hỏi có thể trả lời từ một đoạn văn duy nhất, nhiều câu hỏi trong HotpotQA yêu cầu kết hợp thông tin từ hai hoặc nhiều tài liệu hỗ trợ. Bộ dữ liệu cũng cung cấp các supporting facts đã được gán nhãn, nhờ đó có thể đánh giá không chỉ câu trả lời cuối cùng mà cả việc hệ thống có tìm đúng bằng chứng cần thiết hay không.

CloudHop RAG sử dụng thiết lập **Distractor** của HotpotQA. Trong thiết lập này, mỗi câu hỏi đi kèm các tài liệu hỗ trợ cần thiết cùng với nhiều tài liệu nhiễu không liên quan. Các câu hỏi gồm cả dạng **bridge**, trong đó một bằng chứng dẫn đến bằng chứng tiếp theo, và dạng **comparison**, trong đó thông tin về nhiều đối tượng phải được kết hợp trước khi đưa ra câu trả lời.

Đây là một bài toán retrieval phù hợp để đánh giá RAG. Việc tìm được một tài liệu có mức độ liên quan cao chưa chắc đã đủ; hệ thống còn phải thu thập các bằng chứng bổ sung và tiếp tục tìm kiếm dựa trên những thông tin đã thu được khi cần thiết. Vì vậy, HotpotQA cung cấp cho CloudHop RAG một môi trường có kiểm soát để đánh giá một mục tiêu cốt lõi của dự án: **liệu hệ thống retrieval có thể tìm đúng các bằng chứng nằm trên nhiều tài liệu trước khi chuyển context cho mô hình ngôn ngữ sinh câu trả lời cuối cùng hay không.**

## CloudHop RAG

CloudHop RAG kết hợp lexical retrieval, semantic retrieval và multi-hop retrieval để xử lý dạng câu hỏi này. **BM25** hoạt động hiệu quả khi câu hỏi chứa tên riêng, thuật ngữ hoặc cụm từ cũng xuất hiện trong tài liệu hỗ trợ, trong khi **BGE-M3** giúp tìm các bằng chứng liên quan về mặt ngữ nghĩa ngay cả khi cách diễn đạt khác nhau.

Thay vì xem retrieval là một lần tìm kiếm duy nhất, pipeline có thể sử dụng các bằng chứng đã tìm được để định hướng cho những bước truy xuất tiếp theo. Các tài liệu ứng viên sau đó được kết hợp và rút gọn thành một context tập trung hơn trước khi được chuyển cho mô hình ngôn ngữ để sinh câu trả lời cuối cùng.

## Từ câu hỏi đến câu trả lời

![Sơ đồ tổng quan kiến trúc AWS CloudHop RAG](/images/5-Workshop/5.1-Workshop-Overview/rag_diagram.png)

Khi người dùng gửi câu hỏi, giao diện web được triển khai trên **AWS Amplify** gửi request qua **Amazon API Gateway** đến FastAPI backend chạy trên **Amazon EC2**. Backend thực hiện BM25 retrieval bằng các artifact được lưu trong **Amazon S3**, đồng thời thực hiện dense retrieval trên **Amazon S3 Vectors** bằng embedding BGE-M3.

Các bằng chứng thu được tiếp tục được xử lý trong multi-hop RAG pipeline và tổng hợp thành context gửi đến Groq LLM API. Câu trả lời được sinh ra cùng các nguồn hỗ trợ sau đó được trả về frontend thông qua API Gateway.


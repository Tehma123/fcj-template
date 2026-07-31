---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

## Tổng quan dự án

Trong quá trình thực tập AWS First Cloud AI Journey, tôi và nhóm đã đề xuất **AWS CloudHop RAG**, một hệ thống Retrieval-Augmented Generation (RAG) được thiết kế cho các câu hỏi cần thông tin từ nhiều hơn một tài liệu. Dự án tập trung vào bài toán hỏi-đáp đa bước (multi-hop question answering), trong đó việc tìm được một đoạn văn bản liên quan thường là chưa đủ để đưa ra câu trả lời chính xác.

Chúng tôi chọn **HotpotQA** làm bộ dữ liệu chính vì các câu hỏi trong đó được thiết kế riêng cho suy luận đa bước và có kèm theo bằng chứng hỗ trợ đã được gán nhãn. Điều này tạo ra một môi trường có kiểm soát để phát triển pipeline truy xuất và đánh giá xem hệ thống có thể tìm đúng các tài liệu cần thiết để trả lời từng câu hỏi hay không.

Dự án được lên kế hoạch như một ứng dụng end-to-end hoàn chỉnh, chứ không chỉ dừng lại ở một thử nghiệm truy xuất. Bên cạnh việc phát triển và đánh giá pipeline RAG, nhóm sẽ triển khai các thành phần chính của ứng dụng trên AWS và cung cấp một giao diện web đơn giản để gửi câu hỏi và xem câu trả lời được sinh ra cùng các nguồn hỗ trợ đi kèm.

## Vấn đề và động lực

Retrieval-Augmented Generation cải thiện khả năng hỏi-đáp bằng cách truy xuất thông tin liên quan từ một nguồn tri thức bên ngoài trước khi yêu cầu mô hình ngôn ngữ sinh câu trả lời. Cách này giúp mô hình dựa vào bằng chứng đã truy xuất được thay vì chỉ phụ thuộc hoàn toàn vào những gì nó đã biết sẵn.

Tuy nhiên, nhiều hệ thống RAG chỉ thực hiện truy xuất một lần duy nhất. Cách này hoạt động tốt khi thông tin cần để trả lời câu hỏi nằm gọn trong một tài liệu liên quan, nhưng trở nên kém tin cậy hơn khi cần kết nối nhiều bằng chứng khác nhau.

Một câu hỏi đa bước có thể yêu cầu hệ thống trước tiên tìm một tài liệu, xác định một người, địa điểm, tổ chức hoặc mối quan hệ quan trọng từ bằng chứng đó, rồi dùng thông tin này để tìm ra một tài liệu khác. Nếu thiếu một trong hai phần bằng chứng, câu trả lời có thể trở nên không đầy đủ hoặc không chính xác.

HotpotQA cung cấp một benchmark hữu ích cho bài toán này vì nó chứa cả **câu hỏi bridge** (bridge questions), trong đó một bằng chứng dẫn tới bằng chứng khác, lẫn **câu hỏi so sánh** (comparison questions), trong đó thông tin từ nhiều thực thể cần được kết hợp lại.

Vì vậy, với dự án này, chúng tôi muốn tìm hiểu liệu việc kết hợp truy xuất từ vựng (lexical retrieval), truy xuất ngữ nghĩa (semantic retrieval), và các bước truy xuất bổ sung có thể cung cấp bằng chứng đầy đủ hơn cho các câu hỏi đa bước hay không, trong khi vẫn khả thi để triển khai như một ứng dụng AWS trong thực tế.

## Mục tiêu và phạm vi

### Mục tiêu dự án

Các mục tiêu chính của AWS CloudHop RAG bao gồm:

1. Xây dựng một pipeline RAG có khả năng trả lời các câu hỏi đa bước bằng bằng chứng truy xuất từ nhiều tài liệu.
2. Tìm hiểu cả phương pháp truy xuất từ vựng lẫn truy xuất ngữ nghĩa, và kết hợp điểm mạnh của chúng thông qua truy xuất hybrid.
3. Hỗ trợ thêm các bước truy xuất bổ sung khi bằng chứng từ lượt tìm kiếm ban đầu chưa đủ.
4. Đánh giá chất lượng truy xuất tách biệt với chất lượng câu trả lời cuối cùng, để có thể xác định rõ ràng các lỗi truy xuất.
5. Triển khai ứng dụng hoàn chỉnh bằng các dịch vụ AWS và cung cấp một giao diện đơn giản để tương tác với hệ thống.
6. Xây dựng dự án theo hướng có thể tái tạo được, để các artifact truy xuất, kết quả đánh giá và các bước triển khai đều có thể được tạo lại và ghi lại đầy đủ.

### Phạm vi dự án

Dự án sẽ tập trung vào chế độ **HotpotQA Distractor** như môi trường phát triển và đánh giá chính.

Phạm vi dự kiến bao gồm:

- chuẩn bị dữ liệu HotpotQA;
- truy xuất từ vựng và truy xuất dense;
- truy xuất hybrid;
- truy xuất bằng chứng đa bước;
- xếp hạng bằng chứng và xây dựng ngữ cảnh;
- sinh câu trả lời bằng LLM;
- đánh giá truy xuất và câu trả lời;
- lưu trữ và tìm kiếm vector trên AWS;
- triển khai backend trên cloud;
- tích hợp API và frontend;
- kiểm thử chức năng và tài liệu kỹ thuật.

Dự án được xác định ở quy mô triển khai và đánh giá cho một kỳ thực tập, chứ không phải một dịch vụ production phục vụ số lượng lớn người dùng. Lưu lượng truy cập quy mô lớn, xác thực cấp doanh nghiệp, và triển khai trên tập tài liệu cực lớn nằm ngoài phạm vi chính của dự án.

## Giải pháp và kiến trúc đề xuất

### Phương pháp RAG đề xuất

Pipeline được đề xuất kết hợp nhiều chiến lược truy xuất khác nhau thay vì chỉ dựa vào một phương pháp tìm kiếm duy nhất.

Luồng xử lý tổng thể như sau:

**Câu hỏi → Phân tích truy vấn → Truy xuất từ vựng và ngữ nghĩa → Truy xuất đa bước → Xếp hạng bằng chứng → Xây dựng ngữ cảnh → Sinh câu trả lời bằng LLM → Câu trả lời và nguồn hỗ trợ**

Đối với truy xuất từ vựng, dự án sẽ sử dụng **BM25**, phương pháp hiệu quả khi các tên riêng, thực thể hoặc cụm từ quan trọng trong câu hỏi cũng xuất hiện trực tiếp trong tài liệu nguồn.

Đối với truy xuất ngữ nghĩa, dự án sẽ sử dụng **BGE-M3 embeddings** để biểu diễn văn bản dưới dạng vector dense. Điều này cho phép hệ thống tìm được bằng chứng liên quan ngay cả khi cách diễn đạt của câu hỏi và tài liệu nguồn khác nhau.

Kết quả từ truy xuất từ vựng và truy xuất ngữ nghĩa sẽ được gộp lại thành một tập candidate chung. Với các câu hỏi cần nhiều thông tin, hệ thống cũng sẽ thực hiện thêm các bước truy xuất bổ sung dựa trên bằng chứng đã tìm được trước đó trong quá trình xử lý.

Sau khi truy xuất, các bằng chứng mạnh nhất sẽ được xếp hạng và thu gọn thành một ngữ cảnh tập trung trước khi đưa vào mô hình ngôn ngữ. Phản hồi cuối cùng sẽ bao gồm cả câu trả lời được sinh ra lẫn các nguồn hỗ trợ đã dùng để xây dựng ngữ cảnh đó.

### Kiến trúc AWS đề xuất

Pipeline RAG sẽ được triển khai như một ứng dụng web, sử dụng nhiều dịch vụ AWS với trách nhiệm tách biệt nhau.

![Kiến trúc AWS CloudHop RAG được đề xuất](/images/2-Proposal/AWS-RAG.drawio.png)

Luồng xử lý ứng dụng dự kiến như sau:

**Người dùng → AWS Amplify → Amazon API Gateway → Amazon EC2 → Amazon S3 / Amazon S3 Vectors → Groq API → Câu trả lời**

| Thành phần | Vai trò dự kiến |
| --- | --- |
| **AWS Amplify** | Host frontend web dùng để gửi câu hỏi và hiển thị câu trả lời |
| **Amazon API Gateway** | Cung cấp API HTTPS giữa trình duyệt và backend |
| **Amazon EC2** | Chạy backend FastAPI và điều phối pipeline RAG |
| **Amazon S3** | Lưu trữ corpus đã xử lý, artifact BM25, mapping và manifest |
| **Amazon S3 Vectors** | Lưu trữ và tìm kiếm vector dense BGE-M3 |
| **AWS IAM** | Kiểm soát quyền truy cập giữa các tài nguyên AWS |
| **AWS Systems Manager** | Hỗ trợ quản trị và truy cập backend EC2 |
| **Groq API** | Cung cấp inference mô hình ngôn ngữ cho pipeline RAG |

Các artifact truy xuất sẽ được chuẩn bị trước khi phục vụ truy vấn người dùng. Điều này giữ cho các bước tiền xử lý tốn kém như chuẩn bị tài liệu, lập chỉ mục và sinh embedding nằm ngoài luồng xử lý request online.

Khi chạy, backend EC2 sẽ tải các artifact truy xuất từ vựng cần thiết từ Amazon S3 và truy vấn Amazon S3 Vectors để truy xuất ngữ nghĩa. Bằng chứng truy xuất được sau đó sẽ được pipeline RAG xử lý và gửi tới mô hình ngôn ngữ để sinh câu trả lời cuối cùng.

## Kế hoạch dự án

Dự án được lên kế hoạch như một nỗ lực của cả nhóm, đi từ nền tảng AWS và nghiên cứu RAG đến phát triển truy xuất, đánh giá, và triển khai ứng dụng hoàn chỉnh. Các thành phần khác nhau có thể được phát triển song song khi phù hợp, nhưng trình tự tổng thể được thiết kế sao cho pipeline truy xuất được xác thực trước khi tích hợp vào ứng dụng AWS cuối cùng.

### Các giai đoạn phát triển

**Giai đoạn 1 – Nền tảng AWS và xác định dự án**

Nhóm trước tiên xây dựng hiểu biết chung về các dịch vụ AWS cần thiết cho dự án, đồng thời xác định bài toán, mục tiêu, kiến trúc và phương pháp đánh giá của CloudHop RAG. Giai đoạn này cũng bao gồm việc tìm hiểu về RAG, text embedding, hỏi-đáp đa bước, và cấu trúc của HotpotQA.

**Giai đoạn 2 – Chuẩn bị dữ liệu và baseline truy xuất**

HotpotQA sẽ được kiểm tra và chuyển đổi sang định dạng nhất quán cho các thử nghiệm truy xuất. Các phương pháp truy xuất từ vựng và dense ban đầu sẽ được phát triển để thiết lập baseline và xác định những khó khăn truy xuất chính của câu hỏi đa bước.

**Giai đoạn 3 – Truy xuất đa bước nâng cao**

Pipeline truy xuất sẽ được mở rộng với BM25, embedding BGE-M3, truy xuất hybrid, biểu diễn tài liệu parent-child, phân rã truy vấn (query decomposition), truy xuất đa bước thích nghi, và rerank bằng chứng. Mục tiêu của giai đoạn này là cải thiện khả năng thu thập bằng chứng bổ sung cho nhau từ nhiều tài liệu của hệ thống.

**Giai đoạn 4 – Đánh giá và chuẩn bị artifact**

Nhóm sẽ đánh giá chất lượng truy xuất bằng bằng chứng hỗ trợ của HotpotQA và đo chất lượng câu trả lời bằng Exact Match và F1. Các artifact truy xuất như corpus đã xử lý, chỉ mục BM25, mapping tài liệu, embedding và manifest cũng sẽ được chuẩn bị ở định dạng có thể tái sử dụng cho việc triển khai.

**Giai đoạn 5 – Backend AWS và triển khai truy xuất**

Các artifact truy xuất đã được xác thực sẽ được chuyển lên AWS. Amazon S3 sẽ lưu trữ corpus và các artifact truy xuất, trong khi Amazon S3 Vectors cung cấp khả năng tìm kiếm vector dense. Backend RAG sẽ được triển khai trên Amazon EC2 và cấu hình với các quyền cần thiết để truy cập dịch vụ lưu trữ và tìm kiếm vector.

**Giai đoạn 6 – Tích hợp API và frontend**

Amazon API Gateway sẽ được dùng để expose backend qua một API HTTPS. Một frontend web sẽ được triển khai qua AWS Amplify và kết nối với API để người dùng có thể gửi câu hỏi và xem câu trả lời được sinh ra cùng các nguồn hỗ trợ.

**Giai đoạn 7 – Xác thực hệ thống và hoàn thiện**

Ứng dụng hoàn chỉnh sẽ được kiểm thử xuyên suốt từ frontend qua API, backend, các dịch vụ truy xuất, cho đến mô hình ngôn ngữ. Nhóm sẽ xác thực chức năng, hành vi truy xuất, chất lượng câu trả lời và thời gian phản hồi trước khi hoàn thiện workshop triển khai và tài liệu dự án.

### Tiến độ dự án theo tuần

| Tuần | Hoạt động dự kiến của nhóm |
| --- | --- |
| **Tuần 1**<br>08/06 – 12/06 | **Nền tảng AWS và định hướng dự án.** Ôn lại kiến thức nền tảng AWS, AWS Console và CLI, EC2, cùng các yêu cầu của kỳ thực tập. Thảo luận các hướng dự án AI khả thi và chuẩn bị môi trường phát triển. |
| **Tuần 2**<br>15/06 – 19/06 | **Lưu trữ, bảo mật và networking trên AWS.** Học và thực hành với Amazon S3, IAM, VPC, security group, và quyền dịch vụ. Xây dựng nền kiến thức AWS cần thiết cho kiến trúc ứng dụng sau này. |
| **Tuần 3**<br>22/06 – 26/06 | **Xác định dự án và nghiên cứu RAG.** Tìm hiểu embedding, semantic search, RAG, và hỏi-đáp đa bước. Chọn HotpotQA làm benchmark và xác định kiến trúc, mục tiêu, chiến lược đánh giá ban đầu của CloudHop RAG. |
| **Tuần 4**<br>29/06 – 03/07 | **Chuẩn bị dữ liệu và baseline truy xuất.** Chuẩn bị dữ liệu HotpotQA, gắn kết câu hỏi với bằng chứng hỗ trợ, và phát triển các phương pháp truy xuất từ vựng, dense ban đầu để thiết lập baseline. |
| **Tuần 5**<br>06/07 – 10/07 | **Phát triển truy xuất nâng cao.** Phát triển truy xuất BM25 và BGE-M3, tìm kiếm hybrid, biểu diễn tài liệu parent-child, phân rã truy vấn, truy xuất đa bước thích nghi, và rerank bằng chứng. |
| **Tuần 6**<br>13/07 – 17/07 | **Xây dựng pipeline và chuẩn bị đánh giá.** Tổ chức các module và cấu hình dự án có thể tái sử dụng, xác thực sự khớp của dữ liệu, xây dựng artifact truy xuất có phiên bản, và chuẩn bị quy trình benchmark, đánh giá. |
| **Tuần 7**<br>20/07 – 24/07 | **Đánh giá và chuẩn bị triển khai AWS.** Đánh giá chất lượng truy xuất và câu trả lời, phân tích độ trễ, hoàn thiện kiến trúc production, upload artifact truy xuất lên Amazon S3, chuẩn bị Amazon S3 Vectors, và cấu hình môi trường backend Amazon EC2. |
| **Tuần 8**<br>27/07 – 31/07 | **Tích hợp AWS hoàn chỉnh và hoàn thiện dự án.** Hoàn tất triển khai backend FastAPI trên EC2, kết nối Amazon S3 và S3 Vectors, cấu hình quyền truy cập IAM và Systems Manager, expose backend qua Amazon API Gateway, triển khai frontend với AWS Amplify, xác thực toàn bộ ứng dụng end-to-end, tổng hợp kết quả đánh giá, và hoàn thiện workshop cùng tài liệu kỹ thuật. |

## Ước tính ngân sách

AWS CloudHop RAG được lên kế hoạch như một workload thực tập và demo có quy mô nhỏ, với dung lượng lưu trữ và số lượng request tương đối thấp. Phần lớn các dịch vụ managed được ứng dụng sử dụng tính phí theo mức dùng, trong khi instance compute cho backend được dự kiến sẽ chiếm phần lớn nhất trong chi phí vận hành.

| Tài nguyên | Mức sử dụng dự kiến | Ghi chú về chi phí |
| --- | --- | --- |
| **Amazon EC2** | Một instance backend trong suốt quá trình phát triển và demo | Chi phí compute liên tục chính |
| **Amazon S3** | Lưu trữ corpus và artifact truy xuất | Chi phí lưu trữ thấp ở quy mô dự án |
| **Amazon S3 Vectors** | Lưu trữ và truy vấn vector dense | Phụ thuộc vào số vector lưu trữ và mức truy vấn |
| **Amazon API Gateway** | Số lượng request API thấp | Tính phí theo request |
| **AWS Amplify** | Host một frontend web nhỏ | Tính theo build, lưu trữ và băng thông |
| **AWS IAM** | Kiểm soát quyền tài nguyên AWS | Không tính phí trực tiếp |
| **AWS Systems Manager** | Quản trị EC2 | Gần như không tốn chi phí trực tiếp với mức dùng dự kiến |
| **Groq API** | Inference LLM | Phụ thuộc vào model và mức sử dụng token |

Dự án sẽ giữ quy mô triển khai nhỏ và tránh để các tài nguyên không cần thiết chạy liên tục. Tài nguyên compute có thể được dừng khi không cần dùng, trong khi các artifact truy xuất bền vững vẫn được lưu trữ riêng trong S3 và S3 Vectors.

Vì vậy, ngân sách sẽ được quản lý chủ yếu bằng cách kiểm soát thời gian chạy của EC2, hạn chế các request không cần thiết, và dọn dẹp tài nguyên sau khi workshop và quá trình đánh giá hoàn tất.

## Rủi ro và biện pháp giảm thiểu

| Rủi ro | Tác động có thể xảy ra | Biện pháp giảm thiểu dự kiến |
| --- | --- | --- |
| Không truy xuất được bằng chứng hỗ trợ liên quan | LLM nhận ngữ cảnh không đầy đủ và có thể đưa ra câu trả lời sai | Kết hợp truy xuất từ vựng và ngữ nghĩa, đồng thời đánh giá độ phủ bằng chứng hỗ trợ |
| Truy xuất đa bước làm tăng thời gian phản hồi | Truy vấn có thể mất quá nhiều thời gian để hoàn tất | Giới hạn độ sâu truy xuất và kích thước candidate khi cần thiết |
| Giới hạn rate limit hoặc lỗi tạm thời của LLM API | Quá trình đánh giá hoặc sinh câu trả lời có thể bị gián đoạn | Kiểm soát tốc độ gửi request, dùng cơ chế retry, và đánh giá có thể tiếp tục lại |
| Dataset hoặc artifact truy xuất không nhất quán | Kết quả đánh giá có thể không phản ánh đúng chất lượng truy xuất | Xác thực sự khớp của dataset và tính toàn vẹn của artifact trước khi benchmark |
| Cấu hình hoặc quyền AWS không chính xác | Các thành phần ứng dụng có thể không giao tiếp được với nhau | Sử dụng IAM role, quyền theo nguyên tắc least-privilege, và kiểm thử chức năng theo từng giai đoạn |
| Tài nguyên cloud tiêu tốn nhiều hơn dự kiến | Chi phí dự án có thể tăng | Giữ quy mô triển khai nhỏ, dừng compute không dùng đến, và dọn dẹp tài nguyên sau khi sử dụng |

## Kết quả kỳ vọng

Khi kết thúc dự án, nhóm kỳ vọng sẽ có một ứng dụng RAG đa bước hoạt động được, có khả năng truy xuất bằng chứng từ HotpotQA, kết hợp thông tin từ nhiều tài liệu khi cần thiết, và sinh câu trả lời dựa trên ngữ cảnh đã truy xuất.

Các kết quả đầu ra chính dự kiến bao gồm:

- một quy trình chuẩn bị dữ liệu HotpotQA có thể tái sử dụng;
- các thành phần truy xuất từ vựng và ngữ nghĩa;
- một pipeline truy xuất hybrid và đa bước;
- một quy trình đánh giá có thể tái tạo được;
- các artifact truy xuất có thể lưu trữ và tái sử dụng độc lập với runtime của ứng dụng;
- một môi trường backend và tìm kiếm vector được host trên AWS;
- một giao diện web để gửi câu hỏi và xem câu trả lời cùng nguồn hỗ trợ;
- đánh giá định lượng về chất lượng truy xuất và câu trả lời;
- một workshop triển khai AWS hoàn chỉnh cùng tài liệu kỹ thuật.

Ngoài bản thân ứng dụng cuối cùng, dự án còn nhằm mang lại cho nhóm kinh nghiệm thực tế trong việc kết nối thử nghiệm truy xuất và mô hình ngôn ngữ với hạ tầng cloud. HotpotQA cung cấp một benchmark có kiểm soát cho kỳ thực tập này, trong khi thiết kế RAG tổng thể sau này có thể được điều chỉnh để áp dụng cho các bộ tài liệu khác cần hỏi-đáp dựa trên bằng chứng.

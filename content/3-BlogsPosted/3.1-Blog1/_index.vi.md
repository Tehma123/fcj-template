---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Amazon S3 Annotations: Metadata Có Thể Cập Nhật Và Truy Vấn Cho Từng Object

Amazon S3 đã hỗ trợ nhiều loại metadata để mô tả và quản lý object, chẳng hạn kích thước, storage class, cùng object tags và user-defined metadata cho các nhu cầu quản lý khác nhau.

Tuy nhiên, trong các pipeline xử lý dữ liệu và AI, một object thường còn đi kèm nhiều thông tin được tạo ra sau quá trình xử lý. Hệ thống có thể tạo transcript, kết quả OCR và bản tóm tắt, đồng thời trích xuất các thông tin quan trọng như tên người, tổ chức, ngày tháng hoặc mã tài liệu. Mô hình cũng có thể sinh nhãn phân loại, embedding, confidence score, cờ phát hiện PII hoặc kết quả kiểm duyệt nội dung. Những metadata này thường vẫn phải được lưu trong cơ sở dữ liệu hoặc file riêng.

Cách tổ chức này tạo ra hai vấn đề: ứng dụng phải tự giữ metadata đồng bộ với object, đồng thời phải kết nối nhiều hệ thống để tìm được đúng dữ liệu cần sử dụng.

Được AWS ra mắt vào tháng 6 năm 2026, Amazon S3 Annotations bổ sung một cách mới để trực tiếp gắn các phần metadata có tên riêng với từng object. Mỗi annotation có thể được cập nhật độc lập và đưa vào S3 Metadata để truy vấn bằng công cụ phân tích hoặc AI agent.

## 1. S3 Annotations là gì?

Mỗi annotation là một phần metadata được định danh bằng tên riêng và gắn với một phiên bản cụ thể của S3 object. Nội dung có thể là văn bản thuần hoặc dữ liệu có cấu trúc như JSON, XML và YAML.

Ví dụ, một video có thể đi kèm các annotation sau:

```
videos/documentary-2026.mp4
├── mediainfo
├── transcript
├── ai_summary
├── moderation_result
└── licensing
```

Trong đó, `mediainfo` lưu codec và độ phân giải, `transcript` lưu nội dung hội thoại, `ai_summary` lưu bản tóm tắt do AI tạo, `moderation_result` lưu kết quả kiểm duyệt, còn `licensing` mô tả phạm vi và thời hạn bản quyền.

Mỗi annotation được quản lý độc lập. Pipeline tạo transcript có thể cập nhật `transcript` mà không phải chỉnh sửa video hoặc ảnh hưởng đến metadata do pipeline khác tạo ra.

Mỗi object version có thể có tối đa 1.000 annotation, với dung lượng tối đa 1 MiB cho mỗi annotation, tương đương tối đa 1 GiB metadata annotation. Nhờ đó, annotations có thể chứa những kết quả xử lý chi tiết và có cấu trúc hơn nhiều so với object tags hoặc user-defined metadata.

Object chứa dữ liệu gốc. Annotation cho biết dữ liệu đó là gì, đã được xử lý ra sao và đang ở trạng thái nào.

## 2. Vậy S3 Annotations khác gì với tags và metadata hiện có?

S3 Annotations không thay thế các cơ chế metadata hiện có. Mỗi cơ chế được thiết kế cho một mục đích khác nhau:

### System-defined metadata

Là các thuộc tính do S3 tự quản lý, chẳng hạn kích thước, storage class và thời điểm tạo object. Loại metadata này mô tả đặc điểm của object nhưng không dùng để lưu thông tin tùy chỉnh từ ứng dụng.

### User-defined metadata

Phù hợp với một lượng nhỏ thông tin được xác định khi upload object. Dung lượng bị giới hạn và muốn thay đổi metadata, ứng dụng thường phải ghi lại object.

### Object tags

Phù hợp với các tác vụ vận hành như kiểm soát truy cập, áp dụng lifecycle rule và phân bổ chi phí. Mỗi object version chỉ có thể chứa tối đa 10 cặp key–value ngắn.

### S3 Annotations

Phù hợp với metadata chi tiết và có cấu trúc, chẳng hạn transcript, kết quả OCR, nhãn phân loại, trạng thái xử lý hoặc thông tin cấp phép. Mỗi object version có thể chứa tối đa 1.000 annotation và từng annotation có thể được cập nhật độc lập.

Có thể ghi nhớ sự khác biệt theo cách đơn giản:

Tags chủ yếu phục vụ việc quản lý object. Annotations cung cấp ngữ cảnh để ứng dụng và AI hiểu nội dung cũng như trạng thái của object.

## 3. Từ metadata của từng object đến tìm kiếm trên toàn bucket

Gắn metadata vào từng object chỉ giải quyết một phần bài toán. Khi bucket chứa hàng triệu object, hệ thống còn phải tìm được đúng dữ liệu dựa trên nội dung và trạng thái của chúng.

Khi bật annotation table trong S3 Metadata, S3 tự động đưa annotations vào một bảng Apache Iceberg do AWS quản lý. Bảng này có thể được truy vấn bằng Amazon Athena và các công cụ tương thích với Iceberg.

```
S3 object → annotations → S3 Metadata → Athena / công cụ Iceberg → ứng dụng hoặc AI agent
```

Nhờ đó, một công ty truyền thông có thể tìm các video đã được kiểm duyệt, có phụ đề tiếng Việt và còn quyền phát hành mà không phải mở từng tệp. Tương tự, một hệ thống xử lý tài liệu có thể tìm các hợp đồng chứa PII nhưng chưa được phê duyệt.

AI agent cũng có thể sử dụng lớp metadata này để thu hẹp phạm vi tìm kiếm trước khi đọc dữ liệu gốc. Thay vì chỉ nhìn thấy một tên object khó hiểu như `archive/file_00873142.pdf`, agent có thể dựa vào annotation về loại tài liệu, ngôn ngữ, trạng thái xử lý hoặc mức độ nhạy cảm.

Một yêu cầu bằng ngôn ngữ tự nhiên như:

> Tìm các phim xếp hạng PG, có phụ đề tiếng Tây Ban Nha và được phát hành trong năm 2023.

có thể được chuyển thành truy vấn trên annotations, thay vì phải kết nối và đối chiếu nhiều hệ thống metadata riêng biệt.

## 4. Khi nào S3 Annotations phù hợp?

S3 Annotations đáng cân nhắc khi metadata có một hoặc nhiều đặc điểm sau:

* gắn trực tiếp với một object cụ thể;
* lớn hơn một vài cặp key–value ngắn;
* có cấu trúc như JSON, XML hoặc YAML;
* được nhiều pipeline cập nhật theo thời gian;
* cần được tìm kiếm hoặc phân tích trên nhiều object;
* cung cấp ngữ cảnh cho công cụ phân tích hoặc AI agent.

Một số tình huống điển hình gồm:

* **Xử lý tài liệu bằng AI:** lưu kết quả OCR, phân loại, trích xuất thực thể (entity extraction), phát hiện PII và trạng thái phê duyệt.
* **Quản lý nội dung số:** lưu transcript, phụ đề, kết quả kiểm duyệt, tóm tắt và thông tin bản quyền.
* **Theo dõi pipeline dữ liệu:** lưu phiên bản schema, trạng thái xử lý, kết quả kiểm tra chất lượng và nguồn gốc dữ liệu.

Tuy nhiên, annotations không thay thế mọi hệ thống metadata. Object tags vẫn phù hợp hơn cho access control, lifecycle rule và cost allocation. Cơ sở dữ liệu riêng vẫn cần thiết khi metadata có quan hệ phức tạp, cần giao dịch có độ nhất quán cao hoặc phải cập nhật với độ trễ rất thấp.

## 5. Những điều cần lưu ý

Trước khi triển khai, có bốn điểm quan trọng cần cân nhắc:

* Annotation gắn với từng object version. Các phiên bản khác nhau của cùng một object có annotations độc lập.
* Annotation table không cập nhật tức thời. S3 Metadata phù hợp với khám phá và phân tích dữ liệu hơn là các tác vụ giao dịch thời gian thực.
* Dung lượng lưu trữ annotation được tính theo mức giá S3 Standard. Điều này vẫn áp dụng khi object gốc nằm trong S3 Glacier hoặc storage class khác.
* Dung lượng tối đa không phải mục tiêu sử dụng. Không phải mọi dữ liệu liên quan đều nên được đưa vào annotation; một số nội dung vẫn phù hợp hơn khi được lưu thành object riêng hoặc trong cơ sở dữ liệu chuyên dụng.

Ngoài ra, cần kiểm tra kỹ hành vi của annotations khi copy object, sử dụng S3 Replication hoặc bật versioning, đặc biệt nếu hệ thống yêu cầu object và metadata luôn xuất hiện đồng thời.

## Kết luận

Amazon S3 Annotations cho phép gắn các phần metadata có tên riêng với từng object, cập nhật độc lập và đưa vào S3 Metadata để truy vấn trên quy mô lớn. Thay vì phải quản lý toàn bộ ngữ cảnh ở hệ thống bên ngoài, ứng dụng có thể gắn transcript, kết quả/trạng thái xử lý hoặc thông tin bản quyền với chính object mà chúng mô tả.

S3 Annotations không thay thế object tags hay cơ sở dữ liệu. Nó phù hợp với khoảng trống giữa hai mô hình đó: metadata cần chi tiết và dễ truy vấn hơn tags, nhưng vẫn cần được quản lý cùng dữ liệu trong S3.

*Nguồn: AWS – Amazon S3 User Guide, "Annotating your objects"*

...Hình ảnh...

...Link...

...Hướng dẫn...

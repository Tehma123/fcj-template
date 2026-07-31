---
title: "Các bài viết đã đăng"
date: 2026-07-31
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

Bên cạnh dự án CloudHop RAG, nhóm mình cũng thực hiện ba bài viết kỹ thuật để đăng trên cộng đồng **AWS Study Group VN**. Nhóm xem đây là cơ hội để tìm hiểu thêm những chủ đề AWS nằm ngoài phạm vi trực tiếp của dự án, vì vậy mỗi bài tập trung vào một mảng khá khác nhau trong hệ sinh thái AWS.

Thay vì chỉ tóm tắt lại tài liệu, nhóm cố gắng tìm hiểu đủ sâu để giải thích được cách một tính năng hoặc dịch vụ hoạt động, vì sao nó đáng chú ý và trong những tình huống nào nó có thể thực sự hữu ích. Ba bài viết lần lượt xoay quanh quản lý dữ liệu và metadata, tối ưu chi phí cloud và cơ sở dữ liệu phân tán.

### [Blog 1 - Amazon S3 Annotations: Metadata Có Thể Cập Nhật Và Truy Vấn Cho Từng Object](3.1-Blog1/)

Bài đầu tiên tìm hiểu về **Amazon S3 Annotations**, một tính năng cho phép gắn các phần metadata chi tiết và có thể cập nhật độc lập trực tiếp với từng S3 object.

Nhóm tập trung vào những loại metadata thường xuất hiện trong các data pipeline và AI pipeline như transcript, kết quả OCR, bản tóm tắt do AI tạo, nhãn phân loại hoặc trạng thái xử lý. Bài viết cũng so sánh annotations với object tags và các loại metadata hiện có của S3, đồng thời tìm hiểu cách annotations có thể được truy vấn thông qua S3 Metadata.

### [Blog 2 - Tối Ưu Chi Phí AWS: Đừng Chỉ Nhìn Vào Hóa Đơn](3.2-Blog2/)

Bài thứ hai chuyển sang một vấn đề rộng hơn là **tối ưu chi phí trên AWS**.

Thay vì chỉ xem tối ưu chi phí là việc giảm hóa đơn, nhóm tiếp cận chủ đề này theo AWS Well-Architected Framework: hiểu chi phí phát sinh từ đâu, đặt chi phí cạnh kết quả mà workload tạo ra, điều chỉnh tài nguyên theo nhu cầu thực tế, xác định rõ người chịu trách nhiệm và duy trì việc rà soát chi phí thường xuyên.

Chủ đề này cũng liên quan khá trực tiếp đến quá trình thực tập của nhóm, vì một hệ thống cloud không chỉ cần hoạt động đúng mà còn cần sử dụng tài nguyên một cách hợp lý.

### [Blog 3 - Amazon Aurora DSQL: Khi Cơ Sở Dữ Liệu Phân Tán Không Còn Phải Đánh Đổi Giữa Tốc Độ Và Tính Nhất Quán](3.3-Blog3/)

Bài thứ ba tìm hiểu về **Amazon Aurora DSQL** và các ý tưởng distributed systems phía sau kiến trúc của dịch vụ này.

Nhóm tìm hiểu cách Aurora DSQL tách riêng query processing, transaction coordination, journaling và storage, cũng như cách Optimistic Concurrency Control được sử dụng để hỗ trợ các giao dịch phân tán có tính nhất quán mạnh. Đây cũng là cơ hội để nhóm tìm hiểu một dịch vụ AWS nằm ngoài phạm vi trực tiếp của CloudHop RAG và mở rộng kiến thức về kiến trúc cơ sở dữ liệu phân tán.

Nhìn chung, ba bài viết giúp nhóm mở rộng hiểu biết về AWS ngoài phạm vi dự án chính, đồng thời rèn luyện cách trình bày những chủ đề kỹ thuật theo hướng rõ ràng và có tính ứng dụng hơn.
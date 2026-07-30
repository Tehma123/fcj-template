---
title: "Blog 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.3. </b> "
---

# Hướng Dẫn Chi Tiết: Mở Rộng Amazon CloudWatch Bằng Cribl Stream Cho Mọi Nguồn Dữ Liệu

Trong các môi trường quản lý đám mây lai (hybrid) hoặc đa đám mây (multi-cloud), các tổ chức thường xuyên phải thu thập dữ liệu đo lường từ nhiều nguồn phức tạp như thiết bị mạng độc quyền, công cụ APM tại chỗ (on-premises) hoặc các luồng dữ liệu Apache Kafka.

Mặc dù Amazon CloudWatch đã cung cấp khả năng thu thập log tự nhiên cho hơn 60 dịch vụ AWS và 20 công cụ bên thứ ba, các nguồn dữ liệu nằm ngoài hệ sinh thái này vẫn cần các công đoạn chuẩn hóa phức tạp. Đây là lúc Cribl Stream – một đối tác của AWS – tỏa sáng, cho phép bạn đưa hàng trăm nguồn dữ liệu đa dạng vào CloudWatch một cách liền mạch.

## 1. Cơ Chế Hoạt Động Của Cribl Stream

Cribl Stream đóng vai trò là một đường ống trung gian (pipeline) nằm giữa các hệ thống nguồn và CloudWatch Logs. Dưới đây là những lợi ích kỹ thuật cốt lõi:

- **Hỗ trợ đa dạng nguồn:** Cribl Stream có thể tiếp nhận log từ syslog, Apache Kafka, công cụ APM, tác nhân bảo mật, cũng như các nguồn dựa trên giao thức HTTP, TCP và UDP.
- **Không cần tài nguyên trung gian:** Dữ liệu được ghi trực tiếp vào CloudWatch log group thông qua API `PutLogEvents`. Kiến trúc này loại bỏ hoàn toàn nhu cầu sử dụng bộ nhớ lưu trữ trung gian, hàng đợi SQS hay các hàm Lambda xử lý.
- **Đảm bảo tính toàn vẹn dữ liệu:** Một hàng đợi liên tục (persistent queue) bên trong Cribl Stream sẽ giúp bảo vệ dữ liệu không bị thất thoát khi có lỗi mạng hoặc sự cố tạm thời từ CloudWatch Logs.
- **Khả năng phát lại (Replay):** Quản trị viên có thể phát lại dữ liệu thô phục vụ cho mục đích kiểm toán, điều tra bảo mật hoặc xử lý sự cố.

## 2. Kiến Trúc 3 Lớp Tiêu Chuẩn

Mô hình kết nối này được thiết kế theo 3 lớp kiến trúc tinh gọn:

| Lớp Kiến Trúc | Đặc Điểm Kỹ Thuật |
|---|---|
| **Nguồn (Upstream)** | Mỗi nguồn hệ thống logic nên ghi vào một log group riêng biệt (ví dụ: `/apps/any-company-firewall`) để dễ dàng quản lý quyền truy cập và chính sách lưu giữ. |
| **Cribl Stream** | Có thể chạy trên máy chủ Amazon EC2 từ AWS Marketplace hoặc dưới dạng container trong mạng VPC. Tại đây, log được tự động chuẩn hóa sang định dạng OCSF hoặc OTel. |
| **CloudWatch Logs** | Dữ liệu sau khi chuẩn hóa sẽ lưu tại đây, cho phép kỹ sư thực hiện truy vấn mạnh mẽ bằng Logs Insights QL, OpenSearch PPL hoặc SQL. |

## 3. Các Bước Cấu Hình Tối Ưu

Việc tích hợp không đòi hỏi bạn phải viết mã tùy chỉnh. Thay vào đó, bạn chỉ cần thực hiện các cấu hình sau:

### Triển Khai
- Bạn có thể sử dụng phiên bản phần mềm dạng dịch vụ quản lý (SaaS) Cribl.Cloud Suite hoặc tự triển khai trên AWS Marketplace.
- Đối với môi trường khép kín, hãy đặt Cribl Stream trong private subnet và giao tiếp qua VPC endpoint của CloudWatch Logs (`com.amazonaws.logs`).

### Cấu Hình Log Group & IAM
- **Tiết kiệm chi phí:** Đối với dữ liệu ít truy cập (lưu trữ, tuân thủ), hãy chọn lớp log "Infrequent Access" khi tạo CloudWatch log group.
- **Gắn thẻ (Tags):** Sử dụng các tag `cw:datasource:name` và `cw:datasource:type` để hệ thống CloudWatch dễ dàng nhận diện log.
- **Bảo mật IAM:** Gắn một role IAM cho phiên bản Cribl Stream với các quyền `logs:CreateLogStream`, `logs:PutLogEvents` và `logs:DescribeLogStreams`. Nên giới hạn Resource ARN để Cribl không thể ghi đè ra ngoài các log group được chỉ định.

### Đích Đến (Destination)
- Trong Cribl, hãy cấu hình đích đến trỏ tới CloudWatch Logs với định dạng JSON, xác thực IAM và bật hàng đợi liên tục.
- Nếu lưu lượng log quá lớn, hãy bật nén và tối ưu hóa kích thước batch để tránh vượt quá giới hạn API của `PutLogEvents`.

## 4. Phân Tích Chéo & Hỗ Trợ AI Tác Nhân

Nhờ việc đưa dữ liệu ngoài hệ sinh thái AWS về nằm cạnh các log bản địa của AWS, khả năng điều tra bảo mật trở nên linh hoạt hơn bao giờ hết:

- **Truy vấn chéo hệ thống:** Sử dụng trường `@log` có sẵn để xác định nguồn gốc từng hàng dữ liệu. Bạn có thể dễ dàng truy vết một địa chỉ IP đáng ngờ bằng cách nối dữ liệu từ tường lửa của công ty với luồng Amazon VPC Flow Logs.
- **Phân tích lỗi đa nền tảng:** Kết hợp dữ liệu từ hệ thống định danh (IdP) và AWS CloudTrail để phát hiện các mẫu đăng nhập thất bại một cách tập trung. Hàm `coalesce` trong truy vấn sẽ hỗ trợ việc đọc qua các cấu trúc log khác biệt.

### Sức Mạnh Của Agentic AI

Đáng chú ý, nhờ dữ liệu được chuẩn hóa theo định dạng mở, chúng có thể kết nối với máy chủ Amazon CloudWatch MCP. MCP (Model Context Protocol) là một tiêu chuẩn mở của Linux Foundation giúp liên kết các mô hình ngôn ngữ lớn (LLMs) với dữ liệu thực tế.

Nhờ vậy, các trợ lý AI tương thích như Claude Code, Kiro, hoặc GitHub Copilot có thể phân tích trực tiếp dữ liệu log. Thay vì viết câu lệnh SQL, các kỹ sư trực hệ thống chỉ cần yêu cầu AI: *"Hãy tóm tắt các lần đăng nhập thất bại từ nhà cung cấp danh tính và AWS CloudTrail trong giờ qua"*, và AI sẽ trả về kết quả dựa trên dữ liệu thật.

**Link tham khảo:** <https://aws.amazon.com/blogs/mt/extend-amazon-cloudwatch-beyond-native-connectors-with-cribl-stream/>
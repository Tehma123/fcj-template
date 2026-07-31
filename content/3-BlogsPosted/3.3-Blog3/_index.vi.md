---
title: "Blog 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.3. </b> "
---

# Amazon SQS Fair Queues: Chấm Dứt Vấn Đề "Noisy Neighbor" Trong Hệ Thống Multi-Tenant

![Bài viết đã đăng trên nhóm Facebook AWS Study Group VN](/images/BlogsPosted/blog3.png)
*Đã đăng lên nhóm Facebook AWS Study Group VN.*

Hàng đợi tin nhắn từ lâu đã là xương sống của kiến trúc phân tán — bộ đệm giữ hệ thống không sập dây chuyền khi traffic tăng đột biến. Amazon SQS luôn là lựa chọn quen thuộc nhờ khả năng mở rộng gần như vô hạn.

Nhưng có một vấn đề mà gần như đội nào xây hệ thống multi-tenant trên một hàng đợi dùng chung cũng từng gặp: một tenant "ồn ào" có thể làm chậm tất cả các tenant còn lại.

## 1. Vấn Đề "Noisy Neighbor" Là Gì?

Trong một hệ thống multi-tenant dùng chung một hàng đợi SQS, nếu một tenant đột nhiên gửi lượng tin nhắn khổng lồ hoặc xử lý tin nhắn của mình rất chậm, hàng đợi thông thường vẫn giao tin nhắn theo thứ tự đến trước. Điều này khiến tin nhắn của các tenant khác bị dồn lại phía sau, làm tăng message dwell time (thời gian tin nhắn nằm chờ trong hàng đợi) cho tất cả các tenant khác — dù họ không hề gây ra vấn đề gì. Nói cách khác, một tenant "ồn ào" (noisy neighbor) có thể kéo tụt chất lượng dịch vụ của toàn bộ hệ thống dùng chung.

## 2. Amazon SQS Fair Queues Giải Quyết Vấn Đề Này Như Thế Nào?

Amazon SQS Fair Queues là một tính năng mới giúp tự động giảm thiểu ảnh hưởng của noisy neighbor bằng cách thông minh sắp xếp lại thứ tự giao tin nhắn, ưu tiên các tenant "yên tĩnh" khi phát hiện một tenant đang chiếm dụng phần lớn tài nguyên consumer.

### Định Danh Tenant Qua MessageGroupId

Hệ thống dùng trường `MessageGroupId` có sẵn trên tin nhắn để xác định ranh giới giữa các tenant. Khi gửi tin nhắn, ứng dụng chỉ cần gắn thêm định danh tenant:

```java
SendMessageRequest request = new SendMessageRequest()
    .withQueueUrl(queueUrl)
    .withMessageBody(messageBody)
    .withMessageGroupId("tenant-123");  // Định danh tenant
sqs.sendMessage(request);
```

### Thuật Toán Công Bằng (Fairness Algorithm)

SQS liên tục theo dõi phân bố tin nhắn in-flight (đang xử lý) giữa các tenant. Khi phát hiện mất cân bằng, hệ thống sẽ: (1) xác định tenant đang "ồn ào", (2) tự động ưu tiên lại tin nhắn của các tenant yên tĩnh, và (3) vẫn giữ nguyên throughput tổng thể của hàng đợi.

### Không Cần Sửa Code Phía Consumer

Fair Queues hoạt động hoàn toàn trong suốt (transparent) với consumer — không cần thay đổi logic xử lý tin nhắn, không ảnh hưởng độ trễ API, và không giới hạn throughput.

## 3. Giám Sát Bằng CloudWatch

Tính năng này đi kèm các metric CloudWatch mới dành riêng cho việc theo dõi tính công bằng của hàng đợi:

- `ApproximateNumberOfNoisyGroups`
- `ApproximateNumberOfMessagesVisibleInQuietGroups`
- `ApproximateNumberOfMessagesNotVisibleInQuietGroups`
- `ApproximateNumberOfMessagesDelayedInQuietGroups`
- `ApproximateAgeOfOldestMessageInQuietGroups`

Kết hợp với **CloudWatch Contributor Insights**, đội vận hành có thể xác định chính xác tenant nào đang tiêu tốn tài nguyên bất thường, ngay cả khi hệ thống có tới hàng nghìn message group khác nhau.

## 4. Lợi Ích Mang Lại

- Giữ message dwell time thấp cho các tenant không gây nhiễu, ngay cả khi có traffic tăng đột biến từ một tenant khác.
- Loại bỏ nhu cầu over-provisioning hoặc tự xây giải pháp cách ly tenant riêng.
- Duy trì chất lượng dịch vụ (QoS) một cách trong suốt, không cần thay đổi kiến trúc hiện có.
- Vẫn hỗ trợ throughput không giới hạn, không áp đặt quota cứng cho từng tenant.

## Kết Luận

Amazon SQS Fair Queues giải quyết đúng vào bài toán mà rất nhiều hệ thống multi-tenant gặp phải: một tenant gây nhiễu không nên là lý do khiến toàn bộ tenant khác bị chậm trễ. Bằng cách tận dụng trường `MessageGroupId` sẵn có và một thuật toán công bằng chạy ngầm, tính năng này mang lại khả năng cách ly tenant ngay ở tầng hàng đợi mà không đòi hỏi bất kỳ thay đổi nào ở phía consumer.

**Link tham khảo:** <https://aws.amazon.com/blogs/compute/building-resilient-multi-tenant-systems-with-amazon-sqs-fair-queues/>
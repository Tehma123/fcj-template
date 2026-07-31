---
title: "Blog 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.3. </b> "
---

# Amazon Aurora DSQL: Khi Cơ Sở Dữ Liệu Phân Tán Không Còn Phải Đánh Đổi Giữa Tốc Độ Và Tính Nhất Quán

![Bài viết đã đăng trên nhóm Facebook AWS Study Group VN](/images/BlogsPosted/blog3.png)
*Đã đăng lên nhóm Facebook AWS Study Group VN.*

Xây một cơ sở dữ liệu quan hệ phân tán trên nhiều Region từ trước đến nay luôn buộc đội kỹ thuật phải chọn một trong hai hướng: hoặc chấp nhận eventual consistency để đổi lấy độ trễ thấp, hoặc đồng bộ hoá đồng bộ (synchronous replication) giữa các Region cho mọi lần commit để giữ tính nhất quán mạnh, nhưng phải trả giá bằng độ trễ ghi rất cao. Amazon Aurora DSQL, ra mắt GA từ tháng 5/2025 và liên tục bổ sung tính năng trong suốt năm 2026, được AWS thiết kế để phá vỡ đúng sự đánh đổi kinh điển này: đọc với độ trễ thấp ở bất kỳ Region nào, còn ghi vẫn được xác thực nhất quán trên toàn cầu.

## 1. Kiến Trúc Tách Rời: Bốn Thành Phần Độc Lập Thay Vì Một Khối Instance

Khác với mô hình cơ sở dữ liệu truyền thống nơi compute, storage và transaction coordination gắn chặt vào cùng một instance, Aurora DSQL tách toàn bộ hệ thống thành các thành phần độc lập, có thể mở rộng theo chiều ngang riêng biệt.

- **Query Processor (QP):** Là nơi tiếp nhận và thực thi câu lệnh SQL tương thích PostgreSQL, chạy trong các Firecracker MicroVM và không giữ trạng thái cục bộ nào. Vì không lưu state, QP có thể được cấp phát và thu hồi gần như tức thời theo tải thực tế — đây cũng là lý do Aurora DSQL có thể scale to zero mà không có cold start đáng kể, khác với Aurora Serverless v2 vẫn cần khoảng 15 giây để resume sau khi tạm dừng.
- **Adjudicator:** Là thành phần chịu trách nhiệm xử lý xung đột tại thời điểm commit. Mỗi adjudicator được gán một dải khoá (key range) cụ thể; khi một giao dịch động đến dữ liệu thuộc nhiều dải khoá, các adjudicator liên quan sẽ phối hợp với nhau để quyết định giao dịch có được phép commit hay không.
- **Journal:** Là luồng dữ liệu có thứ tự (ordered stream), nơi các giao dịch đã commit được ghi lại để đảm bảo tính bền vững (durability) và đồng thời dùng để xác nhận việc ghi thành công trở lại cho client.
- **MVCC Storage Replicas:** Lớp lưu trữ này lưu dữ liệu theo mô hình multi-version concurrency control, cho phép nhiều phiên bản dữ liệu cùng tồn tại để phục vụ các giao dịch đọc tại các thời điểm snapshot khác nhau mà không cần khoá.

Vì bốn thành phần này tách rời và mở rộng độc lập, Aurora DSQL không có khái niệm "primary instance" như các cơ sở dữ liệu truyền thống — không có điểm lỗi đơn (single point of failure), và một Region hay Availability Zone gặp sự cố không kéo theo gián đoạn ghi ở nơi khác.

## 2. Optimistic Concurrency Control: Không Khoá, Chỉ Kiểm Tra Xung Đột Khi Commit

Phần lõi giúp Aurora DSQL đạt được nhất quán mạnh mà vẫn giữ độ trễ thấp là cơ chế Optimistic Concurrency Control (OCC), khác căn bản với khoá theo hàng (row-level locking) mà PostgreSQL tiêu chuẩn sử dụng.

Với OCC, một giao dịch được thực thi hoàn toàn mà không cần xin khoá trước — hệ thống "lạc quan" rằng phần lớn giao dịch sẽ không xung đột với nhau. Toàn bộ thao tác đọc trong giao dịch dùng timestamp tại thời điểm giao dịch bắt đầu (Tstart), còn việc kiểm tra xung đột chỉ diễn ra ở thời điểm commit (Tcommit): nếu không có giao dịch nào khác đã commit thay đổi lên cùng dữ liệu trong khoảng Tstart–Tcommit, giao dịch được phép hoàn tất; nếu có, giao dịch bị từ chối với lỗi serialization (SQLSTATE 40001) và ứng dụng cần tự thực hiện retry.

Cách tiếp cận này loại bỏ hoàn toàn deadlock và tình trạng một giao dịch chậm chặn các giao dịch khác — hai vấn đề kinh điển của mô hình khoá truyền thống. Đổi lại, ứng dụng viết trên Aurora DSQL cần được thiết kế để xử lý retry như một phần bình thường của luồng xử lý, đặc biệt với các thao tác ghi lặp lại lên cùng một hàng dữ liệu.

Để việc so sánh Tstart và Tcommit chính xác trên quy mô nhiều Region — nơi đồng hồ vật lý giữa các máy chủ luôn có sai lệch dù rất nhỏ — Aurora DSQL dùng Amazon Time Sync Service kết hợp với ClockBound, một cơ chế không trả về một mốc thời gian chính xác tuyệt đối mà trả về một khoảng thời gian đảm bảo chứa thời điểm thực (ví dụ [10:00:00.123456785, 10:00:00.123456793]). Nhờ biểu diễn thời gian dưới dạng khoảng thay vì điểm, hệ thống có thể xác định chắc chắn sự kiện nào xảy ra trước, hay hai sự kiện có khả năng xảy ra đồng thời — nền tảng để adjudicator đưa ra quyết định commit chính xác mà không cần các node phải liên tục đồng bộ với nhau qua mạng.

Aurora DSQL sử dụng strong snapshot isolation, tương đương mức repeatable read của PostgreSQL, chặt hơn read committed nhưng không đòi hỏi chi phí điều phối toàn cục như serializable isolation ở một số cơ sở dữ liệu phân tán khác. Vì các giao dịch chỉ đọc dùng snapshot tại Tstart, chúng không cần xếp hàng chờ và gần như không bao giờ gặp lỗi OCC.

## 3. Tương Thích PostgreSQL, Nhưng Không Phải "Aurora PostgreSQL Bản Phân Tán"

Aurora DSQL dùng chung parser, planner, optimizer và type system với PostgreSQL 16, nên phần lớn cú pháp SQL, kiểu dữ liệu, phép toán số học giữ hành vi giống hệt PostgreSQL tiêu chuẩn. Tuy nhiên đây là một sản phẩm riêng biệt về kiến trúc, không phải một chế độ vận hành khác của Aurora PostgreSQL, và có một số khác biệt quan trọng người dùng cần nắm trước khi thiết kế schema.

- **DDL chạy bất đồng bộ:** Các lệnh như `CREATE TABLE` hay `ALTER TABLE` được thực thi như tác vụ nền, cho phép đọc/ghi không gián đoạn trong lúc thay đổi schema, khác với PostgreSQL truyền thống nơi một số thao tác DDL có thể khoá bảng. Đổi lại, khi catalog schema được cập nhật bởi một giao dịch khác trong lúc phiên làm việc đang dùng bản cache cũ, phiên đó có thể nhận lỗi OC001 và cần tải lại catalog.
- **Lưu trữ theo thứ tự khoá (key-ordered storage):** Khoá chính quyết định cách dữ liệu được phân bố vật lý và cách việc ghi được phân tán giữa các adjudicator, nên việc chọn khoá chính ảnh hưởng trực tiếp đến hiệu năng ghi ở quy mô lớn.
- **Xác thực qua IAM:** Thay vì chỉ dùng username/password như PostgreSQL thông thường, Aurora DSQL tích hợp xác thực qua AWS IAM, phù hợp với mô hình quản lý danh tính tập trung trong hệ sinh thái AWS.

AWS cũng liên tục thu hẹp khoảng cách tương thích — ví dụ tính năng sequences và identity columns (`GENERATED ALWAYS AS IDENTITY`, `CREATE SEQUENCE`) vốn là một trong những tính năng được yêu cầu nhiều nhất, đã được bổ sung vào đầu năm 2026, cho phép tối đa 5.000 sequence trên mỗi database.

## 4. Thiết Kế Schema Để Tránh Xung Đột: Bài Học Quan Trọng Nhất Khi Dùng OCC

Vì cơ chế OCC chỉ phát hiện xung đột ở cấp độ hàng dữ liệu, một nguyên tắc thiết kế then chốt trên Aurora DSQL là tránh gộp nhiều trường được cập nhật độc lập vào cùng một hàng. Ví dụ, nếu một bảng `account` gộp chung số dư tài khoản, bộ đếm lượt đăng nhập và thời gian cập nhật cuối vào cùng một hàng, thì dù ba giá trị này về mặt logic hoàn toàn độc lập, mọi thao tác cập nhật lên bất kỳ trường nào cũng sẽ tuần tự hoá lẫn nhau vì chúng chia sẻ cùng một hàng — nghĩa là các giao dịch cập nhật đồng thời sẽ liên tục bị từ chối và phải retry, dù về bản chất không hề xung đột nghiệp vụ.

Ngoài ra, giao dịch càng mở lâu và càng chạm vào nhiều hàng thì xác suất bị một giao dịch khác "vượt mặt" trước khi commit càng cao, nên các ứng dụng khai thác tốt Aurora DSQL thường giữ giao dịch ngắn và tập trung. Việc tập trung quá nhiều thao tác ghi vào một khoá hoặc dải khoá cụ thể (hot key/hot key range) cũng làm giảm lợi ích của kiến trúc phân tán, vì về bản chất vẫn dồn tải lên một adjudicator duy nhất.

## 5. Khi Nào Aurora DSQL Là Lựa Chọn Phù Hợp

Kiến trúc active-active đa Region với nhất quán mạnh khiến Aurora DSQL phù hợp với các hệ thống giao dịch tài chính thời gian thực, bảng xếp hạng (leaderboard) cập nhật liên tục, mạng xã hội, hệ thống microservices và nền tảng SaaS cần độ sẵn sàng cao — những nơi ứng dụng cần phục vụ người dùng ở nhiều khu vực địa lý mà vẫn đảm bảo mọi Region đọc được dữ liệu mới nhất, không có Region "chính" duy nhất làm nút thắt cổ chai.

Ngược lại, Aurora DSQL chưa phải lựa chọn tối ưu cho các ứng dụng phụ thuộc nhiều vào tiện ích mở rộng PostgreSQL nâng cao hoặc kiểu dữ liệu tuỳ chỉnh chưa được hỗ trợ đầy đủ, cũng như các workload có đặc điểm ghi liên tục dồn dập lên cùng một hàng dữ liệu — trường hợp này OCC sẽ gây ra tỷ lệ retry cao hơn lợi ích mang lại.

Aurora DSQL hiện có mặt ở 14 AWS Region trên toàn cầu, được các đơn vị như ADP, Cintra, Caylent, DeNA và Robinhood sử dụng cho các hệ thống yêu cầu độ liên tục kinh doanh nghiêm ngặt, đồng thời AWS cung cấp một sandbox trải nghiệm trực tuyến (DSQL Playground) để chạy thử SQL trên một database thật mà không cần tài khoản AWS.

## Kết Luận

Cảm ơn mọi người đã dành thời gian đọc bài viết này. Hy vọng những chia sẻ trên phần nào giúp bạn hiểu rõ hơn về cách Aurora DSQL giải quyết bài toán phân tán kinh điển, và có thêm góc nhìn hữu ích khi cân nhắc cho hệ thống của mình.

**Nguồn tham khảo:**
- <https://aws.amazon.com/blogs/database/concurrency-control-in-amazon-aurora-dsql/>
- <https://aws.amazon.com/blogs/database/dsql-sql-dialect-how-amazon-aurora-dsql-differs-from-single-instance-postgresql/>
- <https://aws.amazon.com/blogs/database/building-scalable-applications-on-amazon-aurora-dsql/>
- <https://aws.amazon.com/blogs/database/everything-you-dont-need-to-know-about-amazon-aurora-dsql-part-2-shallow-view/>
- <https://aws.amazon.com/rds/aurora/dsql/>

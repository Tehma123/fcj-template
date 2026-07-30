---
title: "Blog 2"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.2. </b> "
---

# Tối Ưu Chi Phí AWS: Đừng Chỉ Nhìn Vào Hóa Đơn

Khi hóa đơn AWS tăng, phản ứng đầu tiên của nhiều đội ngũ là tìm tài nguyên nào có thể tắt, cấu hình nào có thể thu nhỏ hoặc dịch vụ nào đang tốn tiền nhất. Những việc này có thể giúp giảm chi phí trước mắt, nhưng chưa đủ để kết luận hệ thống đã được tối ưu.

Một hệ thống rẻ hơn chưa chắc đã tốt hơn. Nếu chi phí giảm nhưng ứng dụng chậm đi, khách hàng gặp lỗi nhiều hơn hoặc đội kỹ thuật phải mất thêm thời gian vận hành, doanh nghiệp có thể đang tiết kiệm một cách sai lầm.

Theo AWS Well-Architected Framework, tối ưu chi phí là một quá trình diễn ra trong suốt vòng đời của workload. Mục tiêu không phải luôn chọn phương án rẻ nhất, mà là sử dụng tài nguyên vừa đủ để đạt được kết quả mong muốn và vẫn đáp ứng các yêu cầu của hệ thống.

Trước khi hỏi "cắt được bao nhiêu?", doanh nghiệp nên trả lời bốn câu hỏi:

* Tiền đang được chi vào đâu?
* Ai đang sử dụng và chịu trách nhiệm cho khoản chi đó?
* Khoản chi này tạo ra kết quả gì?
* Lượng tài nguyên hiện tại còn phù hợp với nhu cầu không?

## 1. Trước hết, phải biết chi phí đến từ đâu

Một hóa đơn AWS có thể bao gồm hàng trăm hoặc hàng nghìn resource thuộc nhiều account, team, dự án và môi trường khác nhau. Nếu chỉ nhìn tổng hóa đơn, rất khó biết phần chi phí nào đang tăng và ai đang phụ trách workload đó. Vì vậy, doanh nghiệp cần biết mỗi tài nguyên thuộc team, sản phẩm, môi trường và người phụ trách nào.

Tags là một cách phổ biến để bổ sung những thông tin này vào resource. Tuy nhiên, việc quản lý chi phí không chỉ phụ thuộc vào tags. Cấu trúc AWS account, AWS Cost Categories và cách tổ chức dữ liệu billing cũng có thể được dùng để phân bổ chi phí theo nhiều góc nhìn khác nhau.

Ví dụ, cùng một database có thể thuộc:

* team Payments;
* sản phẩm Checkout;
* môi trường Production;
* cost center Digital Commerce.

Finance có thể cần xem chi phí theo cost center, engineering cần xem theo workload, còn product owner lại quan tâm đến chi phí của từng sản phẩm. Một hệ thống phân bổ tốt phải phục vụ được những nhu cầu đó thay vì ép mọi người nhìn chi phí theo một chiều duy nhất.

Quan trọng hơn, phân bổ chi phí không chỉ để làm báo cáo. Khi một đội ngũ biết workload mình phụ trách đang phát sinh bao nhiêu chi phí và khoản chi đó đến từ đâu, họ có cơ sở để chủ động điều chỉnh cách sử dụng cloud.

## 2. Đặt chi phí cạnh kết quả workload tạo ra

Tổng chi phí tăng chưa chắc là dấu hiệu xấu.

Giả sử một workload tốn 100 USD để xử lý 10.000 giao dịch, tương đương 0,01 USD cho mỗi giao dịch. Tháng sau, hóa đơn tăng lên 120 USD nhưng hệ thống xử lý được 15.000 giao dịch. Tổng chi phí tăng 20%, trong khi chi phí trên mỗi giao dịch giảm còn 0,008 USD.

Trong trường hợp này, workload có thể đang hoạt động hiệu quả hơn trước. Đó là lý do AWS khuyến nghị đo chi phí cùng với business output. Tùy loại workload, doanh nghiệp có thể theo dõi:

* chi phí trên mỗi giao dịch;
* chi phí trên mỗi khách hàng;
* chi phí trên mỗi tài liệu được xử lý;
* chi phí trên mỗi đơn hàng hoàn tất.

Các chỉ số kỹ thuật như CPU, bộ nhớ và số lượng request vẫn cần thiết để theo dõi hệ thống. Tuy nhiên, chúng không tự cho biết workload có tạo ra giá trị hay không. CPU được sử dụng hiệu quả không có nhiều ý nghĩa nếu chi phí trên mỗi giao dịch vẫn tăng hoặc hệ thống xử lý được ít công việc hơn.

Vì vậy, một câu hỏi tốt hơn "hóa đơn tháng này tăng bao nhiêu?" là:

> Với số tiền đã chi, workload tạo ra được bao nhiêu kết quả có ý nghĩa cho doanh nghiệp?

## 3. Điều chỉnh tài nguyên theo nhu cầu

Một trong những lợi thế lớn nhất của cloud là khả năng tăng hoặc giảm tài nguyên theo nhu cầu. Tuy nhiên, lợi thế này chỉ mang lại hiệu quả khi đội ngũ hiểu workload được sử dụng vào thời điểm nào và mức sử dụng thay đổi ra sao. Chẳng hạn, môi trường development và testing thường không cần chạy liên tục cả tuần. Nếu chỉ được sử dụng trong giờ làm việc, việc lên lịch tắt ngoài giờ có thể giảm đáng kể phần chi phí tính theo thời gian chạy.

Với hệ thống production, nhu cầu có thể thay đổi theo giờ trong ngày, theo mùa, theo số lượng người dùng hoặc theo các chiến dịch kinh doanh. Nếu quy luật sử dụng khá ổn định, tài nguyên có thể được định kỳ tăng giảm trước và sau giờ cao điểm. Nếu lưu lượng biến động khó đoán hơn, Auto Scaling có thể điều chỉnh tài nguyên dựa trên các chỉ số của hệ thống.

Tuy nhiên, mức sử dụng trung bình thấp chưa chắc có nghĩa là tài nguyên đang dư. Hệ thống vẫn có thể cần năng lực dự phòng để đáp ứng giờ cao điểm, duy trì SLA hoặc tiếp tục hoạt động khi một thành phần gặp sự cố.

Tối ưu chi phí không có nghĩa là cấp ít tài nguyên nhất. Mục tiêu là cấp lượng tài nguyên phù hợp với nhu cầu và yêu cầu dịch vụ, tránh cả hai tình trạng:

* **over-provisioning:** cấp nhiều hơn mức cần thiết;
* **under-provisioning:** cấp quá ít, làm giảm hiệu năng hoặc ảnh hưởng khách hàng.

## 4. Chi phí cần có người phụ trách minh bạch

Trong mô hình hạ tầng truyền thống, đội kỹ thuật thường gửi yêu cầu mua sắm, finance phê duyệt ngân sách và operations triển khai thiết bị. Quy trình này có thể mất nhiều thời gian, nhưng trách nhiệm giữa các bên là rõ ràng.

Trên cloud, đội kỹ thuật có thể tạo tài nguyên rất nhanh, nên chi phí cũng thay đổi nhanh hơn và khó được kiểm soát chỉ bằng quy trình ngân sách truyền thống.

Bộ phận tài chính nắm ngân sách và dự báo. Đội kỹ thuật hiểu hệ thống và tác động của từng quyết định kiến trúc. Đội sản phẩm và kinh doanh lại biết các kế hoạch có thể làm nhu cầu sử dụng tăng hoặc giảm. Không nhóm nào có đủ thông tin để tự quản lý toàn bộ chi phí cloud.

Doanh nghiệp vì thế cần chỉ định một người hoặc một nhóm phụ trách việc tối ưu chi phí, đồng thời duy trì các buổi trao đổi định kỳ giữa tài chính, kỹ thuật và kinh doanh. Ở tổ chức nhỏ, đây có thể là một người kiêm nhiệm. Với tổ chức lớn hơn, trách nhiệm này có thể thuộc về FinOps team hoặc một nhóm liên phòng ban.

Điều quan trọng là họ phải có:

* mục tiêu rõ ràng;
* dữ liệu chi phí đủ chi tiết;
* thời gian được phân bổ cho công việc;
* cơ chế review định kỳ;
* sự hỗ trợ từ lãnh đạo.

Nếu không có người chịu trách nhiệm minh bạch, tối ưu chi phí rất dễ trở thành "việc của tất cả mọi người", nhưng cuối cùng lại không phải trách nhiệm chính của ai.

## 5. Tối ưu chi phí phải trở thành một thói quen

Nhiều tổ chức chỉ bắt đầu quan tâm tới vấn đề chi phí khi hóa đơn tăng bất thường. Khi đó, cost optimization đã trở thành một vấn đề khẩn cấp cần xử lý: tìm resource không dùng, giảm instance size hoặc yêu cầu các team cắt chi phí.

Cách làm này có thể tạo ra kết quả ngắn hạn nhưng khó duy trì. AWS khuyến nghị xây dựng cost-aware culture, nghĩa là chi phí được cân nhắc ngay khi:

* thiết kế workload mới;
* thay đổi kiến trúc;
* tạo môi trường thử nghiệm;
* triển khai tính năng;
* lựa chọn service hoặc pricing model.

Điều này không có nghĩa là yêu cầu mọi kỹ sư phải trở thành chuyên gia tài chính. Họ chỉ cần nhìn thấy được tác động chi phí của quyết định mình đưa ra và biết khi nào cần trao đổi với finance hoặc product owner.

Ví dụ, trước khi tăng cấu hình một database, team có thể xem:

* nguyên nhân thực sự có phải thiếu tài nguyên không;
* thay đổi sẽ làm chi phí tăng bao nhiêu;
* mức tăng đó có cải thiện business output hay trải nghiệm khách hàng không;
* có phương án nào khác ít tốn kém hơn không.

Khi cost trở thành một phần của quyết định kỹ thuật, tổ chức sẽ ít phải chạy theo xử lý hậu quả sau khi hóa đơn đã tăng.

## 6. Rà soát định kỳ, nhưng không tối ưu bằng mọi giá

AWS liên tục ra mắt dịch vụ, tính năng và loại tài nguyên mới. Bản thân workload cũng thay đổi theo số lượng người dùng, dữ liệu và yêu cầu kinh doanh. Vì vậy, một cấu hình phù hợp hôm nay có thể không còn phù hợp sau vài tháng.

Những workload có chi phí lớn, thay đổi nhanh hoặc ảnh hưởng trực tiếp đến khách hàng nên được xem lại thường xuyên hơn các workload nhỏ và ổn định. Tuy nhiên, không phải dịch vụ mới nào cũng đáng để chuyển sang ngay. Việc thay đổi kiến trúc cũng tốn thời gian triển khai, kiểm thử, đào tạo và có thể tạo thêm rủi ro vận hành.

Trước khi thực hiện một thay đổi, đội ngũ nên ghi lại chi phí hiện tại và kết quả mà workload đang tạo ra. Sau khi triển khai, đo lại cùng các chỉ số đó để kiểm tra xem thay đổi có thực sự giúp hệ thống hiệu quả hơn hay không.

Cũng có những thời điểm speed-to-market quan trọng hơn việc tìm cấu hình rẻ nhất. Khi cần ra mắt sản phẩm đúng thời điểm, hoàn thành migration trước hạn hoặc thử nghiệm nhanh một ý tưởng, doanh nghiệp có thể chấp nhận cấp dư tài nguyên trong giai đoạn đầu. Điều quan trọng là phải quay lại đánh giá và điều chỉnh sau đó.

Bản thân việc tối ưu cũng tốn thời gian và nguồn lực. Doanh nghiệp không nên dành nhiều tuần để tiết kiệm một khoản rất nhỏ trong khi những workload chiếm phần lớn hóa đơn vẫn chưa được xem xét. Công sức bỏ ra cần tương xứng với lợi ích có thể đạt được.

## Kết luận

Tối ưu chi phí AWS không bắt đầu từ việc tìm tài nguyên nào để cắt. Nó bắt đầu từ việc biết chi phí đến từ đâu, workload tạo ra kết quả gì, lượng tài nguyên hiện tại có phù hợp với nhu cầu hay không và ai chịu trách nhiệm xem lại khi mọi thứ thay đổi.

Khi những câu hỏi này được trả lời thường xuyên, tối ưu chi phí không còn là một đợt cắt giảm ngân sách mỗi khi hóa đơn tăng. Nó trở thành một phần bình thường trong cách doanh nghiệp vận hành cloud.

*Nguồn tham khảo: AWS Well-Architected Framework – Cost Optimization Pillar.*

...Hình ảnh...

...Link...

...Hướng dẫn...

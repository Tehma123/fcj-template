---
title: "Amazon API Gateway"
date: 2026-07-31
weight: 8
chapter: false
pre: " <b> 5.8. </b> "
---

FastAPI backend trên EC2 hoạt động qua HTTP, trong khi frontend được AWS Amplify phân phối qua HTTPS. Trình duyệt không thể gọi trực tiếp HTTP backend từ một trang HTTPS, vì vậy **Amazon API Gateway** được sử dụng làm điểm truy cập HTTPS công khai cho ứng dụng.

API Gateway nhận request từ trình duyệt và chuyển tiếp đến FastAPI service trên EC2. Dịch vụ này đồng thời cung cấp cấu hình CORS cần thiết để frontend trên Amplify có thể gọi backend thành công.

{{% notice info %}}
**Bạn sẽ có gì sau chương này:** một HTTP API với ba route chuyển tiếp về EC2, CORS đã cấu hình cho origin của Amplify, và một lệnh `POST /query` qua HTTPS trả về câu trả lời thật đã được kiểm chứng. Chương này là nơi ba lỗi cấp trình duyệt khác nhau được giải quyết, và mỗi lỗi đều đáng hiểu bản chất chứ không chỉ chép lại cách sửa.
{{% /notice %}}

---

## 1. Vì sao chọn HTTP API thay vì REST API

API Gateway có hai loại. Dự án này dùng **HTTP API**:

| | HTTP API | REST API |
| --- | --- | --- |
| Mục đích ở đây | Kết thúc TLS và proxy đơn giản về EC2 | Quản trị API đầy đủ |
| Chi phí | Khoảng một phần ba giá REST API | Cao hơn |
| Tính năng | Route, integration, CORS, xác thực JWT | Thêm usage plan, API key, kiểm tra request, WAF |
| Mức phù hợp | **Đã chọn** - đủ mọi thứ cần, không thừa | Quá mức cần thiết cho ba route proxy |

Backend vốn đã tự kiểm tra dữ liệu bằng các model Pydantic, nên tính năng validate request của REST API chỉ làm trùng lặp công việc. Application Load Balancer cũng kết thúc TLS được, nhưng nó tính tiền theo giờ bất kể có ai dùng hay không và cần chứng chỉ cùng tên miền riêng; API Gateway cung cấp sẵn endpoint HTTPS có chứng chỉ và tính tiền theo request.

---

## 2. Tạo API và các integration

**Console:** API Gateway → **Create API** → **HTTP API** → Build.

Đặt tên nói rõ nó là gì - trong dự án này là `rag-ec2-backend`.

Đích của integration chính là backend EC2. Và đây là cái bẫy thật sự đầu tiên.

{{% notice warning %}}
**Đừng dùng một integration chung chung.** Trỏ một integration vào URL gốc:

```text
http://<elastic-ip>:8000/
```

sẽ cho ra kết quả này ở mọi lệnh gọi:

```json
{"detail":"Not Found"}
```

Request có tới được FastAPI, nhưng đường dẫn không được chuyển tiếp như bạn tưởng, nên FastAPI bị hỏi một route không tồn tại. Phản hồi trông như lỗi backend nhưng thực ra không phải.

**Cách sửa là mỗi route một integration, mỗi cái trỏ tới đường dẫn backend đầy đủ:**

```text
http://<elastic-ip>:8000/health
http://<elastic-ip>:8000/warmup
http://<elastic-ip>:8000/query
```
{{% /notice %}}

Đây cũng là lúc Elastic IP ở chương 5.7 phát huy tác dụng. Integration lưu lại một **địa chỉ cụ thể**. Không có IP cố định thì mỗi lần EC2 stop/start là API âm thầm trỏ sang một địa chỉ không còn thuộc về instance của bạn nữa - và biểu hiện của lỗi là timeout chứ không phải một thông báo rõ ràng.

---

## 3. Các route

Ba route, khớp với các endpoint từ chương 5.7:

| Route | Đích integration | Mục đích |
| --- | --- | --- |
| `GET /health` | `http://<elastic-ip>:8000/health` | Kiểm tra sống và cấu hình |
| `POST /warmup` | `http://<elastic-ip>:8000/warmup` | Nạp pipeline trước khi có traffic thật |
| `POST /query` | `http://<elastic-ip>:8000/query` | Trả lời một câu hỏi |

`POST /warmup` ở tầng này là tùy chọn - `systemd` vốn đã gọi nó cục bộ ngay trên instance sau mỗi lần khởi động lại (chương 5.7 mục 8). Phơi nó qua API Gateway chỉ hữu ích khi bạn muốn làm nóng dịch vụ từ máy cá nhân hoặc từ một trang quản trị. Thêm vào không tốn gì, và thực sự tiện khi instance vừa được bật lại sau một đêm tắt máy.

<!-- ẢNH 1 - SCREENSHOT.
     API Gateway Console -> API của bạn -> Routes.
     Phải thấy đủ ba route (GET /health, POST /warmup, POST /query) trong danh sách.
     Nếu thấy được cả đích integration bên cạnh mỗi route thì càng tốt - đó là bằng chứng
     cho mục 2. Làm mờ Elastic IP nếu bạn không muốn công khai nó. -->

![Ba route và các integration của chúng](/images/5-Workshop/5.8-API-Gateway/routes.png)

---

## 4. CORS có hai tầng, không phải một

Đây là phần khiến nhiều người mất cả buổi chiều. Trình duyệt khi gọi API từ một origin khác sẽ gửi trước một request **preflight** dạng `OPTIONS`, và **cả** API Gateway lẫn FastAPI đều phải được cấu hình để đáp ứng nó.

Lỗi hiện ra nếu bạn chỉ cấu hình một trong hai:

```text
Access to fetch at 'https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/query'
from origin 'https://<your-amplify-app>.amplifyapp.com'
has been blocked by CORS policy.
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

**Tầng 1 - CORS của API Gateway.** Trong Console, mục **CORS**:

| Thiết lập | Giá trị |
| --- | --- |
| `Access-Control-Allow-Origin` | `https://<your-amplify-app>.amplifyapp.com` |
| `Access-Control-Allow-Methods` | `GET`, `POST`, `OPTIONS` |
| `Access-Control-Allow-Headers` | `content-type` |
| `Access-Control-Allow-Credentials` | Không |

**Tầng 2 - CORS của FastAPI.** Backend có `CORSMiddleware` riêng, và danh sách origin được phép của nó đến từ tham số SSM `cors-allow-origins` đã đặt ở chương 5.7:

```text
http://localhost:5173,http://127.0.0.1:5173,https://<your-amplify-app>.amplifyapp.com
```

Hai origin cục bộ chính là dev server của Vite, nhờ vậy cùng một backend phục vụ được cả môi trường phát triển cục bộ mà không cần cấu hình riêng.

{{% notice note %}}
Hãy để ý thứ **không** được dùng: ký tự đại diện `*`. Origin được phép được nêu tên tường minh ở cả hai tầng. Dùng dấu `*` sẽ cho phép bất kỳ website nào trên internet gọi API này từ trình duyệt của người dùng, và vì API không có xác thực nên đây là khác biệt thực chất chứ không phải hình thức.
{{% /notice %}}

**Kiểm tra preflight trực tiếp** - cách này nhanh hơn nhiều so với ngồi gỡ lỗi qua trình duyệt:

```powershell
curl.exe -i -X OPTIONS "https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/query" `
  -H "Origin: https://<your-amplify-app>.amplifyapp.com" `
  -H "Access-Control-Request-Method: POST" `
  -H "Access-Control-Request-Headers: content-type"
```

Cấu hình đúng sẽ trả về:

```text
HTTP/1.1 204 No Content
access-control-allow-origin: https://<your-amplify-app>.amplifyapp.com
access-control-allow-methods: GET,OPTIONS,POST
access-control-allow-headers: content-type
```

Nếu thiếu ba header đó, trình duyệt sẽ từ chối request thật dù backend có chạy tốt đến đâu.

<!-- ẢNH 2 - SCREENSHOT.
     API Gateway Console -> API của bạn -> trang cấu hình CORS, hiển thị
     Access-Control-Allow-Origin (URL Amplify), Allow-Methods và Allow-Headers.
     Chụp SAU KHI đã bấm lưu, để giá trị hiển thị đúng là giá trị đang có hiệu lực. -->

![Cấu hình CORS trong API Gateway](/images/5-Workshop/5.8-API-Gateway/cors-config.png)

---

## 5. Triển khai API

HTTP API tạo sẵn stage `$default` với **auto-deploy đang bật**, nên thay đổi về route và CORS có hiệu lực ngay mà không cần bước deploy thủ công. Nếu bạn tạo một stage đặt tên riêng thì phải deploy lại sau mỗi lần đổi - một tỉ lệ đáng ngạc nhiên các báo cáo "CORS vẫn hỏng" thực chất chỉ là stage chưa được deploy.

Ghi lại **Invoke URL**:

```text
https://<api-id>.execute-api.ap-southeast-1.amazonaws.com
```

Chỉ duy nhất URL này sẽ được cấu hình cho frontend ở chương 5.9. Không có gì khác về backend bị phơi ra cho trình duyệt.

---

## 6. Kiểm tra qua HTTPS

Health trước - nhanh và không cần pipeline:

```powershell
curl.exe "https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/health"
```

```json
{"status":"ok","pipeline_loaded":false}
```

Rồi đến một câu hỏi thật:

```powershell
'{"question":"Were Scott Derrickson and Ed Wood of the same nationality?"}' | Set-Content -Encoding utf8 query.json

curl.exe -s -X POST "https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/query" `
  -H "Content-Type: application/json" `
  --data-binary "@query.json"
```

```json
{ "answer": "yes", "sources": [ ... ], "timings": { ... } }
```

{{% notice tip %}}
Luôn kiểm tra backend EC2 **trực tiếp** trước (chương 5.7 mục 10), rồi mới qua API Gateway. Nếu gọi trực tiếp chạy được mà qua gateway thì không, lỗi nằm ở chương này - route, đường dẫn integration hoặc CORS. Nếu cả hai đều hỏng, lỗi nằm ở chương 5.7. Kiểm tra tách bạch hai nửa sẽ biến một câu "ứng dụng hỏng rồi" mơ hồ thành một chẩn đoán trong hai phút.
{{% /notice %}}

<!-- ẢNH 3 - SCREENSHOT.
     Terminal hiển thị, nếu được thì trong cùng một tấm:
       1. phản hồi GET /health qua HTTPS
       2. request OPTIONS preflight trả về 204 kèm ba header access-control
     Chữ https:// trong URL chính là điểm mấu chốt của cả chương - chụp cho rõ. -->

![Kiểm tra API qua HTTPS](/images/5-Workshop/5.8-API-Gateway/https-test.png)

---

## 7. Trần thời gian chờ

API Gateway chờ backend khoảng **30 giây**. Khi pipeline chạy lâu hơn, phía gọi nhận được:

```json
{"message":"Service Unavailable"}
```

Thông báo này gây hiểu nhầm - thường thì nó **không** có nghĩa là backend chết. Nó nghĩa là API Gateway đã hết kiên nhẫn chờ. Một request đo được từ frontend mất **26,56 giây**, tức vẫn nằm trong giới hạn nhưng gần như không còn biên an toàn, và request đầu tiên sau khi khởi động lại còn chậm hơn nhiều.

Chính phép đo đó đã dẫn tới hai quyết định đã thực hiện ở chương 5.7:

- `RAG_FAST_MODE=true` để cắt giảm độ trễ ở trạng thái ổn định
- `POST /warmup` được `systemd` gọi sau mỗi lần khởi động lại, để không ai phải trả giá cho lần nạp nguội

Nếu pipeline buộc phải vượt quá 30 giây, API Gateway không còn là hình thái phù hợp cho route đó, và các lựa chọn thay thế là Nginx hoặc ALB kết thúc TLS ngay trên EC2, hoặc mô hình job bất đồng bộ trong đó `/query` trả về ngay một job id còn frontend thì hỏi kết quả sau.

---

## 8. Thế trận bảo mật

Hai điều cần nói thẳng:

- **API không có xác thực.** Bất kỳ ai có Invoke URL đều gửi câu hỏi được, và mỗi câu hỏi đều tiêu tốn token của Groq. Với một bản demo thì chấp nhận được; với bất cứ thứ gì nghiêm túc, HTTP API hỗ trợ JWT authorizer, hoặc Lambda authorizer, hoặc đặt một API key phía trước.
- **API Gateway không che giấu backend.** Cổng `8000` trên instance vẫn tiếp cận trực tiếp được (chương 5.7 mục 4). API Gateway giải quyết vấn đề *HTTPS*, không phải vấn đề *phơi nhiễm*. Muốn backend thực sự riêng tư thì cần **VPC Link** trỏ tới một instance nằm trong private subnet - điều này được ghi nhận như một hạn chế đã biết ở chương 5.13.

---

## 9. Các lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| `Mixed Content ... blocked` trong console trình duyệt | Frontend đang gọi địa chỉ HTTP của EC2 chứ không phải gateway | Đặt `VITE_API_BASE_URL` là Invoke URL HTTPS (chương 5.9) |
| `blocked by CORS policy` | Thiếu origin ở API Gateway, hoặc ở FastAPI, hoặc cả hai | Cấu hình cả hai tầng, rồi chạy lại phép thử preflight ở mục 4 |
| `{"detail":"Not Found"}` | Integration dùng URL chung chung; đường dẫn không được chuyển tiếp | Mỗi route một integration với đường dẫn backend đầy đủ (mục 2) |
| `{"message":"Service Unavailable"}` | Backend không tới được, security group chặn, hoặc truy vấn vượt ~30 giây | Kiểm tra EC2 trực tiếp, làm nóng, rồi thử lại |
| Health chạy được nhưng query timeout | Khởi động nguội | Gọi `POST /warmup` trước |
| Hôm qua chạy được, hôm nay hỏng hết | IP công khai của EC2 đã đổi | Elastic IP (chương 5.7 mục 3) |
| Đổi CORS mà không thấy tác dụng | Stage đặt tên riêng chưa được deploy lại | Deploy stage, hoặc dùng `$default` với auto-deploy |

---

## 10. Kết quả

Backend giờ đã tiếp cận được qua HTTPS tại một URL được quản lý và ổn định, với CORS chỉ chấp nhận đúng một origin trình duyệt. Vấn đề Mixed Content vốn là lý do tồn tại của chương này đã được giải quyết, và mảnh ghép cuối cùng còn lại chính là giao diện.

Chương 5.9 sẽ triển khai frontend React lên Amplify và trỏ nó vào Invoke URL này.

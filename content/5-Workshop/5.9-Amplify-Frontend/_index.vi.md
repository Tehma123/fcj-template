---
title: "Triển khai Frontend với AWS Amplify"
date: 2026-07-31
weight: 9
chapter: false
pre: " <b> 5.9. </b> "
---

Thành phần trực tiếp tương tác với người dùng của CloudHop RAG là một ứng dụng web **React/Vite** được triển khai bằng **AWS Amplify**. Giao diện cho phép người dùng nhập câu hỏi, nhận câu trả lời được sinh ra và xem các nguồn hỗ trợ do RAG backend trả về.

Frontend giao tiếp với backend thông qua HTTPS endpoint được tạo trên Amazon API Gateway. Cách triển khai này tách giao diện web khỏi EC2 service nhưng vẫn cung cấp một điểm truy cập thống nhất cho người dùng.

{{% notice info %}}
**Bạn sẽ có gì sau chương này:** toàn hệ thống chạy được từ trình duyệt - một câu hỏi gõ vào trang web đã host và trả về câu trả lời kèm bằng chứng, HTTPS suốt từ đầu đến cuối. Đây là bước triển khai cuối cùng.
{{% /notice %}}

---

## 1. Thứ được triển khai

Một ứng dụng single-page React 19 build bằng Vite. Toàn bộ giao diện gọn trong một component, gửi câu hỏi đi và hiển thị phản hồi:

| Trường do `/query` trả về | Hiển thị thành |
| --- | --- |
| `answer` | Câu trả lời, đặt ở vị trí nổi bật |
| `sources` | Các tài liệu hỗ trợ đứng sau câu trả lời |
| `timings` | Thời gian truy hồi và tổng thời gian phản hồi |
| `num_candidates` | Số tài liệu đã được xem xét |

Việc hiển thị `sources` không phải để trang trí. Một hệ thống RAG chỉ đưa ra mỗi câu trả lời thì không khác gì một chatbot đoán mò; chính việc phơi bằng chứng thu được mới cho phép người đọc đối chiếu câu trả lời với nguồn của nó.

Amplify Hosting được chọn thay vì S3 + CloudFront vì nó cung cấp sẵn triển khai theo git push, chứng chỉ TLS được quản lý và CDN mà không cần cấu hình thủ công. Bộ S3 + CloudFront tương đương đòi hỏi tự tay tạo bucket, distribution, chứng chỉ và origin access policy - nhiều bộ phận chuyển động hơn cho cùng một kết quả.

---

## 2. Kết nối repository

**Console:** Amplify → **Create new app** → **Deploy from Git** → GitHub → cấp quyền → chọn repository và nhánh.

| Thiết lập | Giá trị trong dự án này |
| --- | --- |
| Tên app | `aws-rag-project` |
| Nhánh | nhánh bạn dùng để triển khai |
| **Monorepo root directory** | **`frontend`** |

Thiết lập monorepo rất quan trọng. Repository này chứa cả `backend/` lẫn `frontend/`, nên phải nói cho Amplify biết ứng dụng nằm trong thư mục con `frontend/`. Không có nó, quá trình build chạy ở thư mục gốc, không tìm thấy `package.json` và thất bại ngay lập tức.

<!-- ẢNH 1 - SCREENSHOT.
     Amplify Console -> app của bạn -> trang thiết lập repository/branch, hiển thị
     repository GitHub đã kết nối, tên nhánh, và monorepo root directory đặt là
     "frontend".
     Trường monorepo chính là điểm mấu chốt của tấm ảnh này - nhớ để nó lộ ra. -->

![Repository và nhánh đã kết nối với Amplify](/images/5-Workshop/5.9-Amplify-Frontend/amplify-repo.png)

---

## 3. Cấu hình build

Vite xuất ra một trang tĩnh vào thư mục `dist/`, và đó chính là thư mục Amplify phải publish:

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm install
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: dist
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

Hai dòng cần đặt cho đúng:

- **`baseDirectory: dist`** - thư mục đầu ra của Vite. Amplify mặc định là `build` (quy ước của Create React App). Để nguyên mặc định sẽ cho ra một bản build thành công rồi kèm theo một trang trắng hoặc lỗi 404, vì Amplify publish một thư mục không tồn tại.
- **`cache: node_modules/**/*`** - lưu cache dependency giữa các lần build, rút ngắn đáng kể thời gian build từ lần thứ hai trở đi.

---

## 4. `VITE_API_BASE_URL` - thiết lập quan trọng nhất

**App settings → Environment variables:**

| Key | Giá trị |
| --- | --- |
| `VITE_API_BASE_URL` | `https://<api-id>.execute-api.ap-southeast-1.amazonaws.com` |

Đây bắt buộc phải là **Invoke URL HTTPS của API Gateway** từ chương 5.8 - tuyệt đối không phải địa chỉ EC2. Dùng địa chỉ EC2 sẽ cho ra đúng thông báo này trong console trình duyệt:

```text
Mixed Content: The page at 'https://<your-amplify-app>.amplifyapp.com/' was loaded
over HTTPS, but requested an insecure resource 'http://<elastic-ip>:8000/query'.
This request has been blocked; the content must be served over HTTPS.
```

Trang vẫn tải hoàn hảo còn mọi câu hỏi đều thất bại. Đây chính là lỗi khiến chương 5.8 phải tồn tại.

{{% notice warning %}}
**Biến môi trường của Vite được nhúng cứng lúc build, không phải đọc lúc chạy.** Frontend đọc như sau:

```javascript
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;
```

Vite thay thế biểu thức đó bằng chuỗi giá trị ngay trong lúc `npm run build`. File JavaScript đã biên dịch chứa URL dưới dạng văn bản - không hề có bước tra cứu lúc chạy.

Hệ quả thực tế: **đổi biến này trong Amplify Console sẽ không có tác dụng gì cho tới khi bạn deploy lại.** Sửa giá trị rồi F5 lại trang thì vẫn thấy URL cũ đang được gọi, điều này gây bối rối nặng nếu bạn quen với cách biến môi trường hoạt động trên máy chủ. Sau mỗi lần đổi, hãy bấm **Redeploy this version**.
{{% /notice %}}

Có một thiết lập đối ứng ở phía bên kia: origin của Amplify phải xuất hiện trong tham số SSM `cors-allow-origins` của backend (chương 5.7) và trong cấu hình CORS của API Gateway (chương 5.8). URL frontend chỉ biết được sau khi app được tạo lần đầu, nên trình tự thông thường là: deploy một lần, sao chép URL Amplify, rồi cập nhật CORS ở cả hai tầng.

<!-- ẢNH 2 - SCREENSHOT.
     Amplify Console -> App settings -> Environment variables, hiển thị
     VITE_API_BASE_URL và giá trị của nó.
     Giá trị phải thấy rõ bắt đầu bằng https:// và có chữ execute-api - đó là bằng chứng
     cho thấy vấn đề Mixed Content đã được giải quyết. Làm mờ api-id nếu bạn muốn. -->

![Biến môi trường chứa URL của API](/images/5-Workshop/5.9-Amplify-Frontend/amplify-env-var.png)

---

## 5. Triển khai

Amplify tự build mỗi khi có push lên nhánh đã kết nối. Muốn triển khai thủ công sau khi đổi thiết lập, dùng **Redeploy this version**.

Quá trình build chạy qua bốn giai đoạn - Provision, Build, Deploy, Verify - và log rất đáng đọc ở lần đầu: nó cho thấy `npm install`, rồi `npm run build`, rồi bước tải artifact lên. Thất bại ở đây gần như luôn là do monorepo root hoặc `baseDirectory`.

Khi thành công, ứng dụng được phục vụ tại:

```text
https://<branch>.<app-id>.amplifyapp.com
```

<!-- ẢNH 3 - SCREENSHOT.
     Amplify Console -> lịch sử triển khai / trang build, hiển thị đủ bốn giai đoạn
     (Provision, Build, Deploy, Verify) với dấu tích xanh, kèm tên nhánh.
     Nếu URL đã triển khai cũng lộ ra trong cùng khung hình thì càng tốt. -->

![Một lần triển khai Amplify thành công](/images/5-Workshop/5.9-Amplify-Frontend/amplify-build.png)

---

## 6. Kiểm chứng từ trình duyệt

Mở URL Amplify. Chính giao diện sẽ báo cho bạn biết nó đã được cấu hình đúng hay chưa - phần đầu trang hiển thị một trong hai trạng thái:

| Hiển thị | Ý nghĩa |
| --- | --- |
| `Connected through AWS API Gateway` | `VITE_API_BASE_URL` đã có mặt lúc build |
| `API URL is not configured` | Biến còn thiếu lúc bản build chạy |

Thông báo thứ hai nghĩa là biến được thêm vào *sau* khi build - hãy deploy lại.

Sau đó gửi một câu hỏi đa bước thật:

```text
Were Scott Derrickson and Ed Wood of the same nationality?
```

Một phản hồi thành công sẽ hiển thị câu trả lời, các nguồn hỗ trợ và thời gian đã tiêu tốn. Chỉ một thao tác đó thôi đã vận hành toàn bộ hệ thống:

```text
Trình duyệt
  → AWS Amplify (HTTPS)
  → Amazon API Gateway (HTTPS)
  → Amazon EC2 - FastAPI
  → Amazon S3 + Amazon S3 Vectors
  → Groq
  → câu trả lời + nguồn bằng chứng
```

{{% notice tip %}}
Nếu câu hỏi đầu tiên sau một quãng nghỉ bị timeout, đó là hiện tượng khởi động nguội đã mô tả ở chương 5.7, không phải lỗi frontend. Hãy gọi `POST /warmup` qua API Gateway, hoặc đơn giản là hỏi lại - câu thứ hai sẽ nhanh. Nên mở sẵn developer console của trình duyệt ở lần thử đầu tiên: lỗi **Mixed Content** và **CORS** hiện ra ở đó kèm giải thích chính xác, trong khi bản thân trang web chỉ báo một lỗi chung chung.
{{% /notice %}}

<!-- ẢNH 4 - SCREENSHOT.
     Ứng dụng đã triển khai, mở trên trình duyệt, sau khi hỏi câu hỏi kiểm thử.
     Phải thấy đồng thời: thanh địa chỉ với URL https:// của Amplify và biểu tượng ổ khóa,
     câu trả lời, và danh sách nguồn hỗ trợ.
     Đây là tấm ảnh quan trọng nhất của cả báo cáo - nó là bằng chứng cho tiêu chí
     "triển khai end-to-end" trong thang điểm. Chương 5.3 cũng tham chiếu đúng tấm này,
     nên chụp một lần rồi dùng cho cả hai chỗ. -->

![Ứng dụng đã triển khai đang trả lời một câu hỏi](/images/5-Workshop/5.9-Amplify-Frontend/deployed-app.png)

---

## 7. Chạy cục bộ trên cùng backend

Vẫn frontend đó chạy được ngay trên máy mà không phải đụng gì tới bản triển khai. Tạo file `frontend/.env.local`:

```env
VITE_API_BASE_URL=http://127.0.0.1:8000
```

```powershell
cd frontend
npm install
npm run dev
```

Cách này chạy được vì `http://localhost:5173` và `http://127.0.0.1:5173` vốn đã nằm trong danh sách CORS origin được phép của backend (chương 5.7). Cả hai trang đều là HTTP khi chạy cục bộ, nên Mixed Content không áp dụng - vấn đề đó chỉ tồn tại khi trang được phục vụ qua HTTPS.

File `.env.local` nằm trong `.gitignore` và phải giữ nguyên như vậy; giá trị của bản triển khai thuộc về Amplify Console, không thuộc về repository.

---

## 8. Các lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| Build hỏng ngay, báo không tìm thấy `package.json` | Chưa đặt monorepo root | Đặt app root là `frontend` |
| Build thành công nhưng trang trắng hoặc 404 | Sai `baseDirectory` | Dùng `dist`, không phải `build` |
| Đầu trang hiện `API URL is not configured` | Biến được thêm sau khi build | Deploy lại |
| `Mixed Content ... blocked` | `VITE_API_BASE_URL` trỏ vào địa chỉ HTTP của EC2 | Dùng URL HTTPS của API Gateway |
| `blocked by CORS policy` | Origin Amplify chưa được cho phép | Thêm vào CORS của API Gateway **và** tham số `cors-allow-origins` |
| Đổi biến mà không thấy tác dụng | Vite nhúng cứng giá trị lúc build | Redeploy this version |
| Câu hỏi đầu bị timeout, các câu sau bình thường | Backend khởi động nguội | Gọi `POST /warmup`, hoặc hỏi lại |
| `{"message":"Service Unavailable"}` | Truy vấn vượt quá timeout của API Gateway | Chương 5.8 mục 7 |

---

## 9. Kết quả

Việc triển khai đã hoàn tất. Người dùng mở một trang HTTPS, gõ vào một câu hỏi đa bước, và nhận về câu trả lời ngắn gọn kèm những tài liệu đã sinh ra nó - trong khi mọi tài nguyên AWS phía sau vẫn ở trạng thái riêng tư, ngoại trừ hai endpoint HTTPS được quản lý.

Chương 5.10 sẽ xác thực toàn bộ đường đi một cách hệ thống, còn chương 5.11 đo xem hệ thống trả lời tốt đến đâu.

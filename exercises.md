# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hồ Trọng Hảo  Mã học viên: 2A202601358

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> *Câu trả lời của bạn:* Khi deploy lên cloud (như Render hoặc Cloud Run) mà quên cấu hình biến môi trường `API_TOKEN`. Nếu để mặc định là "changeme", service vẫn khởi động thành công (dù liveness probe báo xanh) và người ngoài có thể gọi endpoint `/chat` thoải mái với token "changeme", khiến bạn bị lộ API và mất tiền oan cho LLM provider. Nếu fail fast, service sẽ crash ngay lập tức, quá trình deploy sẽ bị báo lỗi (failed), giúp bạn phát hiện ra ngay cấu hình thiếu và ngăn chặn việc service chạy với lỗ hổng bảo mật chết người.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> *Câu trả lời của bạn:* Dòng log: `{"timestamp": "2026-08-10T17:15:00.000Z", "event": "chat_completed", "client_id": "sv-test", "prompt_tokens": 15, "completion_tokens": 20, "usd_cost": 0.005}`
> Hai việc làm được: 
> 1. Có thể đẩy vào hệ thống như Elasticsearch/Kibana hoặc Datadog để filter, group, và tính tổng chi phí (sum `usd_cost`) hoặc đếm số request theo từng `client_id` (vẽ biểu đồ rất dễ). 
> 2. Có thể set alert (cảnh báo tự động) bằng code nếu một `client_id` có `usd_cost` cao đột biến trong thời gian ngắn. `print("đã trả lời xong")` chỉ là chuỗi text, máy không thể parse và tính toán tự động được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~ 1000 MB |
| Multi-stage | ~ 150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> *Câu trả lời của bạn:* Phần chênh lệch (hơn 800MB) bao gồm các công cụ build (gcc, make), pip cache, header files, và các thư viện hệ thống cần thiết chỉ để compile các package C-extensions. Khi dùng multi-stage, stage 2 (production) chỉ copy đúng những file đã được build sẵn (thư mục cài đặt `site-packages`) từ stage 1 (builder) sang, bỏ lại toàn bộ rác và build tools ở stage 1, giúp image cuối cùng cực kỳ nhỏ gọn và an toàn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> *Câu trả lời của bạn:* Mặc định, khi sửa code Python (như `app/main.py`), các layer `COPY requirements.txt` và `RUN pip install` vẫn lấy từ cache vì file `requirements.txt` không bị đổi (chỉ tốn <1s để build). Nếu đưa `COPY . .` lên trước `RUN pip install`, bất kỳ thay đổi nào trong source code dù là nhỏ nhất (như thêm 1 khoảng trắng) cũng làm layer `COPY . .` bị mất cache. Kết quả là Docker phải tải và chạy lại toàn bộ `pip install` từ đầu, làm tăng thời gian build từ vài giây lên vài phút mỗi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> *Câu trả lời của bạn:* Nếu code Python có lỗ hổng (ví dụ RCE hoặc dính thư viện bên thứ 3 chứa mã độc), hacker có thể gửi payload chạy lệnh shell độc hại. Nếu container chạy bằng root, script đó sẽ có quyền root bên trong container, hacker có thể cài thêm malware, đọc biến môi trường bí mật, cài crontab, hoặc khai thác lỗ hổng Linux kernel để thoát ra (container breakout) chiếm quyền máy chủ host. Lệnh `USER appuser` cắt đứt chuỗi này: tiến trình bị hạ cấp xuống quyền user thường, hacker dù có chạy được mã độc cũng không có quyền cài package, không thể ghi đè file hệ thống, bị giam lỏng.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> *Câu trả lời của bạn:* Header `WWW-Authenticate: Bearer` theo chuẩn RFC 6750 giúp báo cho client/trình duyệt biết cơ chế xác thực nào đang được yêu cầu, để trình duyệt có thể tự format lại token hoặc nhắc người dùng. Việc trả **cùng một thông báo lỗi** (invalid or missing) cho mọi trường hợp là để tránh rò rỉ thông tin (Information Disclosure) cho kẻ tấn công. Nếu ta chỉ rõ "Thiếu token", "Sai scheme", hay "Sai vài ký tự ở cuối", hacker có thể lợi dụng phản hồi đó làm căn cứ để chạy Brute-force hoặc Timing Attack dò mật khẩu từ từ.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> *Câu trả lời của bạn:* Với `capacity=10`, dù im lặng 10 phút, client cũng chỉ gửi được tối đa **10 request** liên tiếp vì rổ chứa đã đầy và không thể chứa thêm token nào. Nếu bỏ hàm `min(capacity, ...)`, số token sẽ tích lũy vô hạn. Sau 10 phút im lặng (nạp thêm 10 * 10 = 100), client sẽ dồn được 100 token, cho phép gọi 100 request liên tiếp trong vỏn vẹn vài giây. Việc này phá vỡ khả năng chống "burst traffic", làm sập server do bị dội một lượng tải quá lớn cục bộ.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> *Câu trả lời của bạn:* Với $30/tháng, thiệt hại tối đa là mất trắng **$30** chỉ trong 1 đêm do bot phá hoại chạy vòng lặp, và service sẽ tự động khóa cứng cho đến tận... tháng sau, làm tê liệt người dùng thực sự trong suốt 29 ngày còn lại. Với $1/ngày, thiệt hại tối đa chỉ là **$1**, và quan trọng nhất là service sẽ tự động được "hồi sinh" ngay khi đồng hồ bước sang 0h00 của ngày tiếp theo. Thiết kế $1/ngày giúp khoanh vùng thiệt hại "cháy nổ" rất tốt và giảm thiểu thời gian downtime tối đa.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Câu trả lời của bạn:* Trình tự sự kiện: 1. Redis chết, `/healthz` (đã bị gộp chung logic) check Redis và trả về 503 Fail. 2. Orchestrator (K8s/Docker/Cloud) gọi probe liveness thấy `healthz` rớt, liền kết luận container đang bị treo và cần kill (SIGKILL). 3. Cả 3 container đang chạy bị kill và khởi động lại. 4. Container mới boot lên, lại gọi `healthz`, Redis lúc này vẫn chưa sửa xong -> lại báo 503 -> lại bị kill tiếp. Hệ quả là cả cụm app sụp đổ và vào vòng lặp chết chóc (CrashLoopBackOff) chỉ vì đứt kết nối mạng một lúc, thay vì đơn giản là từ chối traffic (ngắt load balancer) và chờ đợi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn:* Lỗi gặp phải: Connection Refused từ Redis. Khi app chạy trên Render, các API `/chat` hoặc `/readyz` báo lỗi 503 và log in ra `ConnectionError`. 
> Nguyên nhân: Tôi xem log trên dashboard thì thấy app vẫn đang cố gọi kết nối tới `redis://localhost:6379/0` (giá trị mặc định trong code) thay vì chạy vào database thật, vì Cloud không có Redis cài sẵn ở localhost.
> Sửa lỗi: Truy cập vào trang quản lý biến môi trường của Render, tạo thêm một biến `REDIS_URL` với giá trị là `Internal Database URL` lấy từ instance Redis đã tạo. App tự khởi động lại và kết nối thành công.

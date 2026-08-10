# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thành Long  Mã học viên: 2A202601536

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Giả sử khi deploy ứng dụng lên Cloud mà bạn quên cấu hình biến môi trường `API_TOKEN`. 
- Nếu để giá trị mặc định `"changeme"`: Ứng dụng vẫn khởi động bình thường. Kẻ xấu hoặc bot trên mạng rà quét các IP/Domain công khai sẽ thử token mặc định `"changeme"` và gọi thành công API dịch vụ LLM của bạn, gây tiêu tốn hết ngân sách tài khoản API mà bạn không hề hay biết cho đến khi nhận hóa đơn.
- Ngược lại với "fail fast" (không mặc định): Ứng dụng sẽ văng lỗi `ValidationError` và ngắt ngay lúc khởi động (`Application startup failed`). Bạn phát hiện ra ngay trên Dashboard của Cloud và bổ sung secret trước khi ứng dụng tiếp nhận bất kỳ request nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:46:09.123456+00:00", "client_id": "sv01", "prompt_tokens": 1, "completion_tokens": 35, "usd_cost": 0.00002115}`

Hai việc làm được với log JSON:
1. **Lọc và tạo Dashboard cảnh báo tự động trên Cloud (CloudWatch/Google Cloud Logging/Datadog)**: Công cụ theo dõi có thể tự động parse trường `"severity": "ERROR"` hoặc lọc tất cả log có `"usd_cost" > 0.01` để tự động gửi cảnh báo qua Slack/Telegram.
2. **Truy vết và thống kê số liệu (Analytics & Audit)**: Dễ dàng bóc tách các trường `"client_id"`, `"prompt_tokens"`, `"usd_cost"` để tính tổng chi phí theo ngày hoặc thống kê khách hàng nào dùng nhiều nhất bằng SQL/Log Explorer, điều mà câu print text đơn thuần không thể làm được.

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
| 1 stage (bản đầu) | ~1.8 GB |
| Multi-stage | 209 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~1.6 GB) bao gồm:
- Base image đầy đủ `python:3.11` chứa sẵn các bộ biên dịch C/C++ (`gcc`, `g++`, `make`), các thư viện phát triển header system (`python-dev`), và các công cụ hệ thống không cần thiết khi chạy ứng dụng.
- Bản Multi-stage sử dụng `python:3.11-slim` chỉ chứa bộ runtime Python tối giản và loại bỏ hoàn toàn các layer build/dependencies trung gian trong quá trình tạo image runtime cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Layer được dùng lại từ cache**: Layer cài đặt thư viện (`COPY requirements.txt .` và `RUN pip install ...`) được giữ nguyên từ cache vì `requirements.txt` không hề thay đổi.
- **Layer phải chạy lại**: Layer `COPY . .` và các lệnh phía sau (chuyển user, healthcheck, cmd) phải chạy lại vì file `app/main.py` đã bị sửa.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`**: Mỗi lần sửa dù chỉ 1 ký tự code trong `app/main.py`, Docker sẽ coi layer `COPY . .` đã thay đổi và vô hiệu hóa cache của tất cả các layer đứng sau nó. Kết quả là Docker phải tải và chạy lại toàn bộ lệnh `RUN pip install` tốn rất nhiều thời gian (vài phút).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python chứa 1 lỗ hổng Remote Code Execution (RCE) hoặc Command Injection.
2. Kẻ tấn công gửi request chứa payload độc hại để thực thi lệnh hệ thống bên trong container.
3. Nếu container đang chạy bằng user `root`, tiến trình của kẻ tấn công có quyền `root` bên trong container.
4. Kẻ tấn công có thể khai thác tiếp các lỗ hổng container escape (như mounting docker socket, cgroups exploit...) để chiếm toàn bộ quyền `root` của máy host chứa container đó.

Lệnh `USER appuser` cắt đứt chuỗi ở bước 3: Nó giới hạn quyền hạn của container ở một user thường không có đặc quyền (`unprivileged user`). Ngay cả khi kẻ tấn công thực thi được lệnh độc hại trong container, chúng cũng không có quyền ghi các file hệ thống quan trọng hay thực thi các lệnh root để leo thang đặc quyền ra máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- **Vì sao 401 phải kèm `WWW-Authenticate: Bearer`**: Theo chuẩn RFC 6750 / HTTP Protocol, response 401 bắt buộc phải thông báo cho client (trình duyệt, SDK, API Client) biết ứng dụng sử dụng phương thức xác thực nào (ở đây là `Bearer` token) để client biết cách gửi lại request đúng chuẩn (ví dụ thêm header `Authorization: Bearer <token>`).
- **Vì sao trả CÙNG MỘT thông báo lỗi**: Để bảo mật chống dò quét (Information Leakage). Nếu phân biệt lỗi (như "Token không tồn tại", "Sai scheme", "Sai chữ cái đầu"), kẻ tấn công có thể lợi dụng phản hồi này để dò đoán hệ thống (ví dụ biết được username/token nào tồn tại hay chưa). Trả về cùng một thông báo `invalid or missing bearer token` giúp bảo mật tối đa.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- **Số request gửi được trước khi bị 429**: Client im lặng 10 phút xô sẽ tích đầy trần `capacity = 10` token. Do đó client gửi được liên tiếp **10 request** (request thứ 11 sẽ bị từ chối 429).
- **Nếu bỏ `min(capacity, ...)` trong `available()`**:
  Sau 10 phút (600 giây), với tốc độ `refill_per_minute = 10` (10/60 = 1/6 token/giây), xô sẽ nạp thêm 600 * (1/6) = 100 token.
  Số request gửi được sẽ thành **110 request** (10 token ban đầu + 100 token nạp thêm). Việc thiếu `min()` khiến client im lặng lâu tích lũy token vô hạn và có thể xả một đợt bùng nổ (burst attack) làm sập hệ thống.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- **$30/tháng**: Sự cố lúc 2h sáng gọi liên tục có thể đốt sạch toàn bộ $30 ngân sách của cả tháng chỉ trong vài giờ. Thiệt hại tối đa là **$30**. Service sẽ bị khóa và **chỉ tự phục hồi sang tháng sau** (hoặc phải có người vào reset thủ công).
- **$1/ngày**: Sự cố lúc 2h sáng chỉ có thể đốt tối đa **$1** của ngày hôm đó (chặn 402 khi chạm $1). Thiệt hại tối đa bị khoanh vùng xuống **$1** (1/30). Service sẽ **tự động phục hồi vào 00:00 UTC sáng ngày hôm sau** khi nhãn ngày chuyển sang ngày mới (`spend:<client>:<YYYY-MM-DD>`).

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Chuỗi sự kiện xảy ra:
1. Redis mất kết nối 30 giây.
2. Cả 3 container đều chạy liveness probe (đã gộp với readyz) kiểm tra Redis -> Cả 3 container đồng loạt trả về `unhealthy`.
3. Orchestrator (Docker/K8s) thấy cả 3 container `unhealthy` nên tiến hành **restart cả 3 container cùng lúc**.
4. Khi các container vừa bật lại, Redis vẫn chưa quay lại (đang trong khoảng 30s sự cố) -> Container tiếp tục báo `unhealthy` và lại bị restart liên tục (CrashLoopBackOff).
5. Khi Redis phục hồi sau 30s, toàn bộ cụm container vẫn đang ở trạng thái bị sập/khởi động lại, dẫn đến toàn hệ thống sập hoàn toàn thay vì chỉ tạm hoãn nhận traffic.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**: `HTTP/2 500 Internal Server Error` tại endpoint `/readyz` khi deploy ứng dụng lên Railway.
- **Cách tìm ra nguyên nhân**: Kiểm tra lệnh `curl -i https://<app-url>/readyz` trả về 500. Mở tab **Variables** và **Logs** trên Railway Dashboard, phát hiện ứng dụng chưa được gắn kết nối tới biến `REDIS_URL` của Redis Cloud mà đang chạy mặc định trỏ về `localhost:6379`.
- **Cách sửa**: Tạo thêm service **Redis Database** trên Railway, sau đó trong Variables của service `chat` thêm biến `REDIS_URL=${{Redis.REDIS_URL}}` và Redeploy lại app. Endpoint `/readyz` lập tức trả về `200 {"status":"ready","redis":true}`.

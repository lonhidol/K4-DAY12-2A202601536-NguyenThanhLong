# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Thành Long |
| Mã học viên | 2A202601536 |
| Repo | https://github.com/lonhidol/K4-DAY12-2A202601536-NguyenThanhLong |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis  |

| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
(.venv) ryu@MacBook-Air-cua-Ryu K4-Day12-Cloud-Services-And-Deployment % curl -i https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/healthz

HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 09:18:01 GMT
server: railway-hikari
x-railway-request-id: PBlmSfbvSLiAEKmdjq4OvQ
content-length: 64
x-hikari-trace: sin1.tr00
x-railway-edge: sin1

{"status":"ok","service":"day12-chat-service","version":"1.0.0"}% 

(.venv) ryu@MacBook-Air-cua-Ryu K4-Day12-Cloud-Services-And-Deployment % curl -i https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/readyz

HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 09:23:42 GMT
server: railway-hikari
x-railway-request-id: Qe4M6L0-RUmjw9P8nPRhug
content-length: 31
x-hikari-trace: sin1.tr00
x-railway-edge: sin1

{"status":"ready","redis":true}%   

(.venv) ryu@MacBook-Air-cua-Ryu K4-Day12-Cloud-Services-And-Deployment % curl -i -X POST https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'
HTTP/2 401 
content-type: application/json
date: Mon, 10 Aug 2026 09:24:25 GMT
server: railway-hikari
www-authenticate: Bearer
x-railway-request-id: vp_qi6ITR3uWFjiyWUN5dQ
content-length: 44
x-hikari-trace: sin1.nzn2
x-railway-edge: sin1

{"detail":"invalid or missing bearer token"}%

(.venv) ryu@MacBook-Air-cua-Ryu K4-Day12-Cloud-Services-And-Deployment % curl -i -X POST https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'
HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 09:26:38 GMT
server: railway-hikari
x-railway-request-id: 8JVp4wPAThqMFbURljLL4A
content-length: 288
x-hikari-trace: sin1.d1nj
x-railway-edge: sin1
vary: accept-encoding

{"reply":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","client_id":"sv-test","turns_before":0,"usd_cost":2.145e-05,"usage":{"prompt":3,"completion":35}}%                             

(.venv) ryu@MacBook-Air-cua-Ryu K4-Day12-Cloud-Services-And-Deployment % for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://k4-day12-2a202601536-nguyenthanhlong-production.up.railway.app/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429 

```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl


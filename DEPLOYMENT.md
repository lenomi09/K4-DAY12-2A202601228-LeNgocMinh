# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Lê Ngọc Minh |
| Mã học viên | 2A202601228 |
| Repo | https://github.com/lenomi09/K4-DAY12-2A202601228-LeNgocMinh |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://chat-production-eadb.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway (kết nối nội bộ qua `redis.railway.internal`) |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

```
$ curl -i https://chat-production-eadb.up.railway.app/health
HTTP/1.1 200 OK
content-type: application/json
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

$ curl -i https://chat-production-eadb.up.railway.app/ready
HTTP/1.1 200 OK
content-type: application/json
{"status":"ready","redis":true}

$ curl -i -X POST https://chat-production-eadb.up.railway.app/chat \
  -H "Content-Type: application/json" -d '{"message":"Hello"}'
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}

$ curl -i -X POST https://chat-production-eadb.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'
HTTP/1.1 200 OK
{"reply":"Với Deploy là gì, cách làm phổ biến trong production là đặt một lớp
gateway phía trước để lo authentication, rate limiting và bảo vệ chi phí.
(Mình đang nhớ 2 lượt trao đổi trước đó.)","client_id":"sv-test",
"turns_before":2,"usd_cost":3.315e-05,"usage":{"prompt":41,"completion":45}}

$ for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code} " -X POST \
  https://chat-production-eadb.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" -H "X-Client-Id: sv-test" \
  -d '{"message":"test"}'; done
200 200 200 200 200 200 200 200 200 429 429 429 429 429 200
```

Ghi chú: client `sv-test` đã dùng 1 token ở lệnh gọi /chat phía trên nên xô chỉ
còn 9 token khi bắt đầu vòng lặp 15 request — đúng với `BUCKET_CAPACITY=10`.
9 request đầu qua (200), 5 request tiếp bị chặn (429) vì xô cạn, request thứ
15 lại 200 vì `REFILL_PER_MINUTE=10` đã kịp nạp lại ≥1 token.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

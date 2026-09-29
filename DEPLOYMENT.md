# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Châu Tùng Dương |
| Mã học viên | 2A202602822 |
| Repo | https://github.com/ChauTungDuong/K4-L3B-DAY12-Chau-Tung-Duong-2A202602822-Cloud-Service-And-Deployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3b-day12-chau-tung-duong-2a202602822-cloud-s-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway (tự nối qua Add Reference) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Public URL đã dùng trong các lệnh dưới đây:

```bash
URL=https://k4-l3b-day12-chau-tung-duong-2a202602822-cloud-s-production.up.railway.app
```

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i "$URL/health"

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i "$URL/ready"

# 3. Không có API key — mong đợi 401
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST "$URL/ask" \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Kiểm tra ngày 2026-09-29 qua HTTPS:

```text
$ curl -i "$URL/health"
HTTP/1.1 200 OK
Content-Type: application/json
Server: railway-hikari

{"status":"ok","service":"day12-agent","version":"1.0.0"}

$ curl -i "$URL/ready"
HTTP/1.1 200 OK
Content-Type: application/json
Server: railway-hikari

{"status":"ready","redis":true}

$ curl -i -X POST "$URL/ask" -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Server: railway-hikari

{"detail":"invalid or missing API key"}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang Railway của service, thấy tên service,
  trạng thái deploy thành công và domain công khai.
- `screenshots/health.png` — terminal hoặc trình duyệt hiển thị request HTTPS
  tới `/health`, HTTP 200 và JSON `status: ok`.


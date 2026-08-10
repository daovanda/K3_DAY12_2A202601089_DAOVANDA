# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đào Văn Đa |
| Mã học viên | 2A202601089 |
| Repo | https://github.com/daovanda/K3_DAY12_2A202601089_DAOVANDA |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-fbf9.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Chỉ ghi tên biến và nguồn giá trị; giá trị secret không nằm trong repository.

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán; container đọc biến lúc khởi động |
| `AGENT_API_KEY` | ✅ | Secret đặt bằng Railway CLI qua stdin |
| `REDIS_URL` | ✅ | Reference variable `${{Redis.REDIS_URL}}` từ Redis managed service |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
URL=https://day12-agent-production-fbf9.up.railway.app

curl -i "$URL/health"
curl -i "$URL/ready"

curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

```text
GET  /health                  -> 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET  /ready                   -> 200 {"status":"ready","redis":true}
POST /ask (không API key)     -> 401 {"detail":"invalid or missing API key"}
POST /ask (API key hợp lệ)    -> 200, có answer và user_id

Rate limit với cùng một X-User-Id, gọi liên tiếp 15 lần:
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png` — Railway project gồm `day12-agent` và Redis.
- `screenshots/health.png` — kết quả gọi public endpoint `/health`.

## Sự Cố Đã Xử Lý Khi Deploy

Domain Railway ban đầu chưa có target port nên request có lúc trả 502. Runtime log
cho thấy Uvicorn lắng nghe trên cổng Railway cấp là `8080`. Sau khi cập nhật domain
trỏ tới target port `8080`, `/health`, `/ready`, xác thực và rate limit đều hoạt động.

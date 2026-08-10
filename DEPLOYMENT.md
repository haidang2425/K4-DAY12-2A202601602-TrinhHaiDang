# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trịnh Hải Đăng |
| Mã học viên | 2A202601602 |
| Repo | https://github.com/<tên-github-của-bạn>/K4-DAY12-2A202601602-TrinhHaiDang |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | — (chưa deploy lên cloud, xem phần "Phương Án Dự Phòng" bên dưới) |
| Platform | Dự kiến Railway — hiện đang dùng phương án dự phòng Local Fallback (Docker Compose), chưa deploy thật lên Railway/Render |
| Ngày deploy | chưa deploy — cần cập nhật ngày thật sau khi lên cloud |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | — | chưa deploy cloud; local dùng mặc định 8000 |
| `API_TOKEN` | ✅ | đặt trong `.env` cục bộ, không nằm trong repo |
| `REDIS_URL` | ✅ | service `redis` trong `docker-compose.yml` (`redis://redis:6379/0`) |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/readyz

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

Chạy thật ở `http://localhost:8000` (uvicorn cục bộ, `REDIS_URL=fake://`) —
thay bằng output của `docker compose` + Public URL thật khi bạn deploy:

```
$ curl -i http://localhost:8000/healthz
HTTP/1.1 200 OK
content-type: application/json
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

$ curl -i http://localhost:8000/readyz
HTTP/1.1 200 OK
content-type: application/json
{"status":"ready","redis":true}

$ curl -i -X POST http://localhost:8000/chat -H "Content-Type: application/json" -d '{"message":"Hello"}'
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}

$ curl -i -X POST http://localhost:8000/chat -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" -H "X-Client-Id: sv01" -d '{"message":"Docker la gi"}'
HTTP/1.1 200 OK
{"reply":"Ngắn gọn: Docker la gi phụ thuộc vào ba yếu tố...","client_id":"sv01","turns_before":0,"usd_cost":2.265e-05,"usage":{"prompt":3,"completion":37}}

$ for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code} " -X POST http://localhost:8000/chat \
    -H "Content-Type: application/json" -H "Authorization: Bearer $API_TOKEN" -H "X-Client-Id: sv-rate" \
    -d '{"message":"test"}'; done; echo
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform (hoặc `docker compose ps` nếu dùng fallback)
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

**Chưa có ảnh nào trong `screenshots/` — bạn cần tự chụp và thêm vào trước khi
nộp bài**, vì việc này đòi hỏi thao tác trên máy/trình duyệt của bạn.

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
Chưa deploy lên Railway/Render: cần tài khoản cloud thật (đăng nhập, thêm
thẻ/free-tier) — bước này phải làm thủ công, chưa thực hiện tại thời điểm
soạn file này. Đã chuẩn bị sẵn railway.toml/render.yaml và toàn bộ code đã
qua CP1–CP4 (75/75 test tương ứng pass). Việc còn lại để hoàn tất CP5:
  1. `railway login` (hoặc tạo tài khoản Render), deploy bằng `railway up`
     hoặc Blueprint từ render.yaml.
  2. Set các biến môi trường liệt kê ở trên trên dashboard.
  3. Điền Public URL thật vào mục "Service" phía trên.
  4. Chụp màn hình dashboard + /healthz vào screenshots/.
Nếu không kịp deploy cloud: giữ nguyên phương án dự phòng — chạy
`docker compose up -d`, chụp `docker compose ps` và kết quả curl vào
screenshots/, CP5 tối đa 60% điểm theo đúng quy định của lab.
```

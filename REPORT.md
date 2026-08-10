# Báo cáo tình trạng — K4-DAY12-2A202601602-TrinhHaiDang

_Cập nhật: 10/08/2026 — trước khi bắt đầu code._

## Tóm tắt
Repo mới ở trạng thái template gốc (1 commit gốc, 0 commit của học viên).
Tiến độ hiện tại: **0/100 điểm** theo `grade.py` (dự đoán) — chưa file nào
trong `app/` được cài đặt. Đây là báo cáo liệt kê chính xác việc còn thiếu,
dùng làm checklist đối chiếu khi làm xong từng phần.

## Chi tiết theo file

| File | Trạng thái | Việc cần làm |
|---|---|---|
| `app/config.py` | TODO | Khai báo đủ 7 trường trong `Settings`; `api_token: str` không có giá trị mặc định |
| `app/logging_utils.py` | TODO | Cài `emit()` in JSON một dòng, `severity` viết hoa, `ts` ISO, `ensure_ascii=False` |
| `app/main.py` | TODO (3 endpoint `raise NotImplementedError`) | `/healthz` (CP1/CP4), `/readyz` (CP4), `/chat` (CP3/CP4) |
| `app/auth.py` | TODO | `verify_bearer_token()` — tách scheme/token, `secrets.compare_digest`, 401 + `WWW-Authenticate` |
| `app/rate_limiter.py` | TODO | `TokenBucket.available()` (có `min(capacity, ...)`) và `consume()` |
| `app/cost_guard.py` | TODO | `spent()`, `check()` (402), `record()` (`incrbyfloat` + TTL ngày) |
| `app/store.py` | TODO | `ping()` (nuốt exception), `add_turn()` (`rpush`/`ltrim`/`expire`), `history()` |
| `app/lifecycle.py` | TODO | `arm()` đăng ký `SIGTERM`/`SIGINT` giữ handler cũ, `start_draining()` |
| `Dockerfile` | Bản mẫu 1-stage | Chuyển multi-stage, `slim`, `USER` thường, `HEALTHCHECK`, đọc `$PORT` |
| `docker-compose.yml` | Thiếu service `chat` | Thêm service `chat` với env, healthcheck, `depends_on: redis` |
| `.dockerignore` | Chỉ có `.git`, `.gitignore` | Bổ sung `.env`, `__pycache__`, `.venv` |
| `.env` | Chưa tồn tại | `cp .env.example .env` rồi điền `API_TOKEN` riêng |
| `exercises.md` | 0/10 câu trả lời | Trả lời sau khi làm xong checkpoint tương ứng |
| `DEPLOYMENT.md` | Placeholder | Điền sau khi deploy thật ở CP5 |
| `.github/workflows/` | Chưa có | Bonus CI/CD — chỉ làm sau khi CP1–CP5 xong |

## Thứ tự ưu tiên (theo cách chấm điểm)
1. **CP1** (15đ) — Config + logging + `/healthz`. Nền tảng cho mọi phần sau.
2. **CP2** (15đ) — Docker multi-stage + compose.
3. **CP3** (20đ) — Auth + rate limit + cost guard + `/chat`. Điểm cao nhất, nên ưu tiên thời gian.
4. **CP4** (20đ) — Redis store + `/readyz` + graceful shutdown. Cũng 20đ, không kém CP3.
5. **CP5** (15đ) — Deploy thật lên Railway/Render; có phương án dự phòng (`LOCAL_FALLBACK`) nếu kẹt, nhưng tối đa chỉ 9/15đ.
6. **`exercises.md`** (15đ) — chấm theo số câu trả lời, không tốn nhiều thời gian nếu làm ngay sau mỗi CP.
7. **Bonus CI/CD** (+10đ) — chỉ làm khi còn dư thời gian, không đánh đổi lấy CP1–CP5.

## Việc cần kiểm tra sau khi sửa mỗi phần
```bash
pytest tests/test_cp1.py -v
pytest tests/test_cp2.py -v
pytest tests/test_cp3.py -v
pytest tests/test_cp4.py -v
pytest tests/test_cp5.py -v
python grade.py
```

## Rủi ro / lưu ý đã ghi nhận từ đề bài
- Không commit `.env` hoặc token thật — kiểm tra bằng
  `git ls-files | grep "^\.env$"` trước mỗi lần push.
- `REDIS_URL` trong Docker Compose phải là `redis://redis:6379/0` (tên
  service là hostname trong mạng compose), không phải `localhost`.
- `/healthz` tuyệt đối không được gọi Redis; `/readyz` thì được — nhầm lẫn
  hai endpoint này là lỗi phổ biến nhất và bị trừ điểm trực tiếp ở CP4.
- `/chat` phải chặn (rate limit, budget) **trước khi** gọi LLM giả, không
  phải sau.
- Trên cloud: không tự đặt `PORT` — platform tự gán, app chỉ cần đọc
  `${PORT:-8000}`.
- Repo đã đúng định dạng tên `K4-DAY12-2A202601602-TrinhHaiDang` — không có
  rủi ro trừ điểm ở mục này.

## Đánh giá nhanh mức độ sẵn sàng
- Môi trường/tài liệu lab: đầy đủ, không thiếu file nào cần có sẵn.
- Code: 0% hoàn thành — toàn bộ 8 file trong `app/` cần viết từ đầu theo
  đúng đặc tả trong `LAB_GUIDE.md`.
- Deploy: chưa bắt đầu, cần tài khoản Railway/Render trước CP5.
- Không có blocker nào từ phía môi trường (Python, Docker, Git đều sẵn sàng
  theo yêu cầu README) — việc còn lại thuần là code theo checklist ở
  `PLAN.md`.
</content>

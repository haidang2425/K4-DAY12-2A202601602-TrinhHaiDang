# Kế hoạch làm việc — K4-DAY12-2A202601602-TrinhHaiDang

## Mục tiêu
Hoàn thành lab "Hạ Tầng Cloud & Deployment" theo đúng 5 checkpoint (CP1–CP5) +
`exercises.md`, đạt tối thiểu 75/100 điểm, không phạm lỗi trừ điểm (đặt sai
tên repo, lộ `.env`/token, một commit duy nhất phút chót).

## Tình trạng lúc lập kế hoạch (10/08/2026)
- Repo mới có 1 commit gốc từ template lab (`de7b16e` → `cbbf765`), **chưa có
  commit nào của học viên**.
- Toàn bộ file trong `app/` (`config.py`, `main.py`, `auth.py`,
  `rate_limiter.py`, `cost_guard.py`, `store.py`, `lifecycle.py`) vẫn ở dạng
  TODO/`NotImplementedError` — chưa code dòng nào.
- `Dockerfile` vẫn là bản 1-stage, chạy root, chưa đọc `$PORT`, chưa có
  `HEALTHCHECK` (đúng như đề bài cho sẵn).
- `docker-compose.yml` chưa có service `chat`.
- `.dockerignore` mới có `.git`, `.gitignore` — thiếu `.env`, `__pycache__`,
  `.venv`.
- Chưa có file `.env` (chưa copy từ `.env.example`).
- Chưa có `.github/workflows/` (bonus CI/CD chưa bắt đầu).
- `exercises.md` — 10 câu đều chưa trả lời.
- `DEPLOYMENT.md` — vẫn là placeholder, chưa deploy.

→ Đây là điểm xuất phát 0%, khớp với những gì `python grade.py` sẽ báo lúc này.

## Luồng làm việc đề xuất (bám theo lịch trình gốc của README/LAB_GUIDE)

### CP0 — Setup
- [ ] Tạo venv Python 3.11, `pip install -r requirements.txt`.
- [ ] `cp .env.example .env`, sinh `API_TOKEN` riêng bằng
      `python -c "import secrets; print(secrets.token_urlsafe(32))"`.
- [ ] Bật Redis: `docker compose up -d redis` (hoặc tạm `REDIS_URL=fake://`
      nếu chưa cài Docker được ngay).
- [ ] Xác nhận môi trường chạy được: `pytest tests/ -v -m "not docker"`
      (rớt gần hết là **đúng** ở bước này).
- [ ] Commit "Checkpoint 0".

### CP1 — 12-Factor Config, Health & Logging (15đ)
- [ ] `app/config.py`: khai báo 7 trường trong `Settings`
      (`port`, `api_token` **không mặc định**, `redis_url`, `bucket_capacity`,
      `refill_per_minute`, `daily_budget_usd`, `log_level`).
- [ ] `app/logging_utils.py`: `emit()` in **một dòng JSON** ra stdout, khóa
      `severity` viết hoa, `ts` ISO 8601, `ensure_ascii=False` cho tiếng Việt.
- [ ] `app/main.py` → `/healthz`: 200 bình thường, 503 khi
      `shutdown_guard.draining`; **không** được đụng Redis.
- [ ] `pytest tests/test_cp1.py -v` xanh hết.
- [ ] Commit.

### CP2 — Docker (15đ)
- [ ] `Dockerfile`: multi-stage (`builder` → `runtime`), base `python:3.11-slim`,
      `COPY requirements.txt` + `pip install` trước `COPY app`, `USER appuser`
      (uid thường), `HEALTHCHECK` gọi `/healthz`, bind `0.0.0.0`, đọc
      `${PORT:-8000}`.
- [ ] `.dockerignore`: bổ sung `.env`, `__pycache__`, `.venv` (giữ nguyên
      `.git`, `.gitignore`).
- [ ] `docker-compose.yml`: thêm service `chat` — `build: .`, map `8000:8000`,
      `depends_on: redis`, `environment: API_TOKEN=${API_TOKEN}`,
      `REDIS_URL=redis://redis:6379/0`, healthcheck.
- [ ] Build thử, ghi lại dung lượng image (< 400MB) cho câu 3 `exercises.md`.
- [ ] `pytest tests/test_cp2.py -v` xanh hết.
- [ ] Commit.

### CP3 — API Security (20đ)
- [ ] `app/auth.py`: tách `scheme`/`token` từ header `Authorization`, so
      sánh token bằng `secrets.compare_digest`, 401 kèm
      `WWW-Authenticate: Bearer`, thông báo lỗi giống nhau cho mọi trường hợp.
- [ ] `app/rate_limiter.py`: token bucket — `available()` refill theo thời
      gian có `min(capacity, ...)`, `consume()` trừ 1 token + ghi lại `ts`.
- [ ] `app/cost_guard.py`: `spent()` (trả `0.0` khi key rỗng), `check()` (402
      khi vượt), `record()` (`incrbyfloat` + TTL theo ngày).
- [ ] `app/main.py` → `/chat`: đúng thứ tự
      `verify_bearer_token → bucket.consume → guard.check → store.history →
      generate_reply → store.add_turn ×2 → guard.record → emit → response`.
- [ ] `pytest tests/test_cp3.py -v` xanh hết.
- [ ] Commit.

### CP4 — Scaling & Reliability (20đ)
- [ ] `app/store.py`: state hoàn toàn trong Redis (không dict toàn cục),
      `ping()` nuốt exception trả `False`, `add_turn`/`history` dùng
      `rpush`/`ltrim(-N,-1)`/`expire`.
- [ ] `app/main.py` → `/readyz`: draining → 503, Redis chết → 503, ngược lại
      200 (khác `/healthz` ở chỗ có kiểm tra dependency).
- [ ] `app/lifecycle.py`: `arm()` đăng ký `SIGTERM`/`SIGINT` (nhớ lưu handler
      cũ của uvicorn), `start_draining()` chỉ bật cờ rồi gọi lại handler cũ.
- [ ] Test với `docker compose up -d --scale chat=3` — `turns_before` phải
      tăng dần dù đổi container.
- [ ] `pytest tests/test_cp4.py -v` xanh hết.
- [ ] Commit.

### CP5 — Deploy thật (15đ)
- [ ] Chọn Railway (dễ nhất) hoặc Render.
- [ ] Set biến môi trường trên platform: `API_TOKEN`, `BUCKET_CAPACITY`,
      `REFILL_PER_MINUTE`, `DAILY_BUDGET_USD`, `LOG_LEVEL` (không đặt `PORT`).
- [ ] Deploy, lấy Public URL, kiểm tra `/healthz`, `/readyz`, `/chat` (401 khi
      thiếu token, 200 khi có).
- [ ] Điền `DEPLOYMENT.md`: URL, platform, danh sách **tên** biến (không dán
      giá trị token), output các lệnh curl, ảnh `screenshots/dashboard.png`
      và `screenshots/healthz.png`.
- [ ] (Điểm cộng) thêm `DEPLOY_API_TOKEN` vào `.env` local để test CP5 gọi
      thẳng bản deploy có xác thực.
- [ ] Không deploy được → `LOCAL_FALLBACK=true`, `docker compose up -d`,
      chụp màn hình, ghi lý do (tối đa 9/15đ).
- [ ] `pytest tests/test_cp5.py -v`.
- [ ] Commit.

### Wrap-up — `exercises.md` (15đ)
- [ ] Trả lời đủ 10 câu bằng quan sát thật khi chạy code (không chép mẫu).
- [ ] `python grade.py` → mục tiêu ≥ 75/100.
- [ ] `git ls-files | grep "^\.env$"` phải rỗng.
- [ ] Commit cuối, push, nộp link repo (hạn 23h59 cùng ngày).

### Bonus — CI/CD với GitHub Actions (+10đ, không bắt buộc)
Chỉ làm sau khi CP1–CP5 đã ổn.
- [ ] `.github/workflows/ci.yml`: trigger `push`/`pull_request` vào `main`.
- [ ] Job `test`: checkout → setup-python → cài deps → `pytest` (bỏ qua
      `test_cp5.py` và `test_bonus_cicd.py`), truyền `API_TOKEN=ci-dummy`,
      `REDIS_URL=fake://` qua `env:`.
- [ ] Job `build`: `docker build` trên runner sạch.
- [ ] Job `deploy`: `needs: [test, build]`, `if:` chỉ chạy khi push vào
      `main`; secret deploy để trong GitHub Secrets, action ghim version
      (`@v4`, không dùng `@main`).
- [ ] Smoke test sau deploy (`curl -fsS $URL/healthz`).
- [ ] Badge CI trong `README.md`.
- [ ] `pytest tests/test_bonus_cicd.py -v`.

## Công cụ kiểm tra chính
```bash
pytest tests/test_cp1.py -v
pytest tests/test_cp2.py -v
pytest tests/test_cp3.py -v
pytest tests/test_cp4.py -v
pytest tests/test_cp5.py -v
pytest tests/test_bonus_cicd.py -v   # bonus, sau khi CP1–CP5 xanh
python grade.py                      # tổng điểm
python grade.py --no-bonus           # chỉ chấm phần bắt buộc
```

## Ghi chú / rủi ro
- Làm theo đúng thứ tự CP1 → CP2 → CP3 → CP4 → CP5; kẹt quá 10 phút ở block
  nào thì ghi lại và chuyển sang block sau — điểm tính theo test pass được,
  không phải theo có hoàn thành tuần tự hay không.
- `.env` không bao giờ được commit — kiểm tra trước mỗi lần push.
- Repo phải public trước khi nộp; sai tên repo trừ 5đ (tên hiện tại
  `K4-DAY12-2A202601602-TrinhHaiDang` đã đúng định dạng).
- Commit sau mỗi checkpoint, không dồn vào một commit cuối — lịch sử commit
  là bằng chứng tự làm.
</content>

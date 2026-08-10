# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trịnh Hải Đăng  Mã học viên: 2A202601602

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu deploy lên Railway/Render mà quên set biến `API_TOKEN` trong dashboard,
> với `api_token: str` không mặc định thì `Settings()` ném `ValidationError`
> ngay lúc container khởi động — build log đỏ, deploy fail, và tôi biết ngay
> lập tức phải vào dashboard set lại biến. Nếu để mặc định `"changeme"`, app
> vẫn khởi động và trả lời request bình thường; endpoint `/chat` công khai
> lúc đó chỉ cần ai đó thử đúng `Authorization: Bearer changeme` (một giá trị
> hoàn toàn có thể đoán được vì nó nằm trong chính mã nguồn) là gọi được miễn
> phí. Tôi sẽ chỉ phát hiện ra khi nhìn hóa đơn LLM tăng bất thường, tức là
> đã mất tiền rồi mới biết có lỗi. "Chết sớm" biến lỗi cấu hình thành một sự
> cố nhìn thấy ngay lúc deploy, thay vì một lỗ hổng bảo mật âm thầm.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> ```json
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:35:11.150310+00:00", "client_id": "sv-rate", "prompt_tokens": 269, "completion_tokens": 43, "usd_cost": 6.615e-05}
> ```
>
> Hai việc `print("đã trả lời xong")` không làm được:
> 1. **Lọc/tổng hợp theo trường**: vì mỗi dòng là một JSON object có khóa cố
>    định (`client_id`, `usd_cost`...), tôi có thể chạy một câu truy vấn kiểu
>    "tổng `usd_cost` group by `client_id` trong 24h qua" trên hệ thống log
>    (Cloud Logging, Datadog...) để trả lời "client nào tiêu nhiều tiền nhất
>    hôm nay?". Với `print()`, câu trả lời đó phải viết script parse chuỗi
>    tiếng Việt tự do — dễ vỡ mỗi khi đổi câu chữ.
> 2. **Lọc theo mức độ nghiêm trọng**: khóa `severity` viết hoa là quy ước mà
>    Google Cloud Logging (và tương đương ở platform khác) đọc để tô màu và
>    cho phép đặt alert "báo tôi khi có log `ERROR` trong 5 phút qua". Log tự
>    do bằng `print()` không có khái niệm mức độ, nên không thể đặt alert tự
>    động — phải có người ngồi đọc log bằng mắt mới phát hiện sự cố.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> **Chưa đo được số thật**: Docker Desktop trong máy/môi trường tôi soạn bài
> không khởi động được engine (`docker version` treo, không kết nối được
> named pipe `dockerDesktopLinuxEngine`), nên không build được image ở đây để
> lấy số đo. Trước khi nộp bài, tôi cần tự chạy lại 2 lệnh build ở trên trên
> máy có Docker hoạt động và điền số thật vào bảng.
>
> Về mặt giải thích (không phụ thuộc số đo): bản 1-stage build trên
> `python:3.11` đầy đủ (không phải `-slim`), nên trong image cuối cùng vẫn
> còn nguyên compiler (`gcc`, `build-essential` nếu có), header phát triển,
> apt cache, và toàn bộ file mã nguồn kể cả những gì không cần lúc chạy —
> tổng cộng dễ tới 1.5–1.8GB. Bản multi-stage dùng `python:3.11-slim` ở cả
> hai stage, và stage `runtime` chỉ `COPY --from=builder /install /usr/local`
> — tức là chỉ mang theo các gói Python đã cài (`site-packages`), không mang
> theo compiler hay cache của `pip`/`apt` dùng trong lúc build. Phần chênh
> lệch chính là: base image đầy đủ so với `slim` (~700MB khác biệt), cộng với
> toolchain build (`gcc` và các thư viện dev) bị loại khỏi image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại (`COPY requirements.txt .` → `RUN pip install` →
> `COPY app ./app`), sửa một ký tự trong `app/main.py` chỉ làm thay đổi nội
> dung của layer `COPY app ./app` trở đi. Docker so khớp checksum nội dung
> từng layer: layer `COPY requirements.txt .` và `RUN pip install
> --prefix=/install ...` ở stage `builder` không đổi input nên được lấy từ
> cache nguyên vẹn — không cài lại dependency. Chỉ có `COPY --from=builder`,
> `COPY app ./app` và các lệnh sau nó ở stage `runtime` phải chạy lại, và vì
> đó chỉ là copy file + set user, build lại gần như tức thì.
>
> Nếu đảo thành `COPY . .` rồi mới `RUN pip install`, thì layer `COPY . .`
> sẽ đổi checksum mỗi khi *bất kỳ* file nào trong repo thay đổi — kể cả một
> dấu phẩy trong `app/main.py`. Vì `RUN pip install` nằm ngay sau, cache của
> nó cũng bị hủy theo (Docker hủy cache từ layer đầu tiên thay đổi trở đi),
> nên toàn bộ dependency bị cài lại từ đầu mỗi lần build dù `requirements.txt`
> không hề đổi. Với image build local nhanh vài giây thì không thấy rõ, nhưng
> trên CI/CD build hàng chục lần một ngày, đây là khác biệt giữa build 5 giây
> và build 1-2 phút mỗi lần.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) một thư viện Python tôi dùng (hoặc chính code `/chat`)
> có lỗ hổng cho phép thực thi lệnh tuỳ ý — ví dụ deserialize dữ liệu không
> tin cậy hoặc một RCE trong dependency; (2) vì process FastAPI/uvicorn chạy
> với UID 0 (root) bên trong container, lệnh mà kẻ tấn công chèn được cũng
> chạy với quyền root *trong container đó*; (3) container không phải một
> "hộp cách ly tuyệt đối" — nó dùng chung kernel với host, và nếu kẻ tấn công
> tìm được thêm một lỗ hổng thoát container (container breakout, qua misconfig
> volume mount, capability thừa, hay lỗ hổng kernel/runtime), quyền root
> *trong* container rất dễ biến thành quyền root *trên host*, vì không còn
> tầng phân quyền hệ điều hành nào chặn lại nữa.
>
> Lệnh `USER appuser` (UID 10001, không có quyền admin) cắt đứt chuỗi này ở
> bước (2)-(3): dù lỗ hổng ở bước (1) vẫn tồn tại và kẻ tấn công vẫn thực thi
> được lệnh, lệnh đó giờ chạy với quyền của một user thường. Nếu sau đó có
> thêm một lỗ hổng thoát container, thứ kẻ tấn công có được trên host cũng
> chỉ là quyền của user thường 10001 — không phải root. `USER` không vá lỗ
> hổng ở bước (1), nhưng giới hạn triệt để mức thiệt hại nếu bước (1) xảy ra,
> đúng theo nguyên tắc least privilege.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là yêu cầu bắt buộc của chuẩn HTTP (RFC 7235) —
> mọi response 401 phải nói cho client biết *cách* xác thực đúng là gì, để
> client (hoặc lập trình viên gọi client đó) biết cần gửi header nào, theo
> scheme nào, thay vì đoán mò. Đây cũng là thứ các thư viện HTTP client tự
> động đọc để quyết định có nên thử lại với thông tin xác thực khác không.
>
> Trả cùng một thông báo cho cả ba trường hợp (thiếu header / sai scheme /
> sai token) là để không "tặng thông tin" cho kẻ đang dò hệ thống. Nếu API
> trả "sai scheme" riêng và "token không đúng" riêng, một kẻ tấn công thử
> ngẫu nhiên các header có thể suy ra được: header của họ đúng định dạng
> nhưng token sai, tức là họ đã thu hẹp được không gian cần dò (không cần
> thử lại các định dạng header khác nữa, chỉ cần tập trung dò token). Với
> người dùng hợp lệ đã đọc tài liệu API, thông báo chung này không gây khó —
> họ chỉ cần biết "tôi làm sai", việc còn lại là đối chiếu lại đúng cú pháp
> `Authorization: Bearer <token>` trong docs.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với `min(capacity, ...)` (cài đặt đúng): xô chỉ chứa tối đa `capacity = 10`
> token dù im lặng bao lâu. Sau 10 phút im lặng, xô đã đầy từ trước đó rồi
> (chỉ mất 1 phút để nạp từ 0 lên 10 với tốc độ 10/phút) và không tích thêm
> được nữa. Vì vậy client gửi liên tiếp sẽ được đúng **10 request** ở mã 200,
> request thứ 11 trở đi nhận 429.
>
> Nếu bỏ `min(capacity, ...)`: token cứ cộng dồn tuyến tính theo thời gian im
> lặng, không bị chặn trần. Sau 10 phút, xô sẽ có `10 phút × 10 token/phút =
> 100` token (thay vì dừng ở 10). Client khi đó gửi liên tiếp được **100
> request** trước khi bị 429 — gấp 10 lần sức chứa danh nghĩa của "xô 10
> token". Đây chính là lỗ hổng mà `capacity` được thiết kế để ngăn: không có
> trần thì một client im lặng đủ lâu (một ngày, một tuần) sẽ tích được một
> lượng token khổng lồ và xả hết trong vài giây, biến rate limit thành vô
> nghĩa đúng vào lúc traffic dồn dập nhất — thường cũng là lúc hệ thống dễ
> quá tải nhất.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> **Hạn mức $30/tháng**: key chi tiêu chỉ reset đầu tháng sau. Nếu sự cố bắt
> đầu lúc 2h sáng ngày đầu tháng và không ai để ý, client đó có thể tiêu tới
> gần $30 trước khi bị chặn — vì hệ thống chỉ so sánh tổng chi tiêu tích luỹ
> với 30, không quan tâm chi tiêu đó dồn trong bao lâu. Service chỉ "tự hồi
> phục" (tức chi tiêu về 0, được phép gọi lại) vào đầu tháng kế tiếp — có thể
> phải chờ gần 30 ngày nếu sự cố xảy ra ngay đầu tháng.
>
> **Hạn mức $1/ngày**: key `spend:<client>:<YYYY-MM-DD>` gắn với một ngày cụ
> thể. Dù sự cố kéo dài bao lâu trong ngày, client bị chặn (402) ngay khi
> chạm $1 — thiệt hại tối đa của một sự cố là **$1**, tức 1/30 so với hạn mức
> tháng. Sang 00:00 UTC hôm sau, khoá ngày mới bắt đầu từ 0, client tự động
> gọi lại được (nếu sự cố gốc đã được xử lý) mà không cần ai can thiệp thủ
> công — tối đa chờ vài giờ thay vì gần một tháng. Đây là lý do lab chọn hạn
> mức theo ngày: nó giới hạn thiệt hại của *một lần* sự cố xuống một phần nhỏ
> và tự phục hồi nhanh, dù tổng ngân sách cho phép trong tháng có thể ngang
> nhau.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối. Endpoint gộp (đóng vai trò cả liveness lẫn readiness)
>    gọi `store.ping()`, nhận `False`, trả 503 — ở **cả 3 container** cùng
>    lúc, vì cả 3 đều phụ thuộc cùng một Redis.
> 2. Orchestrator (Docker/Railway/K8s) đọc endpoint đó như một **liveness**
>    probe. Nó hiểu 503 là "process này hỏng, cần khởi động lại", không phải
>    "tạm thời chưa sẵn sàng nhận traffic" — nên nó **restart** cả 3 container
>    cùng lúc, dù bản thân process FastAPI hoàn toàn khoẻ mạnh, chỉ có Redis
>    là bên ngoài đang gặp sự cố.
> 3. Trong lúc 3 container đang khởi động lại, không còn container nào chạy
>    để phục vụ request — toàn bộ service down hoàn toàn, dù trước đó chỉ có
>    một dependency (Redis) gặp sự cố tạm thời 30 giây.
> 4. Redis hồi phục sau 30 giây, nhưng 3 container vừa bị kill có thể vẫn
>    đang trong quá trình khởi động lại (kéo container image, chạy lại
>    entrypoint...) — nên ngay cả khi Redis đã sống lại, service vẫn gián
>    đoạn thêm một khoảng thời gian nữa vì phải chờ container lên lại.
>
> Tách riêng `/healthz` (không đụng Redis, luôn 200 nếu process còn sống) và
> `/readyz` (có kiểm tra Redis) tránh được toàn bộ chuỗi trên: khi Redis rớt,
> `/healthz` vẫn 200 nên orchestrator không restart container; chỉ `/readyz`
> trả 503 nên load balancer tạm ngừng đẩy traffic mới vào — 3 container vẫn
> sống, sẵn sàng nhận traffic lại ngay khi Redis hồi phục, không cần khởi
> động lại từ đầu.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Tại thời điểm soạn bài này, tôi chưa hoàn tất bước deploy thật lên
> Railway/Render (cần tạo tài khoản và thao tác trên dashboard — việc tôi sẽ
> tự làm sau, xem `DEPLOYMENT.md`). Lỗi thực tế tôi gặp trong quá trình làm
> lab là ở **CP2 khi build Docker image cục bộ**: Docker Desktop trên máy
> không phản hồi — `docker version` treo và `docker build` báo
> `request returned 500 Internal Server Error for API route ... check if the
> server supports the requested API version`, tức CLI không kết nối được vào
> Docker engine dù tiến trình `Docker Desktop` vẫn đang chạy.
>
> Cách tìm nguyên nhân: chạy `docker info` và `docker version` riêng để xác
> nhận vấn đề nằm ở kết nối tới daemon (named pipe
> `dockerDesktopLinuxEngine`) chứ không phải ở Dockerfile hay lệnh build —
> nếu daemon sống thì các lệnh này trả thông tin ngay, ở đây chúng bị treo
> hoặc lỗi 500. Việc `pytest tests/test_cp2.py -v` (không dùng marker docker)
> vẫn pass hết 14 test cấu trúc cho thấy Dockerfile/compose viết đúng yêu
> cầu; vấn đề chỉ nằm ở môi trường build, không phải ở code.
> Hướng khắc phục: khởi động lại Docker Desktop (hoặc chờ WSL2 backend khởi
> động xong) rồi build lại; nếu vẫn lỗi thì kiểm tra WSL2 integration trong
> Settings → Resources của Docker Desktop. Đây cũng là lý do các số đo dung
> lượng image ở Câu 3 tôi chưa điền được số thật và cần tự đo lại trước khi
> nộp bài.
</content>

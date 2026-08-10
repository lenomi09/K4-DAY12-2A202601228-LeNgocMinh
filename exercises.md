# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng đánh dấu chỗ trống (in nghiêng, bắt đầu bằng `>`)
> bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Ngọc Minh  Mã học viên: 2A202601228

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi tôi deploy lần đầu lên Railway, tôi tạo service `chat` trước rồi mới
> gõ lệnh `railway variables --set API_TOKEN=...` sau. Nếu tôi lỡ deploy
> trước khi set biến, và `api_token` có mặc định kiểu `"changeme"`, thì app
> vẫn khởi động bình thường, `/health` vẫn trả 200, tôi sẽ tưởng deploy
> thành công — trong khi thực tế endpoint `/chat` đang mở cho bất kỳ ai biết
> chuỗi `"changeme"` (một giá trị dễ đoán) gọi vào và tiêu ngân sách LLM của
> tôi. Vì `api_token: str` không có mặc định, pydantic-settings ném
> `ValidationError` ngay khi `Settings()` được khởi tạo (lúc container start),
> container crash-loop, Railway báo build/deploy fail rõ ràng trên dashboard
> — đúng lúc tôi đang nhìn màn hình để xử lý, thay vì phát hiện ra vài ngày
> sau khi xem hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy từ `uvicorn app.main:app --port 8123` rồi gọi `/chat`:
>
> ```json
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T07:56:17.997459+00:00", "client_id": "sv01", "prompt_tokens": 3, "completion_tokens": 37, "usd_cost": 2.265e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Truy vấn/tổng hợp theo trường**: vì mỗi dòng là một object JSON có
>    khóa cố định, tôi có thể đưa log vào bất kỳ hệ thống nào (Grafana Loki,
>    Google Cloud Logging, hay đơn giản là `jq`) và chạy câu hỏi kiểu "tổng
>    `usd_cost` theo `client_id` trong hôm nay" bằng cách group-by trên
>    trường `client_id` rồi sum `usd_cost`. Với `print()`, câu trả lời đó là
>    chuỗi text tự do — muốn lấy số tiền ra phải viết regex đoán mò và dễ vỡ
>    mỗi khi câu print đổi chữ.
> 2. **Lọc và cảnh báo theo mức độ**: khóa `severity` viết hoa đúng quy ước
>    mà Google Cloud Logging (và tương tự ở nhiều platform khác) dùng để tô
>    màu, lọc theo mức log. Tôi có thể đặt cảnh báo "gửi email nếu có dòng
>    `severity: ERROR` trong 5 phút" mà không cần đụng vào code. `print()`
>    không có khái niệm mức độ, nên muốn lọc "chỉ xem lỗi" phải tự quy ước
>    và tự viết công cụ đọc, hoặc đọc tay từng dòng.

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

> Build thật trên máy tôi (`docker images | grep day12-chat`):
>
> | Bản | Dung lượng |
> |-----|-----------|
> | 1 stage (`python:3.11` đầy đủ, `COPY . .` rồi `pip install`, chạy root) | 1.73 GB |
> | Multi-stage (`python:3.11-slim`, tách builder/runtime, non-root) | 270 MB |
>
> Chênh lệch ~1.46GB đến từ ba nguồn:
> 1. **Base image**: `python:3.11` đầy đủ có sẵn toolchain biên dịch
>    (gcc, make...), thư viện dev, tài liệu — nặng hơn `python:3.11-slim`
>    hàng trăm MB dù chưa cài gì.
> 2. **Build-time deps không bị vứt**: bản 1 stage `pip install` ngay trong
>    image cuối cùng, giữ luôn cache pip, các gói build phụ trợ. Bản
>    multi-stage cài trong stage `builder` (có thể nặng tùy ý) rồi chỉ
>    `COPY --from=builder /install /usr/local` — tức chỉ copy các file
>    `.py`/`.so` đã cài xong sang, không mang theo compiler hay cache pip.
> 3. **`COPY . .` trong bản 1 stage** copy luôn cả `.git`, `tests/`,
>    `.venv` nếu có — mọi thứ không cần lúc chạy vẫn nằm trong image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm một dòng comment mới vào cuối `app/main.py` rồi build lại
> (`docker build --progress=plain`). Kết quả log build thật:
>
> ```
> #6  [builder 3/4] COPY requirements.txt .                    CACHED
> #7  [builder 4/4] RUN pip install ... -r requirements.txt     CACHED
> #8  [runtime 3/6] COPY --from=builder /install /usr/local      CACHED
> #9  [builder 2/4] WORKDIR /app                                 CACHED
> #10 [runtime 4/6] RUN useradd --create-home --uid 10001 ...    CACHED
> #11 [runtime 5/6] COPY app ./app                                (chạy lại)
> #12 [runtime 6/6] COPY utils ./utils                            (chạy lại)
> ```
>
> Toàn bộ stage `builder` (copy requirements.txt, pip install) và cả việc
> tạo user trong stage `runtime` đều CACHED — vì nội dung `requirements.txt`
> không đổi. Chỉ `COPY app ./app` bị cache-miss (đúng file thay đổi), và
> `COPY utils ./utils` sau đó cũng phải chạy lại theo — mỗi layer Docker phụ
> thuộc layer trước nó, hễ một layer bị bust cache thì mọi layer đứng sau nó
> trong cùng stage đều phải build lại dù bản thân chúng không đổi.
>
> Nếu đặt `COPY . .` lên **trước** `RUN pip install -r requirements.txt`
> (như bản Dockerfile gốc chưa sửa) thì sửa một dòng bất kỳ trong `app/`
> cũng làm layer `COPY . .` cache-miss, kéo theo `RUN pip install` — vốn là
> bước tốn thời gian nhất (tải và biên dịch hàng chục package) — phải chạy
> lại từ đầu mỗi lần, dù `requirements.txt` không hề đổi. Đặt
> `COPY requirements.txt` + `pip install` trước, `COPY app` sau, giữ được
> cache của bước cài thư viện qua mọi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) một lỗ hổng trong app — ví dụ một endpoint cho phép
> ghi file tùy ý, hoặc một dependency có remote code execution — cho kẻ tấn
> công chạy được lệnh bên trong container; (2) nếu process đang chạy bằng
> root (UID 0) *bên trong* container, lệnh đó cũng chạy với quyền root; (3)
> UID 0 bên trong container ánh xạ thẳng sang UID 0 trên host (namespace
> chia sẻ UID trừ khi bật user-namespace remapping, mà mặc định Docker
> Desktop/hầu hết setup không bật); (4) kẻ tấn công giờ chỉ cần một lỗ hổng
> thoát container (container escape — qua kernel bug, mount volume nhạy
> cảm như `/var/run/docker.sock`, hay misconfiguration) là có quyền root
> thật trên máy host, không chỉ trong sandbox của container.
>
> Lệnh `USER appuser` (với `useradd --uid 10001`) cắt đứt chuỗi này ở bước
> (2)–(3): process bên trong container giờ chạy với UID 10001, không phải
> 0. Dù kẻ tấn công khai thác được lỗ hổng ở bước (1) và thoát được
> container ở bước (4), quyền họ có trên host chỉ là UID 10001 — một user
> thường, không phải root. Thiệt hại bị giới hạn lại đúng bằng quyền hạn
> của user đó, thay vì toàn quyền trên host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate` là bắt buộc theo chuẩn HTTP (RFC 7235) cho mọi response
> 401: nó là cách server nói cho client biết "bạn cần xác thực, và đây là
> *kiểu* xác thực tôi chấp nhận" (ở đây là `Bearer`). Không có header này,
> client (hay thư viện HTTP chung chung) chỉ biết là bị từ chối chứ không
> biết phải làm gì tiếp — có SDK/proxy dựa vào chính header này để tự động
> chọn luồng xác thực đúng.
>
> Việc trả cùng một thông báo cho cả ba trường hợp (thiếu header, sai
> scheme, sai token) là để không rò rỉ thông tin cho người đang dò: nếu
> "thiếu header" và "sai token" trả lỗi khác nhau, kẻ tấn công có thể suy
> ra được là mình đã đoán đúng *cấu trúc* request (có header, đúng scheme
> `Bearer`) chỉ còn thiếu đúng giá trị token — tức đã thu hẹp không gian dò
> từ "mọi thứ" xuống chỉ còn "giá trị token". Với người dùng hợp lệ đang gõ
> nhầm, thông báo mơ hồ hơi bất tiện một chút, nhưng đổi lại kẻ tấn công
> không lấy được tín hiệu nào để tối ưu cách dò — cùng lý do ta dùng
> `secrets.compare_digest` thay vì `==` để không rò rỉ qua thời gian phản
> hồi.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với `min(capacity, ...)` (code hiện tại): sau 10 phút im lặng, xô nạp
> thêm `10 phút × 10 token/phút = 100` token về mặt lý thuyết, nhưng bị
> chặn trên ở `capacity=10`. Vậy client gửi được đúng **10 request** liên
> tiếp trước khi request thứ 11 nhận 429.
>
> Nếu bỏ `min(...)`: `available()` trả thẳng `100` token (tôi tính lại bằng
> tay: `tokens = 0 + (600s × 10/60 token/s) = 100`). Client gửi được **100
> request** liên tiếp trước khi bị chặn — gấp 10 lần sức chứa dự kiến. Lý
> do: token bucket vốn định nghĩa là "im lặng thì tích, cần thì bùng, nhưng
> mức bùng tối đa bị giới hạn bởi kích thước xô". Thiếu bước chặn trên,
> thời gian im lặng càng dài thì số request được phép bùng ra càng lớn
> không giới hạn — một client rảnh rỗi cả tuần có thể bắn hàng chục nghìn
> request dồn dập trong một giây, đúng thứ mà rate limit sinh ra để ngăn.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức **$30/tháng**: key chi tiêu chỉ reset đầu tháng, nên nếu sự cố
> xảy ra ngày 2 của tháng, client có thể tiêu liên tục tới khi chạm $30 —
> thiệt hại tối đa gần như bằng nguyên hạn mức tháng ($30), và nếu không ai
> phát hiện kịp, service vẫn tiếp tục chặn (hoặc tiếp tục cho qua nếu chưa
> chạm) cho tới hết tháng đó mới tự reset — có thể mất tới gần 30 ngày mới
> "tự hồi phục" theo nghĩa ngân sách mới được cấp lại.
>
> Với hạn mức **$1/ngày** (`CostGuard` trong bài, key `spend:<client>:<ngày>`):
> thiệt hại tối đa trong một ngày sự cố là $1 — dù sự cố bắt đầu lúc 2h sáng
> và không ai để ý, tới khi `spent() + estimated_cost > budget` là `check()`
> raise 402 ngay, chặn hoàn toàn request tiếp theo của client đó. Sang
> 00:00 UTC hôm sau, `today()` đổi nhãn ngày, key Redis mới (`spend:<client>:
> <ngày mới>`) bắt đầu từ 0 — service tự phục vụ lại bình thường mà không
> cần ai can thiệp thủ công.
>
> Vậy cùng một tổng ngân sách tháng ($30 ≈ $1×30), hạn mức ngày giới hạn
> thiệt hại của **một lần sự cố** xuống còn 1/30, và thời gian tự hồi phục
> tính bằng giờ (tới nửa đêm) thay vì tính bằng tuần.

---

### Câu 9 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện nếu `/health` (liveness) cũng gọi `store.ping()`:
>
> 1. Redis mất kết nối. Cả 3 container `chat` cùng gọi vào một Redis, nên
>    cả 3 đồng thời thấy `ping()` trả `False` ngay ở lần probe tiếp theo.
> 2. Endpoint gộp trả 503 cho cả 3 container cùng lúc — vì liveness probe
>    giờ phụ thuộc một dependency bên ngoài mà lẽ ra nó không nên biết tới.
> 3. Orchestrator (Docker/K8s/Railway) đọc 503 từ liveness probe theo đúng
>    ngữ nghĩa của nó: "process này hỏng, cần **restart**" — không phải
>    "tạm rút khỏi vòng xoay" như readiness. Cả 3 container bị kill và khởi
>    động lại gần như cùng lúc.
> 4. Trong lúc cả 3 container đang restart (tốn vài giây tới vài chục
>    giây để container mới boot, app khởi động, kết nối lại), **không còn
>    container nào phục vụ request** — kể cả những request hoàn toàn không
>    liên quan tới Redis (ví dụ chỉ hỏi `/health` để kiểm tra sống chết).
> 5. Nếu Redis chỉ mất kết nối 30 giây rồi tự hồi phục, sự cố lẽ ra chỉ nên
>    khiến `/ready` báo not-ready trong 30 giây đó (load balancer tạm
>    ngừng gửi traffic, container vẫn sống, không ai bị kill) — nhưng vì
>    gộp chung, một sự cố Redis 30 giây biến thành cả cụm service chết hẳn
>    trong thời gian dài hơn (30 giây sự cố + thời gian cả 3 container
>    restart), và có nguy cơ restart-loop nếu Redis chưa kịp hồi phục lúc
>    container mới boot lên và probe lại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy đầu tiên lên Railway: build Docker thành công (image push xong),
> nhưng healthcheck fail — dashboard báo:
>
> ```
> Attempt #1 failed with service unavailable. Continuing to retry for 19s
> Attempt #2 failed with service unavailable. Continuing to retry for 8s
> 1/1 replicas never became healthy!
> Healthcheck failed!
> ```
>
> Tôi tìm nguyên nhân bằng `railway logs -s chat -d --latest` (log runtime,
> không phải log build) và thấy container thật ra khởi động rồi chết ngay:
>
> ```
> Error: Invalid value for '--port': '$PORT' is not a valid integer.
> ```
>
> Nguyên nhân: `railway.toml` có `startCommand = "uvicorn app.main:app
> --host 0.0.0.0 --port $PORT"`. Railway chạy `startCommand` này **không**
> qua một shell trung gian để giãn biến môi trường, nên `$PORT` được truyền
> nguyên văn dưới dạng chuỗi `"$PORT"` cho tham số `--port` của uvicorn thay
> vì được thay bằng số cổng thật — trong khi CMD của Dockerfile (dạng
> `sh -c "..."`) thì có shell nên hoạt động đúng. Sửa bằng cách bọc lại
> `startCommand` qua `sh -c` tường minh:
>
> ```toml
> startCommand = "sh -c \"uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}\""
> ```
>
> Deploy lại (`railway up -s chat --ci`) thành công, `/health` và `/ready`
> đều trả 200 ngay sau đó.

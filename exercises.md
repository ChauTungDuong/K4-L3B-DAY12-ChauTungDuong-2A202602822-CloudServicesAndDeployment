# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Châu Tùng Dương  Mã học viên: 2A202602822

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: Khi deploy service lên Render lần đầu, tôi quên không set biến môi trường `AGENT_API_KEY` trong dashboard. Nhờ `agent_api_key` không có giá trị mặc định, container khởi động xong thì crash ngay với lỗi `ValidationError: 1 validation error for Settings - agent_api_key: Field required`. Tôi thấy ngay trong build log và fix được trong vòng 1 phút. Nếu để mặc định `"changeme"`, service vẫn khởi động bình thường — log xanh hoàn toàn — nhưng bất kỳ ai biết thử key `"changeme"` đều gọi được `/ask` và tiêu ngân sách LLM của tôi mà tôi không hay biết, cho đến khi nhận hóa đơn cuối tháng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được khi chạy service cục bộ và gọi /ask:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:22:14.581042+00:00", "user_id": "sv01", "tokens_in": 15, "tokens_out": 32, "cost_usd": 0.0001}`
>
> Hai việc log JSON làm được mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc và truy vấn tự động**: Hệ thống thu thập log (Datadog, CloudWatch, Render Logs) có thể parse JSON và lọc theo field cụ thể — ví dụ `jq '.[] | select(.cost_usd > 0.01)'` để tìm các request đắt tiền bất thường, hoặc vẽ biểu đồ tổng chi phí theo `user_id` theo thời gian thực.
> 2. **Thiết lập alert tự động**: Có thể đặt rule "cảnh báo nếu `level == 'error'`" hoặc "alert nếu tổng `cost_usd` trong 1 giờ vượt $1". Chuỗi `"đã trả lời xong"` không có cấu trúc, không thể phân tích tự động được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu, python:3.11 full) | ~1.05 GB |
| Multi-stage (python:3.11-slim) | ~186 MB |

> Phần dung lượng chênh lệch (~870 MB) đến từ: toàn bộ toolchain build của Debian (gcc, g++, make, binutils, build-essential ~400MB), header files C/C++ dùng khi compile Python extension, cache của pip (~50MB), và base image python:3.11 đầy đủ tích hợp thư viện hệ thống như libssl-dev, libffi-dev, sqlite... mà runtime không cần. Multi-stage chỉ copy kết quả wheel đã cài vào image slim, bỏ lại toàn bộ môi trường build ở stage builder.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại (COPY requirements.txt → pip install → COPY . .):
> - Khi sửa `app/main.py` và build lại, các layer `FROM python:3.11-slim AS builder`, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install` đều được **dùng lại từ cache** vì requirements.txt không đổi.
> - Chỉ có layer `COPY . .` ở stage runtime bị invalidate (vì source thay đổi), và các lệnh sau đó (`USER appuser`, `CMD`) chạy lại — nhưng không có gì nặng.
> - **Nếu đặt `COPY . .` lên trước `RUN pip install`**: Mỗi lần sửa dù 1 ký tự trong code thì Docker cache bị vô hiệu hóa từ bước `COPY . .`, kéo theo `RUN pip install` phải tải và cài lại toàn bộ thư viện từ đầu (~30-60 giây). Mỗi lần dev là mất vài phút chờ build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện nếu chạy root:
> 1. Code Python có lỗ hổng Remote Code Execution (ví dụ: eval() nhận input từ user trong /ask).
> 2. Kẻ tấn công gửi payload khai thác, thực thi lệnh tùy ý trong container.
> 3. Vì container chạy bằng root (uid=0), kẻ tấn công có quyền ghi vào mọi file trong container, bao gồm `/etc/`, cài backdoor, đọc file nhạy cảm.
> 4. Nếu Docker socket `/var/run/docker.sock` được mount hoặc có lỗ hổng container breakout (kernel exploit), kẻ tấn công thoát ra máy host với quyền root.
>
> Lệnh `USER appuser` cắt đứt ở bước 3: kẻ tấn công chỉ có quyền của user thường (uid=1000), không thể ghi vào `/etc/`, không cài phần mềm hệ thống, không đọc file root-only. Phạm vi tấn công bị giới hạn trong thư mục `/app` mà appuser sở hữu.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request** trong 2 giây liên tiếp với fixed-window 10/phút:
> - Giây 10:00:59 (cuối phút 1): gửi 10 request → hết quota phút 1, nhưng vẫn hợp lệ.
> - Giây 10:01:00 (đầu phút 2): counter reset về 0, gửi tiếp 10 request → hợp lệ.
> - Tổng: 20 request trong khoảng 2 giây (59 → 00), gấp đôi hạn mức.
>
> Sliding window 60s ngăn được điều này: tại thời điểm 10:01:01, cửa sổ nhìn lại 10:00:01–10:01:01 vẫn thấy 10 request từ giây 59, nên request mới bị chặn cho đến khi các request cũ ra khỏi cửa sổ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác biệt**: Rate limit kiểm soát *tần suất* (số request / đơn vị thời gian). Cost guard kiểm soát *tổng chi phí* (USD / tháng). Một là đo tốc độ, một là đo ngân sách tích lũy.
>
> **Tình huống rate limit cho qua, cost guard chặn**: User gửi 1 request/giờ (tần suất rất thấp, qua rate limit dễ dàng), nhưng mỗi request kèm tài liệu 100,000 token → chi phí ~$2/request. Sau 5 request trong tháng, tổng $10 vượt ngân sách → cost guard trả 402.
>
> **Tình huống cost guard cho qua, rate limit chặn**: User mới đầu tháng, chưa tiêu đồng nào (spent=0, qua cost guard). Nhưng user gửi 20 request trong 1 phút (limit=10/phút) → request thứ 11 bị rate limiter chặn 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Chuỗi sự kiện nếu /health kiểm tra Redis:
> 1. Redis mất kết nối. `store.ping()` trả False.
> 2. /health của cả 3 container trả 503.
> 3. Docker/K8s đọc /health thấy 503 → kết luận container đã "chết" → gửi SIGTERM rồi khởi động lại container.
> 4. Container mới khởi động, thử kết nối Redis → vẫn không được → /health 503 → bị restart lại.
> 5. Cả 3 container rơi vào crash-loop liên tục. Toàn bộ traffic vào hệ thống trả 502 Bad Gateway.
> 6. Sau 30 giây Redis hồi phục, nhưng các container đang bị restart nên cần thêm thời gian khởi động lại mới phục vụ được.
>
> Tách /health (không dùng Redis) và /ready (dùng Redis) giải quyết điều này: khi Redis chết, /health vẫn 200 (container không bị restart), chỉ /ready trả 503 để load balancer tạm dừng chuyển traffic. Khi Redis hồi phục, /ready 200 → traffic được chuyển vào ngay, không cần restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis (stateless đúng nghĩa): `history_length` tăng đều và nhất quán qua mỗi request — request 1: 0, request 2: 2, request 3: 4... — bất kể request nào vào instance A, B hay C, vì tất cả đọc chung từ Redis.
>
> Nếu dùng dict Python trong RAM: `history_length` sẽ nhảy loạn. Ví dụ request 1 vào Instance A (history_length=0, lưu vào RAM của A), request 2 vào Instance B (thấy dict rỗng → history_length=0), request 3 vào Instance A (thấy 2 message → history_length=2), request 4 vào Instance C (thấy dict rỗng → history_length=0). Agent "mất trí nhớ" ngẫu nhiên, ảnh hưởng nghiêm trọng đến chất lượng hội thoại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp phải**: Khi deploy lên Render lần đầu, service build Docker thành công nhưng health check timeout — Render báo "Health check failed: Service did not respond to health check at /health within 60s".
>
> **Tìm nguyên nhân**: Xem tab "Logs" trên Render dashboard thấy lỗi `ValidationError: 1 validation error for Settings / agent_api_key / Field required`. App crash ngay khi khởi động vì chưa set `AGENT_API_KEY` trong Render Environment Variables — đây chính là "fail fast" đang hoạt động đúng.
>
> **Cách sửa**: Vào Render Dashboard → Service `day12-agent` → Environment → Add `AGENT_API_KEY` với giá trị key cá nhân → Manual Deploy. Lần này log hiện `service_started` và `/health` trả 200 bình thường. Bài học: luôn kiểm tra Logs ngay sau deploy, không chỉ nhìn vào trạng thái build.

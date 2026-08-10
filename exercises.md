# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder in nghiêng ngay dưới mỗi câu hỏi bằng
> nội dung trả lời thật của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Huy Hoàng  Mã học viên: 2A202601113

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway ở CP5, tôi tạo project mới, thêm Redis, nhưng quên
> qua tab Variables của service `agent` để set `AGENT_API_KEY` — chỉ thêm
> Redis rồi tưởng thế là đủ. Kết quả: `/ready` và `/ask` trả `500 Internal
> Server Error` ngay từ request đầu tiên, vì `Settings()` (được gọi bên trong
> `get_settings()`) ném `ValidationError` do thiếu trường bắt buộc. Nhờ vậy
> tôi phát hiện thiếu cấu hình chỉ vài phút sau khi deploy, trước khi có ai
> gọi thật vào service.
>
> Nếu `agent_api_key` có mặc định `"changeme"`, app sẽ khởi động "thành công"
> bình thường — `/health` xanh, dashboard không báo lỗi gì. Nhưng vì
> `"changeme"` là giá trị công khai (ai đọc code lab cũng biết), bất kỳ ai
> cũng gửi được header `X-API-Key: changeme` và gọi `/ask` miễn phí trên chi
> phí của tôi. Tệ hơn nữa: tôi sẽ không biết mình đang bị lộ key cho tới khi
> nhận hóa đơn hoặc thấy `cost_guard` báo vượt ngân sách bất thường — lúc đó
> đã quá muộn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy từ `docker compose logs agent` khi tôi gọi `/ask`:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:44:10.123456+00:00", "user_id": "anonymous", "tokens_in": 37, "tokens_out": 45, "cost_usd": 3.255e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc/truy vấn theo trường cụ thể** — ví dụ `docker compose logs agent |
>    grep ask_completed | jq 'select(.cost_usd > 0.0001)'` để tìm các lần gọi
>    tốn nhiều tiền bất thường, hoặc lọc theo `user_id` để xem một user cụ thể
>    đã hỏi gì. `print` chỉ ra một chuỗi tự do, muốn lọc phải viết regex đoán mò.
> 2. **Tổng hợp/tính toán tự động** — vì mỗi dòng là JSON hợp lệ, một hệ thống
>    log (Datadog, CloudWatch...) có thể tự cộng dồn `cost_usd` theo `user_id`
>    theo ngày và bắn cảnh báo khi vượt ngưỡng, hoàn toàn không cần code thêm ở
>    phía app. `print` không có cấu trúc nên không công cụ nào tổng hợp được.

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
| 1 stage (bản đầu) | 1190 MB (1.19 GB) |
| Multi-stage | 183 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch hơn 1000MB. Phần lớn là do base image: `python:3.11` (bản đầy đủ)
> mang theo toàn bộ toolchain build (gcc, make, các thư viện dev...) để có thể
> compile bất kỳ package Python nào cần biên dịch từ source, cộng thêm rất
> nhiều tiện ích dòng lệnh không dùng tới khi chạy app (editor, debugger,
> locale...). Bản multi-stage dùng `python:3.11-slim` cho stage runtime — stage
> `builder` (cũng dựa trên `slim`) là nơi duy nhất chạy `pip install`, sau đó
> chỉ có `/root/.local` (kết quả cài đặt) được copy sang stage runtime bằng
> `COPY --from=builder`. Toolchain build, cache của pip, và mọi thứ stage
> `builder` từng dùng để cài đặt đều bị bỏ lại, không lọt vào image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi đổi `SERVICE_VERSION = "1.0.0"` thành `"1.0.1"` trong `app/main.py` rồi
> `docker build` lại. Kết quả thật:
>
> - **CACHED**: `WORKDIR /app`, `COPY requirements.txt .`,
>   `RUN pip install --user -r requirements.txt` (cả stage builder), và ở
>   stage runtime: `RUN useradd ...`, `COPY --from=builder /root/.local ...`
> - **Chạy lại (không cache)**: `COPY . .` (vì nội dung thư mục đã đổi) và
>   `RUN chown -R appuser:appuser /app` (đứng sau `COPY . .` nên cache của nó
>   cũng mất theo — Docker cache theo layer tuần tự, một layer bị invalidate
>   thì mọi layer sau nó cũng phải chạy lại dù bản thân lệnh không đổi)
>
> Nếu đặt `COPY . .` lên **trước** `RUN pip install`: bất kỳ lần nào sửa dù
> chỉ một dòng code, Docker sẽ thấy `COPY . .` đổi nội dung → cache layer đó
> mất → toàn bộ `RUN pip install` (đứng sau) cũng phải chạy lại từ đầu, dù
> `requirements.txt` không hề đổi. Với repo này chỉ mất vài giây do ít
> dependency, nhưng với dự án thật có hàng chục thư viện (một số cần compile),
> việc cài lại toàn bộ mỗi lần sửa 1 dòng code có thể biến build 10 giây thành
> build vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) một lỗ hổng trong code Python (ví dụ một dependency bị
> compromise, hoặc một lỗi deserialize cho phép chạy lệnh tùy ý) cho phép kẻ
> tấn công thực thi mã bên trong container; (2) nếu process đang chạy bằng
> root, đoạn mã tùy ý đó cũng chạy với quyền root **bên trong container**; (3)
> container không phải một cỗ máy ảo hoàn toàn tách biệt — nó dùng chung kernel
> với host, chỉ cách ly bằng namespace/cgroup; nếu có thêm một lỗ hổng ở tầng
> container runtime hoặc container được chạy với cấu hình thiếu chặt (mount
> `/var/run/docker.sock`, `--privileged`, thiếu `no-new-privileges`...), quyền
> root-trong-container có thể "thoát" (container breakout) thành quyền root
> thật trên host.
>
> Lệnh `USER appuser` cắt đứt chuỗi này ngay ở bước (2): dù code Python có lỗ
> hổng và kẻ tấn công chạy được lệnh tùy ý, lệnh đó chỉ chạy được với quyền
> của `appuser` (một user thường, không có quyền ghi ra ngoài thư mục
> `/app` đã `chown`) — không còn là root ngay từ bên trong container, nên dù
> tầng runtime có sơ hở, kẻ tấn công cũng không có sẵn quyền root để khai
> thác tiếp.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt được: gửi đúng 10 request vào
> giây thứ 59 của phút hiện tại (dùng hết hạn mức của phút đó), rồi gửi tiếp
> 10 request vào giây thứ 00–01 của phút kế tiếp (bộ đếm vừa reset về 0 vì
> sang phút mới). Với cách đếm theo phút đồng hồ, hai phút là hai "hộp" hoàn
> toàn độc lập — hệ thống không biết 20 request đó thực chất dồn vào một
> khoảng 2 giây liên tiếp, mỗi hộp đều thấy đúng 10/10, hợp lệ. Đây chính là
> lỗ hổng mà sliding window (đếm số request trong 60 giây *gần nhất*, không
> quan tâm ranh giới phút) giải quyết được — `hit_count()` luôn xóa các entry
> cũ hơn `now - WINDOW_SECONDS` bất kể mốc phút, nên không có "ranh giới" nào
> để lợi dụng.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **số lượng** request trong một khoảng thời gian (đếm
> bằng ZSET), không quan tâm mỗi request tốn bao nhiêu tiền. Cost guard giới
> hạn **số tiền** cộng dồn trong tháng (bằng `INCRBYFLOAT`), không quan tâm
> tốc độ gửi.
>
> - **Rate limit cho qua, cost guard phải chặn**: một user chỉ gửi 2–3
>   request/phút (rất dưới hạn mức 10/phút) nhưng mỗi câu hỏi rất dài kèm
>   lịch sử hội thoại 20 tin nhắn (đã cắt ở `HISTORY_MAX_MESSAGES` nhưng vẫn
>   nhiều token) — vài chục request như vậy trong tháng đã đủ vượt
>   `MONTHLY_BUDGET_USD`, dù không bao giờ vi phạm rate limit.
> - **Cost guard cho qua, rate limit phải chặn**: một user gửi 15 request
>   trong 10 giây, mỗi câu hỏi chỉ vài từ ("test", "hi"...) — tổng chi phí cả
>   15 request cộng lại vẫn rất nhỏ so với ngân sách tháng, nhưng vượt hạn mức
>   10 request/phút nên bị `429` từ request thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Tôi từng tự kiểm chứng riêng phần "Redis chết thì /health vẫn 200, /ready
> mới 503" bằng cách chạy `docker compose stop redis` — đúng như thiết kế,
> `/health` không hề bị ảnh hưởng. Nếu gộp chung làm một endpoint (liveness
> probe cũng gọi `store.ping()`), thứ tự sự kiện với cụm 3 container sẽ là:
>
> 1. Redis mất kết nối.
> 2. Lần probe kế tiếp trên **cả 3** container đều gọi `store.ping()` bên
>    trong endpoint liveness → `ping()` trả `False` → endpoint trả về lỗi
>    (503 hoặc exception) thay vì 200.
> 3. Orchestrator hiểu nhầm "endpoint liveness fail" nghĩa là **process đã
>    chết**, không phải "process sống nhưng phụ thuộc đang down" — nó
>    **restart cả 3 container cùng lúc**, dù cả 3 process Python đều hoàn
>    toàn khỏe mạnh, chỉ có Redis là bên ngoài bị lỗi.
> 4. Trong lúc 3 container đang bị kill và khởi động lại (tốn vài giây đến
>    vài chục giây mỗi container), **không còn container nào phục vụ được
>    request** — outage toàn cụm, dù bản chất chỉ là một sự cố Redis 30 giây.
> 5. Nếu Redis hồi phục đúng lúc 3 container đang trong quá trình restart,
>    downtime thực tế còn kéo dài hơn 30 giây ban đầu, vì phải đợi cả 3
>    container khởi động xong.
>
> Với thiết kế tách riêng: `/health` vẫn 200 trong suốt 30 giây đó nên
> orchestrator không restart gì cả; chỉ `/ready` báo 503 khiến load balancer
> tạm ngừng đẩy traffic mới vào — khi Redis hồi phục, `/ready` quay lại 200
> ngay lập tức, không cần chờ restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với `ConversationStore` lưu trong Redis (bản tôi đã code ở CP4), khi gọi
> `/ask` liên tiếp cùng một `X-User-Id` — kể cả khi có 3 replica `agent` phía
> sau load balancer — `history_length` luôn tăng đúng thứ tự (0, 2, 4, 6...)
> vì bất kể request rơi vào replica nào, cả 3 đều đọc/ghi chung một Redis.
> Tôi đã kiểm chứng hành vi này ở quy mô 1 container (`history_length` tăng
> đúng 0 → 2 giữa 2 lần gọi liên tiếp).
>
> Nếu lịch sử được lưu trong một `dict` Python ngay trong process thay vì
> Redis: mỗi trong 3 container có một bộ nhớ RAM hoàn toàn riêng biệt, không
> container nào nhìn thấy dict của container khác. Load balancer phân phối
> request luân phiên (round-robin) giữa 3 container, nên `history_length`
> trả về sẽ **không tăng đều đặn** mà nhảy lung tung — ví dụ 0, 0, 1, 0, 1,
> 2... tùy request đó rơi đúng vào container nào đã từng phục vụ user này
> trước đó. Agent trông như liên tục "quên" một phần cuộc hội thoại, đúng
> hiện tượng mà thiết kế stateless (CP4) nhắm tới việc loại bỏ.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp: deploy lên Railway báo `Healthcheck failed!` — log chỉ có
> `====Starting Healthcheck====`, 2 lần thử đều `service unavailable`, `1/1
> replicas never became healthy!`, sau đó im lặng hoàn toàn — không hề có
> dòng `Uvicorn running on...` hay bất kỳ log runtime nào của app.
>
> Cách tìm nguyên nhân: tôi build và chạy thử chính image đó trên máy local,
> mô phỏng đúng cách Railway gán cổng động (`docker run -e PORT=3000 ...`) và
> cả trường hợp thiếu `AGENT_API_KEY`. Cả hai lần chạy local đều lên đủ log
> khởi động và `/health` trả `200` bình thường trong dưới 1 giây — loại được
> 2 giả thuyết phổ biến nhất (không đọc đúng `$PORT`, thiếu API key làm
> crash). Vì local chạy tốt còn Railway thì hoàn toàn im lặng, nghi ngờ
> chuyển sang phần khác biệt duy nhất giữa hai môi trường: `railway.toml` có
> khai báo riêng `startCommand = "uvicorn app.main:app --host 0.0.0.0 --port
> $PORT"`, ghi đè hẳn `CMD` của Dockerfile (vốn chạy qua
> `sh -c "... --port ${PORT:-8000}"` — có shell nội suy biến, có giá trị dự
> phòng). `startCommand` viết tay không đảm bảo được chạy qua shell để `$PORT`
> được thay thế đúng — nếu không, uvicorn nhận literal chuỗi `"$PORT"` làm
> cổng và crash ngay khi khởi động, không kịp in bất kỳ log nào.
>
> Cách sửa: xóa hẳn dòng `startCommand` trong `railway.toml`, để Railway dùng
> thẳng `CMD` đã có sẵn trong Dockerfile (đã kiểm chứng chạy đúng trên nhiều
> kịch bản khác nhau). Sau khi push thay đổi này và deploy lại, log runtime
> xuất hiện đầy đủ và `/health`, `/ready`, `/ask` đều phản hồi đúng qua URL
> công khai.

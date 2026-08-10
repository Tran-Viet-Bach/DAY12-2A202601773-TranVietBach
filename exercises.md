# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng chỗ trống dưới mỗi câu bằng câu trả lời của mình.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Việt Bách  Mã học viên: 2A202601773

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống em gặp thật khi deploy lên Railway: em tạo service `agent` xong
> nhưng đặt biến môi trường bị lỗi cú pháp, nên `AGENT_API_KEY` thực tế chưa hề
> được set.
>
> Nếu `Settings` để mặc định `"changeme"`, app sẽ khởi động bình thường,
> `/health` trả 200, healthcheck của Railway xanh, và service lên production
> trong trạng thái "trông như ổn". Nhưng khoá lúc đó là `"changeme"` — một
> chuỗi ai cũng đoán được, mà endpoint `/ask` thì đang nằm trên URL công khai.
> Bất kỳ ai thử `X-API-Key: changeme` đều gọi được LLM bằng tiền của em, và em
> chỉ phát hiện khi nhìn hoá đơn hoặc khi `cost_guard` chặn ở mức 402. Sai lầm
> lúc cấu hình biến thành lỗ hổng bảo mật im lặng.
>
> Vì không có mặc định, `Settings()` ném `ValidationError: agent_api_key Field
> required` — sai cấu hình lộ ra ngay dưới dạng lỗi rõ ràng, không có cách nào
> chạy tiếp trong trạng thái nửa vời.
>
> Một điều em quan sát thêm và thấy đáng nói: hôm đó service vẫn hiện "Online"
> và `/health` vẫn trả 200 dù thiếu khoá. Lý do là `get_settings()` chỉ được gọi
> **lúc có request** thông qua `Depends`, chứ không phải lúc khởi động — nên
> `/ready` và `/ask` trả 500 còn `/health` thì không. Nghĩa là "fail fast" ở đây
> mới chỉ fail sớm ở mức *request đầu tiên*, chưa phải ở mức *khởi động*. Muốn
> fail đúng lúc boot thì phải gọi `get_settings()` một lần trong `lifespan`.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy từ `docker compose logs agent`:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:55:43.299032+00:00", "user_id": "sv-bach", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
> ```
>
> **Việc 1 — cộng tiền theo từng user mà không cần đụng vào database.** Vì
> `cost_usd` và `user_id` là hai trường riêng biệt chứ không phải chữ trong một
> câu, em lọc và cộng được trực tiếp trên log:
>
> ```bash
> docker compose logs agent | grep -o '{"event".*}' \
>   | jq -s 'map(select(.event=="ask_completed"))
>            | group_by(.user_id)
>            | map({user: .[0].user_id, tong: (map(.cost_usd) | add)})'
> ```
>
> Với `print("đã trả lời xong")` thì không có số nào để cộng — muốn biết ai tiêu
> bao nhiêu phải viết lại code và deploy lại.
>
> **Việc 2 — đặt cảnh báo tự động theo ngưỡng.** Các nền tảng log (Railway,
> Datadog, CloudWatch) parse được JSON thành field, nên em viết được điều kiện
> kiểu `event = "ask_completed" AND cost_usd > 0.05` để bắn cảnh báo khi có một
> request bất thường đắt. Câu chữ tự do không tạo được điều kiện như vậy, cùng
> lắm chỉ tìm theo từ khoá.
>
> Ngoài ra `timestamp` ở dạng ISO-8601 UTC giúp gộp log của cả 3 container theo
> đúng trình tự thời gian — chạy `--scale agent=3` thì mỗi dòng log đến từ một
> process khác nhau, không có mốc thời gian chuẩn thì không dựng lại được diễn
> biến của một sự cố.

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
| 1 stage (bản đầu) | ~1020 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo thật của bản multi-stage là **270MB** (`docker images day12-agent:prod`),
> trong đó riêng base `python:3.11-slim` đã chiếm **189MB** — nghĩa là toàn bộ
> thư viện của em chỉ thêm khoảng 81MB. Con số ~1020MB của bản 1 stage em lấy
> theo dung lượng công bố của base `python:3.11` bản đầy đủ (~1.01GB), chưa build
> lại để đo trực tiếp.
>
> Phần chênh khoảng 750MB gồm hai nhóm:
>
> **Nhóm 1 — thứ base image đầy đủ mang sẵn mà runtime không cần.** `python:3.11`
> dựng trên Debian bản đủ, kèm bộ biên dịch (`gcc`, `g++`, `make`), header của
> thư viện C (`build-essential`, `libc6-dev`), Git, và nhiều công cụ dòng lệnh.
> Những thứ này chỉ cần lúc **cài** package có phần biên dịch, không cần lúc
> **chạy**. Bản `slim` cắt bỏ chúng.
>
> **Nhóm 2 — rác của quá trình build.** Cache của pip, file nguồn `.tar.gz` tải
> về, thư mục build tạm. Trong Dockerfile của em, chúng nằm ở stage `builder`
> tại `/build` và `/install`, còn stage runtime chỉ `COPY --from=builder /install
> /usr/local` — tức chỉ lấy thư viện đã cài xong, bỏ lại toàn bộ phần còn lại.
> Stage `builder` không hề đi vào image cuối.
>
> Ngoài dung lượng, phần bị bỏ lại còn là **bề mặt tấn công**: có `gcc` và Git
> trong container production nghĩa là kẻ tấn công vào được đã có sẵn công cụ để
> biên dịch và tải mã độc. Image nhỏ vừa deploy nhanh hơn vừa ít thứ để khai thác.
>
> Một con số nữa em thấy đáng chú ý: Docker báo `CONTENT SIZE` là 63.7MB — đây
> mới là lượng byte thật sự phải tải khi `docker pull`, vì các layer được nén.
> 270MB là dung lượng sau khi giải nén trên đĩa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Docker cache theo layer và **vô hiệu hoá theo dây chuyền**: một layer đổi thì
> mọi layer sau nó đều phải chạy lại, bất kể chúng có liên quan hay không.
>
> Với Dockerfile của em, sửa `app/main.py` cho kết quả:
>
> | Layer | Kết quả |
> |---|---|
> | `FROM python:3.11-slim AS builder` | cache |
> | `COPY requirements.txt .` | cache — file này không đổi |
> | `RUN pip install --prefix=/install ...` | **cache** — đây là layer đắt nhất |
> | `FROM python:3.11-slim` (runtime) | cache |
> | `COPY --from=builder /install /usr/local` | cache |
> | `COPY app/ app/` | **chạy lại** — nội dung đã đổi |
> | `COPY utils/ utils/` | chạy lại |
> | `RUN adduser ...`, `USER`, `HEALTHCHECK`, `CMD` | chạy lại |
>
> Các layer chạy lại đều là thao tác chép file và metadata, gần như tức thời.
> Em quan sát đúng hiện tượng này khi sửa dòng `RUN adduser` để bỏ chữ
> `--disabled-password`: build lại chỉ mất vài giây vì `pip install` được dùng
> lại từ cache.
>
> **Nếu đặt `COPY . .` trước `RUN pip install`:** mọi thay đổi trong bất kỳ file
> nào của repo — kể cả sửa một dấu chấm trong README — đều làm checksum của layer
> `COPY . .` thay đổi. Layer đó hỏng cache thì `RUN pip install` nằm sau cũng
> hỏng theo, và toàn bộ thư viện phải tải và cài lại từ đầu, mất vài phút mỗi lần
> build.
>
> Nguyên tắc rút ra: **xếp layer theo tần suất thay đổi, ít đổi nhất lên trước.**
> `requirements.txt` vài tuần mới sửa một lần, còn code sửa mỗi ngày — nên chép
> `requirements.txt` riêng và cài thư viện trước, rồi mới chép mã nguồn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy bằng root:
>
> 1. Code Python có lỗ hổng cho phép thực thi lệnh — ví dụ một chỗ truyền dữ
>    liệu người dùng vào `subprocess` hoặc `pickle.loads`, hoặc một thư viện phụ
>    thuộc dính CVE.
> 2. Kẻ tấn công gửi request khai thác, giành được quyền chạy lệnh tuỳ ý **bên
>    trong container, với uid 0**.
> 3. Là root trong container, họ ghi được vào mọi thư mục hệ thống: sửa
>    `/usr/local/lib/python3.11/site-packages` để cài backdoor, thay binary, đọc
>    mọi file cấu hình và biến môi trường — trong đó có `AGENT_API_KEY` và
>    `REDIS_URL` kèm mật khẩu.
> 4. Root trong container cũng dùng được các capability của kernel mà user
>    thường không có (`CAP_SYS_ADMIN`, thao tác mount, raw socket) — đây là bàn
>    đạp cho bước tiếp theo.
> 5. Nếu có thêm một điểm yếu về cấu hình — Docker socket bị mount vào
>    container, một volume gắn từ host, hoặc một CVE cho phép thoát container —
>    thì uid 0 trong container **chính là uid 0 trên host**, vì namespace user
>    mặc định không được bật. Kẻ tấn công thành root trên máy chủ.
>
> **`USER appuser` cắt chuỗi ở bước 3.** Từ bước 2 trở đi, mã của kẻ tấn công
> chạy dưới một uid thường:
>
> - Không ghi được vào `/usr`, `/etc` → không cài được backdoor bền vững, restart
>   container là sạch.
> - Không có capability đặc quyền → phần lớn kỹ thuật thoát container không dùng
>   được.
> - Kể cả thoát ra được host, họ mang theo một uid không đặc quyền chứ không phải
>   root — thiệt hại giới hạn ở những gì uid đó chạm tới.
>
> Điểm cần nhấn: `USER` **không ngăn** lỗ hổng ở bước 1 và 2. Nó là lớp phòng thủ
> chiều sâu — giả định rằng sớm muộn sẽ có người vào được, và làm cho việc vào
> được đó ít giá trị nhất có thể.
>
> Chi tiết nhỏ nhưng quan trọng trong Dockerfile của em: `USER appuser` đặt
> **sau** các lệnh `COPY` và `RUN` cần quyền ghi, nhưng **trước** `CMD`. Nếu đặt
> ngay đầu, các bước cài đặt sẽ hỏng vì không đủ quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút?

> **20 request trong 2 giây** — gấp đôi hạn mức danh nghĩa.
>
> Cách đạt được: bộ đếm theo phút đồng hồ tạo ra một ranh giới cứng, và ranh
> giới đó reset bộ đếm về 0 bất kể vừa xảy ra gì trước đó.
>
> | Thời điểm | Hành động | Bộ đếm phút |
> |---|---|---|
> | 10:00:59.0 → 10:00:59.9 | gửi 10 request | 10/10 — vừa chạm trần |
> | 10:01:00.0 | sang phút mới, bộ đếm reset | 0/10 |
> | 10:01:00.0 → 10:01:00.9 | gửi tiếp 10 request | 10/10 |
>
> Tổng: 20 request trong khoảng 2 giây, mà xét theo luật thì cả hai phút đều
> "đúng hạn mức 10/phút". Server phải chịu tải tức thời gấp đôi thiết kế.
>
> Sliding window không có ranh giới nào để lợi dụng, vì mốc so sánh luôn là
> `now - 60` chứ không phải giây :00. Trong `hit_count`, em xoá các entry cũ hơn
> cửa sổ rồi mới đếm phần còn lại:
>
> ```python
> self.client.zremrangebyscore(key, 0, now - WINDOW_SECONDS)
> return self.client.zcard(key)
> ```
>
> Tại 10:01:00.5, cửa sổ trải từ 10:00:00.5 tới hiện tại — 10 request lúc
> 10:00:59 vẫn nằm trong đó, nên request thứ 11 bị chặn ngay. Muốn gửi thêm phải
> đợi chúng trôi ra khỏi cửa sổ, tức đúng 60 giây sau khi chúng được ghi.
>
> Chi tiết em suýt làm sai: member của ZSET phải là chuỗi **duy nhất**
> (`f"{now}:{uuid4().hex}"`). Nếu chỉ dùng timestamp làm member, hai request đến
> cùng một mốc thời gian sẽ ghi đè nhau trong sorted set và bộ đếm bị hụt.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau ở **đơn vị đo** và **chu kỳ**:
>
> | | Rate limit | Cost guard |
> |---|---|---|
> | Đo cái gì | số request | số tiền (USD) |
> | Chu kỳ | 60 giây trượt | theo tháng |
> | Bảo vệ khỏi | quá tải tức thời | hoá đơn cuối tháng |
> | Mã lỗi | 429 Too Many Requests | 402 Payment Required |
>
> Điểm mấu chốt: **số request không tỉ lệ với số tiền.** Chi phí phụ thuộc số
> token, mà một request có thể mang 3 token hoặc 50.000 token.
>
> **Rate limit cho qua, cost guard chặn:** một user gửi 3 request/phút — thoải
> mái dưới hạn 10 — nhưng mỗi request đính kèm một tài liệu dài, tốn khoảng
> 4 USD tiền token. Sau ba request là 12 USD, vượt `MONTHLY_BUDGET_USD=10`. Rate
> limit không thấy gì bất thường vì nó chỉ đếm được "3 lần gọi". Cost guard chặn
> ở request thứ ba với 402.
>
> **Cost guard cho qua, rate limit chặn:** user viết một vòng lặp gọi `/ask` liên
> tục với câu hỏi ngắn, mỗi lần khoảng 0,000025 USD. Em đo thật trên bản deploy:
> `cost_usd` là `2.505e-05`, nghĩa là phải gọi khoảng 400.000 lần mới chạm ngân
> sách 10 USD — cost guard sẽ không bao giờ chặn kịp. Nhưng rate limit chặn ngay
> ở request thứ 11. Đây là kết quả em chạy 15 lần liên tiếp trên URL công khai:
>
> ```
> 200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
> ```
>
> Đúng 10 lần đầu qua, 5 lần sau bị chặn.
>
> Vì vậy hai cơ chế bổ sung nhau chứ không thay thế nhau, và trong `/ask` cả hai
> đều được gọi **trước** `ask_llm`. Lý do là tiền chỉ mất ở bước gọi LLM — chặn
> sau khi đã gọi thì vừa mất tiền vừa trả lỗi cho người dùng.
>
> Một hạn chế em nhận ra ở thiết kế hiện tại: `guard.check(user_id)` gọi với
> `estimated_cost` mặc định là 0, nên nó chỉ chặn dựa trên tiền **đã tiêu**, chưa
> tính tiền của request sắp chạy. Một request cực đắt vẫn lọt qua nếu số dư còn
> đúng 0,01 USD. Muốn chặt hơn thì phải ước lượng chi phí từ độ dài prompt rồi
> truyền vào `estimated_cost`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Diễn biến theo thứ tự:
>
> 1. **Giây 0** — Redis mất kết nối. Cả 3 container vẫn khoẻ: process sống,
>    RAM bình thường, code không lỗi.
> 2. **Giây 0-10** — liveness probe của orchestrator gọi endpoint gộp. Nó chạy
>    `store.ping()`, ping thất bại → trả 503. **Cả 3 container cùng lúc**, vì cả
>    ba đều trỏ tới một Redis duy nhất.
> 3. **Giây ~10-30** — probe thất bại đủ số lần `retries`. Orchestrator kết luận
>    "container hỏng, cần restart" và **giết cả 3 container**.
> 4. **Giây ~30** — container khởi động lại. Nhưng Redis vẫn chưa hồi, nên probe
>    lại thất bại ngay từ lần đầu.
> 5. **Vòng lặp** — container vào trạng thái `CrashLoopBackOff`: khởi động, bị
>    giết, khởi động lại, khoảng nghỉ giữa các lần tăng dần.
> 6. **Redis hồi phục** — nhưng dịch vụ **chưa** trở lại ngay: các container đang
>    trong khoảng chờ backoff, phải đợi hết chu kỳ mới được khởi động, rồi còn
>    thời gian cold start.
>
> Kết quả: **Redis chập 30 giây gây ra sự cố kéo dài vài phút.** Tệ hơn, mọi
> request đang xử lý dở lúc bị giết đều đứt giữa chừng, và khi cả 3 container
> cùng khởi động lại chúng đồng loạt mở kết nối tới Redis vừa hồi — tạo thêm một
> đợt tải dồn có thể làm nó ngã tiếp.
>
> Tách hai endpoint thì mọi chuyện dừng ở bước 2:
>
> - `/health` không đụng Redis → vẫn 200 → orchestrator **không restart** gì cả.
> - `/ready` trả 503 → load balancer ngừng đẩy traffic vào, nhưng container vẫn
>   sống nguyên vẹn.
> - Redis hồi → `/ready` tự trả 200 trở lại → traffic quay lại **ngay lập tức**,
>   không cần khởi động lại gì.
>
> Em quan sát đúng kịch bản này trên Railway khi service Redis bị hỏng: `/health`
> trả 200 `{"status":"ok"}` trong khi `/ready` trả 503
> `{"status":"not ready","redis":false}`. Container `agent` không hề bị restart,
> và sau khi em khôi phục Redis bằng `railway redeploy --from-source`, `/ready`
> trả 200 lại mà không cần deploy lại `agent`.
>
> Nguyên tắc: **liveness trả lời "có cần restart tôi không", readiness trả lời
> "có nên gửi traffic cho tôi không".** Restart không sửa được một Redis đang
> chết, nên liveness không được phụ thuộc vào Redis.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> **Với Redis (bản em làm):** `history_length` tăng đều 0 → 2 → 4 → 6..., mỗi
> lượt hỏi thêm 2 message (một `user`, một `assistant`). Nó tăng đúng như vậy bất
> kể nginx đẩy request vào container nào, vì cả 3 container đọc chung một key
> `history:<user_id>` trong Redis. Kết quả thật em đo trên bản deploy:
>
> ```json
> {"history_length": 0, ...}   ← lần 1
> {"history_length": 2, ...}   ← lần 2, và câu trả lời có thêm
>                                 "(Mình đang nhớ 2 lượt trao đổi trước đó.)"
> ```
>
> `tokens_in` cũng tăng từ 3 lên 48 vì lịch sử được nạp vào prompt — bằng chứng
> agent thực sự đọc được ngữ cảnh cũ.
>
> **Nếu lưu trong dict Python:** mỗi container có một dict riêng trong RAM của
> process nó, ba container là ba bộ nhớ hoàn toàn tách biệt. nginx chia request
> theo vòng (round-robin), nên cùng một `X-User-Id` sẽ rơi vào các container khác
> nhau:
>
> | Lần gọi | Container | `history_length` thấy được |
> |---|---|---|
> | 1 | A | 0 |
> | 2 | B | 0 — B chưa từng thấy user này |
> | 3 | C | 0 |
> | 4 | A | 2 — A nhớ được lần 1 của chính nó |
> | 5 | B | 2 |
>
> Con số nhảy loạn xạ và tăng chậm hơn thực tế khoảng 3 lần. Từ góc nhìn người
> dùng, agent "mất trí nhớ" một cách ngẫu nhiên — lúc nhớ lúc không, tuỳ vào việc
> họ tình cờ rơi vào container nào.
>
> Còn hai hệ quả nữa:
>
> - **Restart là mất sạch.** Container bị giết hoặc deploy bản mới là toàn bộ
>   lịch sử biến mất, không có cách nào khôi phục.
> - **Không scale được.** Muốn thêm container thứ 4 để chịu tải thì lại càng làm
>   trí nhớ vụn hơn — đúng nghịch lý mà stateless sinh ra để giải quyết.
>
> Với Redis, container trở thành thứ có thể vứt đi và thay thế bất cứ lúc nào,
> vì không có gì quan trọng nằm trong nó. Đó là điều kiện để scale ngang, để
> deploy không downtime, và để orchestrator tự do restart khi cần.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi: app không đọc được `$PORT`.**
>
> Thông báo lỗi trong log deploy của Railway, lặp lại liên tục rồi `Deploy failed`:
>
> ```
> Starting Container
> Usage: uvicorn [OPTIONS] APP
> Error: Invalid value for '--port': '$PORT' is not a valid integer.
> ```
>
> **Cách tìm ra nguyên nhân:** chi tiết quan trọng nhất là uvicorn nhận được đúng
> chuỗi ký tự `$PORT` chứ không phải một con số. Nghĩa là biến môi trường không
> được thay thế — mà việc thay thế `$PORT` là công việc của shell. Suy ra: lệnh
> khởi động đang chạy **không qua shell**.
>
> Em kiểm tra `Dockerfile` thì thấy `CMD` của mình đã xử lý đúng:
>
> ```dockerfile
> CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
> ```
>
> Có `sh -c` nên biến chắc chắn nở được. Vậy lệnh đang chạy không phải `CMD` này.
> Em tìm tiếp và thấy trong `railway.toml`:
>
> ```toml
> [deploy]
> startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"
> ```
>
> `startCommand` của Railway **ghi đè `CMD` của Dockerfile** và được thực thi
> trực tiếp, không qua shell — nên `$PORT` giữ nguyên dạng văn bản.
>
> **Cách sửa:** xoá hẳn dòng `startCommand`, để Railway dùng `CMD` trong
> Dockerfile. Em chọn xoá thay vì bọc `sh -c` vào `startCommand` vì hai lý do:
> lệnh khởi động chỉ nên định nghĩa ở một nơi, và bản chạy ở máy với bản chạy
> trên cloud giống hệt nhau thì mới tránh được cảnh "máy tôi chạy được".
>
> Còn một hệ quả nữa nếu giữ `startCommand`: nó thiếu `exec`. Không có `exec`,
> `sh` giữ PID 1 còn uvicorn là tiến trình con, nên SIGTERM từ Railway sẽ đến
> `sh` chứ không đến Python — và toàn bộ phần graceful shutdown em làm ở CP4 sẽ
> không bao giờ chạy. Xoá `startCommand` sửa luôn cả vấn đề này.
>
> **Lỗi thứ hai xuất hiện ngay sau đó**, đáng ghi lại vì nó dạy em một chuyện
> khác:
>
> ```
> /bin/sh: 1: exec: docker-entrypoint.sh: not found
> ```
>
> `docker-entrypoint.sh` là entrypoint của image Redis, không có trong image
> Python của em. Chạy `railway status` thì thấy:
>
> ```
> Linked service
> Redis
>     image:  redis:8.2.1
>     volume: redis-volume · /data
> ```
>
> Em đã `railway up` trong khi CLI đang link vào **service Redis** — vì lúc đó
> project mới chỉ có mỗi service này, do `railway add --database redis` tạo ra,
> và CLI tự chọn nó. Toàn bộ image Python của em bị đẩy đè lên Redis. Dấu vết rõ
> nhất là ID volume trong log của cả hai lần deploy đều giống nhau, và đó chính
> là volume `/data` của Redis.
>
> Sửa: tạo service riêng bằng `railway add --service agent`, deploy bằng
> `railway up --service agent`, rồi khôi phục Redis bằng
> `railway redeploy --service Redis --from-source` — cờ `--from-source` kéo lại
> từ image gốc `redis:8.2.1` thay vì chạy lại bản deploy hỏng.
>
> Bài học chung của cả hai lỗi: trên cloud, **cấu hình ở tầng nền tảng ghi đè
> cấu hình trong repo**, và nó không nói cho bạn biết. Log chỉ hiện triệu chứng;
> phải đối chiếu xem lệnh thực sự đang chạy là lệnh nào và chạy ở đâu.

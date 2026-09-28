# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời chi tiết cho từng câu hỏi bên dưới.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Chí Vi  Mã học viên: 2A202602968

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống thực tế: Khi deploy ứng dụng lên môi trường Cloud/Production (như Railway hoặc Kubernetes), người phụ trách cấu hình vô tình quên thêm biến môi trường `AGENT_API_KEY` hoặc gõ sai tên biến (ví dụ gõ nhầm thành `API_KEY`). Nếu trong mã nguồn ta để giá trị mặc định là `"changeme"` hoặc chuỗi rỗng `""`, ứng dụng vẫn sẽ khởi động thành công, endpoint `/health` vẫn trả về 200 OK. Kết quả là service đi vào hoạt động công khai nhưng lại được bảo vệ bằng mật khẩu mặc định `"changeme"`. Bất kỳ ai trên internet hoặc các bot quét tự động đều có thể gửi hàng nghìn request với header `X-API-Key: changeme` để gọi LLM miễn phí, đốt sạch toàn bộ hạn mức tài chính của dự án trước khi đội ngũ kỹ thuật kịp phát hiện. Ngược lại, khi không có giá trị mặc định, Pydantic `Settings` sẽ raise `ValidationError` ngay lúc ứng dụng vừa nạp file cấu hình (Fail Fast). Container lập tức crash và nền tảng cloud sẽ gắn cờ `Deploy Failed` hoặc `CrashLoopBackOff`, buộc kỹ sư phải cung cấp đúng API key trước khi dịch vụ được cấp phát traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thực tế thu được từ hệ thống:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:23:41.258901+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 35, "cost_usd": 0.00012}
```

Hai việc làm được với dòng log có cấu trúc (JSON structured logging) mà log văn bản thuần túy không thể làm được:
1. **Truy vấn, lọc và tổng hợp định lượng tự động bằng máy (Machine Parsable & Queryable)**: Các hệ thống quản lý log tập trung (như Datadog, Grafana Loki, AWS CloudWatch) hoặc công cụ dòng lệnh như `jq` có thể bóc tách các trường dữ liệu tức thì mà không cần viết biểu thức chính quy (regex). Ví dụ ta có thể chạy `jq 'select(.cost_usd > 0.001)'` để lọc các request tốn kém bất thường, hoặc nhóm và tính tổng token tiêu thụ theo từng `user_id` trong một khoảng thời gian nhất định.
2. **Thiết lập cảnh báo thời gian thực và kích hoạt tự động hóa (Real-time Metric Alerting)**: Có thể dễ dàng định nghĩa các quy tắc giám sát tự động dựa trên các trường JSON. Ví dụ, nếu trường `level` chuyển thành `"error"` vượt quá 5 lần/phút hoặc trường `cost_usd` của một user vượt ngưỡng ngân sách đột ngột, hệ thống giám sát có thể kích hoạt webhook gửi cảnh báo Slack/PagerDuty cho kỹ sư trực hệ thống hoặc tự động block tạm thời user đó.

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
| 1 stage (bản đầu) | ~1.05 GB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch khổng lồ (~865 MB) giữa hai phiên bản bao gồm:
1. **Các công cụ biên dịch và header hệ thống (Build Tools & Compilers)**: Base image đầy đủ `python:3.11` chứa sẵn trình biên dịch C/C++ (GCC, G++), thư viện `build-essential`, các header file của Linux và các công cụ phát triển phần mềm. Chúng chỉ cần thiết khi pip cài đặt và biên dịch các thư viện Python, nhưng hoàn toàn vô dụng khi chạy ứng dụng (runtime).
2. **Bộ nhớ đệm và file rác sinh ra khi cài đặt (Pip Cache & Wheel Archives)**: Trong quá trình chạy `pip install`, pip tải về các file nén `.whl`, `.tar.gz` và lưu trữ trong cache. Ở bản multi-stage, toàn bộ các file cache này nằm ở stage `builder` và bị hủy bỏ hoàn toàn.
3. **Sự tinh giản của base image runtime**: Stage runtime sử dụng `python:3.11-slim`, loại bỏ hầu hết các tiện ích bổ sung không cần thiết của Linux (như tài liệu man pages, debug symbols) và chỉ sao chép đúng thư mục `/usr/local` chứa các gói Python đã cài hoàn chỉnh từ builder, tạo ra một image siêu nhẹ và an toàn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với cấu trúc Dockerfile tối ưu đã thiết lập:
- **Các layer được dùng lại từ cache (CACHED)**:
  1. `FROM python:3.11-slim AS builder`
  2. `WORKDIR /build`
  3. `COPY requirements.txt .`
  4. `RUN pip install --no-cache-dir --prefix=/install -r requirements.txt`
  5. `FROM python:3.11-slim AS runtime`
  6. `WORKDIR /app`
  7. `COPY --from=builder /install /usr/local`
  8. `RUN useradd --create-home --uid 10001 appuser`
  Toàn bộ các layer trên hoàn toàn không bị ảnh hưởng vì `requirements.txt` không thay đổi, Docker tái sử dụng cache 100% trong 0 giây.
- **Các layer phải chạy lại**:
  Chỉ từ lệnh `COPY app ./app` trở đi (bao gồm `COPY app`, `COPY utils`, `RUN chown`, `USER`, `EXPOSE`, `HEALTHCHECK`, `CMD`) mới bị invalidate cache và phải chạy lại, toàn bộ quá trình build chỉ mất khoảng 1 đến 2 giây.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`**:
  Khi sửa dù chỉ một ký tự trong `app/main.py`, checksum của thư mục hiện tại thay đổi làm layer `COPY . .` bị cache miss (vỡ cache). Do tính chất phụ thuộc tuần tự của Docker, toàn bộ các layer phía sau nó đều bị hủy cache, buộc lệnh `RUN pip install` phải tải và cài đặt lại toàn bộ thư viện từ internet mỗi lần build. Thời gian build sẽ kéo dài từ 2 giây lên 2-3 phút, gây lãng phí băng thông và làm chậm chu kỳ CI/CD nghiêm trọng.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện leo thang đặc quyền từ ứng dụng ra máy host:
1. **Khai thác lỗ hổng cấp ứng dụng**: Ứng dụng Python xuất hiện một lỗ hổng thực thi mã từ xa (RCE - Remote Code Execution), ví dụ thông qua việc deserialize dữ liệu không an toàn (`pickle`), sử dụng hàm `eval()`/`exec()`, hoặc tiêm lệnh hệ thống qua `os.system()` / `subprocess`. Kẻ tấn công gửi payload độc hại và kích hoạt được một interactive shell (ví dụ `/bin/sh`) bên trong container.
2. **Khai thác quyền hạn trong container**: Mặc định container chạy dưới quyền user `root` (UID 0). Bên trong container, kẻ tấn công có toàn quyền quản trị: chỉnh sửa file cấu hình hệ thống, đọc tất cả biến môi trường bí mật, cài đặt thêm các công cụ tấn công mạng (như `nmap`, `netcat`).
3. **Thoát container (Container Breakout) ra máy host**: Do nhân Linux (Linux Kernel) được chia sẻ chung giữa container và máy host, nếu container có mount một số volume nhạy cảm (như `/var/run/docker.sock`, `/etc`), hoặc container chạy ở chế độ privileged, hay kernel có lỗ hổng (như Dirty COW, cgroup release_agent breakout), quyền root (UID 0) trong container tương ứng với quyền root (UID 0) trên host. Kẻ tấn công có thể ghi đè file hệ thống của host, thoát khỏi cgroup/namespace và nắm quyền kiểm soát toàn bộ máy chủ vật lý.
4. **Vị trí lệnh `USER` cắt đứt chuỗi tấn công**:
   Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi tấn công ngay tại **bước 2**. Dù kẻ tấn công có khai thác thành công RCE trong code Python, shell thu được chỉ là quyền của một người dùng thông thường không có đặc quyền. Kẻ tấn công không thể cài thêm phần mềm, không thể ghi vào các thư mục hệ thống của container, không có các Linux Capabilities nguy hiểm, và việc khai thác các lỗ hổng kernel để thoát container bị vô hiệu hóa hoàn toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Trong 2 giây liên tiếp, một người dùng có thể gửi tối đa **20 request** (gấp đôi hạn mức cho phép trên mỗi phút).

**Cách đạt được con số đó**:
- Thuật toán đếm theo phút đồng hồ (Fixed Window Counter) chia thời gian thành các khung cố định và reset bộ đếm về 0 vào giây thứ 00 của mỗi phút (ví dụ 10:00:00, 10:01:00).
- Người dùng có thể chờ đến giây cuối cùng của khung giờ thứ nhất, cụ thể là lúc `10:00:59`, và gửi dồn dập 10 request. Hệ thống kiểm tra thấy trong phút 10:00 người này mới gọi 10 lần (vừa đúng hạn mức 10/phút) nên cho phép tất cả 10 request đi qua.
- Ngay một giây sau đó, khi đồng hồ điểm `10:01:00`, bước sang khung giờ mới, bộ đếm tự động reset về 0. Người dùng lập tức gửi tiếp 10 request nữa tại giây `10:01:00`. Hệ thống ghi nhận đây là phút 10:01 và người dùng mới dùng 10/10 lượt, nên tiếp tục cho qua.
- Như vậy, trong khoảng thời gian vỏn vẹn 2 giây (từ `10:00:59` đến `10:01:00`), hệ thống đã phải xử lý 20 request từ cùng một người dùng. Đợt traffic tăng đột biến (traffic burst) này có thể làm nghẽn server và cạn kiệt tài nguyên downstream. Thuật toán Sliding Window với Redis Sorted Set giải quyết triệt để lỗi này bằng cách luôn tính chính xác số request trong đúng 60 giây trôi ngược từ thời điểm hiện tại (`now - 60s` đến `now`).

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

**Sự khác biệt cốt lõi**:
- **Rate Limit**: Kiểm soát **tần suất và lưu lượng truy cập (Throughput / Requests Per Minute)** nhằm bảo vệ tính khả dụng và khả năng chịu tải của hạ tầng mạng/máy chủ khỏi nguy cơ bị nghẽn hoặc quá tải tức thời. Đơn vị đo lường là số lần gọi API trong một đơn vị thời gian ngắn (ví dụ: 10 req/phút).
- **Cost Guard**: Kiểm soát **ngân sách tài chính thực tế (Financial Budget / Token Usage)** nhằm ngăn ngừa rủi ro cạn kiệt ngân sách do việc tiêu thụ token LLM của người dùng. Đơn vị đo lường là tiền tệ (USD) tính theo chu kỳ thanh toán (ví dụ: 10.0 USD/tháng).

**Hai tình huống minh họa đối lập**:
1. *Tình huống Rate Limit cho qua nhưng Cost Guard chặn*:
   Một người dùng cả tuần không gửi câu hỏi nào. Đến đầu tuần, họ gửi một request duy nhất (tần suất là 1 req/phút, hoàn toàn nằm trong hạn mức 10 req/phút nên Rate Limit cho qua). Tuy nhiên, người dùng này đã tích lũy chi phí sử dụng trong tháng đạt 9.95 USD (trên tổng hạn mức 10.0 USD), và request lần này gửi kèm một tài liệu khổng lồ (vài chục nghìn tokens) với chi phí ước tính là 0.10 USD. Tổng chi phí sẽ vượt quá ngân sách tháng (9.95 + 0.10 = 10.05 > 10.0 USD) -> Cost Guard lập tức chặn lại và trả về lỗi `402 Payment Required`.
2. *Tình huống Cost Guard cho qua nhưng Rate Limit chặn*:
   Đầu tháng mới, tài khoản của người dùng còn nguyên 10.0 USD hạn mức. Người dùng viết một script tự động gửi liên tục 15 câu hỏi ngắn (mỗi câu chỉ 5 token, chi phí cực nhỏ cỡ $0.00002 cho mỗi câu, tổng chi phí mới chỉ $0.0003, rất xa ngưỡng 10 USD -> Cost Guard thấy tiền vẫn còn nên cho qua). Nhưng vì 15 request này được gửi chỉ trong vòng 10 giây (vượt quá 10 req/phút) -> Rate Limit sẽ kích hoạt từ request thứ 11 và chặn lại với lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Chuỗi phản ứng dây chuyền thảm họa (Cascading Failure) khi gộp chung `/health` và `/ready`:

1. **Giây 0**: Redis gặp sự cố tạm thời (ví dụ: rớt mạng nội bộ, quá tải bộ nhớ, hoặc đang thực hiện snapshot lưu đĩa) và tạm dừng phản hồi trong 30 giây.
2. **Giây 5**: Cơ chế Liveness Probe của orchestrator (Kubernetes, Docker Swarm hoặc nền tảng Cloud) định kỳ gọi endpoint `/health` của cả 3 container agent. Do `/health` kiểm tra kết nối tới Redis và Redis đang chết, cả 3 container đều đồng loạt trả về lỗi (HTTP 500 hoặc 503).
3. **Giây 10**: Nhận thấy `/health` thất bại, orchestrator kết luận rằng cả 3 tiến trình ứng dụng đều đã bị treo/hỏng nghiêm trọng (deadlock hoặc zombie process) và quyết định ra lệnh khởi động lại (restart) toàn bộ 3 container cùng một lúc.
4. **Giây 15 - 25**: 3 container mới được tạo ra và bắt đầu quá trình nạp mã nguồn, khởi động server. Tuy nhiên, lúc này Redis vẫn chưa phục hồi xong (vẫn nằm trong khoảng 30s sự cố). Khi các container mới vừa khởi động xong, orchestrator lại gửi liveness probe vào `/health`, lại tiếp tục thất bại, và orchestrator lại tiếp tục tiêu diệt và restart các container lần thứ hai (rơi vào trạng thái `CrashLoopBackOff`).
5. **Giây 30**: Redis phục hồi hoàn toàn và sẵn sàng hoạt động. Tuy nhiên, lúc này cả 3 container ứng dụng đều đang trong trạng thái restart dở dang hoặc bị orchestrator phạt lùi thời gian khởi động (back-off penalty). Người dùng bên ngoài hoàn toàn không thể truy cập dịch vụ, một sự cố tạm thời của Redis đã bị phóng đại thành một đợt ngừng hoạt động toàn diện (Total Outage) của toàn bộ hệ thống.

*Sự phân tách đúng chuẩn*:
`/health` (Liveness) chỉ kiểm tra tiến trình ứng dụng có còn sống và phản hồi HTTP không (không đụng tới Redis) -> orchestrator sẽ không bao giờ restart container một cách oan uổng. `/ready` (Readiness) kiểm tra kết nối Redis: khi Redis chết, `/ready` trả về 503 để Load Balancer tạm thời cô lập instance, ngừng chuyển traffic vào container. Khi Redis phục hồi ở giây 30, `/ready` tự động trả về 200 và Load Balancer mở lại traffic ngay lập tức mà không có bất kỳ container nào bị restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

**Hiện tượng quan sát được khi lưu state bằng dict trong RAM của từng container**:
- Khi scale lên 3 instance (Container A, Container B, Container C), Load Balancer (hoặc Docker internal round-robin DNS) sẽ phân phối các request kế tiếp nhau lần lượt vào các container khác nhau.
- Mỗi container là một process độc lập với không gian bộ nhớ RAM tách biệt hoàn toàn:
  - **Lần gọi 1**: Request rơi vào Container A. Dict của A chưa có gì -> trả về `history_length = 0`. Container A lưu câu hỏi 1 và câu trả lời 1 vào RAM của nó.
  - **Lần gọi 2**: Request rơi vào Container B. Do B chưa từng nhận request nào của user này, dict của B hoàn toàn rỗng -> trả về `history_length = 0` (thay vì 2). B lưu lượt 2 vào RAM của B.
  - **Lần gọi 3**: Request rơi vào Container C. Dict của C cũng rỗng -> tiếp tục trả về `history_length = 0`. C lưu lượt 3 vào RAM của C.
  - **Lần gọi 4**: Request quay vòng lại Container A. Dict của A chỉ nhớ duy nhất lượt hỏi 1 -> trả về `history_length = 2` (hoàn toàn mất dấu vết của lượt 2 và 3).
  - **Lần gọi 5**: Request rơi vào Container B -> trả về `history_length = 2` (nhưng nội dung ngữ cảnh chỉ có lượt 2).
- **Kết luận**: Giá trị `history_length` sẽ nhảy lộn xộn, bất thường (ví dụ: `0 -> 0 -> 0 -> 2 -> 2 -> 4...`), agent liên tục bị "mất trí nhớ từng phần" tùy thuộc vào việc request rơi ngẫu nhiên vào instance nào.
- Khi chuyển sang kiến trúc Stateless với **Redis tập trung**, tất cả các instance đều đọc và ghi chung vào key `history:{user_id}` trên Redis. Dù request có được điều phối tới bất kỳ container nào, `history_length` luôn tăng trưởng nhất quán và chuẩn xác: `0 -> 2 -> 4 -> 6 -> 8...`.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Thông báo lỗi gặp phải**:
`Application failed to respond / Healthcheck timeout (HTTP 502 Bad Gateway / 503 Service Unavailable)` sau khi triển khai Dockerfile lên nền tảng cloud (Railway/Render).

**Cách tìm ra nguyên nhân**:
1. Truy cập vào mục **Deploy Logs** trên dashboard quản trị của nền tảng cloud để kiểm tra luồng khởi động của ứng dụng.
2. Log hiển thị dòng thông báo: `INFO: Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)`.
3. Phân tích nguyên nhân kỹ thuật:
   - Trong Dockerfile gốc, lệnh khởi chạy được hardcode là `uvicorn app.main:app --host 0.0.0.0 --port 8000`. Khi chạy trên cloud, nền tảng phân bổ một cổng ngẫu nhiên cho container thông qua biến môi trường `$PORT` (ví dụ `PORT=6542`). Do ứng dụng chỉ lắng nghe ở cổng cố định `8000`, bộ định tuyến reverse proxy của cloud không thể kết nối tới ứng dụng tại cổng `$PORT`, dẫn đến việc probe kiểm tra sức khỏe bị timeout và đánh dấu container thất bại.
   - Thêm vào đó, nếu cấu hình host là `127.0.0.1` (loopback interface), server chỉ chấp nhận các kết nối nội bộ sinh ra từ chính container đó, từ chối toàn bộ traffic từ bên ngoài đi qua router của cloud.

**Cách sửa chữa triệt để**:
Cập nhật câu lệnh khởi chạy `CMD` trong Dockerfile để vừa bind vào tất cả các card mạng (`0.0.0.0`), vừa đọc linh hoạt biến môi trường `PORT` do platform cấp phát, có fallback an toàn về 8000 khi chạy local:
```dockerfile
CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
```
Sau khi sửa và kích hoạt build lại, log hiển thị `Uvicorn running on http://0.0.0.0:6542`, liveness probe lập tức nhận được phản hồi 200 OK từ `/health`, trạng thái dịch vụ chuyển sang màu xanh (Active/Running) hoàn hảo.

# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
> Cách trả lời: thay placeholder câu trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Trường Bảo  Mã học viên: 2A202602540

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy ứng dụng lên Cloud (Render/Railway), lập trình viên quên thêm biến môi trường `AGENT_API_KEY` vào mục cấu hình của dashboard.
- Nếu để giá trị mặc định `"changeme"`: Ứng dụng vẫn khởi động bình thường và orchestrator báo service "Healthy". Tuy nhiên, bot quét Internet hoặc người ngoài có thể dễ dàng đoán ra khóa mặc định `"changeme"`, gửi hàng ngàn request gọi vào endpoint `/ask` và sử dụng dịch vụ miễn phí. Hậu quả là tài khoản LLM bị cạn kiệt ngân sách hoặc phát sinh hóa đơn chi phí khổng lồ trước khi lập trình viên kịp nhận ra.
- Với cơ chế Fail-fast (không có giá trị mặc định): Pydantic ném lỗi `ValidationError` ngay lúc khởi động làm container dừng lại lập tức. Lập trình viên sẽ nhìn thấy lỗi deploy ngay trong log và bổ sung key an toàn trước khi bất kỳ ai trên Internet có thể tiếp cận API.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:00:27.420658+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

Hai việc làm được với dòng log có cấu trúc trên mà lệnh `print()` không làm được:
1. **Lọc, tìm kiếm và truy vấn nâng cao tự động:** Các hệ thống thu thập log (như Datadog, Grafana Loki, CloudWatch) có thể parse trực tiếp các trường JSON để lọc ra các request có chi phí cao (`cost_usd > 0.01`), đếm số lượng request theo từng `user_id` cụ thể, hoặc tính toán tổng chi phí phát sinh trong ngày mà không cần phải viết regex bóc tách chuỗi phức tạp.
2. **Thiết lập cảnh báo và biểu đồ giám sát theo thời gian thực:** Có thể tổng hợp metric từ các trường số liệu (`tokens_in`, `tokens_out`, `cost_usd`) để vẽ dashboard theo dõi lưu lượng, đồng thời cài đặt ngưỡng cảnh báo tự động gửi về Slack/Email nếu tổng số token hoặc tỷ lệ request lỗi tăng đột biến trong 5 phút.

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
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~750 MB) bao gồm:
1. Các công cụ và trình biên dịch C/C++ (`gcc`, `build-essential`), python headers và package dev cần dùng khi biên dịch các thư viện Python có C-extensions nhưng hoàn toàn không cần thiết ở môi trường runtime.
2. Toàn bộ file cache tạm thời của pip (`pip cache`), wheel files tải về trong quá trình cài đặt dependencies.
3. Sự khác biệt giữa base image Debian đầy đủ (chứa nhiều tiện ích shell, gói hệ thống thừa) và base image `python:3.11-slim` chỉ giữ lại nhân runtime tối thiểu để chạy Python.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py`:
  - Các layer trước đó trong stage builder gồm `FROM`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install ...` và base image của stage runtime đều được dùng lại hoàn toàn từ cache (`CACHED`).
  - Layer bắt đầu phải chạy lại (cache invalidated) là: `COPY app ./app` và các bước kế tiếp (`COPY utils ./utils`, `RUN useradd`, `USER appuser`).
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  Mỗi lần sửa bất kỳ một ký tự nào trong mã nguồn ứng dụng, layer `COPY . .` sẽ bị thay đổi checksum, làm Docker hủy bỏ toàn bộ cache từ bước đó trở đi. Kết quả là lệnh `RUN pip install` bắt buộc phải chạy lại từ đầu (tải và cài đặt lại toàn bộ thư viện qua mạng, mất thêm từ 1 đến vài phút cho mỗi lần build, làm chậm chu kỳ phát triển và CI/CD).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Kẻ tấn công phát hiện một lỗ hổng thực thi mã từ xa (RCE) trong code Python (ví dụ: command injection hoặc deserialize pickle không an toàn).
2. Kẻ tấn công gửi payload để chiếm quyền điều khiển shell trong container. Vì container mặc định chạy bằng `root` (UID 0), shell này có đầy đủ đặc quyền root bên trong namespace của container.
3. Kẻ tấn công lợi dụng đặc quyền root này để khai thác các lỗ hổng container escape (như cgroup vulnerabilities, kernel exploit hoặc truy cập vào Docker socket nếu bị mount) để vượt ra khỏi ranh giới container và leo thang chiếm quyền root trên chính máy host thực tế.

Lệnh `USER appuser` cắt đứt chuỗi tấn công ở bước 2: Nó chuyển tiến trình ứng dụng sang chạy dưới định danh user thường không đặc quyền (UID 10001). Ngay cả khi mã độc được thực thi, kẻ tấn công cũng chỉ có quyền hạn tối thiểu của `appuser`, không thể can thiệp vào các file nhạy cảm của hệ điều hành container và không đủ quyền để thực hiện các cuộc tấn công container escape.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được con số đó:
- Vào giây `10:00:59` (giây cuối cùng của phút thứ nhất): Người dùng gửi dồn dập 10 request. Bộ đếm ghi nhận 10 request cho phút 10:00 (vẫn đúng luật không vượt quá hạn mức).
- Khi đồng hồ chuyển sang `10:01:00`, bộ đếm theo phút tự động reset về 0.
- Vào giây `10:01:00` hoặc `10:01:01` (giây đầu tiên của phút thứ hai): Người dùng gửi tiếp 10 request nữa. Bộ đếm ghi nhận 10 request cho phút 10:01 (vẫn hợp lệ).
- Tổng kết: Trong vòng 2 giây (từ 10:00:59 đến 10:01:01), hệ thống đã phải gánh tới 20 request, gây quá tải gấp đôi công suất thiết kế. Thuật toán Sliding Window loại bỏ được lỗ hổng này bằng cách luôn tính lùi 60 giây từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Điểm khác nhau:** Rate Limiting kiểm soát **tần suất/số lượng request** trong một đơn vị thời gian (nhằm chống DDoS, bảo vệ tài nguyên tính toán mạng); Cost Guard kiểm soát **chi phí tài chính phát sinh (USD)** tích lũy trong kỳ thanh toán (nhằm ngăn chặn việc vượt trần ngân sách gọi mô hình LLM).
- **Tình huống Rate limit cho qua nhưng Cost guard chặn:**
  Người dùng chỉ gửi đúng 1 request trong ngày (tốc độ 1 req/ngày << 10 req/phút, Rate limit cho qua), nhưng câu hỏi kèm theo văn bản tài liệu dài 100.000 tokens khiến chi phí ước tính vượt quá số dư ngân sách tháng còn lại của người dùng đó -> Cost guard chặn lại với lỗi 402.
- **Tình huống Cost guard cho qua nhưng Rate limit chặn:**
  Người dùng gửi liên tục 15 request câu hỏi siêu ngắn trong 3 giây (mỗi câu chỉ tốn 0.00001$, tổng chi phí 15 câu chỉ khoảng 0.00015$ << 10$ ngân sách tháng, Cost guard cho qua), nhưng vi phạm hạn mức tần suất 10 request/phút -> Rate limit chặn ngay từ request thứ 11 với lỗi 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. **Giây 0:** Redis gặp sự cố mạng hoặc khởi động lại, tạm thời mất kết nối trong 30 giây.
2. **Giây 5:** Orchestrator (Docker/Kubernetes) gửi định kỳ liveness check tới endpoint `/health` của cả 3 container agent. Do `/health` kiểm tra Redis và thấy lỗi kết nối, cả 3 container đều đồng loạt trả về mã lỗi 503 (Unhealthy).
3. **Giây 10:** Orchestrator cho rằng toàn bộ 3 process ứng dụng đã bị tê liệt/hỏng hóc, lập tức gửi lệnh restart cứng cả 3 container cùng lúc.
4. **Giây 15 - 25:** Cả 3 container mới khởi động lại và lại thăm dò Redis (lúc này Redis vẫn đang offline), dẫn đến health check lại tiếp tục fail và orchestrator lại kích hoạt vòng lặp restart (CrashLoopBackOff).
5. **Giây 30:** Khi Redis đã hoạt động bình thường trở lại, toàn bộ cụm container vẫn đang trong trạng thái khởi động lại dở dang, gây ra tình trạng gián đoạn toàn bộ dịch vụ (Outage hoàn toàn) dù bản thân mã nguồn của 3 container không hề có lỗi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử hội thoại được lưu trong một dict Python (bộ nhớ RAM cục bộ của từng process):
- Vì 3 instance agent chạy độc lập trên 3 container với 3 vùng nhớ RAM hoàn toàn riêng biệt:
  - Lượt 1: Request rơi vào container A -> `history_length = 0` (A lưu vào dict của A).
  - Lượt 2: Load balancer phân phối request sang container B -> `history_length = 0` (vì RAM của container B chưa có dữ liệu của user này).
  - Lượt 3: Request rơi vào container C -> `history_length = 0`.
  - Lượt 4: Request quay lại container A -> `history_length = 2` (A đọc từ dict của A).
- Hiện tượng quan sát được: `history_length` sẽ nhảy lộn xộn, lúc tăng lúc giảm về 0 một cách ngẫu nhiên tùy thuộc vào việc load balancer điều hướng request đến container nào, khiến agent bị "mất trí nhớ" bất thường. Khi đưa sang Redis, tất cả container cùng đọc ghi một nguồn dữ liệu duy nhất nên `history_length` luôn tăng đều đặn (0, 2, 4, 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: Lỗi build container trên cloud/Docker với thông báo:
```text
ERROR: Cannot install uvicorn[standard]... because these package versions have conflicting dependencies.
The conflict is caused by: uvicorn[standard] 0.54.0 depends on watchfiles>=0.20
ERROR: ResolutionImpossible: for help visit https://pip.pypa.io/en/latest/topics/dependency-resolution/
```
- **Nguyên nhân tìm ra:** Khi kiểm tra build log, phát hiện base image `python:3.11-slim` đi kèm phiên bản `pip 24.0` cũ. Thuật toán dependency backtracking của pip 24.0 bị lỗi vòng lặp khi giải quyết các phụ thuộc mở rộng của `uvicorn[standard]` (đặc biệt là gói `watchfiles` và `anyio`), dẫn đến việc thử lùi qua hơn 30 phiên bản uvicorn rồi kết luận nhầm là xung đột bất khả thi.
- **Cách sửa:** Trong `Dockerfile` tại stage `builder`, thêm bước nâng cấp `pip` lên phiên bản mới nhất trước khi chạy cài đặt `requirements.txt`:
  ```dockerfile
  RUN pip install --no-cache-dir --upgrade pip && \
      pip install --no-cache-dir --prefix=/install -r requirements.txt
  ```
  Sau khi nâng cấp lên `pip 26.2+`, pip giải quyết toàn bộ cây phụ thuộc trong vòng vài giây mà không xảy ra xung đột hay lỗi backtracking.

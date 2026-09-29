# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các khối placeholder bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Khánh Duy  Mã học viên: 2A202602403

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy ứng dụng lên Cloud (như Railway hoặc Render), lập trình viên vô tình quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard.
- Nếu để giá trị mặc định là `"changeme"`: Service vẫn khởi động thành công và báo trạng thái Healthy. Nhưng lúc này API sử dụng khóa bí mật mặc định đã bị công khai trong code. Các bot quét tự động trên Internet có thể dò ra giá trị phổ biến này và gửi hàng nghìn request gọi vào endpoint `/ask`. Hậu quả là kẻ xấu thoải mái khai thác mô hình LLM mà bạn phải trả tiền, dẫn đến cạn kiệt ngân sách hoặc lộ dữ liệu, và bạn chỉ phát hiện ra khi nhận hóa đơn thẻ tín dụng cuối tháng.
- Khi không có giá trị mặc định (Fail-fast): Ứng dụng lập tức dừng (crash) ngay ở giai đoạn khởi động với lỗi `ValidationError: Field required` của Pydantic. Quá trình deploy lập tức báo đỏ (Failed) trước mắt bạn. Lỗi xuất hiện ngay tại thời điểm deploy giúp bạn nhận ra thiếu sót và bổ sung secret ngay, ngăn chặn triệt để nguy cơ service bị lộ trên Internet mà không có bảo vệ.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được thực tế:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T02:51:33.456128+00:00", "user_id": "sv-123", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.0000288}`

Hai việc làm được với dòng log có cấu trúc này mà lệnh `print()` thuần túy không làm được:
1. **Truy vấn, lọc và thống kê định lượng tự động bằng máy (Structured Querying & Aggregation):** Các công cụ phân tích log tập trung (như CloudWatch, Datadog, Grafana Loki, ELK) có thể tự động parse JSON để thực hiện các phép tính phức tạp như: tính tổng chi phí `cost_usd` của từng `user_id` trong ngày/tháng, vẽ biểu đồ số lượng token tiêu thụ theo thời gian, hoặc lọc nhanh các request có thời gian xử lý bất thường mà không cần viết regex phức tạp và dễ vỡ.
2. **Thiết lập cảnh báo tự động theo ngưỡng (Automated Metric Alerting):** Hệ thống giám sát có thể tạo cảnh báo thời gian thực gửi về Slack/PagerDuty dựa trên các trường cụ thể: ví dụ kích hoạt cảnh báo khẩn cấp khi phát hiện `level == "error"` vượt ngưỡng 5% trong 5 phút, hoặc khi một user tiêu tốn `cost_usd > 0.5` trong một request đơn lẻ.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 168 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~852 MB) bao gồm:
1. **Bộ công cụ phát triển và biên dịch hệ điều hành:** Base image ban đầu `python:3.11` được xây dựng trên bản Debian đầy đủ, chứa trình biên dịch (`gcc`, `g++`, `make`, `build-essential`), thư viện C header (`linux-libc-dev`, `python3-dev`), công cụ gỡ lỗi và gói tiện ích không cần thiết cho môi trường chạy. Bản multi-stage sử dụng `python:3.11-slim` đã loại bỏ hoàn toàn các gói phát triển này.
2. **File rác và bộ nhớ đệm (cache) trong quá trình cài đặt dependencies:** Ở stage `builder`, quá trình `pip install` sinh ra các file wheel trung gian, cache tải về và mã nguồn giải nén. Nhờ cơ chế multi-stage, stage `runtime` chỉ copy thư mục kết quả `/install` sang `/usr/local`, để lại toàn bộ cache và trình biên dịch ở stage trước mà không đưa vào image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại:
  - Các layer từ đầu cho đến `RUN pip install --no-cache-dir ...` đều được **dùng lại hoàn toàn từ cache (`CACHED`)** vì file `requirements.txt` không hề thay đổi.
  - Chỉ có layer `COPY app ./app` và các bước kế tiếp bị invalidate cache và phải chạy lại. Nhờ đó việc build lại chỉ mất chưa đầy 1 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Khi sửa 1 ký tự trong `app/main.py`, checksum của thư mục hiện tại thay đổi khiến layer `COPY . .` bị vỡ cache.
  - Theo nguyên lý của Docker, khi một layer bị vỡ cache thì toàn bộ các layer tiếp theo phía sau nó đều bị ép phải chạy lại từ đầu. Do đó, Docker sẽ phải chạy lại `RUN pip install`, tải lại toàn bộ dependencies qua mạng và cài đặt lại từ đầu (mất từ 30 giây đến vài phút cho mỗi lần sửa dù chỉ 1 ký tự).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện dẫn đến chiếm quyền máy host:
  1. Ứng dụng Python xuất hiện lỗ hổng bảo mật (ví dụ: Command Injection, Insecure Deserialization, hoặc RCE từ một thư viện bên thứ ba).
  2. Kẻ tấn công khai thác lỗ hổng để thực thi lệnh tùy ý. Do container chạy dưới user mặc định `root` (UID 0), tiến trình mã độc sẽ sở hữu toàn quyền root bên trong container.
  3. Từ quyền root container, kẻ tấn công khai thác các lỗ hổng nhân hệ điều hành Linux (kernel exploit như Dirty COW, cgroup escape) hoặc cấu hình sai (như mount `docker.sock` hoặc quyền thừa Capabilities).
  4. Sau khi thoát khỏi ranh giới container (Container Breakout), do Linux ánh xạ UID 0 trong container trùng với UID 0 của máy host (nếu không bật User Namespaces), kẻ tấn công chính thức trở thành root trên máy host, kiểm soát toàn bộ server.
- Lệnh `USER appuser` cắt đứt chuỗi ở đâu:
  - Lệnh này cắt đứt chuỗi ngay tại **Bước 2**: Khi lỗ hổng bị khai thác, mã độc chỉ chạy dưới quyền của `appuser` (UID 10001) - một user bị giới hạn đặc quyền nghiêm ngặt. Kẻ tấn công không thể sửa file hệ thống, không thể tương tác với các socket quản trị và việc leo thang đặc quyền để escape khỏi container từ non-root user gần như là bất khả thi.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.

Giải thích cách đạt được (Lỗ hổng ranh giới - Boundary Problem của Fixed Window):
- Giả sử khung thời gian phút là từ phút $M$ (`10:00:00` đến `10:00:59`) và phút $M+1$ (`10:01:00` đến `10:01:59`).
- Tại giây `10:00:59` (giây cuối cùng của phút $M$), người dùng gửi liên tiếp **10 request**. Hệ thống kiểm tra phút $M$ chưa quá 10 request nên cho qua toàn bộ.
- Ngay sau đó 1 giây, tại `10:01:00` (giây đầu tiên của phút $M+1$), bộ đếm reset về 0. Người dùng lập tức gửi tiếp **10 request**. Hệ thống kiểm tra thấy phút $M+1$ mới bắt đầu nên tiếp tục cho qua cả 10 request.
- Kết quả: Từ `10:00:59` đến `10:01:01` (chỉ trong 2 giây), hệ thống phải hứng chịu tới 20 request (gấp đôi tải thiết kế). Thuật toán Sliding Window (cửa sổ trượt) ngăn chặn triệt để điều này bằng cách luôn tính tổng request trong khoảng trượt 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau cơ bản:
  - **Rate Limit:** Kiểm soát **tần suất và lưu lượng request trong đơn vị thời gian ngắn** (ví dụ: 10 request/phút) để bảo vệ server khỏi quá tải, chống nghẽn mạng và tấn công DoS.
  - **Cost Guard:** Kiểm soát **tổng chi phí tài chính tích lũy trong khoảng thời gian dài** (ví dụ: $10.0/tháng) để bảo vệ túi tiền của chủ hệ thống trước các chi phí phát sinh từ việc gọi mô hình LLM.
- Tình huống Rate limit cho qua nhưng Cost guard chặn:
  - Một user chỉ gửi 1 request trong vòng 10 phút (tần suất rất thấp, Rate Limit cho qua). Nhưng request này yêu cầu xử lý tài liệu khổng lồ với 100.000 token, hoặc user này trước đó đã tiêu hết $9.98 / $10.0 ngân sách tháng. Lần gọi này dự tính tốn thêm $0.05 ➔ Cost Guard phát hiện vượt ngân sách và lập tức chặn với mã `402 Payment Required`.
- Tình huống Cost guard cho qua nhưng Rate limit chặn:
  - Một user mới tạo tài khoản, ngân sách tháng còn nguyên $10.0. User dùng vòng lặp script gửi liên tục 15 request trong 3 giây (mỗi request chỉ hỏi một từ "hello" tốn $0.00001). Tổng chi phí chỉ vài phần nghìn cent không đáng kể đối với ngân sách tháng, nhưng việc spam request gây nghẽn luồng xử lý của server ➔ Rate Limit lập tức chặn ở request thứ 11 với mã `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự chuỗi sự kiện thảm họa (Cascading Failure):
1. Kết nối giữa các container và Redis bị đứt quãng hoặc Redis bị khởi động lại trong 30 giây.
2. Bộ kiểm tra sức khỏe của Orchestrator (Docker/Kubernetes/Platform) định kỳ gọi probe tới endpoint chung này của cả 3 container agent.
3. Vì endpoint kiểm tra Redis và Redis đang chết, cả 3 container agent đều phản hồi lỗi 503 (hoặc timeout).
4. Do endpoint này đóng vai trò Liveness (sống/chết), Orchestrator cho rằng cả 3 process ứng dụng đều đã hỏng hoàn toàn và đồng loạt ra lệnh **restart cả 3 container**.
5. Trong 30 giây Redis chưa phục hồi, các container vừa khởi động lại tiếp tục bị probe đánh fail và lại bị restart liên tục (CrashLoopBackOff). Toàn bộ hệ thống sập hoàn toàn và không còn container nào trực chiến.
6. Ngay cả khi Redis đã bình thường trở lại, hệ thống vẫn không thể phục vụ ngay lập tức vì các container đang kẹt trong chu kỳ khởi động lại.
(Ngược lại, nếu tách riêng: `/health` vẫn trả 200 để giữ container sống, còn `/ready` trả 503 để Load Balancer chỉ tạm thời ngắt traffic, khi Redis hồi phục hệ thống sẽ lập tức hoạt động lại bình thường mà không cần restart).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless):
  - Mọi request dù được Load Balancer phân bổ ngẫu nhiên vào bất kỳ container nào trong 3 container (A, B hay C), `history_length` vẫn **tăng đều đặn và liên tục**: 0 ➔ 2 ➔ 4 ➔ 6... vì tất cả container đều cùng đọc và ghi chung vào một cơ sở dữ liệu tập trung là Redis.
- Nếu lưu trong một dict Python (Stateful trong RAM):
  - Giá trị `history_length` sẽ **thay đổi thất thường, không đồng nhất và bị nhảy cóc**:
    - Request 1 vào container A: `history_length = 0` (lưu vào RAM của A).
    - Request 2 vào container B: `history_length = 0` (vì RAM của B chưa có dữ liệu của user này ➔ agent bị mất trí nhớ).
    - Request 3 vào container C: `history_length = 0` (RAM của C cũng trống).
    - Request 4 quay lại container A: `history_length = 2`.
  - Hậu quả là AI agent liên tục quên ngữ cảnh người dùng vừa nói ở câu trước, và nếu một container bị restart thì toàn bộ lịch sử của những người dùng lưu trên container đó sẽ mất sạch.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Lỗi Health Check Timeout do ứng dụng cố định cổng lắng nghe thay vì đọc biến môi trường `$PORT` do Cloud Platform chỉ định.
- **Thông báo lỗi trong deploy log:**
  `Timed out waiting for container to become healthy on port 10000` hoặc `Application failed to respond to health check`.
- **Cách tìm ra nguyên nhân:**
  - Kiểm tra tab Runtime Logs trên Dashboard của Render/Railway, thấy dòng log `Uvicorn running on http://0.0.0.0:8000`.
  - Nhận thấy nền tảng cloud tự động cấp phát một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=10000`) và bộ cân bằng tải của cloud sẽ thăm dò `/health` ở cổng này. Tuy nhiên, lệnh CMD trong Dockerfile ban đầu lại cố định cổng 8000 (`--port 8000`). Vì ứng dụng lắng nghe ở 8000 trong khi cloud kiểm tra ở `$PORT`, router không nhận được tín hiệu trả lời và đánh giá container bị treo.
- **Cách sửa:**
  - Sửa lại chỉ thị CMD trong Dockerfile:
    `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  - Việc bọc qua `sh -c` cho phép shell nội suy giá trị biến `$PORT` từ môi trường cloud, đồng thời vẫn giữ giá trị mặc định 8000 khi chạy ở môi trường máy cục bộ. Sau khi sửa và deploy lại, health check thành công ngay lập tức.

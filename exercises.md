# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng gợi ý bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lương Quang Huy  Mã học viên: 2A202602698

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên Cloud (Render/Railway), người cấu hình quên điền biến môi trường `AGENT_API_KEY` vào bảng cấu hình Environment Variables trên dashboard.
- Nếu để mặc định `agent_api_key = "changeme"`: Ứng dụng vẫn khởi động thành công và báo trạng thái xanh (healthy). Khi đó, hệ thống chạy ở môi trường public với khóa bảo vệ mặc định là "changeme". Bất kỳ bot dò quét nào trên internet cũng có thể dùng khóa "changeme" để gọi endpoint `/ask`, gửi hàng nghìn prompt phức tạp và tiêu sạch toàn bộ quota ngân sách mà chủ sở hữu không hề hay biết cho đến khi nhận hóa đơn.
- Với cơ chế "fail fast" (không có giá trị mặc định): Pydantic sẽ ném ngay ngoại lệ `ValidationError` ngay lúc khởi động container. Quá trình deploy lập tức thất bại (crash), hiển thị cảnh báo đỏ trên trang quản lý. Kỹ sư phát hiện ra ngay để bổ sung khóa bảo mật trước khi service nhận traffic, ngăn chặn triệt để rủi ro rò rỉ và lạm dụng API.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:15:30.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.0000288}
```

Hai việc làm được với dòng log có cấu trúc:
1. **Lọc và phân tích số liệu tự động (Aggregation & Metrics):** Các hệ thống thu thập log tập trung (Datadog, Grafana Loki, CloudWatch) có thể bóc tách các trường JSON để chạy truy vấn thống kê, ví dụ: *"Tính tổng chi phí (cost_usd) của user_id='sv-test' trong ngày hôm nay"* hoặc *"Thống kê số lượng token tiêu thụ trung bình theo từng giờ"*.
2. **Thiết lập cảnh báo tự động theo ngưỡng (Automated Alerting):** Có thể cài đặt rule cảnh báo thời gian thực: nếu xuất hiện log có `cost_usd > 0.05` hoặc tỷ lệ log có `level == "error"` vượt quá 5% trong vòng 5 phút, hệ thống sẽ tự động kích hoạt cảnh báo tới Slack/Telegram/PagerDuty của đội vận hành để can thiệp kịp thời.

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
| 1 stage (bản đầu) | ~1050 MB |
| Multi-stage | ~271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (giảm gần 800 MB) bao gồm:
1. **Base image tinh gọn:** Stage runtime sử dụng `python:3.11-slim` (chỉ chứa nhân Debian tối giản và Python runtime) thay vì `python:3.11` bản đầy đủ vốn chứa rất nhiều công cụ phát triển, tài liệu hướng dẫn, và tiện ích hệ thống không cần thiết lúc chạy.
2. **Loại bỏ công cụ build và cache tạm thời:** Ở stage builder, các công cụ biên dịch (compiler), file tạm của `pip` (cache wheel, build artifacts) đã được cách ly. Khi sang stage runtime, ta chỉ copy thư mục kết quả `/install` sang `/usr/local`, hoàn toàn bỏ lại stage builder và toàn bộ rác phát sinh trong quá trình build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Các layer được dùng lại từ cache (CACHED):**
  + Toàn bộ stage `builder`: `FROM python:3.11-slim`, `WORKDIR /build`, `COPY requirements.txt .`, `RUN pip install ...` (vì `requirements.txt` không đổi).
  + Ở stage `runtime`: `FROM python:3.11-slim`, `WORKDIR /app`, `RUN useradd ...`, `COPY --from=builder /install /usr/local`.
- **Các layer phải chạy lại:**
  + `COPY app ./app` (vì file `app/main.py` bị sửa đổi, mã hash của thư mục `app` thay đổi làm mất cache từ layer này).
  + Các layer tiếp theo bên dưới nó: `COPY utils ./utils`, `RUN chown -R appuser:appuser /app`, `USER appuser`, `HEALTHCHECK`, `CMD`.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  Mỗi khi sửa một ký tự trong `app/main.py`, layer `COPY . .` sẽ bị invalid cache. Do tính chất phân tầng của Docker, layer `RUN pip install` nằm bên dưới cũng bắt buộc phải chạy lại từ đầu. Kết quả là máy tính phải tải lại và cài đặt lại toàn bộ thư viện mỗi lần sửa code, làm thời gian build tăng từ 1–2 giây lên mất 2–5 phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện leo thang đặc quyền khi chạy root:**
  1. Kẻ tấn công phát hiện một lỗ hổng trong ứng dụng Python (ví dụ: Remote Code Execution qua deserialization hoặc command injection).
  2. Kẻ tấn công thực thi mã độc và chiếm được shell điều khiển bên trong container.
  3. Vì container chạy mặc định bằng quyền `root` (UID 0), kẻ tấn công sở hữu toàn bộ đặc quyền root trong container.
  4. Nếu container có mount các volume từ máy host (như Docker socket `/var/run/docker.sock`, filesystem host) hoặc có lỗ hổng thoát container (container escape / kernel vulnerability), quyền root trong container sẽ tương đương với quyền root trên máy host Linux. Kẻ tấn công kiểm soát toàn bộ server host và đánh cắp dữ liệu của các container khác.
- **Lệnh `USER appuser` cắt đứt chuỗi ở bước 3:** Khi chuyển sang user thường không có đặc quyền (UID 10001), kẻ tấn công nếu chiếm được shell chỉ có quyền của user bị cô lập: không thể ghi vào các file hệ thống, không thể truy cập socket của Docker daemon, không thể mount thiết bị, từ đó ngăn chặn hoàn toàn việc leo thang chiếm quyền root máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
- **Cách đạt được:**
  + Người dùng gửi 10 request ở giây cuối cùng của phút thứ nhất: lúc `10:00:59`. Vì trong phút 10:00 mới chỉ có 10 request nên hệ thống cho qua toàn bộ.
  + Ngay khi đồng hồ điểm sang `10:01:00`, bộ đếm của hệ thống bị reset về 0.
  + Lúc `10:01:01`, người dùng gửi tiếp 10 request nữa. Hệ thống tính đây là thuộc phút 10:01 nên vẫn cho qua toàn bộ.
  + Tổng cộng: từ `10:00:59` đến `10:01:01` (chỉ trong 2 giây), hệ thống phải chịu tải tới 20 request dồn dập (gấp đôi hạn mức cho phép trên lý thuyết). Cơ chế sliding window (cửa sổ trượt) luôn xét đúng 60 giây gần nhất kể từ thời điểm gửi nên loại bỏ hoàn toàn kẽ hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Sự khác nhau:**
  + **Rate limit:** Kiểm soát **tần suất và số lượng** request trong một đơn vị thời gian ngắn (ví dụ: tối đa 10 request / 60 giây) nhằm bảo vệ tính sẵn sàng và tránh quá tải CPU/network của server.
  + **Cost guard:** Kiểm soát **tổng số tiền chi phí** phát sinh trong một chu kỳ tài chính (ví dụ: tối đa $10.0 / tháng) nhằm bảo vệ ngân sách chi trả cho nhà cung cấp LLM.
- **Tình huống Rate limit cho qua nhưng Cost guard phải chặn:**
  User chỉ gửi đúng 1 request trong 10 phút (tần suất rất thấp, rate limit cho qua thoải mái). Tuy nhiên, câu hỏi kèm context rất dài (100.000 token) khiến chi phí vượt quá ngân sách $10 còn lại của tháng. Lúc này Cost Guard sẽ lập tức chặn lại và trả lỗi `402 Payment Required`.
- **Tình huống Cost guard cho qua nhưng Rate limit phải chặn:**
  Đầu tháng, user còn nguyên $10 ngân sách. User viết script spam 50 request trong vòng 3 giây với câu hỏi siêu ngắn "hi" (tổng chi phí chỉ tốn $0.0001, ngân sách vẫn còn rất nhiều). Nhưng vì tần suất vượt quá 10 req/phút nên Rate limit sẽ chặn từ request thứ 11 và trả về mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Mạng nội bộ chập chờn hoặc Redis bị restart/quá tải dẫn đến mất kết nối trong 30 giây.
2. Endpoint `/health` của cả 3 container agent đồng loạt gọi thử tới Redis và thất bại (trả về lỗi hoặc timeout).
3. Bộ điều phối (Orchestrator như Docker Swarm / Kubernetes / Cloud platform) coi việc `/health` (Liveness probe) thất bại là tiến trình ứng dụng đã chết hoặc bị treo hoàn toàn (deadlock).
4. Orchestrator lập tức ra lệnh **kill và restart đồng loạt cả 3 container agent**.
5. Trong khi các container đang restart, Redis quay trở lại hoạt động bình thường, nhưng lúc này không còn bất kỳ container agent nào đang sống để phục vụ người dùng.
6. Toàn bộ người dùng truy cập vào hệ thống đều gặp lỗi 502/503 Bad Gateway. Một sự cố gián đoạn tạm thời của dependency (Redis) đã bị phóng đại thành sụp đổ toàn bộ hệ thống (**cascading failure**).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Nếu lưu trong Redis (Stateless):** Con số `history_length` sẽ tăng dần đều và liên tục sau mỗi lượt hỏi: `0 -> 2 -> 4 -> 6 -> ...` bất kể request được bộ cân bằng tải phân phối vào container nào trong 3 container.
- **Nếu lưu trong dict Python (Stateful trong RAM container):**
  Con số `history_length` sẽ nhảy loạn xạ và không đồng nhất. Ví dụ:
  + Lượt 1 vào Container A: `history_length` = 0 (A ghi nhớ câu 1, RAM của A có độ dài 2).
  + Lượt 2 bị Load Balancer đẩy sang Container B: `history_length` = 0 (vì B không có dữ liệu của A).
  + Lượt 3 bị đẩy sang Container C: `history_length` = 0.
  + Lượt 4 quay lại Container A: `history_length` = 2.
  Người dùng sẽ thấy AI liên tục bị "mất trí nhớ", lúc thì nhớ lúc thì quên tùy thuộc vào việc request rơi ngẫu nhiên vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Lỗi `Health check timeout / Service failed to bind port` khi container khởi động trên nền tảng Cloud (Render/Railway).
- **Thông báo lỗi trong Runtime Log:** `Timed out waiting for container to become healthy. Service did not listen on port 8000.` hoặc `Application failed to respond on PORT 10000`.
- **Cách tìm ra nguyên nhân:** Đọc log khởi động của platform trên dashboard. Nhận thấy nền tảng cloud tự động cấp phát một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=10000`), nhưng lệnh khởi động cũ trong Dockerfile lại ép cứng `--port 8000` và bind vào `127.0.0.1`.
- **Cách sửa:** Cập nhật lại lệnh CMD trong `Dockerfile` thành: `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`. Lệnh này đảm bảo uvicorn luôn lắng nghe trên tất cả các interface (`0.0.0.0`) và tự động nhận cổng từ biến `$PORT` của Cloud cung cấp (nếu không có thì mặc định về 8000).

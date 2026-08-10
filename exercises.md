# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Mạnh Cường  Mã học viên: 2A202601361

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Render, tôi đã thực sự gặp lỗi `ValidationError: 1 validation error for Settings` vì service chưa có `AGENT_API_KEY`. Ứng dụng dừng ngay trong startup nên tôi phát hiện và thêm secret trong màn hình Environment trước khi có request thật nào được nhận. Nếu code có mặc định `"changeme"`, service có thể vẫn báo deploy thành công nhưng endpoint `/ask` lại được bảo vệ bằng một khóa công khai, dễ đoán; người khác có thể gọi API và làm phát sinh chi phí trước khi tôi nhận ra.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi tạo bằng chính `log_event()` của service là: `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T05:30:03.794058+00:00", "user_id": "sv01", "tokens_in": 43, "tokens_out": 47, "cost_usd": 3.465e-05}`. Với log này, tôi có thể (1) nhóm theo `user_id` rồi cộng `cost_usd` để tìm user tiêu nhiều tiền nhất và (2) lọc/đếm theo `event`, `level`, `timestamp` để tính tỷ lệ lỗi hoặc tạo cảnh báo trong một khoảng thời gian. Dòng `print("đã trả lời xong")` không có các trường máy đọc được để làm hai việc đó.

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
| 1 stage (bản đầu) | khoảng 1.59 GB phần base đã tải |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Image multi-stage build thành công và `docker images` báo đúng 270 MB. BuildKit inspection của bản single-stage dùng `python:3.11` đầy đủ cho thấy chuỗi layer base đã tải khoảng 1.59 GB (các layer chính 930.3, 270.1, 183.3, 94.61, 90.63 và 26.04 MB); build đối chứng bị dừng ở bước `pip install`, nên đây là số layer base thực tế đã inspect chứ không phải kích thước image hoàn chỉnh. Phần chênh lệch chủ yếu là hệ điều hành/base image đầy đủ và các công cụ chỉ cần lúc build. Bản multi-stage chỉ đưa thư viện đã cài từ builder sang runtime `python:3.11-slim`, không mang toàn bộ môi trường builder vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Trong output/cache BuildKit tôi thấy các layer base, `WORKDIR`, `COPY requirements.txt` và layer cài dependency được dùng lại; thay đổi source chỉ làm `COPY app ./app` và các layer đứng sau nó phải tạo lại. Lý do là Dockerfile copy `requirements.txt` và chạy `pip install` trước khi copy `app`. Nếu đặt `COPY . .` trước `RUN pip install`, chỉ một ký tự đổi trong `app/main.py` cũng làm checksum của layer `COPY` thay đổi, kéo theo layer cài toàn bộ dependency phải chạy lại; build chậm hơn rất nhiều dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: một lỗi trong ứng dụng Python cho phép kẻ tấn công thực thi lệnh trong container; nếu process chạy bằng root thì lệnh đó có UID 0 trong container; kết hợp với mount/socket/quyền kernel cấu hình sai, kẻ tấn công có cơ hội sửa file nhạy cảm hoặc leo ra host với quyền cao. `USER appuser` (UID 10001) cắt chuỗi ở bước thực thi lệnh: kể cả chiếm được process, kẻ tấn công trước hết chỉ có quyền của user thường và không thể tùy ý ghi vào các vùng thuộc root. Nó giảm mạnh hậu quả nhưng không thay thế việc vá lỗ hổng và cấu hình container an toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi 20 request trong 2 giây: gửi 10 request vào khoảng 10:00:59, sau đó bộ đếm phút reset ở 10:01:00 và gửi tiếp 10 request vào khoảng 10:01:01. Mỗi phút riêng vẫn chỉ có 10 request nhưng tải thực tế là 20 request sát nhau. Sliding window 60 giây của bài giữ cả hai nhóm trong cùng cửa sổ nên không cho vượt kiểu này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong 60 giây, còn cost guard giới hạn tổng số tiền theo user và tháng. Trường hợp rate limit cho qua nhưng cost guard chặn: user chỉ gửi request đầu tiên trong phút, nhưng trước đó đã tiêu gần/hết 10 USD hoặc request dự kiến làm vượt ngân sách. Trường hợp ngược lại: user gửi request thứ 11 trong 60 giây, mỗi request rất rẻ và tổng chi phí vẫn thấp; cost guard còn cho phép nhưng rate limiter trả 429. Khi thử API tôi cũng thấy mỗi response có `cost_usd` và số này được cộng riêng, trong khi rate limit dùng số lượt gọi.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp hai endpoint, Redis mất kết nối làm cả ba container cùng báo health check thất bại; orchestrator coi process của cả ba bị hỏng và lần lượt restart chúng; trong lúc Redis chưa trở lại, các container mới vẫn fail health rồi tiếp tục bị restart; khi Redis phục hồi có thể chưa còn instance ổn định nào đang nhận traffic, nên sự cố Redis 30 giây bị khuếch đại thành downtime toàn service. Với thiết kế hiện tại, `/health` vẫn 200 để container không bị restart, còn `/ready` trả 503 để load balancer chỉ tạm ngừng gửi request.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Trong lần chạy thật trên Render với cùng `X-User-Id: sv01`, tôi quan sát `history_length` lần lượt là 0, 2 rồi 4; ảnh `screenshots/ask.png` ghi lại giá trị 4. Test CP4 cũng mô phỏng hai `ConversationStore` khác nhau dùng chung fake Redis và container thứ hai đọc được dữ liệu container thứ nhất. Nếu dùng dict Python riêng trong từng instance, request bị load balancer chuyển sang instance khác sẽ thấy 0 hoặc một con số thấp hơn; kết quả sẽ nhảy thất thường theo instance thay vì tăng đều 0, 2, 4.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thật tôi gặp là Render trả 500 cho `/ready`, còn log ghi `pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings`. Tôi mở Render Logs và đối chiếu trang Environment, thấy chỉ có `LOG_LEVEL`, `MONTHLY_BUDGET_USD`, `RATE_LIMIT_PER_MINUTE`, `REDIS_URL` mà thiếu `AGENT_API_KEY`. Tôi thêm `AGENT_API_KEY` dưới dạng secret, lưu và redeploy. Sau đó `/health` trả `200 {"status":"ok",...}`, `/ready` trả `200 {"status":"ready","redis":true}` và `/ask` có key trả lời thành công.

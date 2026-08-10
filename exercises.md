# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đào Văn Đa  Mã học viên: 2A202601089

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu quên `AGENT_API_KEY` mà code có mặc định
> `"changeme"`, container vẫn có thể lên trạng thái hoạt động và người ngoài có
> thể đoán khóa để gọi `/ask`. Với trường bắt buộc hiện tại, cấu hình được kiểm
> tra trong startup lifecycle nên deployment dừng ngay; lỗi xuất hiện trong log
> lúc mình còn đang theo dõi deploy, trước khi service nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật mình thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:49:29.356831+00:00", "user_id": "exercise-log", "tokens_in": 4, "tokens_out": 36, "cost_usd": 2.22e-05}`.
> Với JSON này mình có thể lọc/đếm các event theo `user_id` và khoảng thời gian,
> đồng thời cộng `cost_usd` hoặc đặt cảnh báo khi chi phí vượt ngưỡng. Chuỗi
> `print("đã trả lời xong")` không mang các trường có cấu trúc để làm hai việc đó.

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
| 1 stage (bản đầu) | 431.5 MB |
| Multi-stage | 70.9 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Mình đo bằng `docker image inspect`: single-stage là 431,498,641 byte, còn
> multi-stage là 70,916,079 byte, giảm khoảng 360.6 MB (xấp xỉ 83.6%). Phần
> chênh chủ yếu đến từ base `python:3.11` đầy đủ và các thành phần hệ điều hành,
> công cụ/build artifacts không cần ở runtime. Bản multi-stage dùng
> `python:3.11-slim` và chỉ copy virtual environment cùng source cần chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, thay đổi `app/main.py` không làm thay đổi checksum của
> `requirements.txt`, nên các layer tạo venv và `pip install` được lấy từ cache;
> Docker chỉ cần chạy lại `COPY app ./app` và các layer sau nó rồi export image.
> Nếu đặt `COPY . .` trước `RUN pip install`, bất kỳ thay đổi source nào cũng làm
> layer COPY đổi checksum, kéo theo `pip install` phải chạy lại dù dependency
> không đổi. Trong lúc kiểm tra mình còn gặp Docker Hub trả 429 khi resolve base
> image; đây là lỗi registry tạm thời, không phải lỗi cache hay Dockerfile.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng xử lý input có thể cho phép thực thi mã trong process Python. Nếu
> container chạy root, mã đó có quyền root trong container; kết hợp thêm lỗi
> escape container, socket Docker bị mount hoặc capability quá rộng, kẻ tấn công
> có thể chạm tới host với quyền cao. `USER appuser` cắt chuỗi ở bước thực thi
> sau khi chiếm process: attacker chỉ nhận quyền của user thường, giảm đáng kể
> file/capability có thể truy cập dù vẫn phải vá lỗ hổng gốc.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong khoảng 2 giây: gửi 10 request ở cuối phút, ví dụ
> `10:00:59`, rồi ngay sau khi bộ đếm reset ở giây 00 gửi tiếp 10 request ở
> `10:01:00`. Mỗi bucket phút riêng vẫn chỉ thấy 10 request. Sliding window 60
> giây sẽ nhìn thấy cả hai nhóm trong cùng cửa sổ và chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit bảo vệ tốc độ/số request trong cửa sổ ngắn; cost guard bảo vệ tổng
> tiền tích lũy theo user trong cả tháng. Một user gửi 1 request rất dài khi vẫn
> còn quota tốc độ có thể qua rate limit nhưng phải bị cost guard chặn nếu ngân
> sách tháng đã gần hết. Ngược lại, user còn đủ ngân sách nhưng bắn 11 request
> nhỏ trong một phút sẽ bị rate limit chặn request thứ 11 dù chi phí rất thấp.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp và để liveness kiểm tra Redis, khi Redis mất 30 giây cả ba container
> lần lượt trả liveness 503. Orchestrator coi process chết và restart đồng loạt;
> container mới lại kiểm tra Redis đang lỗi, tiếp tục fail/restart, nên không còn
> instance ổn định để phục vụ endpoint không phụ thuộc Redis. Tách riêng giúp
> `/health` vẫn 200 để process không bị restart, còn `/ready` trả 503 để load
> balancer tạm rút instance khỏi traffic; khi Redis hồi phục, `/ready` tự về 200.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, lượt đầu của cùng `X-User-Id` trả
> `history_length=0`, lượt sau trả `2`, rồi `4`... bất kể request tới instance
> nào vì mỗi lượt thêm một message user và một message assistant. Nếu dùng dict
> trong RAM, ba instance có ba lịch sử riêng: số có thể nhảy/lặp như 0, 0, 2, 0,
> 2 tùy load balancer, và sau restart lại về 0. Test stateless của mình cũng xác
> nhận hai object store khác nhau nhìn thấy cùng dữ liệu qua một Redis.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thật mình gặp là public domain Railway lúc mới tạo chưa có `targetPort`,
> nên `/health` thỉnh thoảng trả 502 dù deployment báo SUCCESS. Mình đọc runtime
> log thấy Uvicorn đang nghe `0.0.0.0:8080`, còn domain report
> `targetPort: null`; request 502 cũng không xuất hiện trong app log nên lỗi nằm
> trước container. Mình cập nhật Railway domain trỏ tới port 8080, sau đó kiểm
> tra lại `/health` 200, `/ready` 200, request thiếu key 401, request có key 200
> và 15 lần gọi rate limit cho kết quả 10 lần 200 rồi 5 lần 429.

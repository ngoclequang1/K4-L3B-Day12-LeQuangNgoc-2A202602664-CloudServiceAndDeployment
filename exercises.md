# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bên dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Quang Ngọc  Mã học viên: 2A202602664

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu quên cấu hình khóa khi deploy public, ứng dụng có giá trị mặc định
> `"changeme"` vẫn chạy và người lạ có thể đoán khóa để gọi API. Việc fail fast
> làm deployment dừng ngay với lỗi thiếu `AGENT_API_KEY`, nên tôi phát hiện cấu
> hình sai trước khi service nhận traffic và phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi thu được khi gọi `/ask` là:
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T02:39:09.539910+00:00","user_id":"sv-test","tokens_in":4,"tokens_out":36,"cost_usd":2.22e-05}`.
> Từ log này tôi có thể nhóm và cộng `cost_usd` theo `user_id`, đồng thời tính
> số request hoặc tỷ lệ lỗi theo từng khoảng thời gian. Dòng `print` chung chung
> không có các trường có cấu trúc để lọc hay tổng hợp như vậy.

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
| 1 stage (bản đầu) | 1.7 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại cả hai image bằng Docker Desktop: bản dùng `python:3.11` đầy đủ
> là 1.7 GB, còn bản multi-stage dùng `python:3.11-slim` là 271 MB. Phần chênh
> lệch chủ yếu là hệ điều hành/base image đầy đủ, công cụ build và các thành
> phần chỉ cần lúc cài dependency; runtime không cần mang chúng theo.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `COPY requirements.txt` và
> `RUN pip install` được dùng lại từ cache; chỉ layer copy source và các layer
> phía sau phải chạy lại. Nếu `COPY . .` nằm trước `pip install`, thay đổi một
> ký tự source cũng làm mất cache của layer cài dependency, khiến pip chạy lại
> dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗi cho phép thực thi lệnh, kẻ tấn công trước hết chiếm
> quyền của process trong container. Khi process chạy root, lỗi cấu hình mount,
> capability hoặc lỗ hổng runtime có thể giúp quyền đó tác động mạnh hơn tới
> host. `USER appuser` giới hạn process bị chiếm ở UID 10001 không đặc quyền,
> giảm quyền đọc/ghi và cắt chuỗi leo thang ngay tại lớp container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi 20 request trong khoảng 2 giây: gửi 10 request ngay trước giây 00
> của phút mới, rồi gửi tiếp 10 request ngay sau giây 00. Bộ đếm theo phút coi
> đó là hai cửa sổ khác nhau, còn sliding window 60 giây vẫn nhìn thấy cả 20
> request và chặn phần vượt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất request trong 60 giây, còn cost guard giới hạn
> tổng tiền theo user trong tháng. Mười request rất dài có thể vẫn đúng rate
> limit nhưng vượt ngân sách nên cost guard chặn. Ngược lại, một user gửi nhiều
> request rất rẻ trong vài giây có thể còn ngân sách nhưng bị rate limiter chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm endpoint chung trả 503; orchestrator hiểu nhầm process
> đã chết và restart cả ba container. Các container mới vẫn chưa nối được Redis
> nên tiếp tục unhealthy/restart, khiến cả cụm mất khả năng phục vụ. Tách
> `/health` giúp process vẫn sống, còn `/ready` chỉ yêu cầu load balancer tạm
> ngừng gửi traffic cho tới khi Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Trong test mô phỏng hai instance dùng chung Redis, request đầu trả
> `history_length=0` và request sau trả `2`, dù store được tạo ở instance khác.
> Khi scale ba container, con số vẫn tăng theo 0, 2, 4... vì mọi instance đọc
> cùng Redis. Nếu dùng dict Python, request đi qua container khác sẽ thấy lịch
> sử riêng nên số có thể nhảy lùi hoặc lặp lại 0/2 một cách ngẫu nhiên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp là `/ready` trả 500 và deploy log ghi `ValueError: Redis URL must
> specify one of the following schemes (redis://, rediss://, unix://)`. Tôi gọi
> riêng `/health` và `/ready`, rồi xem traceback trong Railway Deploy Logs để
> xác định `REDIS_URL` không phải URL đã resolve. Tôi xóa giá trị nhập tay, tạo
> lại bằng **Add Reference → day12-redis → REDIS_URL** và redeploy. Sau đó
> `/ready` trả 200 với `{"status":"ready","redis":true}`.

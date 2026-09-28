# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thành Duy  Mã học viên: 2A202602804

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy service lên cloud (như Railway hoặc Render), nếu ta quên thiết lập biến môi trường `AGENT_API_KEY`:
> - Nếu để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường, báo trạng thái healthy và sẵn sàng nhận traffic. Các bot tự động quét internet có thể dễ dàng gọi API bằng khóa mặc định `"changeme"` để sử dụng tài nguyên và bào mòn ngân sách LLM của bạn mà bạn không hề hay biết cho đến khi nhận hóa đơn.
> - Khi không có giá trị mặc định (Fail Fast), Pydantic Settings lập tức ném lỗi `ValidationError` ngay lúc ứng dụng khởi chạy container. Việc container chết ngay lập tức sẽ hiển thị thông báo lỗi trực tiếp trên dashboard deploy, giúp lập trình viên phát hiện và khắc phục ngay trước khi service tiếp nhận bất kỳ request nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:00:54.678123+00:00", "user_id": "sv-test", "tokens_in": 92, "tokens_out": 45, "cost_usd": 0.0000408}`
>
> Hai việc làm được với dòng log JSON mà `print("đã trả lời xong")` không làm được:
> 1. **Truy vấn, lọc và phân tích dữ liệu có cấu trúc tự động**: Các hệ thống quản lý log tập trung (như Datadog, Grafana Loki, CloudWatch) có thể lập tức parse các trường JSON để lọc ra các request lỗi (`level == "error"`), hoặc tính tổng chi phí `sum(cost_usd)` theo từng `user_id` cụ thể trong ngày.
> 2. **Thiết lập cảnh báo thời gian thực (Alerting & Metrics)**: Dễ dàng cấu hình cảnh báo tự động gửi về Slack/Discord nếu số token tiêu thụ (`tokens_in + tokens_out`) hoặc chi phí của một request vượt ngưỡng cho phép, điều mà chuỗi văn bản thuần túy không thể đo lường tự động được.

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
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~750 MB) bao gồm:
> - Các công cụ biên dịch và xây dựng phần mềm của Linux (như `gcc`, `g++`, `make`, các thư viện header C/C++ cần thiết khi compile).
> - Thư mục cache tải về của pip (`~/.cache/pip`) và các file wheel trung gian sinh ra trong quá trình cài đặt dependencies.
> - Các gói package hệ thống, tài liệu man pages của Debian/Ubuntu đầy đủ mà image `slim` ở stage runtime đã loại bỏ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - **Với Dockerfile hiện tại**: Các layer từ đầu cho đến `COPY requirements.txt .` và `RUN pip install ...` ở stage builder cũng như việc cài đặt user ở stage runtime được tái sử dụng 100% từ cache. Chỉ có layer `COPY app ./app` và các layer bên dưới nó là phải chạy lại. Quá trình build diễn ra gần như tức thì (1-2 giây).
> - **Nếu đặt `COPY . .` lên trước `RUN pip install`**: Khi sửa dù chỉ một ký tự trong mã nguồn, checksum của thư mục thay đổi làm mất hiệu lực (bust cache) của layer `COPY . .`. Do đó, Docker bắt buộc phải thực hiện lại lệnh `RUN pip install`, tức là phải tải và cài lại toàn bộ danh sách thư viện từ đầu ở mỗi lần sửa code, gây lãng phí thời gian và băng thông.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện leo thang đặc quyền:
> 1. Kẻ tấn công phát hiện một lỗ hổng trong code Python (ví dụ: Remote Code Execution - RCE hoặc Command Injection).
> 2. Kẻ tấn công thực thi mã độc trong container. Do container mặc định chạy quyền root (UID 0), tiến trình độc hại có toàn quyền root trong môi trường container.
> 3. Kẻ tấn công khai thác tiếp các lỗ hổng container breakout (như khai thác kernel Linux dùng chung giữa container và host, hoặc can thiệp docker socket nếu bị mount nhầm).
> 4. Do tiến trình có UID 0 tương ứng với UID 0 của máy host, kẻ tấn công chiếm toàn quyền kiểm soát máy chủ host.
>
> **Lệnh `USER appuser` cắt đứt chuỗi ở bước 2**: Tiến trình Python bị giới hạn chạy dưới quyền user thường (UID 10001). Ngay cả khi code Python bị chiếm quyền điều khiển, kẻ tấn công không thể chỉnh sửa file hệ thống của container, không thể leo thang đặc quyền và không thể thoát ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Một người dùng có thể gửi tối đa **20 requests** trong 2 giây liên tiếp.
>
> Cách đạt được con số đó:
> - Người dùng gửi liên tiếp 10 request ở giây thứ 59 của phút trước (ví dụ lúc `10:00:59`). Hệ thống kiểm tra thấy chưa quá hạn mức 10 request của phút 10:00 nên cho qua cả 10.
> - Ngay 2 giây sau, lúc `10:01:01` (bước sang phút mới), bộ đếm đồng hồ vừa được reset về 0. Người dùng gửi tiếp 10 request nữa và tiếp tục được cho qua vì thuộc hạn mức của phút mới 10:01.
> - Kết quả: Trong khoảng thời gian chỉ 2 giây (từ `10:00:59` đến `10:01:01`), người dùng đã gửi thành công 20 request, gây ra đột biến lưu lượng (traffic burst) làm quá tải server.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - **Điểm khác nhau**: Rate limit kiểm soát **số lượng request trong một khoảng thời gian ngắn** (ví dụ 10 request/phút) để bảo vệ server khỏi nghẽn mạng và từ chối dịch vụ. Cost guard kiểm soát **tổng số tiền/chi phí token tiêu thụ trong khoảng thời gian dài** (ví dụ 10 USD/tháng) để bảo vệ ngân sách tài chính.
> - **Rate limit cho qua nhưng Cost guard chặn**: Người dùng chỉ gửi 2 request trong cả giờ (tần suất cực thấp, hoàn toàn không vi phạm rate limit), nhưng mỗi request gửi vào văn bản tài liệu khổng lồ 100.000 token khiến chi phí ước tính vượt quá ngân sách 10 USD của tháng $\rightarrow$ Cost guard chặn với mã 402 Payment Required.
> - **Cost guard cho qua nhưng Rate limit chặn**: Đầu tháng tài khoản người dùng còn nguyên ngân sách 10 USD chưa tiêu đồng nào, nhưng người dùng dùng script gửi liên tiếp 15 request chỉ trong vòng 5 giây $\rightarrow$ Rate limit chặn ngay ở request thứ 11 với mã 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra:
> 1. Khi Redis mất kết nối, endpoint gộp này lập tức kiểm tra thất bại và trả về mã lỗi 503 trên cả 3 container.
> 2. Bộ điều phối container (Orchestrator như Docker Swarm / Kubernetes / Cloud platform) đọc endpoint này như một Liveness Probe và kết luận rằng cả 3 container ứng dụng đều đã hỏng.
> 3. Orchestrator lập tức cưỡng chế khởi động lại (restart) toàn bộ cả 3 container cùng lúc.
> 4. Trong suốt 30 giây Redis chưa phục hồi, các container khởi động lên lại kiểm tra thấy Redis chết $\rightarrow$ báo unhealthy $\rightarrow$ lại bị restart liên tục tạo thành vòng lặp crashloop.
> 5. Đến khi Redis hoạt động trở lại sau 30 giây, không có container nào đang chạy sẵn sàng phục vụ lưu lượng, biến sự cố mất kết nối cơ sở dữ liệu tạm thời thành sự cố sập toàn bộ hệ thống (Cascading Failure).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu lịch sử trong một dict Python nội bộ của tiến trình:
> - Vì 3 container chạy độc lập và có vùng nhớ RAM tách biệt, Load Balancer sẽ phân phối các request kế tiếp của cùng một user luân phiên vào container 1, container 2 hoặc container 3.
> - Người dùng sẽ thấy `history_length` **thay đổi lộn xộn, nhảy cóc không thể đoán trước** (ví dụ: lần 1 gọi vào container 1 trả về 0, lần 2 vào container 2 lại trả về 0, lần 3 quay lại container 1 thì nhảy lên 2, lần 4 vào container 3 lại về 0). Agent sẽ bị hiện tượng "mất trí nhớ" ngẫu nhiên giữa các câu hỏi.
> - Ngược lại, khi lưu vào Redis ngoài process (Stateless), cả 3 container cùng truy xuất một kho dữ liệu chung, giúp `history_length` luôn tăng đều đặn (0, 2, 4, 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Lỗi gặp phải**: Lỗi `500 Internal Server Error` khi gọi vào endpoint `/ready` sau khi cấu hình biến môi trường trên Railway.
> - **Thông báo lỗi trong log**: `ValueError: Port could not be cast to integer value as '6379 '`.
> - **Cách tìm ra nguyên nhân**: Chạy lệnh `railway logs -s day12-agent` để kiểm tra log runtime của container trên Railway. Traceback báo lỗi xảy ra tại thư viện `urllib.parse` khi phân tích cú pháp của biến `REDIS_URL`, cho thấy ở cuối cổng `6379 ` bị dính một ký tự khoảng trắng thừa (trailing space) do sơ suất khi copy paste connection string.
> - **Cách sửa**: Sử dụng lệnh `railway variables -s day12-agent --set "REDIS_URL=redis://..."` để gán lại giá trị chuẩn xác không có khoảng trắng ở cuối. Railway tự động redeploy lại container và endpoint `/ready` lập tức trả về `200 {"status":"ready","redis":true}`.

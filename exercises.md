# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Cao Đức Anh  Mã học viên: 2A202602754

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy service lên môi trường Cloud (như Render hoặc Railway), lập trình viên có thể sơ suất quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard. Nếu trường này có giá trị mặc định, ứng dụng vẫn khởi động bình thường và báo trạng thái xanh. Khi đó, các bot quét mạng hoặc kẻ tấn công có thể dễ dàng dò ra và gọi vào API bằng khóa mặc định, âm thầm tiêu tốn ngân sách LLM của mà chỉ phát hiện ra khi nhận hóa đơn dịch vụ vào cuối tháng. Ngược lại, việc không đặt giá trị mặc định buộc Pydantic ném lỗi `ValidationError` và dừng ứng dụng ngay lúc khởi động, giúp lập trình viên phát hiện và khắc phục ngay trong lúc deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:36:00.123456+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}`

Hai việc làm được với định dạng log JSON mà `print()` thông thường không làm được:
1.Lọc, tổng hợp và thống kê tự động: Các hệ thống tập trung log có thể parse cấu trúc JSON để tính toán tổng chi phí theo ngày, vẽ biểu đồ số token tiêu thụ, hoặc lọc ra danh sách top người dùng tiêu tốn nhiều tiền nhất theo trường `user_id`.
2.Cảnh báo tự động theo ngưỡng: Có thể thiết lập quy tắc tự động gửi cảnh báo ngay khi trường `cost_usd` vượt quá ngưỡng an toàn trong một request, điều mà việc in chuỗi văn bản thuần túy không thể thực hiện một cách chính xác và tự động.

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
| 1 stage (bản đầu) | 1050 MB |
| Multi-stage | 195 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~855 MB) bao gồm:
1. Base image đầy đủ (`python:3.11`) chứa toàn bộ các gói hệ thống Debian tiêu chuẩn cùng các công cụ biên dịch mã nguồn C/C++ (`gcc`, `g++`, `make`, `build-essential`), file header và bộ nhớ đệm của trình quản lý gói `apt`.
2. Trong multi-stage build, stage `builder` thực hiện biên dịch và cài đặt thư viện vào thư mục `/install`. Stage `runtime` sử dụng image `python:3.11-slim` chỉ sao chép các gói Python đã cài đặt từ `/install` sang `/usr/local` mà vứt bỏ toàn bộ compiler, apt cache và các công cụ build trung gian, giúp image sản phẩm cuối cùng trở nên gọn nhẹ và an toàn hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

1. Với Dockerfile tối ưu hiện tại: Do `requirements.txt` không thay đổi, Docker sẽ tái sử dụng cache cho các layer từ đầu đến hết `COPY requirements.txt .`, `RUN pip install ...`, và `RUN useradd ...`. Chỉ có các layer từ `COPY app ./app` trở đi mới bị mất cache và phải thực thi lại, giúp quá trình build lại chỉ mất vài giây.
2. Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa dù chỉ một ký tự trong code, checksum của toàn bộ thư mục thay đổi làm Docker hủy cache từ layer `COPY . .` trở đi. Hệ quả là câu lệnh `RUN pip install` bị ép phải tải và cài đặt lại toàn bộ các thư viện từ đầu, khiến thời gian build kéo dài thêm vài phút mỗi lần thay đổi mã nguồn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python tồn tại lỗ hổng bảo mật.
2. Kẻ tấn công khai thác lỗ hổng để kích hoạt shell thực thi mã bên trong container. Vì container mặc định chạy quyền `root` (UID 0), kẻ tấn công lập tức có toàn quyền root bên trong không gian container đó.
3. Từ quyền root này, kẻ tấn công khai thác các lỗ hổng container breakout/escape.
4. Kẻ tấn công thoát khỏi ranh giới container và chiếm quyền kiểm soát máy host bên ngoài với đặc quyền cao nhất.

Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: Tiến trình ứng dụng bị ép chạy dưới định danh người dùng thường (UID 10001) không có đặc quyền. Khi đó kẻ tấn công bị tước quyền root trong container, không thể ghi đè các file hệ thống nhạy cảm, không thể chạy các lệnh quản trị đặc quyền và ngăn chặn triệt để nguy cơ thoát container ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 requests trong 2 giây liên tiếp.
Giải thích: Với cơ chế đếm theo phút đồng hồ (Fixed Window reset lúc giây 00), người dùng có thể gửi 10 request ở giây `10:00:59`. Ngay 1 giây sau, khi đồng hồ chuyển sang `10:01:00`, bộ đếm được reset về 0, người dùng gửi tiếp 10 request nữa. Kết quả là trong khoảng thời gian chỉ 2 giây, hệ thống đã chấp nhận 20 request mà không hề kích hoạt rate limit, gây ra hiện tượng spike lưu lượng làm quá tải máy chủ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau:
- Rate Limit: Giới hạn số lượng request/tần suất gọi trong một khoảng thời gian ngắn nhằm bảo vệ tính sẵn sàng của hạ tầng và chống tấn công làm nghẽn dịch vụ (DoS).
- Cost Guard: Giới hạn tổng chi phí tài chính tích lũy trong một chu kỳ dài nhằm kiểm soát ngân sách chi trả cho nhà cung cấp LLM.

Ví dụ tình huống:
- Rate limit cho qua nhưng Cost guard chặn: Người dùng gửi chỉ 1 request/phút, nhưng tài khoản này đã tích lũy chi phí gọi API đạt ngưỡng 10.0 USD trong tháng. Cost guard sẽ chặn request với mã lỗi `402 Payment Required`.
- Cost guard cho qua nhưng Rate limit chặn: Người dùng mới sử dụng dịch vụ vào đầu tháng, số dư ngân sách còn nguyên 10.0 USD, nhưng gửi liên tục 15 request chỉ trong vòng 10 giây (vượt ngưỡng 10 req/phút). Rate limit sẽ chặn từ request thứ 11 với mã lỗi `429 Too Many Requests` để tránh nghẽn server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố gián đoạn kết nối trong 30 giây.
2. Endpoint liveness check (`/health`) của cả 3 container đồng loạt trả về lỗi (503) do phụ thuộc vào Redis.
3. Bộ điều phối (Docker Compose / Kubernetes) kiểm tra liveness probe thấy thất bại, cho rằng cả 3 container đã bị treo/hỏng tiến trình và phát lệnh kill để khởi động lại (restart) cả 3 container.
4. Việc restart hàng loạt khiến mọi request đang được xử lý dở dang của người dùng bị ngắt đột ngột, gây ra lỗi `502 Bad Gateway`.
5. Khi 3 container vừa khởi động lại xong, Redis vẫn chưa phục hồi, liveness probe lại tiếp tục thất bại và chu kỳ restart lặp lại liên tục (rơi vào trạng thái `CrashLoopBackOff`), biến một sự cố mất kết nối tạm thời của dịch vụ phụ thuộc thành sự cố sụp đổ toàn bộ hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

1. Khi lưu bằng Redis (Stateless): Giá trị `history_length` tăng đều đặn qua mỗi lượt tương tác (0, 2, 4, 6...) bất kể request được điều phối tới container nào trong cụm, vì cả 3 container đều đọc/ghi lịch sử chung từ Redis.
2. Nếu lưu trong dict Python: Vì mỗi container có một vùng nhớ RAM riêng biệt, khi Load Balancer phân phối các request luân phiên qua từng container:
   - Request 1 vào container 1: `history_length = 0`, container 1 ghi nhận vào RAM của nó.
   - Request 2 vào container 2: Do RAM của container 2 chưa có dữ liệu user này, `history_length` lại trả về `0` (agent bị mất trí nhớ).
   - Request 3 vào container 3: `history_length` tiếp tục trả về `0`.
   - Request 4 quay lại container 1: Lúc này mới đọc được dữ liệu của request 1, `history_length` nhảy lên `2`.
   Con số `history_length` sẽ nhảy lộn xộn, tăng giảm bất thường và mô hình AI liên tục bị mất ngữ cảnh của cuộc trò chuyện.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: Trong quá trình cấu hình deploy ban đầu lên Render, endpoint `/ready` trả về mã lỗi `HTTP 503 Service Unavailable` kèm nội dung `{"status":"not ready","redis":false}`.
- Cách tìm ra nguyên nhân: Mở tab Logs của service `day12-agent` trên Render Dashboard, nhận thấy ứng dụng báo lỗi không thể kết nối tới máy chủ Redis do biến môi trường `REDIS_URL` ban đầu mang giá trị `redis://localhost:6379/0` (giá trị mặc định chạy ở máy local, không thể kết nối tới Redis trên cloud).
- Cách sửa: Sử dụng tệp cấu hình `render.yaml` (Blueprint), khai báo trường `fromService` cho biến `REDIS_URL` liên kết trực tiếp tới connection string của service `day12-redis` (`property: connectionString`). Khi Render liên kết đúng đường truyền nội bộ giữa hai service, endpoint `/ready` lập tức trả về `HTTP 200 OK` với `{"status":"ready","redis":true}`.

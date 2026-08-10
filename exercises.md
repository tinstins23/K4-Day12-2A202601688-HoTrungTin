# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hồ Trung Tín  Mã học viên: 2A202601688

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu `api_token` mặc định là `"changeme"`, mình quên set `API_TOKEN` trên Render
> thì app vẫn khởi động “ổn”. Ai đoán được (hoặc đọc repo mẫu) token mặc định
> đều gọi được `/chat` công khai → tốn ngân sách / lộ dữ liệu. Không có mặc định
> thì deploy thiếu biến là **crash ngay lúc start**, mình thấy lỗi trên dashboard
> trước khi traffic vào, buộc phải set secret đúng trước khi service nhận request.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi  
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`  
không làm được.

> ```
> {
>     "reply": "Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 12 lượt trao đổi trước đó.)",
>     "client_id": "sv01",
>     "turns_before": 12,
>     "usd_cost": 7.74e-05,
>     "usage": {
>         "prompt": 336,
>         "completion": 45
>     }
> }
> ```
>
> Hai việc log JSON làm được mà `print("đã trả lời xong")` không:
>
> 1. **Lọc/đếm theo field** (ví dụ đếm mọi `event=chat_completed` của `client_id=sv01`, hoặc sum `usd_cost`) trong Cloud Logging / Datadog.
> 2. **Cảnh báo theo mức** nhờ khóa `severity` (chỉ alert khi ERROR/WARNING), hoặc theo ngưỡng chi phí — chuỗi tự do không có cấu trúc để query ổn định.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```


| Bản               | Dung lượng |
| ----------------- | ---------- |
| 1 stage (bản đầu) | ~1850 MB   |
| Multi-stage       | ~280 MB    |


Giải thích: phần dung lượng chênh lệch đó là những gì?

> Sau khi build, `docker images` cho khoảng **1850 MB** (1 stage, base đầy đủ /
> mang theo lớp cài đặt) so với **~280 MB** (multi-stage + `python:3.11-slim`).
> Phần chênh ~1.5 GB chủ yếu là toolchain và rác build không cần lúc chạy:
> compiler, cache pip, artifact trung gian trong stage `builder`. Multi-stage chỉ
> copy `/install` sang runtime slim → image chỉ còn Python + dependency + code app.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình sửa một ký tự trong `app/main.py` rồi `docker build` lại. Log build cho thấy:
>
> - **CACHED:** `FROM ... AS builder`, `COPY requirements.txt`, `RUN pip install`,
>   `FROM ... AS runtime`, `COPY --from=builder /install` — vì `requirements.txt`
>   không đổi.
> - **Chạy lại:** `COPY app ./app`, `COPY utils ./utils`, rồi `useradd` / `USER` /
>   các layer sau (vì checksum của `app/` đổi).
>
> Nếu đặt `COPY . .` trước `RUN pip install`, sửa một dòng code cũng làm mất
> cache từ layer COPY → Docker cài lại toàn bộ thư viện mỗi lần → build lâu hơn
> rõ rệt so với tách `requirements.txt` ra trước.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện (rút gọn):
>
> 1. Lỗ hổng (RCE / path traversal / deserialize…) cho phép thực thi lệnh trong process app.
> 2. Process đang **root trong container** → kẻ tấn công thành root namespace container
>   (cài tool, đọc secret mount, chỉnh network…).
> 3. Kết hợp lỗ hổng escape container / mount nguy hiểm / Docker socket → leo quyền
>   lên **host** với đặc quyền cao.
>
> Lệnh `USER appuser` cắt ở bước 2: app (và payload sau khi exploit) chỉ chạy với
> user thường → giảm bề mặt leo thang ngay cả khi code bị chiếm.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là quy ước HTTP/RFC 6750: báo client biết phải xác thực
> kiểu Bearer (SDK/trình duyệt hiểu và gửi lại header đúng).
>
> Cùng một message vì nếu phân biệt “thiếu header” / “sai scheme” / “sai token”
> thì kẻ dò sẽ biết mình đang tiến gần token đúng (oracle). Thông báo chung giảm
> lộ thông tin; client hợp lệ xem docs là đủ.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Im lặng 10 phút: xô đã đầy tới `capacity` nên tối đa **10 request** liên tiếp
> rồi request thứ 11 → 429 (refill trong lúc burst gần như không kịp nếu gửi sát nhau).
>
> Bỏ `min(capacity, ...)`: im 10 phút tích ~`10 token/phút × 10 phút = 100` token
> (cộng phần còn lại nếu chưa hết) → có thể burst khoảng **~100 request** (hoặc hơn
> nếu còn token cũ) trước 429 — đúng cái “tích vô hạn rồi xả một lần” mà lab cảnh báo.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> - **$30/tháng:** từ 2h sáng có thể đốt gần như cả hạn mức tháng → thiệt hại tối đa
> khoảng **$30** trong tháng đó; chỉ “hồi” khi sang chu kỳ tháng mới (hoặc admin
> tăng hạn mức thủ công).
> - **$1/ngày:** cùng sự cố chỉ đốt tối đa khoảng **$1** trong ngày đó; sang ngày UTC
> mới khóa `spend:...:YYYY-MM-DD` khác → hạn mức tự về đủ $1, không cần ai mở lại.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
>
> 1. Redis đứt → probe “health” (đã gộp) của cả 3 container fail vì phụ thuộc Redis.
> 2. Orchestrator tưởng **process chết** → restart / thay thế cả 3 instance (dù app
>   Python vẫn sống).
> 3. Trong lúc restart hàng loạt: mất capacity, request fail; Redis về lại cũng bị
>   storm restart. Nếu tách đúng: `/healthz` vẫn 200 (liveness), chỉ `/readyz` 503
>    → LB ngừng đẩy traffic, **không** restart oan cả cụm.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi trên Render: deploy báo **health check failed / build fail** vì service
> không trả lời được `/healthz` sau khi lên.
>
> Cách tìm: xem log deploy trên dashboard — image build từ GitHub nhưng code trên
> remote chưa đủ (chưa push hết endpoint `/healthz` và phần liên quan), nên
> container start xong probe health check không 200.
>
> Cách sửa: push đủ code đã làm ở máy lên Git, Render build lại từ commit mới →
> `/healthz` OK, deploy xanh.


# Phiếu Phản Ánh — K3 Ngày 12
# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Các câu trả lời dưới đây được viết dựa trên quá trình chạy và kiểm tra thực tế của em.

**Họ và tên:** Vương Trần Hoàn  
**Mã học viên:** 2A202601481

---

### Câu 1 — Fail fast (CP1)

> Khi deploy lên cloud, nếu em quên cấu hình `AGENT_API_KEY`, ứng dụng sẽ báo lỗi ngay lúc khởi động thay vì chạy với một khóa mặc định không an toàn. Nếu dùng `"changeme"`, service vẫn có thể chạy và em có thể không nhận ra rằng API đang sử dụng một secret yếu, khiến người khác có khả năng truy cập API. Cách fail fast giúp phát hiện lỗi cấu hình ngay từ đầu và tránh đưa một service chưa được bảo mật vào môi trường production.

---

### Câu 2 — Log cho máy đọc (CP1)

> Ví dụ một dòng log JSON em quan sát được có dạng:

```json
{"event":"request_completed","method":"POST","path":"/ask","status_code":401,"user_id":"anonymous"}
---

### Câu 3 — Kích thước image (CP2)

Em đã build hai phiên bản Docker image để so sánh:

```bash
docker build -f Dockerfile.single -t agent:single .
docker build -t agent:multi .
docker images agent
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
agent        single    d1d006b65079   17 seconds ago   287MB
agent        multi     ce6c9305e1bd   14 minutes ago   270MB

Image multi-stage nhỏ hơn image 1-stage khoảng 17 MB. Nguyên nhân là multi-stage tách quá trình cài đặt dependency ở stage builder và chỉ copy phần cần thiết (/install) sang stage production. Các thành phần phục vụ quá trình build không cần thiết trong image production được loại bỏ khỏi final image. Vì vậy image cuối gọn hơn và giảm lượng thành phần không cần thiết khi triển khai.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

> Khi em sửa một ký tự trong `app/main.py` rồi build lại, Docker vẫn sử dụng cache cho các layer không bị ảnh hưởng. Cụ thể, layer `FROM python:3.11-slim`, `WORKDIR /build`, `COPY requirements.txt .` và `RUN pip install --no-cache-dir --prefix=/install -r requirements.txt` ở stage builder có thể được dùng lại vì `requirements.txt` không thay đổi.
>
> Ở stage production, các layer `FROM python:3.11-slim`, `WORKDIR /app` và `COPY --from=builder /install /usr/local` cũng có thể được dùng lại. Khi `app/main.py` thay đổi, layer `COPY app/ app/` phải chạy lại và các layer phía sau nó cũng được build lại nếu Dockerfile có thay đổi tương ứng.
>
> Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần sửa một file bất kỳ trong project thì layer `COPY . .` bị thay đổi, kéo theo `RUN pip install` phải chạy lại. Điều này làm thời gian build lâu hơn dù `requirements.txt` không thay đổi. Vì vậy Dockerfile hiện tại đặt `COPY requirements.txt` và `RUN pip install` trước khi copy source code để tận dụng Docker cache tốt hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

> Nếu code Python có một lỗ hổng, kẻ tấn công có thể lợi dụng lỗ hổng đó để thực hiện các thao tác trái phép bên trong container. Nếu ứng dụng đang chạy bằng `root`, process bị khai thác cũng có quyền `root` trong container, nên mức độ ảnh hưởng sẽ lớn hơn nếu kẻ tấn công tiếp tục khai thác các lỗ hổng hoặc cấu hình container để tìm cách truy cập tài nguyên của host.
>
> Trong Dockerfile của em có:
>
> ```dockerfile
> RUN adduser --no-create-home appuser
> USER appuser
> ```
>
> Lệnh `USER appuser` khiến ứng dụng không chạy với quyền `root` mà chạy bằng user thường. Vì vậy nếu ứng dụng bị khai thác, quyền của kẻ tấn công bị giới hạn theo quyền của `appuser`, giúp giảm đáng kể nguy cơ ảnh hưởng đến hệ thống host.

---

### Câu 6 — Cửa sổ trượt (CP3)

> Nếu rate limit được tính theo phút đồng hồ và reset tại giây `00`, một người dùng có thể gửi tối đa **20 request trong 2 giây liên tiếp** khi giới hạn là 10 request/phút.
>
> Ví dụ, người dùng gửi 10 request vào khoảng `12:00:59`, sau đó ngay khi sang `12:01:00` gửi tiếp 10 request. Như vậy trong khoảng thời gian rất ngắn có thể có 20 request được chấp nhận vì 10 request đầu thuộc phút trước và 10 request sau thuộc phút mới.
>
> Sliding window 60 giây khắc phục vấn đề này bằng cách xét số request thực tế trong 60 giây gần nhất thay vì phụ thuộc vào mốc `00` của đồng hồ.

---

### Câu 7 — Rate limit và cost guard (CP3)

> Rate limit và cost guard đều giúp bảo vệ API nhưng kiểm soát hai vấn đề khác nhau. **Rate limit** giới hạn số lượng request trong một khoảng thời gian, còn **cost guard** kiểm soát mức chi phí sử dụng và ngân sách.
>
> Một tình huống rate limit cho qua nhưng cost guard chặn là khi người dùng gửi ít request, nên chưa vượt giới hạn 10 request/phút, nhưng mỗi request sử dụng rất nhiều token khiến chi phí dự kiến vượt `MONTHLY_BUDGET_USD`.
>
> Ngược lại, người dùng có thể gửi quá nhiều request nhỏ trong thời gian ngắn nên bị rate limit chặn, mặc dù tổng chi phí của các request đó vẫn chưa vượt ngân sách. Khi đó rate limit chặn trước còn cost guard chưa cần chặn.

---

### Câu 8 — /health khác /ready (CP4)

> Nếu gộp `/health` và `/ready` thành một endpoint và endpoint đó bắt buộc kiểm tra Redis, khi Redis mất kết nối thì cả ba container sẽ lần lượt bị ảnh hưởng.
>
> Đầu tiên, cả ba container vẫn có thể đang chạy process bình thường nhưng endpoint kiểm tra sẽ không kết nối được Redis. Sau đó hệ thống quản lý container hoặc load balancer có thể đánh dấu các container là không sẵn sàng và ngừng gửi traffic đến chúng. Nếu cơ chế restart được cấu hình dựa trên health check, các container có thể tiếp tục bị restart trong thời gian Redis mất kết nối.
>
> Khi Redis kết nối lại sau 30 giây, các container có thể kiểm tra Redis thành công và được đưa trở lại trạng thái sẵn sàng.
>
> Vì vậy nên tách `/health` và `/ready`: `/health` chỉ kiểm tra ứng dụng còn sống, còn `/ready` kiểm tra ứng dụng đã sẵn sàng phục vụ và các dependency như Redis hoạt động bình thường.

---

### Câu 9 — Stateless (CP4)

> Khi chạy:
>
> ```bash
> docker compose up --scale agent=3
> ```
>
> các request có thể được phân phối cho ba container khác nhau.
>
> Nếu lịch sử được lưu trong một `dict` Python, mỗi container sẽ có một `dict` riêng trong bộ nhớ của chính nó. Vì vậy `history_length` phụ thuộc vào container nhận request. Ví dụ, một request có thể trả về `history_length = 3`, nhưng request tiếp theo được chuyển sang container khác có thể chỉ trả về `history_length = 1` hoặc một giá trị khác.
>
> Nếu lịch sử được lưu trong Redis, cả ba container cùng sử dụng một nơi lưu trữ chung. Do đó `history_length` được duy trì nhất quán hơn khi request được phân phối sang các container khác nhau. Đây là một đặc điểm quan trọng của kiến trúc stateless khi scale nhiều instance.

---

### Câu 10 — Deploy thật (CP5)

> Trong quá trình làm CP5, em chưa deploy được lên cloud nên sử dụng phương án `LOCAL_FALLBACK=true` với Docker Compose.
>
> Lỗi cụ thể em gặp là test CP5 báo:
>
> ```text
> AssertionError: phương án dự phòng cần ít nhất 1 ảnh trong screenshots/
> (terminal đang chạy `docker compose ps` và kết quả gọi API)
> ```
>
> Em kiểm tra file `tests/test_cp5.py` và thấy khi `LOCAL_FALLBACK=true`, test `test_co_anh_chup_man_hinh` yêu cầu thư mục `screenshots/` phải có ít nhất một file ảnh `.png`, `.jpg` hoặc `.jpeg`.
>
> Em đã chụp màn hình kết quả kiểm tra service và lưu vào thư mục:
>
> ```text
> screenshots/healt_ready.png
> ```
>
> Sau đó em chạy lại:
>
> ```bash
> python -m pytest tests/test_cp5.py -q
> ```
>
> Kết quả:
>
> ```text
> 8 passed, 5 skipped in 0.31s
> ```
>
> Tiếp theo em chạy `python grade.py`. CP1, CP2, CP3 và CP4 đều đạt toàn bộ test. CP5 cũng đạt theo phương án local fallback nhưng bị giới hạn điểm vì chưa deploy public cloud. Tổng điểm tự động của bài là **79/100**.

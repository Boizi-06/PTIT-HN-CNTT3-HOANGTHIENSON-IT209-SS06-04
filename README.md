# BÁO CÁO BÀI TẬP THỰC HÀNH - SESSION 06
## BÀI 4: QUẢN LÝ TIẾN TRÌNH NỀN VỚI NOHUP VÀ TÍN HIỆU KILL

---

## 📋 THÔNG TIN HỌC VIÊN & BÀI THỰC HÀNH

- **Môn học:** IT209 - Hệ thống & Quản lý Cấu hình Mã nguồn
- **Học viên:** Hoàng Thiện Sơn
- **GitHub Username:** Boizi-06
- **Lớp / Khóa:** HN-KS24-CNTT3 / IT209
- **Email:** hson05542@gmail.com
- **Session:** Session 06 - Triển khai và Bảo mật Mạng Máy chủ
- **Bài tập:** Bài 4 - Quản lý tiến trình nền với nohup và tín hiệu Kill
- **Đường dẫn nộp bài:** `homework/session_06/ex4/`

---

## 🎯 MỤC TIÊU BÀI THỰC HÀNH

1. **Khởi chạy tiến trình nền độc lập (Background Process):** Sử dụng công cụ `nohup` (No Hang Up) kết hợp toán tử `&` để đưa chương trình chạy nền, không bị ảnh hưởng khi phiên làm việc SSH kết thúc hoặc bị mất kết nối mạng.
2. **Giám sát & Quản lý tiến trình hệ thống:** Sử dụng thành thạo các công cụ `ps aux`, `pgrep`, `top`/`htop` để tra cứu thông tin chi tiết và định danh (PID) của tiến trình.
3. **Hiểu rõ cơ chế Tín hiệu Hệ thống (POSIX Signals):** Phân biệt bản chất và trường hợp sử dụng của các tín hiệu điều khiển tiến trình quan trọng: `SIGHUP (1)`, `SIGINT (2)`, `SIGTERM (15)` và `SIGKILL (9)`.
4. **Tắt tiến trình an toàn (Graceful Shutdown):** Áp dụng quy trình chuẩn: luôn ưu tiên gửi tín hiệu `SIGTERM (15)` để tiến trình giải phóng tài nguyên và dọn dẹp file an toàn trước khi cưỡng chế bằng `SIGKILL (9)`.

---

## 🔬 PHÂN TÍCH KỸ THUẬT: CƠ CHẾ NOHUP VÀ CÁC TÍN HIỆU KILL

### 1. Tại sao cần `nohup` khi chạy ứng dụng trên máy chủ từ xa?
- Khi bạn đăng nhập vào máy chủ qua SSH và chạy một lệnh thông thường (foreground), tiến trình đó là con của shell session hiện tại.
- Khi bạn đóng Terminal hoặc mất mạng, shell gửi tín hiệu **`SIGHUP` (Signal Hang Up - Mã 1)** đến tất cả các tiến trình con của nó $\rightarrow$ Các tiến trình này sẽ bị tiêu diệt ngay lập tức.
- Lệnh **`nohup`** (No Hang Up) chặn tín hiệu `SIGHUP`. Kết hợp với toán tử **`&`** (chạy nền), tiến trình sẽ tiếp tục chạy liên tục và trở thành tiến trình độc lập không phụ thuộc vào Terminal.

```
[ Đóng Terminal SSH ]
        │
        ├──> Gửi SIGHUP ──> [ Tiến trình thường ] ──> ❌ BỊ DỪNG NGAY LẬP TỨC
        │
        └──> Gửi SIGHUP ──> [ nohup + & ] ───────> ✅ BỎ QUA SIGHUP & TIẾP TỤC CHẠY
```

### 2. So sánh các tín hiệu (Signals) điều khiển tiến trình
| Tín hiệu | Mã số (Signal Number) | Bản chất hoạt động | Khả năng can thiệp (Catch / Block) | Khi nào nên dùng? |
|:---|:---:|:---|:---:|:---|
| **SIGHUP** | `1` | Ngắt kết nối terminal / Yêu cầu reload cấu hình | Có thể bắt | Khi tắt session hoặc reload cấu hình service |
| **SIGINT** | `2` | Ngắt từ bàn phím (`Ctrl + C`) | Có thể bắt | Dừng nhanh ứng dụng đang chạy ở foreground |
| **SIGTERM** | `15` | Yêu cầu dừng tiến trình nhẹ nhàng (**Graceful Termination**) | **Có thể bắt (Catchable)** | **MẶC ĐỊNH & ƯU TIÊN SỐ 1**: Cho phép app lưu dữ liệu, đóng socket, ngắt kết nối DB rồi tắt an toàn |
| **SIGKILL** | `9` | Cưỡng chế dừng ngay lập tức bởi Linux Kernel | **KHÔNG THỂ BẮT HOẶC CHẶN** | **CHỈ DÙNG KHI KHẨN CẤP**: Khi tiến trình bị treo, dead-lock, không phản hồi với `SIGTERM` |

---

## 🛠️ QUY TRÌNH THỰC HIỆN CHI TIẾT TỪNG BƯỚC TRÊN VPS

### Bước 1: Tạo kịch bản giám sát `loop-monitor.sh`

Đăng nhập vào VPS (`ssh -p 2222 root@165.101.46.48`) và tạo file kịch bản:

```bash
cat << 'EOF' > loop-monitor.sh
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
EOF
```

Gán quyền thực thi cho file script:
```bash
chmod +x loop-monitor.sh
```

---

### Bước 2: Khởi chạy kịch bản dưới nền độc lập với `nohup`

Chạy script với lệnh `nohup` và chuyển hướng luồng output vào `/dev/null`:

```bash
nohup ./loop-monitor.sh > /dev/null 2>&1 &
```

*Giải thích cú pháp:*
- `nohup`: Miễn nhiễm với tín hiệu ngắt kết nối `SIGHUP`.
- `./loop-monitor.sh`: Script cần thực thi.
- `> /dev/null`: Chuyển hướng Standard Output (stdout) để không in ra màn hình.
- `2>&1`: Gộp Standard Error (stderr) vào stdout.
- `&`: Đưa tiến trình xuống chạy ngầm dưới background.

*Output nhận được trên VPS:*
```text
[1] 34189
```
*(Số `34189` là Process ID - PID được cấp phát cho tiến trình).*

---

### Bước 3: Kiểm tra tiến trình đang chạy và xác định PID

**Dùng lệnh `pgrep`:**
```bash
pgrep -f loop-monitor.sh
```
*Output thực tế:*
```text
34189
```

---

### Bước 4: Kiểm tra tệp tin log được ghi liên tục

Kiểm tra nội dung được ghi vào `/tmp/monitor.log`:

```bash
tail -n 10 /tmp/monitor.log
```

*Output nhận được:*
```text
System time: Tue Oct  6 08:35:21 PM +07 2026
```

---

### Bước 5: Tắt tiến trình an toàn bằng tín hiệu `SIGTERM (15)`

Sử dụng lệnh `kill -15` kèm theo PID `34189`:

```bash
kill -15 34189
```

---

### Bước 6: Kiểm tra xác nhận tiến trình đã dừng hoàn toàn

Chạy lệnh kiểm tra danh sách tiến trình:

```bash
ps aux | grep loop-monitor.sh
```

*Kết quả sau khi kill:*
```text
root       34205  0.0  0.1   6616  2216 pts/0    S+   20:35   0:00 grep --color=auto loop-monitor.sh
[1]+  Terminated              nohup ./loop-monitor.sh > /dev/null 2>&1
```
*(Tiến trình PID `34189` đã nhận tín hiệu SIGTERM và kết thúc an toàn với trạng thái `Terminated`, chỉ còn lại lệnh grep đang tra cứu).*

---

## 📊 TỔNG HỢP CÁC LỆNH QUẢN LÝ TIẾN TRÌNH

| Mục đích | Lệnh thực hiện |
|:---|:---|
| Chạy tiến trình nền độc lập | `nohup <command> > output.log 2>&1 &` |
| Tìm nhanh PID theo tên lệnh | `pgrep -f <process_name>` |
| Tìm PID và tên đầy đủ | `pgrep -a -f <process_name>` |
| Xem danh sách tiến trình chi tiết | `ps aux \| grep <process_name>` |
| Xem trực quan các tiến trình tiêu thụ tài nguyên | `top` hoặc `htop` |
| Tắt an toàn bằng SIGTERM (khuyên dùng) | `kill -15 <PID>` hoặc `kill <PID>` |
| Cưỡng chế dừng ngay lập tức bằng SIGKILL | `kill -9 <PID>` |
| Tắt toàn bộ tiến trình theo tên | `pkill -f <process_name>` |

---

## 💡 BÀI HỌC VÀ KINH NGHIỆM THỰC TẾ

1. **Nguyên tắc "SIGTERM trước, SIGKILL sau":** Không bao giờ lạm dụng `kill -9` ngay từ đầu. `SIGKILL` dừng đột ngột tiến trình mà không cho phép dọn dẹp bộ nhớ đệm, có thể gây hỏng file log, rò rỉ lock file hoặc làm hỏng dữ liệu cơ sở dữ liệu.
2. **Quản lý chuyển hướng output:** Khi dùng `nohup`, nếu không chỉ định chuyển hướng file log (`> file.log 2>&1`), hệ thống sẽ tự động tạo file `nohup.out` trong thư mục hiện hành.
3. **Ứng dụng trong môi trường Production:** Đối với các ứng dụng thực tế (Node.js, Python, Java), ngoài `nohup`, trong thực tế các kỹ sư DevOps thường sử dụng các Process Manager chuyên nghiệp hơn như **systemd service**, **PM2** hoặc **Docker** để tự động khởi động lại (auto-restart) khi gặp sự cố.

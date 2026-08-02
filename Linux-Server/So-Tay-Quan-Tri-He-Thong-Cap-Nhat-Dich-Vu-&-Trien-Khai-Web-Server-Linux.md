## 1. Kết nối và Quản trị Cơ bản (SSH & Power Management)
Kết nối từ xa qua SSH (Truy cập vào Terminal của Server Linux từ máy trạm Windows):
```bash
ssh vutd@192.168.50.130
```

Tắt máy chủ an toàn (Shutdown):
```bash
sudo poweroff
```
> ⚠️ **Production Rule:** Tuyệt đối không tắt ngang cửa sổ VMware (rút phích cắm đột ngột). Lệnh này giúp hệ thống dọn dẹp RAM, đóng các dịch vụ ngầm an toàn trước khi ngắt nguồn.

## 2. Quy trình Bảo trì & Cập nhật Hệ thống (APT Management)
Cập nhật danh sách gói ("Thực đơn"):
```bash
sudo apt update
```
* **Bản chất:** Chỉ tải về danh sách/phiên bản mới nhất của các phần mềm từ Kho chứa (Repositories), không cài đặt hay thay đổi file hệ thống.

Nâng cấp phần mềm thực tế:
```bash
sudo apt upgrade -y
```
* **Bản chất:** Đối chiếu với danh sách cũ, tải bản mới về cài đè lên. Tham số `-y` (Yes) dùng để tự động đồng ý, phục vụ tư duy tự động hóa.
> ⚠️ **Production Rule:** Không chạy lệnh upgrade toàn diện trên hệ thống đang chạy thật (Production) để tránh xung đột thư viện làm chết App của Dev. Chỉ nâng cấp gói cụ thể khi có lỗi bảo mật bằng lệnh `sudo apt install --only-upgrade <tên_gói>`.

## 3. Triển khai Dịch vụ Lõi (Nginx Web Server)
Cài đặt Nginx:
```bash
sudo apt install nginx -y
```
* **Bản chất:** Tải Nginx Web Server về máy. Hệ thống tự động phân tách đặc quyền bảo mật, chạy dịch vụ bằng user hạn chế quyền `www-data` để bảo vệ tài nguyên hệ thống (Root).

Kiểm tra trạng thái hoạt động:
```bash
systemctl status nginx
```
* **Thao tác:** Nhìn dòng `Active: active (running)` màu xanh lá là đạt. Nhấn phím `q` để thoát giao diện xem log.
* **Nghiệm thu (Network Verification):** Mở trình duyệt gõ `http://192.168.50.130` (Port 80). Nếu thấy giao diện "Welcome to nginx!" tức là mạng thông suốt.

### 🛠️ Troubleshooting Web Server
* **Tình huống:** Nginx báo `active` nhưng gõ IP không vào được web (báo Timed Out).
* **Nguyên nhân:** Tường lửa nội bộ (UFW) đang chặn cổng 80.
* **Lệnh khắc phục:**
  ```bash
  sudo ufw allow 'Nginx Full'
  # Hoặc: sudo ufw allow 80/tcp
  ```

## 4. Quản trị Phân quyền File System (chown & chmod)
Giải mã Cấu trúc Phân quyền Linux (`ls -l`):
Ví dụ file: `-rw-r--r-- 1 vutd vutd 623 Jul 22 02:39 index.nginx-debian.html`
* `(-)`: Loại tài nguyên (File thông thường; `d` là thư mục).
* `(rw-)`: Quyền Chủ sở hữu (Owner) -> Đọc (r), Ghi/Sửa (w).
* `(r--)`: Quyền Nhóm sở hữu (Group) -> Chỉ đọc (r).
* `(r--)`: Quyền Người dùng khác (Others) -> Chỉ đọc (r).

### 🛠️ Troubleshooting Phân quyền ("Permission Denied")
* **Tình huống:** Báo lỗi `-bash: index.nginx-debian.html: Permission denied`.
* **Giải pháp:** Chuyển quyền làm chủ file về đúng User/Group cần thao tác bằng `chown`:
  ```bash
  sudo chown vutd:vutd index.nginx-debian.html
  ```

### Quy chuẩn Phân quyền Doanh nghiệp (Security Standard)
> ⚠️ **Production Rule:** Tuyệt đối KHÔNG DÙNG `chmod 777` trên môi trường Production. Việc cấp quyền Execute (x) và Write (w) cho nhóm Others sẽ tạo lỗ hổng cho Hacker tải mã độc.

| Đối Tượng | Mức Quyền (chmod) | Ý Nghĩa Sử Dụng |
| :--- | :---: | :--- |
| **Thư mục Web** | `755` (rwxr-xr-x) | Owner full quyền; Group & Others chỉ xem/truy cập. |
| **File Web** (HTML, PHP) | `644` (rw-r--r--) | Owner đọc/sửa; Nginx đọc để render web cho khách. |
| **File Config / DB Password** | `600` (rw-------) | Duy nhất Owner có quyền đọc/sửa. Cách ly 100%. |

Giám sát Log Nginx (Troubleshooting Web Errors realtime):
```bash
# Log lỗi dịch vụ (502, 500)
sudo tail -f /var/log/nginx/error.log

# Log lịch sử truy cập (HTTP 200, 404)
sudo tail -f /var/log/nginx/access.log
```
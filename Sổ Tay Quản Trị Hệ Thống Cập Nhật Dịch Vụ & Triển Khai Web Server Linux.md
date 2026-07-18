
## 1. Kết nối và Quản trị Cơ bản (SSH & Power Management)
*   **Kết nối từ xa qua SSH:**
    ```bash
    ssh vutd@192.168.50.130
    ```
    *Mục đích:* Truy cập vào Terminal của Server Linux từ máy trạm Windows thông qua mạng nội bộ.
*   **Tắt máy chủ an toàn (Shutdown):**
    ```bash
    sudo poweroff
    ```
    *⚠️ Production Rule:* Tuyệt đối không tắt ngang cửa sổ VMware (rút phích cắm đột ngột). Lệnh này giúp hệ thống dọn dẹp RAM, đóng các dịch vụ ngầm an toàn trước khi ngắt nguồn.

---

## 2. Quy trình Bảo trì & Cập nhật Hệ thống (APT Management)
*   **Cập nhật danh sách gói ("Thực đơn"):**
    ```bash
    sudo apt update
    ```
    *Bản chất:* Chỉ tải về danh sách/phiên bản mới nhất của các phần mềm từ Kho chứa (Repositories), không cài đặt hay thay đổi file hệ thống.
*   **Nâng cấp phần mềm thực tế:**
    ```bash
    sudo apt upgrade -y
    ```
    *Bản chất:* Đối chiếu với danh sách cũ, tải bản mới về cài đè lên. Tham số `-y` (Yes) dùng để tự động đồng ý với các câu hỏi xác nhận, phục vụ tư duy tự động hóa (Automation script).
    *⚠️ Production Rule:* Không chạy lệnh upgrade toàn diện trên hệ thống đang chạy thật (Production) để tránh xung đột thư viện làm chết App của Dev. Chỉ nâng cấp gói cụ thể khi có lỗi bảo mật bằng lệnh `sudo apt install --only-upgrade <tên_gói>`.

---

## 3. Triển khai Dịch vụ Lõi (Nginx Web Server)
*   **Cài đặt Nginx:**
    ```bash
    sudo apt install nginx -y
    ```
    *Bản chất:* Tải Nginx Web Server về máy. Trình quản lý gói APT tự động giải quyết cây phụ thuộc (Dependency Resolution). Hệ thống tự động phân tách đặc quyền bảo mật, chạy dịch vụ bằng user hạn chế quyền `www-data` để bảo vệ tài nguyên hệ thống (Root).
    *Log hệ thống:* Tự động tạo liên kết động (Symlink) vào `multi-user.target.wants` chuyển dịch vụ sang chế độ `enabled` (Tự động khởi động cùng hệ điều hành).

*   **Kiểm tra trạng thái hoạt động:**
    ```bash
    systemctl status nginx
    ```
    *Thao tác:* Nhìn dòng `Active: active (running)` màu xanh lá là đạt. Nhấn phím `q` để thoát giao diện xem log.
    *Xác thực kiến trúc tiến trình ngầm (CGroup):* Hệ thống sinh ra 1 tiến trình điều phối tối cao `master process` (Main PID: 1785) và 2 tiến trình con `worker process` (PID: 1788 và 1789) chạy bất đồng bộ để phân phối tải, chứng minh cấu trúc Event-driven tối ưu RAM/CPU của Nginx hoạt động đúng thiết kế.

*   **Nghiệm thu thực tế (Network Verification):**
    *   *Thao tác:* Mở trình duyệt trên máy thật: `http://192.168.50.130` (Cổng mặc định HTTP: Port 80).
    *   *Kết quả đạt:* Giao diện hiển thị thành công trang mặc định **"Welcome to nginx!"**, xác thực luồng mạng thông suốt từ ngoài vào trong, tường lửa không chặn mạch kết nối TCP.

---

## 🛠️ Troubleshooting (Xử lý sự cố thực chiến)
*   **Tình huống:** Nginx báo `active (running)` nhưng máy Windows gõ IP không vào được web (báo Timed Out).
*   **Nguyên nhân:** Tường lửa nội bộ (Firewall) của Ubuntu chặn cổng 80.
*   **Lệnh khắc phục:** 
    ```bash
    sudo ufw allow 'Nginx Full'
    ```
    hoặc 
    ```bash
    sudo ufw allow 80/tcp
    ```
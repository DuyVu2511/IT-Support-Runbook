# TỔNG QUAN NGINX & TRIỂN KHAI BẰNG DOCKER CONTAINER

---

## 📌 PHẦN 1: BẢN CHẤT VÀ VAI TRÒ CỦA NGINX (ENTERPRISE LEVEL)
Nginx không chỉ là một phần mềm cài cho vui, nó là hạ tầng mạng cốt lõi (Infrastructure Software) với 3 vai trò sinh tử trong doanh nghiệp:

*   **Web Server (Trạm phục vụ Web):** Lưu trữ và phân phối trực tiếp các nội dung tĩnh (giao diện HTML, CSS, hình ảnh) đến trình duyệt người dùng khi họ gõ IP/Domain.
*   **Reverse Proxy (Lễ tân bảo mật - Quan trọng nhất):** Đứng ở "tiền tuyến" nhận luồng truy cập từ Internet, sau đó ẩn danh và gọi các ứng dụng nội bộ (như Zabbix, Backend API) ở phía sau. Giúp giấu kín kiến trúc hệ thống khỏi Hacker.
*   **Load Balancer (Điều phối giao thông):** Khi có hàng trăm ngàn người truy cập, Nginx sẽ chia đều tải trọng (traffic) ra nhiều Server phía sau, đảm bảo hệ thống không bị quá tải cục bộ.

---

## 📌 PHẦN 2: LAB THỰC CHIẾN - ĐÓNG GÓI NGINX BẰNG DOCKER

### 1. Giải quyết bài toán bằng Docker (Containerization)
*   **Vấn đề:** Tránh xung đột môi trường, thư viện phần mềm trên máy chủ vật lý (Bare-metal OS).
*   **Giải pháp:** Bọc ứng dụng thành các Container độc lập, khởi động trong 0.5s, dùng chung Kernel nhưng cách ly hoàn toàn về tiến trình.

### 2. Cài đặt nền tảng Docker Engine (Trên Ubuntu)
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable --now docker
```

### 3. Triển khai Web Server Container siêu tốc
Yêu cầu: Kéo bản Nginx mới nhất về, chạy ngầm, không để xung đột với Port 80 gốc của máy chủ.
```bash
sudo docker run -d --name my-web-container -p 8888:80 nginx
```

### 4. Cấu hình tự động sống lại sau khi Reboot (Restart Policy)
Đảm bảo Container tự động khởi động cùng hệ thống khi Server bị tắt đột ngột:
```bash
sudo docker update --restart unless-stopped my-web-container
```

### 5. Lệnh quản trị & Troubleshoot khẩn cấp
```bash
# Xem danh sách Container đang chạy và Port đang chiếm
sudo docker ps

# Đọc Log lỗi của ứng dụng bên trong (Kỹ năng Debug)
sudo docker logs my-web-container

# Tắt / Bật / Xóa một Container
sudo docker stop my-web-container
sudo docker start my-web-container
sudo docker rm my-web-container
```
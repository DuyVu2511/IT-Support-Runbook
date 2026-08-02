# TỔNG QUAN NGINX & TRIỂN KHAI BẰNG DOCKER CONTAINER

## 📌 PHẦN 1: BẢN CHẤT VÀ VAI TRÒ CỦA NGINX (ENTERPRISE LEVEL)
Nginx là hạ tầng mạng cốt lõi (Infrastructure Software) với 3 vai trò sinh tử:
1. **Web Server (Trạm phục vụ Web):** Lưu trữ và phân phối nội dung tĩnh.
2. **Reverse Proxy (Lễ tân bảo mật - Quan trọng nhất):** Đứng ở "tiền tuyến" nhận luồng truy cập, ẩn danh và gọi các ứng dụng nội bộ (Zabbix, API) phía sau.
3. **Load Balancer (Điều phối giao thông):** Chia đều tải trọng ra nhiều Server phía sau.

## 📌 PHẦN 2: LAB THỰC CHIẾN - ĐÓNG GÓI NGINX BẰNG DOCKER

### 1. Giải quyết bài toán bằng Docker (Containerization)
* **Vấn đề:** Xung đột môi trường, thư viện trên máy chủ vật lý.
* **Giải pháp:** Bọc ứng dụng thành Container độc lập, cách ly tiến trình hoàn toàn.

### 2. Cài đặt Docker Engine (Ubuntu)
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable --now docker
```

### 3. Triển khai Web Server Container siêu tốc
Kéo bản Nginx mới nhất về, chạy ngầm, map port 8888 ra ngoài để không xung đột với Port 80 gốc:
```bash
sudo docker run -d --name my-web-container -p 8888:80 nginx
```

### 4. Cấu hình tự động sống lại (Restart Policy)
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

### 6. Cấu hình Nginx nâng cao: Lột mặt nạ Reverse Proxy (Bắt IP thật)
* **🚨 Vấn đề (Bẫy mạng nội bộ):** Khi Nginx làm Reverse Proxy, Backend khi đọc Access Log sẽ chỉ thấy IP của Nginx (`127.0.0.1`), mất dấu IP thật của hacker/client.
* **✅ Giải pháp chuẩn SysAdmin:** Ép Nginx đính kèm IP thật của Client vào Header. Thêm 2 dòng sau vào block `location` của cấu hình Nginx:
  ```nginx
  location / {
      # Các cấu hình proxy_pass hiện tại...
      
      # Bắt và forward IP thật của Client ra phía sau
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
  ```
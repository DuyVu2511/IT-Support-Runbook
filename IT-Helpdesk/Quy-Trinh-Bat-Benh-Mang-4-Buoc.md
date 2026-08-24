# Quy trình 4 bước "Bắt bệnh" Mạng & Dịch vụ (Sysadmin Troubleshooting)

Khi mất kết nối đến Server hoặc Dịch vụ, luôn tuân thủ quy trình kiểm tra theo mô hình OSI từ dưới lên trên (Từ kết nối vật lý/LAN lên đến tầng Ứng dụng).

## 1. Kiểm tra IP & Gateway (Lớp Mạng)
```cmd
ipconfig /all
```
- **Mục đích:** Xác định xem máy trạm đã nhận đúng dải IP và Default Gateway cấp bởi DHCP hay chưa.

## 2. Kiểm tra thông luồng mạng (Ping)
```cmd
ping <IP_Gateway>
ping 8.8.8.8
```
- **Mục đích:** Ping Gateway để xem mạng LAN nội bộ có thông không. Ping `8.8.8.8` để kiểm tra kết nối ra Internet có bị Firewall chặn hay rớt gói (Packet loss) không.

## 3. Kiểm tra phân giải tên miền (DNS)
```cmd
nslookup <tenmien.com>
```
- **Mục đích:** Đảm bảo máy chủ DNS đang hoạt động và phân giải đúng IP của hệ thống đích (Tránh lỗi do trỏ nhầm IP cũ).

## 4. Kiểm tra Cổng dịch vụ (Telnet/TCP Handshake)
```cmd
telnet <IP_hoặc_Tên_Miền> <Port>
```
- **Mục đích:** Kiểm tra xem Firewall có chặn Port không và dịch vụ (Web/DB) có đang lắng nghe (Listening) không.

> [!WARNING] Dấu hiệu nghiệm thu cực kỳ quan trọng
> - **Lỗi Timeout / Connect failed:** Mạng chưa thông hoặc Firewall đang chặn Port chặn IP.
> - **Màn hình đen xì:** Quá trình TCP 3-way handshake **THÀNH CÔNG**. Port đang mở toang.
> - **Lỗi `HTTP/1.1 400 Bad Request` hoặc `Connection to host lost`:** Dấu hiệu **RẤT TỐT**. Báo hiệu dịch vụ (Nginx/Apache) đang chạy bình thường. Lỗi này xuất hiện do thao tác gõ phím bừa bãi trong màn hình Telnet khiến Web Server nhận diện đây là một request rác (không tuân thủ chuẩn cấu trúc HTTP) nên tự động ngắt kết nối bảo vệ server.
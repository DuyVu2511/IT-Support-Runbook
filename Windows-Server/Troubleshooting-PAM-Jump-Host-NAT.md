# Runbook: Xử Lý Lỗi Định Tuyến Khi Tích Hợp Jump Host Vào PAM (Môi Trường Không VPN)

## 1. Hiện Tượng Lỗi (Symptom)
* **Ngữ cảnh:** Tích hợp máy Jump Host đặt tại site đối tác/khách hàng vào hệ thống PAM quản lý tập trung đặt tại HQ.
* **Lỗi:** Khi mở phiên RDP/SSH từ web PAM, hệ thống báo lỗi `Connection timed out` hoặc `Server unreachable`.

## 2. Phân Tích Nguyên Nhân (Root Cause)
* **Lỗi định tuyến (Routing):** Giữa HQ và Client Site **không có đường truyền Site-to-Site VPN**. 
* Khi khai báo **IP Private** của Jump Host lên PAM, gói tin đi đến Router biên bị drop do không có tuyến đường (route) trỏ vào dải mạng nội bộ của site đối tác.

## 3. Giải Pháp Khắc Phục (Resolution)
1. **Phía Network (Client Site):** Cấu hình NAT / Port Forwarding trên Firewall biên. Ánh xạ cổng dịch vụ (RDP 3389 / SSH 22) của IP Private bên trong ra một **IP Public** đại diện.
2. **Phía System (PAM):** Cập nhật Asset trên PAM sang địa chỉ **IP Public** đã được NAT.
3. **Kiểm tra luồng (Troubleshooting Command):** 
   Do gói tin qua Firewall bị SNAT, đứng tại máy Jump Host chạy lệnh PowerShell sau để kiểm tra phiên kết nối:
```powershell
Get-NetTCPConnection -LocalPort 3389 -State Established | Select-Object LocalAddress, LocalPort, RemoteAddress, State
```

## 4. Best Practices: Chuẩn Bị Môi Trường Quản Trị
Trang bị các công cụ an toàn (Open Source / Freeware) trên Jump Host:
* **Quản lý mật khẩu:** Sử dụng **KeePass Portable** (`.kdbx`). Quản lý offline cục bộ, không đồng bộ cloud. Cấu hình tự động khóa database khi không hoạt động.
* **Quản lý Session:** Cài đặt **Microsoft RDCMan** (gom tab RDP) hoặc **mRemoteNG** (gom tab SSH/RDP). Không dùng MobaXterm Home Edition để tránh vi phạm chính sách bản quyền doanh nghiệp.
* **Trình duyệt:** Gom nhóm toàn bộ URL quản trị nội bộ (vCenter, Storage, Web Service) vào thư mục Bookmark trên trình duyệt của máy Jump.

## 5. Tư Duy SysAdmin (Phỏng Vấn & Thực Chiến)
"Khi triển khai Jump Host tích hợp PAM khác site vật lý mà không có kênh truyền VPN riêng, việc trỏ IP Private sẽ gây lỗi định tuyến. Cần phối hợp với Network cấu hình NAT qua IP Public trên Firewall biên, mở đúng cổng tối thiểu để thu hẹp bề mặt tấn công. Trên máy Jump Host, triển khai KeePass và các công cụ gom tab chính quy nhằm tối ưu hóa thao tác vận hành và bảo vệ an toàn thông tin."
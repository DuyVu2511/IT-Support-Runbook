# Kiến Thức Nền Tảng: WSUS & Docker Container Trong Hạ Tầng Doanh Nghiệp

## I. DỊCH VỤ CẬP NHẬT TẬP TRUNG (WSUS - Windows Server Update Services)
* **Bài toán Doanh nghiệp:**
  1. Tiết kiệm Băng thông: Tránh việc 300 máy tính cùng lúc ra Internet tải bản vá Win 11 (300 x 5GB = Sập mạng). Với WSUS, chỉ Server tải 1 lần duy nhất rồi đẩy qua LAN.
  2. Quản trị Rủi ro (BSOD Avoidance): Không cho máy con tự Update. SysAdmin phải kiểm thử (Test) bản vá trước, nếu an toàn mới bấm Approve đẩy xuống máy trạm qua GPO.
* **Lệnh Troubleshoot Máy Trạm Không Nhận WSUS:**
  * gpupdate /force (Ép nhận GPO trỏ về WSUS)
  * USOClient StartScan (Ép máy trạm quét bản vá ngay lập tức)

## II. ẢO HÓA CONTAINER (Docker vs VMware)
* **So Sánh Cốt Lõi:**
  * VMware/Hyper-V: Ảo hóa cấp Phần cứng (Hardware-level). Mỗi VM mang theo 1 Guest OS riêng -> Nặng, tốn RAM/Disk, khởi động lâu.
  * Docker: Ảo hóa cấp Hệ điều hành (OS-level). Các Container dùng chung Kernel với máy Host -> Siêu nhẹ, khởi động trong 1-2 giây.
* **Mô hình triển khai chuẩn trong Enterprise (Hybrid):**
  * Dùng VMware tạo các VM Linux -> Cài Docker Engine lên các VM đó -> Chạy các ứng dụng Web/Nginx dạng Container bên trong.
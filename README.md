# 🛡️ Kho Tài Liệu Vận Hành Hệ Thống (IT Support & SysAdmin Runbook)

Chào mừng đến với kho lưu trữ Runbook cá nhân của tôi. Đây là nơi tôi tài liệu hóa (document) toàn bộ các quy trình vận hành chuẩn (SOP), kịch bản tự động hóa (PowerShell/Bash) và hướng dẫn khắc phục sự cố (Troubleshooting) trong quá trình xây dựng HomeLab mô phỏng hạ tầng doanh nghiệp.

## 📂 Cấu trúc thư mục (Directory Structure)

Kho lưu trữ này được chia thành các phân hệ lõi của một hệ thống IT:

*   **`/Windows-Server/`**: Tài liệu quản trị Active Directory, cấu hình WSUS, phân quyền AGDLP và các script PowerShell tự động hóa (Cấp phát user hàng loạt, truy vết log Event Viewer).
*   **`/Linux-Server/`**: Cấu hình và vận hành Nginx Web Server trên môi trường Ubuntu, quản lý phân quyền (chmod/chown) và theo dõi tiến trình (systemctl).
*   **`/IT-Helpdesk/`**: Kịch bản xử lý sự cố mạng cơ bản (Subnetting, VLAN), fix lỗi Print Spooler và cứu hộ dữ liệu (Data First) bằng WinPE.
*   **`/Media-Assets/`**: Nơi lưu trữ tài nguyên hình ảnh/video minh họa cho các bài Lab.

## 🚀 Các tài liệu nổi bật (Featured Runbooks)

1.  [Tự động hóa cấp phát tài khoản AD bằng PowerShell](./Windows-Server/Script-Tao-User-Hang-Loat-PowerShell.md)
2.  [Truy vết sự cố tài khoản qua Event Viewer Log](./Windows-Server/Truy-Vet-Log-Active-Directory-EventViewer.md)
3.  [Quy hoạch Subnet & VLAN cho mạng LAN](./IT-Helpdesk/Quy-Hoach-Subnet-Va-VLAN-Networking.md)
4.  [Bản chất Ảo hóa Container (Docker) vs System (WSUS)](./Windows-Server/Ban-Chat-SysAdmin-Docker-Vs-WSUS.md)

---
*Tài liệu được xây dựng dựa trên tiêu chuẩn thực hành "Data First" và hướng tới tự động hóa quy trình vận hành IT.*

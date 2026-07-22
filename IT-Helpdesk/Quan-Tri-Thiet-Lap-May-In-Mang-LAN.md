# Sổ Tay Quản Trị & Troubleshooting Máy In Mạng LAN (Network Printer)

## 1. Quy trình 3 bước Thiết lập Máy in qua IP

* **Cấu hình IP Tĩnh (Static IP):**
    * *Thao tác:* Đặt IP tĩnh cố định trực tiếp tại màn hình cảm ứng/phím bấm của Máy in (Tắt DHCP).
    * *Bản chất:* Tránh việc Router cấp đổi IP khác khi máy in khởi động lại làm mất kết nối toàn bộ công ty.

* **Kiểm tra thông mạng (Ping Test):**
    * *Thao tác:* Mở CMD trên máy trạm Windows và gõ:
        ```cmd
        ping 192.168.50.250
        ```
    * *Nghiệm thu:* Nhận phản hồi `Reply from...` là mạng nội bộ đã thông suốt.

* **Add Printer trên Windows qua TCP/IP Port:**
    * *Thao tác:* Control Panel -> Devices and Printers -> Add printer using an IP address or hostname -> Chọn `TCP/IP Device` -> Điền IP -> Chọn Driver hãng (hoặc `Generic / Text Only`).

---

## 2. Troubleshooting Thực Chiến (Xử Lý Sự Cố Máy In)

* **Tình huống 1: Máy in vẫn Ping thấy IP nhưng Windows báo "Offline"**
    * *Nguyên nhân:* Windows tự động bật tính năng giám sát giao thức SNMP nhưng máy in không hỗ trợ.
    * *Cách fix:* `Printer Properties` -> Tab **Ports** -> Chọn Cổng IP -> Bấm **Configure Port...** -> **Bỏ tích chọn `SNMP Status Enabled`** -> Bấm OK.

* **Tình huống 2: Lệnh in bị treo/kẹt (Print Queue Stuck), xóa không được**
    * *Nguyên nhân:* Tiến trình ngầm `Print Spooler` bị treo, file tạm lệnh in bị khóa trong hệ thống.
    * *Quy trình xả kẹt chuẩn SysAdmin:*
        Mở **CMD (Run as Administrator)** và chạy lệnh dừng dịch vụ:
        ```cmd
        net stop spooler
        ```
        *Xóa toàn bộ file tạm kẹt bên trong thư mục:* `C:\Windows\System32\spool\PRINTERS`
        
        Chạy lệnh khởi động lại dịch vụ:
        ```cmd
        net start spooler
        ```

---
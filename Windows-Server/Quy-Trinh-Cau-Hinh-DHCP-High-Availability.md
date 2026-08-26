# QUY TRÌNH CẤU HÌNH DHCP HIGH AVAILABILITY (FAILOVER)

## 1. Yêu cầu chuẩn bị
- 02 máy chủ đã cài đặt Role DHCP Server (Ví dụ: `DC01` và `DC02`).
- Cả 2 máy đã Join Domain.
- Sử dụng tài khoản có quyền **Enterprise Admin** hoặc **Domain Admin** để thực hiện.

## 2. Các bước thực hiện

### Bước 1: Authorize DHCP Server trong Active Directory
- Mở Server Manager -> DHCP.
- Chuột phải vào tên máy chủ (IPv4) -> Chọn **Authorize** để hệ thống AD cho phép máy chủ này được quyền cấp phát IP (Ngăn chặn Rogue DHCP).

### Bước 2: Tạo Scope trên Node Primary (`DC01`)
- Chuột phải IPv4 -> **New Scope...**
- Khai báo dải IP (Ví dụ: 10.10.10.10 đến 10.10.10.200).
- Cấu hình Subnet mask, Default Gateway và DNS Server (Trỏ về IP của máy DC).

### Bước 3: Cấu hình Failover đồng bộ sang Node Secondary (`DC02`)
- Chuột phải vào Scope vừa tạo trên `DC01` -> Chọn **Configure Failover...**
- Tích chọn Select all (hoặc chọn Scope cụ thể) -> Next.
- Ở mục *Partner Server*, nhấn Add Server và trỏ tới tên máy `DC02`.
- Cấu hình chế độ chạy (Mode):
  - **Load Balance (Mặc định):** Hai máy cùng chia nhau cấp IP theo tỷ lệ 50-50 (Active-Active).
  - **Hot Standby:** Máy 1 cấp chính, máy 2 chờ. Khi máy 1 sập, máy 2 mới nhảy lên cấp IP (Active-Passive).
- Nhập **Shared Secret** (Mật khẩu đồng bộ giữa 2 máy).
- Bấm Finish. Kiểm tra trên `DC02` đã xuất hiện Scope y hệt là thành công.
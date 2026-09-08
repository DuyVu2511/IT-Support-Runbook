# Cẩm Nang Triển Khai Jump Host & Tích Hợp Hệ Thống PAM (JumpServer)

> **Thông tin chung:**
> - **Người thực hiện:** `[My_Account]`
> - **Dự án:** Triển khai hạ tầng truy cập từ xa cho `[Client_Site]`
> - **Phân loại:** Windows-Server / PAM / Network-Troubleshooting
> - **Trạng thái:** Hoàn tất cấu hình hệ thống (100%) - Đang chờ Network thông luồng Site-to-Site.

---

## 1. Mô hình Môi trường (Topology & Architecture)
- **Site A (HQ - Công ty):**
  - Cổng Bastion / PAM tập trung: JumpServer v4.x (IP: `[IP_Jump_HQ]`).
  - Phân quyền vận hành: Giao diện Web Console (System Admin) và Web Workbench (End-User).
- **Site B (Client Site):**
  - Máy chủ máy nhảy (Jump Host): Windows Server (IP tĩnh: `[IP_Jump_Client]`, Default Gateway: `[IP_Gateway]`).
  - Vị trí mạng: Nằm trong phân vùng hạ tầng `[VLAN_INFRA]`.
- **Ranh giới kết nối (Dependencies):**
  - Đường hầm bảo mật: Site-to-Site VPN / Routing do Network Team quản lý.
  - Luồng cho phép: `Source: [IP_Jump_HQ]` $\rightarrow$ `Dest: [IP_Jump_Client]`, giao thức `TCP`, Port `3389` (RDP).

---

## 2. Chuẩn Hóa & Cài Đặt Jump Host (OS & Firewall)
1. **Cấu hình mạng cơ bản:**
   - Đặt IP tĩnh `[IP_Jump_Client]/24` và trỏ Gateway chuẩn về `[IP_Gateway]`.
   - Cài đặt và xác minh trạng thái hoạt động của **VMware Tools**.
2. **Kích hoạt Remote Desktop & Mở Tường Lửa:**
   - Xử lý bật Remote Desktop và mở policy tường lửa cục bộ bằng PowerShell:
   ```powershell
   # Bật tính năng Remote Desktop qua Registry
   Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -name "fDenyTSConnections" -value 0

   # Mở luồng cho dịch vụ Remote Desktop trên Windows Firewall
   Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
   ```

---

## 3. Phân Tích Dữ Liệu Hạ Tầng & Khảo Sát Định Tuyến (Infrastructure Audit)
- **Khối lượng xử lý dữ liệu:** Hợp nhất, làm sạch và đối soát chéo dữ liệu từ **6 file CSV** trích xuất từ vCenter (tổng cộng **[X] máy ảo**).
- **Kết quả khảo sát thông luồng nội bộ:**
  - **Cụm Hạ tầng Mới (`[Zone_Moi]`):** Đạt tỷ lệ thông suốt định tuyến **98.4%** số lượng VM có IP hoạt động, trải đều trên 12 dải mạng `/24`.
  - **Cụm Kế thừa Cũ (`[Zone_Cu]`):** Xác định chỉ gồm 3 máy ảo thuộc dải `[IP_Range_Old]` đang bị cô lập. Đã làm rõ đây là cụm phụ, không làm ảnh hưởng đến tiến độ nghiệm thu cụm chính.
- **Nghiệm thu kết nối cục bộ (Local Testing):**
  - Từ máy Jump Host `[IP_Jump_Client]`, kiểm thử kết nối thành công tới các dịch vụ lõi:
    - Quản trị vCenter nội bộ: `[IP_vCenter]`
    - Cụm lưu trữ đối tượng MinIO: `[IP_MinIO]:9000`
    - Các máy chủ Database và Application thuộc dải nội bộ.

---

## 4. Tích Hợp Hệ Thống Quản Trị Đặc Quyền (JumpServer PAM)
Thao tác cấu hình trên Web Console của JumpServer tổng (`[IP_Jump_HQ]`):

1. **Khởi tạo Tài sản (Asset Registration):**
   - **Tên hiển thị:** `Jump-[Client_Site]-[IP_Jump_Client]`
   - **Địa chỉ IP:** `[IP_Jump_Client]`
   - **Nền tảng (Platform):** `Windows` (tự động gắn giao thức RDP port `3389`).
   - **Cây thư mục (Asset Tree):** `/DEFAULT/[Client_Site]`.
2. **Cấu hình Tài khoản Quản trị (Account Mapping):**
   - Tài khoản: `Administrator` (đặt cờ `Privileged` / Tài khoản đặc quyền).
   - Thiết lập mật khẩu khớp với mật khẩu quản trị cục bộ của Windows Server.
   - *Lưu ý an toàn:* Không kích hoạt cờ `Push now` khi mạng chưa thông.
3. **Chính sách Phân quyền Truy cập (Authorizations):**
   - Tên chính sách: `Perm-[Client_Site]-Jump`
   - **Users / Groups:** Gán quyền cho nhóm kỹ thuật `[Team_Name]` và tài khoản cá nhân `[My_Account]`.
   - **Assets:** Ánh xạ vào máy `Jump-[Client_Site]-[IP_Jump_Client]`.
   - **Accounts:** Cho phép sử dụng tài khoản `Administrator`.
   - **Hành động cho phép:** Bật đủ quyền `Connect`, `Clipboard` (Copy/Paste), và `Transfer` (Truyền file).
4. **Xác thực kết quả trên Màn hình làm việc (Workbench):**
   - Asset đã xuất hiện trực quan trên giao diện Workbench của người dùng.
   - Khi bấm Connect, hệ thống tự động khởi tạo session Luna RDP và dừng ở trạng thái `Connection failed (server unreachable?)`. Đây là phản hồi chính xác xác nhận phần mềm đã cấu hình chuẩn, chỉ chờ thông tuyến mạng vật lý.

---

## 5. Nhật Ký Xử Lý Sự Cố & Tối Ưu (Troubleshooting & Effort)
- **Xử lý Tường lửa (Firewall Policy):** Ban đầu máy Windows Server mặc định drop toàn bộ gói tin kết nối cổng 3389 từ mạng ngoài. Đã can thiệp trực tiếp bằng PowerShell để mở rule Terminal Server mà không cần khởi động lại máy.
- **Rà soát dữ liệu định tuyến:** Thay vì chỉ đứng từ một máy để ping ngẫu nhiên, đã chủ động lọc và vẽ lại bản đồ mạng cho toàn bộ [X] VMs từ dữ liệu vCenter, giúp chứng minh hệ sinh thái đã sẵn sàng tới 98.4%.
- **Tách biệt ranh giới sự cố (Issue Isolation):** Phân định rạch ròi giữa việc "Cấu hình ứng dụng PAM đã hoàn tất 100%" và "Lỗi kết nối do hạ tầng định tuyến Site-to-Site của Network Team", giúp việc báo cáo tiến độ minh bạch và đúng trách nhiệm.

---

## 6. Kế Hoạch Tiếp Theo (Next Steps & Handover)
1. **Chờ tín hiệu Network:** Đội Network kích hoạt xong tuyến định tuyến giữa 2 site.
2. **Kiểm thử nghiệm thu (UAT):**
   - Truy cập giao diện PAM Workbench `[IP_Jump_HQ]` $\rightarrow$ Click vào Asset `Jump-[Client_Site]-[IP_Jump_Client]`.
   - Xác nhận vào thẳng màn hình Desktop của Windows Server trên trình duyệt.
   - Thực hiện lệnh test End-to-End từ Jump Host chọc tiếp vào các máy ảo nội bộ.
3. **Bàn giao chính thức:** Chụp ảnh màn hình phiên kết nối thành công để báo cáo nghiệm thu với `[Mentor_Name]`.
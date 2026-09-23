# Runbook: Triển Khai Toàn Diện PAM Gateway & RemoteApp Qua Mạng Cô Lập

## 1. Bối cảnh & Mục tiêu (Overview)
- **Bối cảnh:** Tích hợp hệ thống quản trị truy cập đặc quyền (PAM) vào một cụm mạng cô lập (Air-Gapped / DMZ) của đối tác. Máy Proxy trung gian không có Internet và không có DNS nội bộ.
- **Mục tiêu:** Xây dựng trạm trung chuyển (Jump Host) chuẩn Zero-Trust. Cung cấp truy cập SSH/RDP (Layer 4) và đặc biệt là truy cập Web nội bộ (Layer 7 - RemoteApp/Seamless Browser) mà không để lộ Desktop của máy proxy, đồng thời phân quyền và ghi hình được từng phiên làm việc.

---

## 2. Giai đoạn 1: Chuẩn bị Hệ điều hành (Máy Proxy)

### Xử lý lỗi sập nguồn máy Proxy mỗi 60 phút
- **Hiện tượng:** Máy chủ Windows dùng làm Proxy liên tục tự tắt nguồn. Event Log báo tiến trình `wlms.exe` (Windows Licensing Monitoring Service) kích hoạt.
- **Nguyên nhân:** Windows Server bản Evaluation đã hết hạn dùng thử (Grace period).
- **Cách khắc phục:** Mở CMD (Run as Admin), chạy lệnh `slmgr.vbs /rearm` và khởi động lại máy để làm mới bộ đếm thời gian.

---

## 3. Giai đoạn 2: Thiết lập Cầu nối SSH (Layer 4 Gateway)

### 3.1. Cài đặt OpenSSH Server Offline
- **Vấn đề:** Không thể cài đặt qua `Add-WindowsCapability` (lỗi timeout `0x8024402c`) do máy Proxy nằm trong mạng cô lập, bị chặn Windows Update.
- **Cách khắc phục:** Tải gói `Win32-OpenSSH.zip` offline, giải nén và chạy script `install-sshd.ps1` bằng PowerShell.

### 3.2. Cấu hình SSH Daemon (sshd_config)
- Mở file `C:\ProgramData\ssh\sshd_config`.
- Thay đổi Port mặc định sang Port custom (VD: `22022`).

> [!danger] Quan trọng: Cấu hình ForceCommand
> Bắt buộc phải tìm và comment out (thêm dấu `#` ở đầu) dòng `ForceCommand`.
> Nếu để nguyên, hệ thống PAM sẽ bị chặn và không thể đẩy lệnh điều khiển (Playbook) vào máy Proxy.

- Khởi động lại dịch vụ SSH:
```powershell
Restart-Service sshd
```

### 3.3. Xử lý Luồng Mạng (Tường lửa pfSense/Router Biên)
Yêu cầu Network Team cấu hình 3 luồng định tuyến bắt buộc:
1. **Inbound SSH (Layer 4):** NAT Port custom (VD: `22022`) từ WAN vào máy Proxy để tạo đường hầm ngầm.
2. **Inbound RDP (Layer 7):** NAT Port `3389` từ WAN vào máy Proxy để truyền luồng hình ảnh trình duyệt ảo.
3. **Outbound API:** Policy Routing cho phép máy Proxy đi ra ngoài Internet (Port 80/443) tới đúng IP của PAM Core Server. Nếu thiếu luồng này, tiến trình `tinkerd` sẽ báo lỗi ngắt kết nối.

---

## 4. Giai đoạn 3: Tích hợp PAM & Khởi tạo RemoteApp Proxy

### 4.1. Khởi tạo Service Account Gateway

> [!warning] Chú ý Quy tắc Đặt tên (Naming Convention)
> Tuyệt đối không đặt tên User có chứa các tiền tố mặc định của PAM (Ví dụ: `js_gateway`, `jms_admin`).
> **Lý do:** Các script bảo trì chạy ngầm của PAM (dưới quyền SYSTEM) sẽ tự động quét và xóa sạch các tài khoản này vì cho rằng đó là phiên làm việc bị rác, dẫn đến lỗi `Authentication failed` ngẫu nhiên.

**Cách khắc phục:**
- Tạo tài khoản độc lập hoàn toàn (Ví dụ: `gw_service`).
- Thiết lập `PasswordNeverExpires = True`.
- Tuân thủ Least Privilege: Không đưa vào nhóm Administrators nếu không bắt buộc, chỉ cần quyền Logon qua SSH/RDP.

### 4.2. Triển khai RemoteApp (Tinker)
- Trên giao diện PAM, tạo cấu hình RemoteApp Publisher trỏ về IP và cấu hình của máy Proxy.
- Thực thi Playbook/Deploy. Kiểm tra log trả về trạng thái `failed=0` và tiến trình `tinkerd` running là thành công. Lúc này máy Proxy đã sẵn sàng khởi tạo Chrome ảo.

---

## 5. Giai đoạn 4: Định tuyến Web Asset & Xử lý Dữ liệu Hàng Loạt

### 5.1. Gán Tag Điều Hướng

> [!info] Bảo toàn cấu trúc Node Path
> Khi ép hàng chục Web Asset đi qua máy Proxy, nếu dùng tính năng Bulk Update thông thường và tích sai ô, toàn bộ cấu trúc phân cấp (Node Path) của hệ thống sẽ bị ghi đè thành 1 thư mục duy nhất.

**Thao tác an toàn:**
1. Bôi đen danh sách thiết bị cần sửa -> Chọn **Edit Selected**.
2. Tại hàng Attributes, **BỎ TÍCH TẤT CẢ** các ô (Node, Platform, Active...).
3. **CHỈ TÍCH DUY NHẤT** ô `[x] Tags`.
4. Thêm Tag RemoteApp (VD: `AppletHostOnly:[Tên_Proxy]`) và lưu lại. Luồng mạng sẽ được cập nhật mà cấu trúc thư mục giữ nguyên 100%.

### 5.2. Khắc phục lỗi DNS cho hệ thống SSO (vCenter)
- **Vấn đề:** Các Web Asset thông thường dùng IP sẽ truy cập thành công. Tuy nhiên, các hệ thống dùng Single Sign-On (như vCenter) sẽ tự động Redirect trình duyệt sang tên miền FQDN nội bộ (VD: `vcsa.domain.local`). Máy Proxy không có DNS nội bộ nên trình duyệt ảo báo lỗi *Server Not Found*.
- **Cách khắc phục:** Bypass DNS Server bằng cách map FQDN trực tiếp vào file `hosts` cục bộ của máy Proxy.

Mở PowerShell (Run as Admin) trên máy Proxy và nạp bản ghi:

```powershell
$hosts = "C:\Windows\System32\drivers\etc\hosts"
Add-Content -Path $hosts -Value "`n10.x.x.x`tvcsa.domain.local"
Add-Content -Path $hosts -Value "`n192.168.x.x`tmgmt.domain.local"
```

- Quay lại giao diện web, tải lại trang, hệ thống SSO sẽ phân giải thành công và hiển thị giao diện đăng nhập mượt mà.
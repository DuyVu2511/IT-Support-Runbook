# Cẩm Nang Triển Khai & Troubleshooting Zabbix Proxy 7.0 (WSL 1 & VMware NAT)

## 1. Mô hình Môi trường (Topology)
- **Zabbix Server:** Máy ảo Ubuntu 24.04 trên VMware Workstation (IP: `192.168.225.130`), phiên bản Zabbix Server 7.0.
- **Zabbix Proxy:** Chạy trên Ubuntu 22.04 (WSL 1) của máy Windows thật (IP LAN: `192.168.1.137`), cơ sở dữ liệu SQLite3, chế độ **Active Proxy**.
- **Zabbix Agent:** Cài đặt trực tiếp trên máy Windows (`WIN-9S72QH9DL1S`) bằng bộ cài đặt `.msi` chính thức, chịu sự giám sát qua Zabbix Proxy.
- **Port Forwarding:** Gói tin từ mạng ngoài kết nối qua cổng NAT của VMware trên máy Host (`192.168.1.125:10051`) để chuyển tiếp vào Zabbix Server (`192.168.225.130:10051`).

---

## 2. Quy trình Cài đặt & Triển khai (Deployment)

### Phase 1: Cài đặt Zabbix Agent trên Windows Host (qua file MSI)
1. **Tải bộ cài:** Truy cập trang chủ [Zabbix Download](https://www.zabbix.com/download_agents), chọn:
   - OS Distribution: **Windows**
   - Version: **7.0 LTS** (hoặc 6.0 LTS tương thích)
   - Archive: **MSI Installer**
2. **Cấu hình trong trình cài đặt (Setup Wizard):**
   - **Host name:** `WIN-9S72QH9DL1S` (khớp với tên khai báo trên Web sau này).
   - **Zabbix server IP / Proxy IP (`Server` & `ServerActive`):** Điền `127.0.0.1` hoặc `192.168.1.137` (địa chỉ của Zabbix Proxy chạy trên WSL).
   - **Agent listen port:** `10050`.
   - Tích chọn **Enable PSK** nếu cần mã hóa, hoặc để mặc định.
   - Tích chọn **Add agent location to the PATH**.
3. **Mở cổng Firewall cho Agent (Chạy trên PowerShell Admin của Windows):**
   ```powershell
   New-NetFirewallRule -DisplayName "Zabbix Agent 10050" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 10050
   ```
4. **Khởi động dịch vụ:** Dịch vụ `Zabbix Agent` sẽ tự động đăng ký vào Windows Services và chạy ngầm.

---

### Phase 2: Cài đặt Zabbix Proxy (SQLite3) trên Ubuntu WSL 1
*Thực hiện trên Terminal Ubuntu WSL:*
```bash
# 1. Nạp kho repository Zabbix
wget [https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu22.04_all.deb](https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu22.04_all.deb)
sudo dpkg -i zabbix-release_latest_7.0+ubuntu22.04_all.deb
sudo apt update

# 2. Cài đặt Zabbix Proxy SQLite3 và công cụ kiểm tra
sudo apt install -y zabbix-proxy-sqlite3 zabbix-sql-scripts zabbix-get

# 3. Cấu hình file /etc/zabbix/zabbix_proxy.conf
sudo nano /etc/zabbix/zabbix_proxy.conf
```
*Các tham số cốt lõi trong `zabbix_proxy.conf`:*
```ini
# Trỏ về IP Host làm nhiệm vụ Port Forwarding vào Server
Server=192.168.1.125

# Tên định danh Proxy (phải khớp tuyệt đối với tên tạo trên Web UI)
Hostname=vutd_zabbix_proxy

# Đường dẫn file dữ liệu SQLite3
DBName=/var/lib/zabbix/zabbix_proxy.db
```

*Cấp quyền thư mục lưu cơ sở dữ liệu và kích hoạt dịch vụ:*
```bash
sudo mkdir -p /var/lib/zabbix
sudo chown -R zabbix:zabbix /var/lib/zabbix

# Khởi chạy dịch vụ Proxy trên WSL 1
sudo service zabbix-proxy start
```

---

### Phase 3: Khai báo trên Giao diện Zabbix Web
1. **Đăng ký Proxy:**
   - Vào **Administration** $\rightarrow$ **Proxies** $\rightarrow$ chọn **Create proxy**.
   - **Proxy name:** `vutd_zabbix_proxy`
   - **Proxy mode:** `Active`
   - **Proxy address:** *Để trống (tránh bị lọc IP nguồn khi qua NAT)*
   - Nhấn **Add**.
2. **Gán Host cho Proxy quản lý:**
   - Vào **Data collection** $\rightarrow$ **Hosts** $\rightarrow$ chọn máy `WIN-9S72QH9DL1S`.
   - **Monitored by proxy:** Chọn `vutd_zabbix_proxy`.
   - **Interface:** `192.168.1.137:10050`.
   - **Template:** Gắn `Windows by Zabbix agent`.
   - Nhấn **Update**.

---

## 3. Nhật ký Xử lý Sự cố Tầng sâu (Troubleshooting Log)

### Sự cố 1: WSL bị crash khi khởi động (Lỗi Mount File System)
- **Hiện tượng:** Truy cập WSL báo lỗi:
  `An error occurred mounting one of your file systems. Please run 'dmesg' for more details.` Khởi động lại Windows không hết.
- **Nguyên nhân:** Cơ chế DrvFs của WSL 1 tự động mount các phân vùng Windows vào `/mnt/`. Khi gặp ổ đĩa ảo hoặc phân vùng có cờ `dirty bit`, tiến trình mount bị kẹt ở Kernel khiến toàn bộ phiên Linux không thể khởi động.
- **Giải pháp:** Can thiệp Registry Windows để tắt tính năng Automount của bản phân phối này.
- **Thực thi (PowerShell Administrator trên Windows):**
  ```powershell
  # Tìm key Registry của bản Ubuntu2204
  $distroKey = Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss\*' | Where-Object { $_.DistributionName -eq 'Ubuntu2204' }

  # Chuyển Flags từ 7 về 3 (Tắt bit 4 - ENABLE_DRIVE_MOUNTING)
  Set-ItemProperty -Path $distroKey.PSPath -Name "Flags" -Value 3

  # Khởi động lại dịch vụ WSL
  Restart-Service LxssManager

  # Đăng nhập lại bình thường
  wsl -d Ubuntu2204
  ```

### Sự cố 2: Lệch phiên bản Zabbix Server và Proxy (Major Version Mismatch)
- **Hiện tượng:** Log Proxy xuất hiện thông báo:
  `The requested task is disabled. Proxy major version does not match server major version.` Server từ chối nạp cấu hình.
- **Nguyên nhân:** Zabbix Server chạy bản 7.0 nhưng Proxy đang chạy bản 6.0.48. Cơ chế Zabbix 7.0 bắt buộc Server và Proxy phải cùng Major Version.
- **Giải pháp:** Nâng cấp Proxy lên bản 7.0.30.
- **Thực thi trên WSL:**
  ```bash
  sudo apt --only-upgrade install -y zabbix-proxy-sqlite3 zabbix-get
  # Khi được hỏi về file zabbix_proxy.conf -> Chọn 'N' để giữ lại cấu hình cũ.
  sudo service zabbix-proxy restart
  ```
  *Lưu ý:* SQLite không hỗ trợ nâng cấp cấu trúc database cũ. Zabbix Proxy 7.0 sẽ tự động xóa file `.db` bản 6.0 và tạo lại database mới tương thích hoàn toàn.

### Sự cố 3: Lệch định tuyến Asymmetric Routing (Lỗi "empty string received")
- **Hiện tượng:** Proxy báo lỗi liên tục:
  `cannot obtain configuration data from server at "192.168.1.125": empty string received`
  Trên Zabbix Web, Proxy hiển thị màu đỏ (**Offline**).
- **Bắt bệnh (Packet Capture trên Server):**
  Chạy lệnh trên Zabbix Server:
  ```bash
  sudo tcpdump -n -i any port 10051
  ```
  Thấy gói tin bắt tay `SYN` từ Proxy (`192.168.1.137`) đẩy đến được Server, nhưng Server không hề gửi gói phản hồi `ACK` quay lại (`Out`). Lệnh `ip route get 192.168.1.137` cho thấy gói tin trả lời bị đẩy nhầm vào gateway khác không có đường về dải `192.168.1.x`.
- **Giải pháp:** Thêm Static Route cố định vào Netplan của Server, ép toàn bộ dữ liệu trả lời cho dải `192.168.1.0/24` đi qua Gateway NAT của VMware (`192.168.225.2`).
- **Thực thi trên Zabbix Server:**
  Chỉnh sửa file `/etc/netplan/50-cloud-init.yaml`:
  ```yaml
  network:
      version: 2
      ethernets:
          ens33:
              dhcp4: true
              routes:
                  - to: 192.168.1.0/24
                    via: 192.168.225.2
  ```
  Áp dụng cấu hình:
  ```bash
  sudo netplan apply
  # Kiểm tra lại đường đi
  ip route get 192.168.1.137
  # Kết quả chuẩn: 192.168.1.137 via 192.168.225.2 dev ens33 src 192.168.225.130
  ```

---

## 4. Nghiệm thu Kết quả (Verification)
1. **Kiểm tra log trên Proxy (WSL):**
   ```bash
   sudo service zabbix-proxy restart
   sudo tail -n 15 /var/log/zabbix/zabbix_proxy.log
   ```
   *Xác nhận dòng kết nối thành công:*
   `received configuration data from server at "192.168.1.125", loading configuration`
2. **Kiểm tra Zabbix Web UI:**
   - **Administration > Proxies:** Proxy `vutd_zabbix_proxy` đổi sang **Online**, Version hiển thị **7.0.30**, Last seen đếm về 1s.
   - **Monitoring > Hosts:** Máy trạm Windows `WIN-9S72QH9DL1S` chuyển biểu tượng **ZBX** sang màu xanh lá cây, dữ liệu giám sát CPU/RAM được thu thập đầy đủ.
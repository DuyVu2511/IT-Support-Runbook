# CẨM NANG HỆ THỐNG GIÁM SÁT ZABBIX (DEPLOYMENT & TROUBLESHOOTING)

---

## 📌 PHẦN 1: QUY TRÌNH TRIỂN KHAI (DEPLOYMENT)

### Phase 1: Xây dựng Trạm chỉ huy (Zabbix Server trên Ubuntu)

**1. Nạp kho lưu trữ (Repository) của Zabbix:**
```bash
wget [https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb](https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb)
sudo dpkg -i zabbix-release_7.0-2+ubuntu24.04_all.deb
sudo apt update
```

**2. Cài đặt các gói cốt lõi (Server, Web Frontend, Nginx, MySQL):**
```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent mysql-server -y
```

**3. Khởi tạo Cơ sở dữ liệu (Database):**
Truy cập MySQL bằng quyền root:
```bash
sudo mysql -u root
```
Thực thi chuỗi lệnh SQL sau trong dấu nhắc `mysql>`:
```sql
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'MatKhauKho@123';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```

**4. Bơm cấu trúc dữ liệu (Schema) vào Database:**
*(Nhập password `MatKhauKho@123` khi hệ thống hỏi)*
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

**5. Khai báo mật khẩu Database cho Zabbix Server:**
Mở file cấu hình cốt lõi:
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
* Bấm `Ctrl + W` tìm dòng `# DBPassword=`
* Bỏ dấu `#` và sửa thành: `DBPassword=MatKhauKho@123`
* Bấm `Ctrl + O` -> `Enter` để lưu, `Ctrl + X` để thoát.

**6. Cấu hình Port cho Nginx (Web Dashboard):**
Mở file cấu hình Nginx:
```bash
sudo nano /etc/zabbix/nginx.conf
```
* Bỏ dấu `#` ở 2 dòng `listen` và `server_name`.
* Đổi `server_name` thành IP máy Ubuntu (VD: `192.168.50.100`).

**7. Kích hoạt và bật tự chạy cùng OS (Enable on boot):**
```bash
sudo mysql -u root -e "set global log_bin_trust_function_creators = 0;"
sudo systemctl restart zabbix-server zabbix-agent nginx php8.3-fpm mysql
sudo systemctl enable zabbix-server zabbix-agent nginx php8.3-fpm mysql
```

---

### Phase 2: Triển khai Agent lên Windows Server (Active Directory)

1. **Cài đặt:** Tải file `Zabbix Agent 2 (MSI)` từ trang chủ Zabbix chạy trên Windows Server. Nhập IP của Ubuntu Server vào ô `Zabbix Server IP` trong quá trình cài.
2. **Đục tường lửa (PowerShell Admin):** Cho phép Agent gửi dữ liệu về Server qua port 10050:
```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent 10050" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 10050
```

---

### Phase 3: Thu hoạch dữ liệu trên Web Dashboard

1. Truy cập trình duyệt: `http://<IP-Ubuntu>` -> Thực hiện Setup Wizard -> Nhập password DB `MatKhauKho@123`.
2. Đăng nhập mặc định: Username `Admin` / Password `zabbix`.
3. Thêm Host: `Data collection` -> `Hosts` -> `Create Host`:
   * **Host name:** `DUYVU-DC01`
   * **Templates:** `Windows by Zabbix agent`
   * **Interfaces:** Add -> Agent -> Điền IP Windows Server -> Port `10050`.
4. Kiểm tra: Vào `Monitoring` -> `Hosts` -> Xem `Graphs` tài nguyên CPU/RAM/Disk.

---

## 📌 PHẦN 2: CẨM NANG KHẮC PHỤC SỰ CỐ (TROUBLESHOOTING)

### 🚨 Quy trình xử lý khi ZBX báo Đỏ (Mất kết nối / Lỗi dịch vụ)

**1. Đọc nguyên nhân lỗi trên UI:**
* Rà chuột (hover) vào biểu tượng **ZBX** đỏ để đọc chi tiết error message.

**2. Xử lý lỗi Mạng (`Network is unreachable`):**
* Kiểm tra kết nối L3 từ Ubuntu sang Agent:
  ```bash
  ping <IP_Windows_Server>
  ```
* Nếu không ping được: Kiểm tra lại card mạng trên ESXi, cấu hình Netplan hoặc IP động bị đổi.

**3. Xử lý lỗi Dịch vụ Down (`Zabbix server is not running` / `Connection refused`):**
* Kiểm tra trạng thái service:
  ```bash
  sudo systemctl status zabbix-server mysql
  ```
* Khởi động lại dịch vụ theo thứ tự (Database trước, App sau):
  ```bash
  sudo systemctl restart mysql
  sudo systemctl restart zabbix-server zabbix-agent
  ```

**4. Gỡ "Bom nổ chậm" (Dịch vụ chết sau khi Reboot OS):**
* Nguyên nhân: Dịch vụ bị ở trạng thái `disabled`.
* Bắt buộc chạy lệnh sau để kích hoạt khởi động cùng hệ điều hành:
  ```bash
  sudo systemctl enable zabbix-server zabbix-agent mysql
  ```
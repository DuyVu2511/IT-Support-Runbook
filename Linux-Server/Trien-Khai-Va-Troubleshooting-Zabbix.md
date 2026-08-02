# CẨM NANG HỆ THỐNG GIÁM SÁT ZABBIX (DEPLOYMENT & TROUBLESHOOTING)

## 📌 PHẦN 1: QUY TRÌNH TRIỂN KHAI (DEPLOYMENT)

### Phase 1: Xây dựng Trạm chỉ huy (Zabbix Server trên Ubuntu)
1. **Nạp kho lưu trữ (Repository):**
```bash
wget [https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb](https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb)
sudo dpkg -i zabbix-release_7.0-2+ubuntu24.04_all.deb
sudo apt update
```

2. **Cài đặt các gói cốt lõi:**
```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent mysql-server -y
```

3. **Khởi tạo Database:**
```bash
sudo mysql -u root
```
*(Trong dấu nhắc mysql>)*
```sql
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'MatKhauKho@123';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```

4. **Bơm Schema vào Database:**
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

5. **Khai báo Database cho Zabbix Server:**
Sửa file `sudo nano /etc/zabbix/zabbix_server.conf`:
Bỏ dấu `#` và cấu hình: `DBPassword=MatKhauKho@123`

6. **Cấu hình Port cho Nginx (Web Dashboard):**
Sửa file `sudo nano /etc/zabbix/nginx.conf`:
Đổi `server_name` thành IP máy Ubuntu.

7. **Bật tự chạy cùng OS (Enable on boot):**
```bash
sudo mysql -u root -e "set global log_bin_trust_function_creators = 0;"
sudo systemctl restart zabbix-server zabbix-agent nginx php8.3-fpm mysql
sudo systemctl enable zabbix-server zabbix-agent nginx php8.3-fpm mysql
```

### Phase 2: Triển khai Agent lên Windows Server (Active Directory)
* **Cài đặt:** Tải Zabbix Agent 2 (MSI), điền IP Ubuntu Server.
* **Đục tường lửa (PowerShell Admin):**
```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent 10050" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 10050
```

### Phase 3: Thu hoạch dữ liệu trên Web Dashboard
* **Truy cập:** `http://<IP-Ubuntu>` -> Setup Wizard.
* **Tài khoản mặc định:** Admin / zabbix.
* **Thêm Host:** `Data collection` -> `Hosts` -> Gắn Template `Windows by Zabbix agent` và trỏ Interface về IP Windows (Port 10050).

---

## 📌 PHẦN 2: CẨM NANG KHẮC PHỤC SỰ CỐ (TROUBLESHOOTING)

### 🚨 Quy trình cơ bản khi ZBX báo Đỏ
1. **Đọc log UI:** Hover chuột vào biểu tượng ZBX đỏ để đọc chi tiết error.
2. **Xử lý lỗi Mạng:** Ping kiểm tra kết nối L3 từ Ubuntu sang Agent (`ping <IP_Windows>`).
3. **Xử lý Dịch vụ Down:**
```bash
sudo systemctl status zabbix-server mysql
# Restart Database trước, App sau
sudo systemctl restart mysql
sudo systemctl restart zabbix-server zabbix-agent
```
4. **Gỡ "Bom nổ chậm" (Dịch vụ chết sau khi Reboot):**
```bash
sudo systemctl enable zabbix-server zabbix-agent mysql
```

### 🔥 Kỹ thuật nâng cao: Cứu hộ tầng sâu (Advanced Troubleshooting)
**1. Bắt bệnh khi `systemctl status` báo `Active: failed`**
* Khi restart không lên, KHÔNG làm bừa. Phải đọc "di chúc" log hệ thống:
```bash
sudo journalctl -xe
# Hoặc soi log tĩnh của riêng Zabbix:
cat /var/log/zabbix/zabbix_server.log
```

**2. Cấp cứu khi CPU Server bị vắt kiệt (Alert 95% CPU)**
* **Nguyên tắc Data First:** Tuyệt đối KHÔNG chạy lệnh Backup hay Restart bừa bãi khi CPU đang quá tải tránh làm treo cứng Server (Deadlock).
* **Thao tác "Phẫu thuật":**
```bash
# Mở bảng xếp hạng tiến trình real-time
htop 
# (Hoặc dùng lệnh top)

# Tìm số PID đang cắn CPU cao nhất và "tiêu diệt" nó để cứu server:
sudo kill -9 <PID>
```
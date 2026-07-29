

---

### 1. Tạo kịch bản sao lưu (Bash Script)
* Thư mục chứa backup: `/opt/db_backups`
* Kịch bản `/opt/backup_zabbix.sh`:
```bash
#!/bin/bash

# 1. Khai báo biến
BACKUP_DIR="/opt/db_backups"
DATE=$(date +"%Y%m%d_%H%M%S")
DB_USER="zabbix"
DB_PASS="MatKhauKho@123"
DB_NAME="zabbix"

# 2. Xuất dữ liệu Database
mysqldump -u$DB_USER -p$DB_PASS $DB_NAME > $BACKUP_DIR/${DB_NAME}_backup_${DATE}.sql

# 3. Tự động xóa file backup cũ hơn 7 ngày (Tránh đầy disk)
find $BACKUP_DIR -type f -name "*.sql" -mtime +7 -exec rm {} \;
```

* Cấp quyền thực thi cho file script:
```bash
sudo chmod +x /opt/backup_zabbix.sh
```

---

### 2. Thiết lập Lập lịch tự động (Cronjob)
* Mở trình quản lý crontab: `sudo crontab -e`
* Cấu hình chạy ngầm lúc **2h00 sáng hằng ngày**:
```plaintext
0 2 * * * /opt/backup_zabbix.sh
```
* Kiểm tra danh sách tiến trình tự động: `sudo crontab -l`
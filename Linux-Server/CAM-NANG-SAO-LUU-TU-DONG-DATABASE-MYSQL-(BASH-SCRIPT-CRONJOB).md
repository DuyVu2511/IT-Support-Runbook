## 1. Tạo kịch bản sao lưu (Bash Script)
Thư mục chứa backup: `/opt/db_backups`

Tạo và chỉnh sửa kịch bản `/opt/backup_zabbix.sh`:
```bash
#!/bin/bash

# 1. Khai báo biến
BACKUP_DIR="/opt/db_backups"
DATE=$(date +"%Y%m%d_%H%M%S")
DB_USER="zabbix"
DB_PASS="MatKhauKho@123"
DB_NAME="zabbix"

# 2. Xuất dữ liệu Database (Sử dụng đường dẫn tuyệt đối để tránh lỗi Cronjob)
/usr/bin/mysqldump -u$DB_USER -p$DB_PASS $DB_NAME > $BACKUP_DIR/${DB_NAME}_backup_${DATE}.sql

# 3. Tự động xóa file backup cũ hơn 7 ngày (Tránh đầy disk)
/usr/bin/find $BACKUP_DIR -type f -name "*.sql" -mtime +7 -exec rm {} \;
```

Cấp quyền thực thi cho file script:
```bash
sudo chmod +x /opt/backup_zabbix.sh
```

## 2. Thiết lập Lập lịch tự động (Cronjob)
Mở trình quản lý crontab:
```bash
sudo crontab -e
```

Cấu hình chạy ngầm lúc 2h00 sáng hằng ngày:
```bash
0 2 * * * /opt/backup_zabbix.sh
```

Kiểm tra danh sách tiến trình tự động:
```bash
sudo crontab -l
```

## 🚨 Troubleshooting (Xử lý sự cố thực chiến)
### ⚠️ Bẫy kinh điển: Cronjob chạy ngầm bị rớt (Environment Path Issue)
* **Tình huống:** Chạy script thủ công bằng `./backup_zabbix.sh` thì sinh ra file backup, nhưng để Cronjob tự chạy thì file trống trơn.
* **Nguyên nhân:** Cronjob chạy ngầm không tự load biến môi trường `$PATH`. Nó không biết lệnh `mysqldump` hoặc `find` nằm ở đâu.
* **Giải pháp chuẩn:** Luôn dùng **Đường dẫn tuyệt đối (Absolute Path)** trong script (Ví dụ: `/usr/bin/mysqldump` thay vì `mysqldump`). Có thể dùng lệnh `which mysqldump` trên Terminal để lấy chính xác đường dẫn này.
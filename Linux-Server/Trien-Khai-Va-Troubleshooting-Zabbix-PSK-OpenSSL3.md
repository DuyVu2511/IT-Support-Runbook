# Runbook: Triển Khai Zabbix Proxy Mã Hóa PSK & Khắc Phục Lỗi OpenSSL 3.0

## 1. Bối cảnh & Mục tiêu (Overview)
- **Bối cảnh:** Mặc định từ Ubuntu 22.04 (OpenSSL 3.0+), chính sách TLS 1.3 rất khắt khe. Cấu hình Zabbix Proxy PSK dễ gặp lỗi `handshake failure` do xung đột Cipher hoặc file key dính ký tự ẩn (newline).
- **Mục tiêu:** Xây dựng Executable Runbook chuẩn để cấu hình PSK thành công 100%, fix triệt để lỗi `error:0A000076`.

---

## 2. Quy trình Thực thi (SOP)

```bash
#!/bin/bash
# ==============================================================================
# GIAI ĐOẠN 1: THAO TÁC TRÊN ZABBIX PROXY (Terminal)
# ==============================================================================

# BƯỚC 1: TẠO FILE KEY PSK "SẠCH" (Dài đúng 64 bytes)
# 1. Sinh chuỗi Hex 64 ký tự và loại bỏ dấu xuống dòng (tr -d '\n')
openssl rand -hex 32 | tr -d '\n' | sudo tee /etc/zabbix/zabbix_proxy.psk

# 2. Phân quyền bắt buộc cho user Zabbix
sudo chown zabbix:zabbix /etc/zabbix/zabbix_proxy.psk
sudo chmod 600 /etc/zabbix/zabbix_proxy.psk

# 3. Kiểm tra dung lượng file (BẮT BUỘC in ra số 64, nếu ra 65 là lỗi)
wc -c /etc/zabbix/zabbix_proxy.psk

# 4. In Key ra màn hình để copy (Dùng dán lên Web GUI ở Bước 3)
cat /etc/zabbix/zabbix_proxy.psk


# BƯỚC 2: BƠM CẤU HÌNH & "VACCINE" VÀO ZABBIX PROXY
# (Thay <NHAP_TEN_IDENTITY> bằng tên thực tế, VD: Niigata-Prod-2026)
sudo tee -a /etc/zabbix/zabbix_proxy.conf << 'EOF'

# --- CẤU HÌNH PSK MẶC ĐỊNH ---
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=<NHAP_TEN_IDENTITY>
TLSPSKFile=/etc/zabbix/zabbix_proxy.psk

# --- VACCINE FIX LỖI OPENSSL 3.0 ---
TLSCipherPSK13=TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
EOF

# 5. Khởi động lại Proxy
sudo systemctl restart zabbix-proxy


# ==============================================================================
# GIAI ĐOẠN 2: THAO TÁC TRÊN WEB GUI VÀ ZABBIX SERVER
# ==============================================================================

# BƯỚC 3: KHAI BÁO TRÊN WEB GUI (Làm trên Trình duyệt)
# 1. Đăng nhập Web -> Administration -> Proxies -> Click tên Proxy.
# 2. Tab Encryption:
#    - Connections to/from proxy: Chọn PSK
#    - PSK identity: Nhập giống hệt <NHAP_TEN_IDENTITY> ở Bước 2.
#    - PSK: Dán chuỗi 64 ký tự đã copy ở Bước 1 (Không khoảng trắng).
# 3. Bấm Update.


# BƯỚC 4: ÁP DỤNG CẤU HÌNH VÀ KIỂM TRA
# 1. Tại Terminal ZABBIX SERVER: Ép nạp cấu hình mới (Xóa Cache OpenSSL)
sudo systemctl restart zabbix-server

# 2. Tại Terminal ZABBIX PROXY: Kiểm tra Log
sudo tail -f /var/log/zabbix/zabbix_proxy.log

# DẤU HIỆU THÀNH CÔNG: Hiện dòng `received configuration data from server at...`
```

---

## 3. Khắc Phục Sự Cố (Troubleshooting)

| Dấu hiệu (Log Error)                                 | Nguyên nhân (Root Cause)                                                | Cách khắc phục (Resolution)                                                                         |
| :--------------------------------------------------- | :---------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------- |
| `alert 40: TLS read fatal alert "handshake failure"` | Sai tên Identity hoặc Web GUI chưa được Update. Cache Server bị kẹt.    | Kiểm tra lại từng chữ cái Identity. Đảm bảo đã ấn Update trên Web và **Restart Zabbix Server**.     |
| `alert 40: ...` (Kèm theo lỗi file Size)             | File PSK bị dư ký tự ẩn (65 bytes thay vì 64 bytes).                    | Chạy lại lệnh sinh key với `tr -d '\n'` như Bước 1.                                                 |
| `error:0A000076: no suitable signature algorithm`    | OpenSSL 3.0 từ chối thuật toán do HĐH ép dùng Cipher không tương thích. | Bắt buộc phải thêm dòng `TLSCipherPSK13` chứa thuật toán SHA256 vào file config Proxy (Xem Bước 2). |
# Runbook: Triển Khai Zabbix Proxy Mã Hóa PSK & Khắc Phục Lỗi OpenSSL 3.0

## 1. Bối cảnh & Mục tiêu (Overview)

> [!info] Bối cảnh
> Mặc định từ Ubuntu 22.04 (OpenSSL 3.0+), chính sách TLS 1.3 rất khắt khe. Cấu hình Zabbix Proxy PSK thường gặp lỗi `handshake failure` hoặc `no suitable signature algorithm` do xung đột Cipher, dính ký tự ẩn (newline) trong file key, hoặc do Firewall/IPS chặn luồng thuần PSK.

> [!goal] Mục tiêu
> Xây dựng Executable Runbook chuẩn để cấu hình PSK thành công 100%, sử dụng "Dummy Certificate Workaround" để lách qua cơ chế kiểm tra chữ ký khắt khe của OpenSSL 3.0.2.

## 2. Quy trình Thực thi (SOP)

### Giai đoạn 1: Thao tác trên Zabbix Proxy (Terminal)

#### Bước 1: Tạo file key PSK "sạch" (dài đúng 64 bytes)

```bash
# 1. Sinh chuỗi Hex 64 ký tự và loại bỏ dấu xuống dòng (tr -d '\n')
openssl rand -hex 32 | tr -d '\n' | sudo tee /etc/zabbix/zabbix_proxy.psk

# 2. Phân quyền bắt buộc cho user Zabbix
sudo chown zabbix:zabbix /etc/zabbix/zabbix_proxy.psk
sudo chmod 600 /etc/zabbix/zabbix_proxy.psk

# 3. Kiểm tra dung lượng file (BẮT BUỘC in ra số 64, nếu ra 65 là lỗi)
wc -c /etc/zabbix/zabbix_proxy.psk

# 4. In Key ra màn hình để copy (Dùng dán lên Web GUI ở Bước 4)
cat /etc/zabbix/zabbix_proxy.psk
```

#### Bước 2: Tạo chứng chỉ "chim mồi" (Dummy Certificate - fix OpenSSL 3.0)

Tạo chứng chỉ tự ký thời hạn 10 năm để lọt qua bước Handshake.

```bash
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /etc/zabbix/dummy.key -out /etc/zabbix/dummy.crt -subj "/CN=DummyProxy"
sudo chown zabbix:zabbix /etc/zabbix/dummy.*
```

#### Bước 3: Bơm cấu hình & "vaccine" vào Zabbix Proxy

Thay `<NHAP_TEN_IDENTITY>` bằng tên thực tế, VD: `Niigata-Prod-2026`.

```bash
sudo tee -a /etc/zabbix/zabbix_proxy.conf << 'EOF'

# --- CẤU HÌNH PSK MẶC ĐỊNH ---
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=<NHAP_TEN_IDENTITY>
TLSPSKFile=/etc/zabbix/zabbix_proxy.psk

# --- VACCINE FIX LỖI OPENSSL 3.0 (Cipher & Dummy Cert) ---
TLSCipherPSK13=TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
TLSCAFile=/etc/zabbix/dummy.crt
TLSCertFile=/etc/zabbix/dummy.crt
TLSKeyFile=/etc/zabbix/dummy.key
EOF
```

Khởi động lại Proxy:

```bash
sudo systemctl restart zabbix-proxy
```

### Giai đoạn 2: Thao tác trên Zabbix Server & Web GUI

#### Bước 4: Khai báo trên Web GUI (làm trên trình duyệt)

1. Đăng nhập Web → **Administration** → **Proxies** → Click tên Proxy.
2. Tab **Encryption**:
   - **Connections to/from proxy**: Chọn `PSK`
   - **PSK identity**: Nhập giống hệt `<NHAP_TEN_IDENTITY>` ở Bước 3.
   - **PSK**: Dán chuỗi 64 ký tự đã copy ở Bước 1 (Không khoảng trắng).
3. Bấm **Update**.

#### Bước 5: Tạo chứng chỉ Dummy và áp dụng trên Zabbix Server

1. Tại Terminal **ZABBIX SERVER**: Tạo chứng chỉ chim mồi tương tự Proxy

```bash
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /etc/zabbix/dummy.key -out /etc/zabbix/dummy.crt -subj "/CN=DummyServer"
sudo chown zabbix:zabbix /etc/zabbix/dummy.*
```

2. Thêm cấu hình vào file `zabbix_server.conf`

```bash
sudo tee -a /etc/zabbix/zabbix_server.conf << 'EOF'
TLSCipherPSK13=TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
TLSCAFile=/etc/zabbix/dummy.crt
TLSCertFile=/etc/zabbix/dummy.crt
TLSKeyFile=/etc/zabbix/dummy.key
EOF
```

3. Ép nạp cấu hình mới (Xóa Cache OpenSSL)

```bash
sudo systemctl restart zabbix-server
sudo zabbix_server -R config_cache_reload
```

#### Bước 6: Kiểm tra log trên Zabbix Proxy

```bash
sudo tail -f /var/log/zabbix/zabbix_proxy.log
```

> [!success] Dấu hiệu thành công
> Hiện dòng `received configuration data from server at...`

## 3. Khắc Phục Sự Cố (Troubleshooting)

| Dấu hiệu (Log Error) | Nguyên nhân (Root Cause) | Cách khắc phục (Resolution) |
| --- | --- | --- |
| `alert 40: TLS read fatal alert "handshake failure"` | Sai tên Identity hoặc Web GUI chưa được Update. Cache Server bị kẹt. | Kiểm tra lại từng chữ cái Identity. Đảm bảo đã ấn Update trên Web và Restart Zabbix Server. |
| `alert 40: ...` (Kèm theo lỗi file Size) | File PSK bị dư ký tự ẩn (65 bytes thay vì 64 bytes). | Chạy lại lệnh sinh key với `tr -d '\n'` như Bước 1. |
| `error:0A000076: no suitable signature algorithm` hoặc `alert certificate required` | Bug Handshake của OpenSSL 3.0 trên Ubuntu 22.04 từ chối kết nối nếu không tìm thấy thuật toán chữ ký ban đầu (hoặc do Firewall IPS chặn PSK thuần). | Bắt buộc phải tạo và khai báo Dummy Certificate (`dummy.crt`/`dummy.key`) trên cả Server và Proxy (Xem Bước 2 & 5) để đánh lừa bộ đệm Handshake. |
# Runbook: Quy trình Triển khai VM Ubuntu (LVM & Tường lửa) Chuẩn Doanh nghiệp

## 1. Bối cảnh & Tiêu chuẩn (Overview)

> [!info] Mục tiêu
> Tài liệu này hướng dẫn cách tạo hàng loạt máy chủ ảo (VM) Ubuntu Server an toàn. Tránh tình trạng tràn ổ cứng chết OS (bằng LVM) và thiết lập Zero-Trust Firewall (UFW) nội bộ theo yêu cầu bảo mật của dự án.

### Tiêu chuẩn Tài nguyên (Mặc định)

- **OS:** Ubuntu Server 22.04 LTS / 24.04 LTS
- **Hardware:** Tùy chọn (Thường là 8 vCPU / 12GB RAM)
- **Disk Controller:** `VMware Paravirtual` (Tối ưu IOPS cho Linux)
- **Network Adapter:** `VMXNET3`

---

## 2. Quy hoạch Lưu trữ (LVM Partitioning)

> [!warning] Cảnh báo rủi ro
> Tuyệt đối KHÔNG gộp chung toàn bộ dung lượng vào ổ Root (`/`). Phải chia ổ `/data` riêng biệt để ứng dụng/database không làm tràn hệ điều hành.

Với ổ cứng **200GB**, quy hoạch Custom Storage Layout như sau:

1. `1GB` -> `ext4` -> mount tại `/boot`
2. `4GB` -> `swap`
3. `40GB` -> `ext4` -> mount tại `/` (Ổ hệ điều hành Root)
4. `100GB` -> `ext4` -> mount tại `/data` (Nơi Dev chứa Source code & DB)
5. **Dự phòng (Free Space):** ~`55GB` để trống (Unallocated). Dùng để extend nóng vào `/data` sau này nếu cần.

---

## 3. Tạo Golden Image & Sysprep (Tẩy não OS)

Sau khi cài đặt xong OS theo cấu trúc LVM trên (và chọn cài OpenSSH), KHÔNG clone ngay. Phải thực hiện xóa định danh mạng để các máy clone ra không bị đá IP nhau.

Đăng nhập vào máy ảo gốc (Server01) và chạy Script sau:

```bash
# Cập nhật hệ thống & Cài VMware Tools
sudo apt update && sudo apt install open-vm-tools net-tools -y

# Tẩy não (Sysprep) - Xóa Machine ID
sudo cloud-init clean --logs
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

# Xóa lịch sử bash và tắt máy để làm Template/Clone
cat /dev/null > ~/.bash_history && history -c
sudo shutdown -h now
```

---

## 4. Rẽ nhánh (Clone) & Cấu hình Mạng (Netplan)

Sau khi máy gốc tắt, tiến hành Clone trên vCenter ra các node tiếp theo (VD: Server02). Bật các máy lên và thực hiện đổi định danh.

### 4.1. Đổi Hostname

```bash
sudo hostnamectl set-hostname <tên_máy_mới>
sudo reboot
```

### 4.2. Cấu hình IP Tĩnh (Netplan)

Mở file cấu hình: `sudo nano /etc/netplan/00-installer-config.yaml`

> [!danger] Chú ý cú pháp YAML
> Bắt buộc dùng phím SPACE (khoảng trắng) để thụt lề. Tuyệt đối KHÔNG dùng phím TAB.

```yaml
network:
  version: 2
  ethernets:
    ens192:
      dhcp4: no
      addresses:
        - 10.100.10.50/24      # Thay IP tại đây
      routes:
        - to: default
          via: 10.100.10.1     # Thay Gateway tại đây
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

Lưu lại và áp dụng:

```bash
sudo netplan apply
```

---

## 5. OS Hardening & Tường lửa (UFW)

> [!info] Nguyên tắc
> Áp dụng Default Deny Incoming. Chỉ mở port cho đúng IP đích danh cần gọi vào (Least Privilege).

### Mẫu 1: Máy chủ Web (Public DMZ)

Mở cửa cho Internet truy cập HTTPS, nhưng bảo vệ port SSH.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22/tcp            # Hoặc giới hạn: sudo ufw allow from <IP_Jump_Server> to any port 22
sudo ufw allow 443/tcp           # HTTPS
sudo ufw enable
```

### Mẫu 2: Máy chủ Database/Backend (Private / Server Farm)

Chặn toàn bộ Internet. Chỉ cho phép duy nhất IP của máy Web chọc vào các port của DB/App.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp

# Chỉ cho phép Máy Web (103.16.1.190) truy cập port dịch vụ
sudo ufw allow from 103.16.1.190 to any port 3306 proto tcp      # MySQL
sudo ufw allow from 103.16.1.190 to any port 9200,9300 proto tcp # Elastic
# Thêm các port khác tương tự...

sudo ufw enable
sudo ufw status numbered
```
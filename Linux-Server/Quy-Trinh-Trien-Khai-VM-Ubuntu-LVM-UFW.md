# Runbook: Triển khai Ubuntu VM, LVM, UFW & Troubleshooting Chuẩn Doanh nghiệp

## 1. Bối cảnh & Tiêu chuẩn (Overview)

> [!info] Mục tiêu
> Tài liệu này hướng dẫn cách khởi tạo, cấu hình IP/Firewall và nghiệm thu máy chủ ảo (VM) Ubuntu Server. Bao gồm cả các bước kiểm tra định tuyến Layer 2/Layer 3 khi phối hợp với đội Network.

### Tiêu chuẩn Tài nguyên

- **OS:** Ubuntu Server 22.04 LTS / 24.04 LTS
- **Hardware:** Theo yêu cầu dự án (Thường là 8 vCPU / 12GB RAM)
- **Disk Controller:** `VMware Paravirtual` (Bắt buộc để tối ưu IOPS Linux)
- **Network Adapter:** `VMXNET3`
- **Hostname:** Tên máy ảo bên trong OS phải **trùng khớp 100%** với tên VM trên vCenter.

---

## 2. Quy hoạch Lưu trữ (LVM Partitioning)

> [!warning] Best Practice vs. Thực tế Dự án
> - **Tiêu chuẩn (Khuyên dùng):** Cắt riêng ổ `/data` để chứa Source Code/DB nhằm bảo vệ OS Root không bị tràn.
> - **Thực tế (Theo chỉ đạo dự án AI):** Gom toàn bộ dung lượng trống vào ổ Root (`/`) để tiện quản lý.

**Cấu hình LVM dồn toàn bộ vào Root (Dung lượng 200GB):**

1. Màn hình Storage -> Chọn `Custom storage layout`.
2. Tạo Partition: `1GB` -> `ext4` -> Mount tại `/boot`.
3. Tạo Partition: `4GB` -> Format `swap` (RAM ảo).
4. Khởi tạo LVM (`Create volume group`) -> Chọn toàn bộ ~195GB còn lại.
5. Tạo Logical Volume trong VG -> Tên `root-lv`, **Size để trống** (Tự ăn full), Mount tại `/`.

---

## 3. Sysprep (Tẩy não OS) & Tạo Template

Để Clone VM không bị trùng MAC/IP/Machine-ID, **bắt buộc** chạy script sau trên máy gốc trước khi Clone:

```bash
# Cập nhật hệ thống & Cài VMware Tools
sudo apt update && sudo apt install open-vm-tools -y

# Tẩy não (Sysprep) - Xóa Machine ID và cấu hình Cloud-init
sudo cloud-init clean --logs
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

# Xóa lịch sử bash và tắt nguồn để Clone
cat /dev/null > ~/.bash_history && history -c
sudo shutdown -h now
```

---

## 4. Cấu hình Mạng (Netplan) & Đổi Hostname

Sau khi Clone, bật VM mới và thực hiện đổi định danh.

### 4.1. Đổi Hostname

```bash
sudo hostnamectl set-hostname <tên_máy_mới>
sudo reboot
```

### 4.2. Cấu hình IP Tĩnh (Netplan)

> [!tip] Mẹo mở nhanh file Netplan
> Trong một số trường hợp tên file cấu hình bị đổi, dùng lệnh: `sudo nano /etc/netplan/*.yaml`

```yaml
network:
  version: 2
  ethernets:
    ens192:
      dhcp4: no
      addresses:
        - 103.16.1.191/24      # Đổi IP tại đây
      routes:
        - to: default
          via: 103.16.1.1      # Đổi Gateway tại đây
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

Lưu lại (Ctrl+O) và Apply: `sudo netplan apply`

---

## 5. Bắt bệnh Mạng (Network Troubleshooting)

Nếu cấu hình Netplan chuẩn nhưng ping Gateway báo `Destination Host Unreachable`, thực hiện kiểm tra Layer 2 (ARP/MAC).

> [!danger] Lệnh kiểm tra
> ```bash
> ip neigh
> ```
> - Nếu báo `REACHABLE` / `STALE`: Mạng Layer 2 thông, lỗi do Firewall chặn ping.
> - Nếu báo `FAILED` / `INCOMPLETE`: Máy ảo không tìm thấy MAC của Gateway. Lỗi 100% do cắm nhầm Port Group (VLAN) trên vCenter hoặc đứt cáp ảo. Yêu cầu đội Network cung cấp lại tên VLAN chuẩn.

---

## 6. OS Hardening & Tường lửa Local (UFW)

> [!info] Phối hợp Defense in Depth
> Tường lửa vòng ngoài do đội Network cấp. Tường lửa OS (fw local) do Sysadmin cấu hình theo cơ chế Default Deny Incoming.

### Máy chủ Web (VD: IP 103.16.1.191)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### Máy chủ DB/Backend (VD: IP 10.100.10.191)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp

# Chỉ cho phép duy nhất IP máy Web truy cập Port Dịch vụ
sudo ufw allow from 103.16.1.191 to any port 1433 proto tcp
sudo ufw allow from 103.16.1.191 to any port 3306 proto tcp
sudo ufw allow from 103.16.1.191 to any port 9200,9300 proto tcp
sudo ufw enable
```

---

## 7. Nghiệm thu Hệ thống (Testing & Handover)

Trải qua 3 bài test trước khi bàn giao cho khách hàng:

1. **Test Internet:** Đứng từ VM chạy `sudo apt update` thành công.
2. **Test Luồng Nội Bộ:** Đứng từ máy Web check port máy DB: `nc -zv 10.100.10.191 3306` (Báo `succeeded!` là pass).
3. **Test VPN:** Bật VPN Client, SSH bằng phần mềm cá nhân vào 2 máy thành công.
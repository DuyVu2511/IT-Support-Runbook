# Sổ Tay Quy Hoạch Mạng Doanh Nghiệp: Subnetting & VLAN

## I. CHIA MẠNG SUBNETTING (IPv4 & CIDR)

### 1. Nguyên tắc cốt lõi & Công thức vàng
- Địa chỉ IPv4 gồm 32 bit. Ký hiệu Prefix (`/24`, `/26`...) cho biết số bit bị khóa làm Network ID.
- Số bit dành cho Host (máy tính): $N = 32 - \text{Prefix}$
- **Công thức tính số IP sử dụng được:**
  $$\text{Số IP khả dụng} = 2^N - 2$$
- **⚠️ Tại sao phải trừ đi 2?**
  - **IP đầu tiên (Network Address):** Dùng làm địa chỉ đại diện cho cả mạng.
  - **IP cuối cùng (Broadcast Address):** Dùng để phát thanh tín hiệu tới tất cả máy trong mạng.

---

### 2. Bảng Tra Cứu Subnet Thực Chiến (Cheat Sheet)

| Prefix | Subnet Mask | Số IP dùng được | Mục đích sử dụng thực tế (Use Case) |
| :--- | :--- | :---: | :--- |
| **/24** | `255.255.255.0` | **254** IP | Mạng tiêu chuẩn cho toàn công ty. |
| **/25** | `255.255.258.128` | **126** IP | Chia đôi dải /24 (Chia theo tầng/khu vực lớn). |
| **/26** | `255.255.255.192` | **62** IP | Chia tư dải /24 (Phòng Marketing, Khối Văn phòng). |
| **/27** | `255.255.255.224` | **30** IP | Phòng họp, dải IP cho phòng Lab nhỏ. |
| **/28** | `255.255.255.240` | **14** IP | Phòng 10-12 máy, dải IP cho Camera/Máy in. |
| **/30** | `255.255.255.252` | **2** IP | **Link nối giữa 2 Router/Switch.** Vừa đủ 2 đầu, chống lãng phí IP. |

---

## II. MẠNG LAN ẢO (VLAN - VIRTUAL LAN)

### 1. Khái niệm & Mục đích
- **Khái niệm:** Dùng phần mềm chia 1 Switch vật lý thành nhiều Switch ảo độc lập ở Layer 2.
- **Lý do công ty phải dùng VLAN:**
  1. **Bảo mật:** Cô lập lưu lượng giữa các phòng ban (Khách/Guest không thể nhìn thấy máy Kế toán hay Server IT).
  2. **Chống nghẽn mạng:** Bóp nhỏ vùng quảng bá (Broadcast Domain), ngăn chặn sự cố bão Broadcast gây treo Switch.

---

### 2. Phân biệt Cổng Access (Access Port) và Cổng Trunk (Trunk Port)

> **Quy tắc vàng:** 
> - **Access** = Cắm thiết bị cuối (End-Device) thuộc **1 phòng ban/1 VLAN**.
> - **Trunk** = Đường ống kết nối hạ tầng chuyên chở **Nhiều VLAN cùng lúc**.

| Tiêu chí                  | Cổng Access (Access Port)                 | Cổng Trunk (Trunk Port)                                                                  |
| :------------------------ | :---------------------------------------- | :--------------------------------------------------------------------------------------- |
| **Số VLAN chạy qua**      | Duy nhất **1** VLAN.                      | **Nhiều** VLAN cùng lúc.                                                                 |
| **Loại thiết bị cắm vào** | PC, Laptop, Máy in, Camera IP.            | Switch nối Switch, Switch nối Router/Firewall, Server VMware ESXi, Wifi AP Doanh nghiệp. |
| **Cơ chế thẻ (Tagging)**  | Tháo nhãn VLAN (Untagged) để PC đọc được. | Dán nhãn VLAN (Tagged - Chuẩn `802.1Q`) để không bị lẫn lộn giữa các mạng.               |

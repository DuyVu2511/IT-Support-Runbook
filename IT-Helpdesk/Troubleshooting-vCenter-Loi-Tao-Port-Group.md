# [TROUBLESHOOTING] Lỗi "A specified parameter was not correct" khi tạo Port Group trên vCenter

## 1. Triệu chứng
Trong quá trình triển khai Lab trên VMware vCenter, khi thực hiện tạo thêm một **Port Group** mới (có gắn VLAN ID) trên vSwitch0 để cách ly mạng cho máy ảo, hệ thống báo lỗi đỏ: `A specified parameter was not correct` và không cho phép tạo.

## 2. Phân tích nguyên nhân (Root Cause)
- Môi trường vCenter của công ty (Production) thường có các chính sách bảo mật (Security Policy) khắt khe trên Switch ảo (vSwitch) hoặc bị khóa từ Switch vật lý bên dưới.
- Hệ thống từ chối việc tự ý tạo VLAN/Trunking Port vô tội vạ nhằm mục đích:
  - Chống bão mạng (Broadcast Storm).
  - Ngăn chặn **Rogue DHCP** (máy chủ DHCP tự dựng đi cấp phát IP nhầm cho các máy tính của phòng ban khác làm sập mạng lưới công ty).

## 3. Cách xử lý (Best Practice)
Tuyệt đối không cố tìm cách bypass (vượt quyền) hoặc cắm tạm máy ảo vào các Port Group đang chạy thực tế của công ty.
- **Bước 1:** Chuẩn bị sẵn thông tin dải IP Lab (Ví dụ: cần dải 10.10.10.x, cấp DHCP nội bộ).
- **Bước 2:** Báo cáo với Network Admin hoặc Mentor quản lý hạ tầng vCenter.
- **Bước 3:** Nhờ cấp phát một Port Group thuộc **VLAN riêng biệt (Host-only hoặc Isolated)** biệt lập hoàn toàn với mạng công ty.
- **Bước 4:** Gán Network Adapter của máy ảo vào đúng Port Group vừa được cấp và tiếp tục bài Lab.
# Runbook: Troubleshooting Veeam Backup - Lỗi chuỗi liên hoàn (Chain Failure)

## 1. Bối cảnh & Triệu chứng (Overview)
- **Hệ thống:** Veeam Backup & Replication, VMware vCenter, Synology NAS (Repository).
- **Triệu chứng:** Hàng loạt Backup Jobs báo `Failed` đồng loạt trong đêm.
- **Mức độ:** High (Ảnh hưởng toàn bộ tiến trình an toàn dữ liệu doanh nghiệp).

## 2. Phân tích Nguyên nhân gốc rễ (Root Cause Analysis - RCA)
Sự cố này là một chuỗi 4 lỗi liên hoàn (Domino effect) từ nhiều tác động khác nhau trên hệ thống. Việc đọc kỹ Log của Veeam là chìa khóa để bóc tách từng lớp lỗi.

### Lỗi 1: Mất kết nối giao thức chia sẻ file (SMB)
- **Log báo:** `Could not perform threshold check... Error: The network path was not found.`
- **Nguyên nhân:** Dịch vụ SMB trên NAS (Repository IP: `10.xxx.xxx.102`) bị tắt. Veeam không tìm thấy đường dẫn mạng.

### Lỗi 2: Sai thông tin xác thực NAS (Credentials)
- **Log báo:** `Task failed. Error: Cannot complete login due to an incorrect user name or password.`
- **Nguyên nhân:** Sau khi bật SMB, Veeam kết nối được nhưng bị từ chối quyền ghi. Pass của User NAS (`[Masked_User_Mailbak]`) đã bị đổi trước đó nhưng chưa được update vào Veeam.

### Lỗi 3: vCenter Authentication Expired
- **Log báo:** Veeam không thể connect vào vCenter/ESXi host để lấy snapshot máy ảo.
- **Nguyên nhân:** Mật khẩu của tài khoản `administrator@vsphere.local` bị hết hạn hoặc đã thay đổi theo Security Policy.

### Lỗi 4: Lỗi mất dấu ID Máy ảo (MoRef ID changed)
- **Log báo:** `Virtual Machine [Masked_VM_Name_IP_186] is unavailable and will be skipped from processing.`
- **Nguyên nhân:** Máy ảo vẫn đang bật (Powered On) trên vCenter. Tuy nhiên, do thao tác Remove/Add lại inventory hoặc vMotion/Restore trước đó, mã ẩn **MoRef ID** của VM đã bị thay đổi. Veeam tìm theo ID cũ nên báo không thấy.

---

## 3. Các bước xử lý chuẩn (Resolution Steps)

Cần xử lý tuần tự theo đúng mô hình OSI (Từ dưới lên):

1. **Khôi phục đường truyền (Layer 3/SMB):**
   - Đăng nhập giao diện Web quản trị NAS.
   - Truy cập `Control Panel` > `File Services` > `SMB`.
   - Tick chọn **Enable SMB service** -> Apply.

2. **Cập nhật Credentials cho Backup Repository:**
   - Tại Veeam: `Backup Infrastructure` > `Backup Repositories`.
   - Chuột phải NAS Repository > `Properties` > Tab `Share/Credentials`.
   - Edit và cập nhật Password mới nhất của User NAS.

3. **Cập nhật Credentials cho vCenter:**
   - Tại Veeam: `Backup Infrastructure` > `Managed Servers` > `VMware vSphere` > `vCenter Servers`.
   - Chuột phải vCenter > `Properties` > Tab `Credentials`.
   - Edit và cập nhật Password quản trị vCenter.

4. **Map lại ID Máy ảo (Fix lỗi Skipped/Unavailable):**
   - Tại Veeam: `Home` > `Jobs` > Chuột phải vào Job bị lỗi > `Edit`.
   - Tab `Virtual Machines`: Chọn VM báo lỗi -> Bấm **Remove**.
   - Bấm **Add** -> Duyệt lại cây thư mục vCenter -> Chọn lại đúng VM đó.
   - Bấm Finish.

5. **Nghiệm thu (Verification):**
   - Bôi đen các Jobs lỗi -> Chuột phải chọn **Retry** (Không chọn Start để tối ưu tài nguyên, chỉ chạy lại các phần lỗi).
   - Theo dõi log: Data transfer bình thường, trạng thái báo `Waiting for backup infrastructure resources availability` (Đang xếp hàng do proxy bận) -> **Hệ thống đã thông suốt, trạng thái an toàn.**

---

## 4. Kinh nghiệm rút ra (Lessons Learned & Pro-tips)
- **Không nhắm mắt bấm Start:** Trong môi trường Production, luôn dùng tính năng **Retry** cho các failed jobs để tránh overload hệ thống (Veeam sẽ chỉ quét các block bị lỗi).
- **Communication Flow:** Khi gặp lỗi hệ thống, báo cáo log/lỗi công khai trên Group chat của Team, nhưng **Password phải được gửi qua kênh PM/Inbox cá nhân** để tuân thủ bảo mật.
- **Xác minh chéo (Cross-check):** Khi Veeam báo VM tắt, luôn mở vCenter để double-check trạng thái nguồn (Power State) thực tế trước khi kết luận.
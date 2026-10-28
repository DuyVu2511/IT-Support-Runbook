# Runbook: Quy trình Phân quyền Helpdesk trên Active Directory và Exchange (EAC)

## 1. Bối cảnh & Yêu cầu (Overview)

- **Hệ thống:** Active Directory Domain Services (Windows Server), Microsoft Exchange Server (EAC / EMS).
- **Yêu cầu:** Cấp quyền cho nhân sự IT Helpdesk (`<Helpdesk_Account_1>`, `<Helpdesk_Account_2>`) được phép **Tạo mới tài khoản** và **Reset mật khẩu** cho người dùng thông thường trong toàn bộ domain.
- **Phạm vi tác động:**
  - Toàn bộ các OU phòng ban nghiệp vụ / chi nhánh (`<OU_Branch_1>`, `<OU_Branch_2>`, `<OU_HQ>`, ...).
  - **Loại trừ tuyệt đối:** Không phân quyền tại Domain Root để bảo vệ các tài khoản Quản trị cấp cao (Domain Admins, Enterprise Admins) và tài khoản dịch vụ hệ thống (Service Accounts / Builtin).

---

## 2. Bản chất kỹ thuật (Key Concepts)

1. **Active Directory (AD DS):**
   - Quản lý đối tượng User và Mật khẩu đăng nhập Windows.
   - Sử dụng cơ chế **Delegate Control** trên từng OU mục tiêu để cấp quyền tạo tài khoản và đổi mật khẩu cho Helpdesk.
2. **Exchange Admin Center (EAC):**
   - Trên web EAC, mọi tài khoản tạo mới mặc định là một **User Mailbox** (đồng thời khởi tạo User trên AD và cấp Hòm thư trên Exchange).
   - Bắt buộc phải cấp Role **`Recipient Management`** qua RBAC thì tài khoản IT Helpdesk mới có thẩm quyền và giao diện để tạo hòm thư trên EAC.

---

## 3. Quy trình triển khai (Step-by-Step Execution)

### Phần 1: Phân quyền trên Active Directory (AD DS)

Thực hiện lần lượt trên **toàn bộ các OU phòng ban nghiệp vụ**:

1. Mở công cụ **Active Directory Users and Computers (ADUC)**.
2. Click chuột phải vào OU mục tiêu -> Chọn **Delegate Control...** -> Bấm **Next**.
3. Tại bước *Users or Groups*: Bấm **Add...** -> Nhập tài khoản IT Helpdesk (hoặc Security Group quản trị) -> Bấm **Next**.
4. Tại bước *Tasks to Delegate*: Tick chọn đúng 2 quyền:
   - [x] `Create, delete, and manage user accounts` (Quyền tạo và quản lý user)
   - [x] `Reset user passwords and force password change at next logon` (Quyền reset mật khẩu)
5. Bấm **Next** -> **Finish**.

> [!note]
> Lặp lại đầy đủ cho các OU chi nhánh / phòng ban còn lại trong phạm vi quản lý.

---

### Phần 2: Phân quyền trên Exchange Server

Mở **Exchange Management Shell (EMS)** trên máy chủ Exchange và thực thi:

```powershell
# Gán quyền Quản lý Người nhận (Recipient Management) cho các tài khoản IT Helpdesk
Add-RoleGroupMember -Identity "Recipient Management" -Member "<Helpdesk_Account_1>"
Add-RoleGroupMember -Identity "Recipient Management" -Member "<Helpdesk_Account_2>"

# (Tùy chọn) Gán thêm quyền Help Desk cơ bản nếu có yêu cầu
Add-RoleGroupMember -Identity "Help Desk" -Member "<Helpdesk_Account_1>"
Add-RoleGroupMember -Identity "Help Desk" -Member "<Helpdesk_Account_2>"
```

---

## 4. Kiểm tra & Nghiệm thu (Verification)

### 4.1. Kiểm tra trên Active Directory

- Trên thanh công cụ ADUC, chọn **View** -> Tick chọn **Advanced Features**.
- Chuột phải vào OU đã phân quyền -> Chọn **Properties** -> Tab **Security**.
- Xác nhận tài khoản IT Helpdesk (hoặc Security Group) đã xuất hiện trong danh sách **Group or user names** với các quyền tương ứng.

### 4.2. Kiểm tra trên Exchange Management Shell

Chạy lệnh kiểm tra danh sách thành viên trong Role Group:

```powershell
Get-RoleGroupMember -Identity "Recipient Management"
```

Xác nhận đầu ra có chứa `<Helpdesk_Account_1>` và `<Helpdesk_Account_2>` với `RecipientType = UserMailbox`.
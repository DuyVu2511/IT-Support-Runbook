# Runbook: Truy Bắt Log Lỗi Đăng Nhập & Khóa Tài Khoản AD (Event Viewer)

## 1. Công Cụ Chẩn Đoán
* **Công cụ:** Event Viewer (eventvwr.msc) trên Domain Controller.
* **Đường dẫn Log:** Event Viewer (Local) -> Windows Logs -> Security.

## 2. Các Event ID Sinh Tử
* **Event ID 4625:** Đăng nhập thất bại (An account failed to log on). Kiểm tra do người dùng nhập sai Pass hay do máy tính/script tự gõ Pass cũ liên tục.
* **Event ID 4740:** Tài khoản bị khóa (A user account was locked out). Mở ADUC -> Tìm User -> Tab Account -> Bỏ tích Unlock account. Soi chi tiết log để biết IP/Máy tính nguồn gây ra lockout.

## 3. Quy Trình Trả Lời Phỏng Vấn (Tư Duy SysAdmin)
> "Khi tài khoản người dùng bị khóa liên tục, em sẽ kiểm tra Event ID 4740 để xác định nguồn gõ sai pass. Sau đó em gỡ khóa tại tab Account trong ADUC. Nếu họ báo lỗi Access Denied vào thư mục, em kiểm tra lại quyền NTFS và mô hình phân quyền AGDLP."
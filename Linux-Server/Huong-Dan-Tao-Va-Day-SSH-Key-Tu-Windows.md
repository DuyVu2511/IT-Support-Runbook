# Hướng dẫn Tạo và Cấu hình Passwordless SSH từ Windows

Áp dụng để quản trị hàng loạt Server Linux mà không cần gõ mật khẩu, đồng thời chuẩn hóa bảo mật chống tool dò quét (Brute-force).

## Bước 1: Tạo cặp khóa mã hóa trên Windows
Mở Windows Terminal (hoặc PowerShell/CMD) và chạy lệnh:
```powershell
ssh-keygen -t rsa -b 4096
```
*(Bấm Enter liên tục để dùng thiết lập mặc định).*

> [!INFO] Bản chất của cặp khóa
> Lệnh này sinh ra 2 file nằm trong `C:\Users\<Tên_User>\.ssh\`:
> - **`id_rsa` (Private Key - Chìa khóa gốc):** Tuyệt đối giữ bí mật trên máy cá nhân, không chia sẻ cho bất kỳ ai.
> - **`id_rsa.pub` (Public Key - Ổ khóa):** File dạng plain text dùng để đẩy lên các Server đích.

## Bước 2: Bắn Public Key sang Server Linux (One-liner PowerShell)
Sử dụng câu lệnh dưới đây để tự động hóa toàn bộ quá trình: Tạo thư mục `.ssh` trên server, chép key vào file `authorized_keys`, và cấp quyền (chmod) chuẩn bảo mật.

Chạy lệnh này trực tiếp trên PowerShell của máy tính Windows:
```powershell
type $env:USERPROFILE\.ssh\id_rsa.pub | ssh user@<IP_Server> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
*(Lưu ý: Thay `user@<IP_Server>` bằng user và IP thực tế của Server. Hệ thống sẽ hỏi mật khẩu 1 lần duy nhất ở bước này).*

## Bước 3: Nghiệm thu kết quả
Đăng nhập lại vào Server:
```powershell
ssh user@<IP_Server>
```
Nếu Terminal nhảy thẳng vào dấu nhắc lệnh của Server mà **không hiện dòng yêu cầu nhập mật khẩu**, quá trình thiết lập SSH Key đã hoàn tất 100%.
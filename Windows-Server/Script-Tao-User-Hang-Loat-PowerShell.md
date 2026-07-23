# Runbook: Tự Động Hóa Tạo User Active Directory Hàng Loạt Qua PowerShell

## 1. Kịch Bản Vận Hành (Scenario)
* **Bài toán:** Nhân sự HR gửi danh sách 50 nhân viên mới dưới dạng file CSV. Yêu cầu khởi tạo tài khoản AD hàng loạt, gán mật khẩu mặc định, ép buộc đổi mật khẩu ở lần đăng nhập đầu tiên và xếp đúng OU phòng ban.
* **Mục tiêu:** Tự động hóa 100%, giảm thời gian thao tác từ 2 tiếng xuống 5 giây, chống sai sót thủ công.

## 2. File Dữ Liệu Đầu Vào (users.csv)
Tạo file CSV cấu trúc như sau:

SamAccountName,Firstname,Lastname,OUPath,Password
vutd,Vu,Tran,"OU=IT,DC=domain,DC=local",Default@2026
namnv,Nam,Nguyen,"OU=KeToan,DC=domain,DC=local",Default@2026

## 3. Cấu Trúc Script PowerShell Cốt Lõi

# 1. Import file CSV
$Users = Import-Csv -Path "C:\Path\To\users.csv"

# 2. Vòng lặp foreach duyệt qua từng dòng
foreach ($User in $Users) {
    # Mã hóa mật khẩu sang chuỗi an toàn
    $SecurePassword = ConvertTo-SecureString $User.Password -AsPlainText -Force
    
    # Tạo User trên Active Directory
    New-ADUser -SamAccountName $User.SamAccountName -GivenName $User.Firstname -Surname $User.Lastname -Name "$($User.Firstname) $($User.Lastname)" -Path $User.OUPath -AccountPassword $SecurePassword -Enabled $true -ChangePasswordAtLogon $true
}

## 4. Bốn Khối Logic Sinh Tử Cần Nhớ Khi Phỏng Vấn
1. Import-Csv: Đọc file danh sách HR gửi.
2. foreach: Vòng lặp quét từng nhân viên.
3. ConvertTo-SecureString: Mã hóa mật khẩu thành dạng mã an toàn.
4. New-ADUser với tham số -ChangePasswordAtLogon $true: Tạo user và bắt buộc đổi mật khẩu ngay lập tức.
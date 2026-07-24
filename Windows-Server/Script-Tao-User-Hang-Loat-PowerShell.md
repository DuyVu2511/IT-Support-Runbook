# Runbook: Tự Động Hóa Tạo User Active Directory Hàng Loạt Qua PowerShell

## 1. Kịch Bản Vận Hành (Scenario)
* **Bài toán:** Phòng HR gửi danh sách nhân viên mới dưới dạng file CSV. Yêu cầu khởi tạo tài khoản AD hàng loạt, gán mật khẩu mặc định, ép buộc đổi mật khẩu ở lần đăng nhập đầu tiên và xếp đúng OU/Phòng ban.
* **Mục tiêu:** Tự động hóa 100%, giảm thời gian thao tác từ 2 tiếng xuống 5 giây, triệt tiêu sai sót thủ công.

---

## 2. File Dữ Liệu Đầu Vào (users.csv)
Cấu trúc file CSV chuẩn hóa từ HR:
```csv
SamAccountName,Firstname,Lastname,Department,Password
vutd,Vu,Tran,IT,Default@2026!
namnv,Nam,Nguyen,KeToan,Default@2026!
```

---

## 3. Cấu Trúc Script PowerShell Cốt Lõi (Standard Production Code)

```powershell
Import-Module ActiveDirectory

# 1. Đọc file CSV và lấy tên Domain tự động
$Users = Import-Csv -Path "C:\LabScript\users.csv"
$Domain = (Get-ADDomain).DNSRoot

# 2. Vòng lặp foreach quét qua từng nhân viên
foreach ($User in $Users) {
    
    # 3. Mã hóa mật khẩu sang chuỗi an toàn (SecureString)
    $SecurePassword = ConvertTo-SecureString $User.Password -AsPlainText -Force
    
    # 4. Gom bảng thuộc tính (Splatting) - Chuẩn SysAdmin
    $UserParams = @{
        SamAccountName        = $User.SamAccountName
        UserPrincipalName     = "$($User.SamAccountName)@$Domain"
        GivenName             = $User.Firstname
        Surname               = $User.Lastname
        Name                  = "$($User.Firstname) $($User.Lastname)"
        Department            = $User.Department
        AccountPassword       = $SecurePassword
        Enabled               = $true
        ChangePasswordAtLogon = $true # Ép đổi mật khẩu lần đầu
    }
    
    # 5. Khởi tạo User có bắt lỗi (Try-Catch)
    try {
        if (-not (Get-ADUser -Filter "SamAccountName -eq '$($User.SamAccountName)'")) {
            New-ADUser @UserParams
            Write-Host "[SUCCESS] Đã tạo User: $($User.SamAccountName)" -ForegroundColor Green
        } else {
            Write-Host "[SKIP] User đã tồn tại: $($User.SamAccountName)" -ForegroundColor Yellow
        }
    } catch {
        Write-Host "[ERROR] Lỗi khởi tạo: $_" -ForegroundColor Red
    }
}
```

---

## 4. Các Khối Logic "Sinh Tử" Cần Thuộc Lòng Khi Phỏng Vấn
1. `Import-Csv`: Đọc dữ liệu thô từ file HR gửi.
2. `foreach`: Vòng lặp duyệt qua từng nhân viên.
3. `ConvertTo-SecureString`: Bắt buộc mã hóa mật khẩu sang chuỗi an toàn thì Active Directory mới chấp nhận gán pass.
4. `New-ADUser` + `@UserParams` (Splatting): Tạo User với tham số `-ChangePasswordAtLogon $true` để ép buộc đổi mật khẩu ở lần đăng nhập đầu tiên (Bảo mật Zero-Trust).
5. `Try-Catch` & `Get-ADUser`: Kiểm tra trùng lặp và bắt lỗi hệ thống để script không bị dừng đột ngột.
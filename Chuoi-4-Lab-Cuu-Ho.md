# 📘 SỔ TAY QUẢN TRỊ & CỨU HỘ HỆ THỐNG (WINPE & WINDOWS SERVER)
**Môi trường:** Ảo hóa VMware Workstation / Windows Server 2022 / Windows 11

---

## 🛠️ Lab 1: Nạp ISO và Boot môi trường WinPE

### 1. Mục tiêu (What & Why)
Khởi tạo môi trường cứu hộ độc lập (WinPE) trên máy ảo VMware. Đây là bước đệm bắt buộc để IT can thiệp sâu vào hệ thống file và phần cứng khi hệ điều hành chính (Windows) bị hỏng, sập hoặc khóa truy cập.

### 2. Các bước thực hiện (How)
*   **Bước 1:** Tắt nguồn máy ảo Windows 11 hoàn toàn.
*   **Bước 2:** Truy cập *Virtual Machine Settings*, tại mục *CD/DVD (SATA)*, trỏ đường dẫn nạp file ISO cứu hộ (ví dụ: `Anhdv_Boot_Free_26.2.iso`).
*   **Bước 3:** Bật nguồn máy ảo, nhanh tay truy cập Boot Manager và chọn ưu tiên khởi động từ ổ đĩa CD-ROM.
*   **Bước 4:** Đợi hệ thống nạp dữ liệu và truy cập thành công vào giao diện Desktop trực quan của WinPE.

### 3. Tình huống ứng dụng (Thực chiến)
Máy tính ảo hóa (VM) của người dùng hoặc máy chủ bị sập hệ điều hành không thể khởi động. Quản trị viên cần mount file ISO cứu hộ vào hạ tầng ảo hóa để chẩn đoán lỗi.

---

## 🛠️ Lab 2: Gỡ bỏ Mật khẩu Windows 11 (Bypass Password)

### 1. Mục tiêu (What & Why)
Khôi phục quyền truy cập máy tính cho người dùng khi họ quên mật khẩu đăng nhập Windows, đảm bảo nhanh chóng và không làm mất dữ liệu hiện có trên màn hình Desktop của họ.

### 2. Các bước thực hiện (How)
*   **Bước 1:** Boot thành công vào môi trường ISO WinPE.
*   **Bước 2:** Mở công cụ can thiệp bảo mật hệ thống: **NTPWEdit**.
*   **Bước 3:** Mở file lưu trữ mật khẩu cục bộ của Windows (file SAM).
*   **Bước 4:** Chọn tài khoản đang bị khóa (`NhanVien_QuenPass`) -> Bấm **Change password**.
*   **Bước 5:** Để trống hoàn toàn 2 ô nhập liệu để tiến hành xóa trắng mật khẩu -> Bấm **Save changes**.
*   **Bước 6:** Khởi động lại máy. Người dùng đăng nhập thành công vào Windows 11 mà không bị hệ thống hỏi mật khẩu.

### 3. Tình huống ứng dụng (Thực chiến)
Nhân viên đi công tác dài ngày về quên mật khẩu đăng nhập máy tính công ty. IT Support dùng công cụ chuyên dụng để gỡ bỏ hoàn toàn mật khẩu cũ, giúp nhân viên vào lại máy làm việc ngay lập tức.

> **🎬 Video minh chứng thực hành:**
> <video src="Video-Lab02.mp4" controls width="100%"></video>

---

## 🛠️ Lab 3: Cứu hộ Dữ liệu khẩn cấp khi gặp lỗi BSOD

### 1. Mục tiêu (What & Why)
Đảm bảo an toàn tuyệt đối cho các file tài liệu quan trọng của công ty khi máy tính đột ngột gặp sự cố Màn hình xanh chết chóc (BSOD - CRITICAL_PROCESS_DIED).

### 2. Các bước thực hiện (How)
*   **Bước 1:** Ghi nhận sự cố BSOD. Buộc tắt máy và boot vào môi trường WinPE.
*   **Bước 2:** Kết nối USB vật lý (Ventoy) từ máy thật (Host) vào máy ảo (Guest).
*   **Bước 3:** Truy cập vào phân vùng ổ C: của Windows cũ, tìm đến màn hình Desktop của người dùng để trích xuất file quan trọng (`Bao_Cao_Tai_Chinh_Q3.txt`).
*   **Bước 4:** Tạo thư mục `DATA_RECOVERY` trên USB và dán file dữ liệu vào đây.
*   **Bước 5:** Mở file text trực tiếp trên USB để kiểm tra nghiệm thu. Dữ liệu đọc tốt, không bị lỗi font hay mã hóa.

### 3. Tình huống ứng dụng (Thực chiến)
Máy tính kế toán đột ngột sập màn hình xanh không thể sửa lỗi. IT Support phải dùng WinPE chui vào ổ cứng hỏng để lấy file báo cáo tài chính ra USB an toàn trước khi quyết định cài lại Windows.

> **🎬 Video minh chứng thực hành:**
><video src="Video-Lab03.mp4" controls width="100%"></video>

---

## 🛠️ Lab 4: Phân quyền File Server Doanh nghiệp (AGDLP)

### 1. Mục tiêu (What & Why)
Xây dựng hạ tầng quản lý tệp tin trên Windows Server 2022. Áp dụng chuẩn bảo mật AGDLP (Accounts -> Global Groups -> Domain Local Groups -> Permissions) trên Active Directory để ngăn chặn nguy cơ rò rỉ thông tin chéo giữa các phòng ban.

### 2. Các bước thực hiện (How)
*   **Bước 1 (Xác thực):** Mở CMD trên Domain Controller, gõ `ipconfig /all` và `whoami` để xác thực máy chủ đang chạy IP tĩnh `192.168.50.10` và nắm quyền quản trị Domain `duyvu.vn`.
*   **Bước 2 (Cấu hình AD):** Thiết lập kiến trúc lồng Group trên ADUC: Đưa Global Group `G_KeToan` vào trong Domain Local Group `DL_KeToan_Modify`.
*   **Bước 3 (Đăng nhập):** Đăng nhập máy Client Win 11 bằng tài khoản nhân viên đã tham gia Domain (`NV_KeToan@duyvu.vn`).
*   **Bước 4 (Kết nối UNC):** Bấm tổ hợp Windows + R, gõ đường dẫn mạng `\\192.168.50.10` để truy cập File Server.
*   **Bước 5 (Nghiệm thu):** 
    *   *Thành công:* Truy cập thư mục `Phong_Ke_Toan`, tạo mới file `Bao_cao_thang_7.txt` và ghi dữ liệu thành công (Đạt quyền Modify).
    *   *Thất bại:* Click truy cập chéo sang thư mục `Phong_Nhan_Su`, hệ thống lập tức khóa và bật bảng cảnh báo **Access Denied**.

### 3. Tình huống ứng dụng (Thực chiến)
Doanh nghiệp yêu cầu quy hoạch lại dữ liệu nội bộ. Giám đốc yêu cầu Kế toán chỉ được làm việc trong thư mục Kế toán, tuyệt đối không được phép xem hoặc chỉnh sửa bảng lương nằm trong thư mục của phòng Nhân sự.

> **🎬 Video minh chứng thực hành:**
><video src="Video-Lab04.mp4" controls width="100%"></video>
# Quy Trình Triển Khai HA: Windows Server Failover Cluster (WSFC) & SQL AlwaysOn

Bài lab mô phỏng kiến trúc High Availability (HA) cấp độ Enterprise dành cho các hệ thống trọng yếu (Ngân hàng, Tài chính, ERP), đảm bảo Uptime 99.9% và Zero Data Loss.

## 1. Yêu cầu Hạ tầng (Topology)
Hệ thống được triển khai hoàn toàn trong vùng mạng cô lập (VLAN riêng), gồm:
- **01 Domain Controller (DC):** Quản lý phân quyền AD, DNS.
- **02 Node SQL Server (SQL01, SQL02):** Chạy Windows Server, đã join Domain.
- **01 File Server:** Làm nơi chứa Quorum (File Share Witness).

## 2. Các bước cấu hình cốt lõi

### Giai đoạn 1: Triển khai Windows Server Failover Cluster (WSFC)
- Cài đặt tính năng `Failover Clustering` trên cả 2 node SQL01 và SQL02.
- Chạy **Cluster Validation Report** để đảm bảo hạ tầng mạng và phần cứng đạt chuẩn Microsoft.
- Khởi tạo Cluster (ví dụ tên: `SQL-Cluster`) kèm 1 IP ảo (VIP) để quản trị.
- **Cấu hình Quorum (File Share Witness):** Trỏ Quorum về một thư mục share trên DC hoặc File Server để làm "người phán xử" (Tie-breaker) chống hiện tượng Split-Brain khi 2 node mất kết nối với nhau.

### Giai đoạn 2: Cấu hình SQL Server AlwaysOn
- Bật tính năng *Enable AlwaysOn Availability Groups* trong SQL Server Configuration Manager trên cả 2 node và khởi động lại dịch vụ `MSSQLSERVER`.
- Khởi tạo **Availability Group (AG)**: Đưa Database cần bảo vệ vào nhóm AG.
- **Cấu hình Replica:**
  - `SQL01`: Primary Replica (Đọc/Ghi).
  - `SQL02`: Secondary Replica (Chỉ Đọc/Đồng bộ).
  - Chế độ đồng bộ: **Synchronous-commit** (Đảm bảo không mất dữ liệu - Zero Data Loss) và **Automatic Failover** (Tự động chuyển quyền khi Node chính sập).

### Giai đoạn 3: Khởi tạo Listener
- Khởi tạo **Availability Group Listener** (ví dụ tên: `App-DB-Listener`) kèm IP ảo và port `1433`.
- **Mục đích:** Ứng dụng (Web/App) sẽ chỉ kết nối vào tên Listener này thay vì IP của từng máy vật lý. Khi có sự cố, Listener tự động định tuyến traffic sang Node đang nắm quyền Primary mà ứng dụng không hề hay biết.

## 3. Kịch bản Troubleshooting & Kiểm thử (Failover Test)
- **Tình huống:** Tắt nóng (Power off) máy SQL01 (Primary).
- **Kết quả mong đợi:** 
  1. Quorum và Node SQL02 phát hiện SQL01 "chết".
  2. SQL02 tự động được đẩy lên làm Primary.
  3. Listener tự động trỏ IP về SQL02. Ứng dụng chỉ bị gián đoạn vài giây (Timeout) rồi hoạt động lại bình thường.
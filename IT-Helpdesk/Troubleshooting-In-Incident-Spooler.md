# Runbook: Troubleshooting Sự Cố Máy In Mạng LAN & Kẹt Lệnh In

## 1. Case 1: Máy in thông Mạng (Ping thông) nhưng Máy tính báo "Offline"
* **Nguyên nhân cốt lõi:** Windows dùng giao thức SNMP để vấn tin trạng thái máy in. Nhiều dòng máy in hoặc tường lửa chặn gói tin SNMP làm Windows hiểu nhầm là máy in đã tắt.
* **Cách khắc phục triệt để:**
  1. Vào Control Panel -> Devices and Printers.
  2. Chuột phải vào máy in -> Printer Properties -> Tab Ports.
  3. Chọn Cổng IP đang dùng -> Bấm Configure Port...
  4. Bỏ tích chọn SNMP Status Enabled -> Bấm OK.

## 2. Case 2: Kẹt Lệnh In (Print Spooler Stuck) & Reset Bằng CMD
* **Triệu chứng:** Hàng dài lệnh in bị kẹt (Status: Printing/Deleting), bấm Cancel không trôi.
* **Quy trình Xử lý Nhanh bằng CMD (Run as Administrator):**

net stop spooler
del /Q /F /S "%systemroot%\System32\Spool\Printers\*.*"
net start spooler
# Hệ Thống Giám Sát An Ninh Mạng (SIEM/IDS) & Phát Hiện Sớm Tấn Công Ransomware

## Giới Thiệu Dự Án
Dự án tập trung vào việc nghiên cứu, xây dựng và triển khai một phân vùng mạng doanh nghiệp giả lập (SOC Lab) tích hợp các giải pháp SIEM/IDS nhằm giám sát, bóc tách log hệ thống chi tiết và đưa ra các cảnh báo, phản ứng sớm đối với các hành vi bất thường, đặc biệt là các biến thể mã độc mã hóa dữ liệu (Ransomware).

---

## Kiến Trúc Hệ Thống & Cấu Hình Thiết Bị

### 1. Tường Lửa Tối Ưu Phân Vùng (pfSense Firewall)
* **Vùng mạng (Interfaces):**
  * **Cổng WAN:** Nhận cấu hình IP động (DHCP Private) từ hạ tầng PNetLab để định tuyến ra Internet.
  * **Cổng DMZ (Interface `em1`):** Cấu hình dải IP tĩnh `10.10.10.1/24` nhằm cách ly các máy chủ công khai.
  * **Vùng INTERNAL (Interface `em2`):** Cấu hình dải mạng nội bộ `192.168.1.1/24` cho các máy chủ/máy trạm doanh nghiệp.
* **Luật Tường Lửa (Firewall Rules) & NAT:**
  * **DMZ Rules:** * *Luật Chặn:* Ngăn chặn triệt để mọi lưu lượng chủ động kết nối từ vùng DMZ (`DMZ net`) sang vùng nội bộ INTERNAL (`LAN net`).
    * *Luật Cho Phép:* Kích hoạt quyền truy cập từ `DMZ net` đi `any` hướng Internet hỗ trợ cập nhật ứng dụng.
  * **WAN NAT (Port Forward):**
    * NAT cổng công khai HTTP/HTTPS về máy chủ Web (`10.10.10.5`).
    * NAT cổng truy cập từ xa `2222` ngoài WAN về cổng mã nguồn SSH `22` của Web Server để hỗ trợ quản trị từ máy thật.

### 2. Máy Chủ Ứng Dụng (Ubuntu Web Server - Vùng DMZ)
* **Thông số Network:** IP Tĩnh `10.10.10.5/24` | Gateway: `10.10.10.1` | DNS: `8.8.8.8`
* **Các dịch vụ cốt lõi đã triển khai:**
  * **Nginx Web Server:** Chuyển đổi định dạng ghi log truy cập (`access_log`) mặc định sang cấu trúc dữ liệu kiểu **JSON** (`/var/log/nginx/access.json`) phục vụ phân tích log tự động trên SIEM.
  * **OpenSSH Server:** Kích hoạt quyền `PermitRootLogin yes` tương thích cho việc cấu hình hạ tầng Lab an toàn qua Terminal máy thật.
  * **Auditd (Linux Auditing System):** Kích hoạt hệ thống kiểm toán nhằm theo dõi tính toàn vẹn của mã nguồn Web và giám sát các hành vi thao túng tệp tin hệ thống.

### 3. Máy Chủ Quản Trị Hệ Thống (Windows Server - Vùng INTERNAL)
* **Thông số Network:** IP Tĩnh `192.168.1.2/24` | Gateway: `192.168.1.1` | DNS: `192.168.1.2`
* **Dịch vụ & Dữ liệu giả lập:**
  * **Active Directory Domain Services (AD DS):** Cấu hình vai trò Domain Controller cho Forest nội bộ (Ví dụ: `soclab.local`).
  * **BadBlood Automation Script:** Khởi chạy thành công kịch bản PowerShell tự động sinh dữ liệu doanh nghiệp quy mô lớn (Hàng trăm OUs, nhóm bảo mật Groups, và các tài khoản người dùng ngẫu nhiên) để tạo môi trường giám sát thực tế.
  * **Mục tiêu Giả Lập (File Share):** Thiết lập vùng thư mục dùng chung `C:\Data_DoAn` (Cấp quyền Full Control cho nhóm `Everyone`) làm đích ngắm bẫy log khi thực hiện các kịch bản Ransomware mã hóa dữ liệu.

---

## 🛠️ Nhật Ký Xử Lý Sự Cố Hạ Tầng (Troubleshooting Log)

Dưới đây là danh sách tổng hợp các lỗi kỹ thuật hệ thống mạng/hệ điều hành đã phát sinh trong quá trình thiết lập Lab và các biện pháp khắc phục tương ứng:

| STT | Mô Tả Lỗi | Nguyên Nhân Gốc (Root Cause) | Giải Pháp Khắc Phục (Resolution) |
| :--- | :--- | :--- | :--- |
| **1** | Thiết bị Linux gửi tin nhắn `DHCPDISCOVER` liên tục nhưng không nhận được IP | Lỗi thiết kế Topology vật lý. Việc kết nối dây "bắc cầu" trực tiếp giữa các máy chủ không qua Switch tạo vòng lặp mạng (Loop) khiến pfSense không tiếp nhận được gói tin Broadcast. | Chuyển đổi hoàn toàn cơ chế nhận cấu hình IP của card mạng sang dạng **IP Tĩnh (Static IP)** định nghĩa trong Netplan. |
| **2** | Lỗi định tuyến hệ thống `Network is unreachable` khi ping ra ngoài Internet | Tệp tin cấu hình mạng `/etc/netplan/00-installer-config.yaml` mới chỉ định nghĩa IP cục bộ mà thiếu thông số định tuyến lối ra (`gateway4`). | Bổ sung trường `gateway4: 10.10.10.1` trỏ thẳng về IP chân cổng nội bộ tương ứng của Tường lửa pfSense. |
| **3** | Lưu lượng Ping ra Internet bị mất sạch (**100% packet loss**) mặc dù mạng Local đã thông | pfSense kích hoạt chế độ mặc định chặn toàn bộ lưu lượng định tuyến từ các dải IP Private thông qua cổng WAN trong môi trường ảo hóa Lab lồng nhau. | Truy cập giao diện quản trị Web GUI $\rightarrow$ **Interfaces** $\rightarrow$ **WAN** $\rightarrow$ Bỏ chọn mục **Block private networks and loopback addresses**. |
| **4** | Lỗi cài đặt gói ứng dụng `403 Forbidden` khi sử dụng trình quản lý APT | Kho lưu trữ Repo mặc định trong template hệ điều hành trỏ về máy chủ Đại học Thanh Hoa (`mirrors.tuna.tsinghua.edu.cn`) bị chặn truy cập từ dải mạng ngoại bang. | Ghi đè tệp tin `/etc/apt/sources.list` hướng luồng tải về cụm máy chủ chính thức của Ubuntu (`vn.archive.ubuntu.com`) và làm sạch cache bằng `apt clean && apt update`. |
| **5** | Hệ thống báo lỗi phân giải tên miền `Could not resolve 'archive.ubuntu.com'` | Sai sót lỗi chính tả trong file cấu hình (gõ nhầm thành `achive`) phối hợp lỗi chưa nhận diện được máy chủ phân giải DNS quốc tế. | Chuẩn hóa lại chuỗi ký tự Repo và thực hiện ép cấu hình DNS của Google thông qua lệnh: `echo "nameserver 8.8.8.8" > /etc/resolv.conf`. |

---

## 📅 Kế Hoạch Triển Khai Tiếp Theo
- [ ] Cài đặt Agent Sysmon và cấu hình Group Policy nâng cao trên môi trường Windows.
- [ ] Join máy trạm Windows Client vào Domain Controller.
- [ ] Triển khai máy chủ giám sát trung tâm Ubuntu SOC (Wazuh Manager & Suricata IDS).

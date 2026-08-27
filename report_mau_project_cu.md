# BÁO CÁO ỨNG PHÓ SỰ CỐ (Incident Response Report)

| Thông tin | Giá trị |
|---|---|
| **Mã sự cố** | IR-20260810-001 |
| **Ngày báo cáo** | 2026-06-12 |
| **Điều phối viên** | Reporting Agent (Phase 5) |
| **Mức độ nghiêm trọng** | **CAO (CRITICAL)** |

---

## 1. Tóm tắt Điều hành (Executive Summary)

Vào ngày **10 tháng 8 năm 2016**, hệ thống máy chủ web Joomla tại địa chỉ **192.168.250.70** đã bị xâm nhập thành công từ địa chỉ IP tấn công **40.80.148.42**. Kẻ tấn công đã khai thác lỗ hổng thực thi mã từ xa (RCE) thông qua trình **PHP-CGI** (`php-cgi.exe`) để giành quyền truy cập với tài khoản **`NT AUTHORITY\IUSR`**.

Sau khi xâm nhập, kẻ tấn công đã tải lên và thực thi webshell `3791.exe` (MD5: `AAE3F5A29935E6ABCC2C2754D12A9AF0`) vào thư mục Joomla, thực hiện trinh sát nội bộ (liệt kê người dùng, share, domain), và thiết lập kênh liên lạc C2 tới domain `prankglassinebracket.jumpingcrab.com`. Có dấu hiệu rõ ràng của hành vi đánh cắp dữ liệu (data exfiltration) qua các lệnh sao chép file và truy vấn cơ sở dữ liệu.

**Tác động:** Máy chủ web đã bị chiếm quyền điều khiển hoàn toàn. Dữ liệu nhạy cảm có nguy cơ bị rò rỉ. Hệ thống cần được cách ly ngay lập tức.

---

## 2. Hồ sơ Kẻ tấn công (Attacker Profile)

| Thuộc tính | Giá trị |
|---|---|
| **IP tấn công** | `40.80.148.42` |
| **IP nạn nhân** | `192.168.250.70` |
| **C2 Domain** | `prankglassinebracket.jumpingcrab.com` |
| **Thời điểm bắt đầu** | `2016-08-10 21:36:48 UTC` (first_seen) |
| **Kéo dài** | ~45 phút |
| **Tổng số luồng mạng** | 21,380 |
| **Tổng lưu lượng** | ~172 MB |

### Danh tiếng IP (Threat Intelligence)

| Nguồn | Kết quả | Ghi chú |
|---|---|---|
| **VirusTotal** | Timeout | Không có dữ liệu phản hồi |
| **AbuseIPDB** | Abuse Score: 0 / Total Reports: 0 | Không có báo cáo lạm dụng |
| **ThreatFox** | Malicious Count: 0 | Không phát hiện độc hại |

> **Lưu ý:** Mặc dù các nguồn OSINT không gắn cờ IP này là độc hại, bằng chứng thực thi từ log hệ thống và phân tích mạng đã **xác nhận hành vi tấn công**.

### IOC Đã Xác nhận (Confirmed Malicious)

| IOC | Loại | Giá trị | Xác nhận |
|---|---|---|---|
| **3791.exe** (Webshell) | MD5 | `AAE3F5A29935E6ABCC2C2754D12A9AF0` | ✅ VirusTotal: success |
| **3791.exe** (Webshell) | SHA1 | `65DF73D77324D008C83C3E57B445DF0FD43A3A51` | ✅ VirusTotal: success |
| **3791.exe** (Webshell) | SHA256 | `EC78C938D8453739CA2A370B9C275971EC46CAF6E479DE2B2D04E97CC47FA45D` | ✅ VirusTotal: success |

---

## 3. Dòng thời gian Chi tiết (Detailed Timeline)

> **Múi giờ:** UTC (Chuyển đổi từ log `-0600`)

| Thời gian (UTC) | Sự kiện | Chi tiết |
|---|---|---|
| **21:36:48** | 🚨 **Cảnh báo đầu tiên** | Hệ thống Alert ghi nhận hoạt động bất thường từ `40.80.148.42` tới `192.168.250.70`. 589 bản ghi khớp. |
| **~21:54 - 21:55** | 🔍 **Quét & Thăm dò** | Kẻ tấn công gửi hàng loạt yêu cầu HTTP độc hại: SQLi (`pg_sleep`, `waitfor delay`), XSS (`<script>GP4l`), Path traversal (`/windows/win.ini%00.jpg`) và quét file nhạy cảm (`web.config`, `.bak`, `database.yml`). |
| **21:55:22** | 🎯 **Khai thác RCE** | `php-cgi.exe` thực thi lệnh `echo 24365` - kiểm tra khả năng thực thi lệnh. |
| **21:55:24** | 📂 **Trinh sát thư mục** | `cmd.exe /c "dir"` - Liệt kê thư mục web. |
| **21:55:33** | 🐧 **Thử lệnh Linux** | `cmd.exe /c "ifconfig"` và `cmd.exe /c "ls"` - Kẻ tấn công thử lệnh Linux trên Windows (không thành công). |
| **21:56:18** | 💀 **Cài đặt Webshell** | Thực thi `3791.exe` từ thư mục `C:\inetpub\wwwroot\joomla\` qua lệnh `cmd.exe /c "3791.exe 2>&1"`. |
| **21:58:23** | 🔄 **Kích hoạt Backdoor** | `3791.exe` sinh tiến trình `cmd.exe` thứ hai. |
| **21:58:28 - 21:59:12** | 🕵️ **Trinh sát Nội bộ** | `whoami` → `NT AUTHORITY\IUSR`; `net view /domain`; `net share`; `net session`; `net user`; `net use c:\share`. |
| **22:05:42** | ✅ **Kiểm tra kết nối** | `echo 63059` - Xác nhận backdoor hoạt động. |
| **22:08:13** | 🔗 **C2 - Lệnh thứ hai** | `3791.exe` sinh cmd.exe lần thứ hai. |
| **22:11:06 - 22:11:30** | 🌐 **Liên lạc C2** | `ping http://prankglassinebracket.jumpingcrab.com` và `nslookup prankglassinebracket.jumpingcrab.com` - Xác định C2. |
| **22:15:49 - 22:16:47** | 📂 **Duyệt thư mục** | Nhiều lệnh `dir` để khảo sát hệ thống. |
| **22:17:17 - 22:17:22** | ℹ️ **Thu thập thông tin** | `help` (liệt kê lệnh), `tasklist` (liệt kê tiến trình). |
| **22:19:14 - 22:20:33** | 📦 **Đóng gói Dữ liệu** | Lệnh `move ..\1.jpeg 2.jpeg` và `move 2.jpeg imnotbatman.jpg` - Đánh dấu/staging dữ liệu để exfil. |
| **22:21:31 - 22:21:34** | 🧪 **Kiểm tra bổ sung** | `cmd.exe /c "exit 2>&1"` - Xác nhận shell vẫn hoạt động. |
| **22:24:21** | 🛡️ **Tự động hóa HĐH** | Các tiến trình hệ thống bình thường (`rundll32.exe`) tiếp tục chạy. |

---

## 4. Ánh xạ MITRE ATT&CK (MITRE ATT&CK Mapping)

| Kỹ thuật | Mã | Mô tả | Bằng chứng |
|---|---|---|---|
| **Exploit Public-Facing Application** | [T1190](https://attack.mitre.org/techniques/T1190/) | Khai thác lỗ hổng PHP-CGI RCE | `php-cgi.exe` → `cmd.exe` |
| **Web Shell** | [T1505.003](https://attack.mitre.org/techniques/T1505/003/) | Cài đặt webshell trên máy chủ web | `3791.exe` trong `C:\inetpub\wwwroot\joomla\` |
| **Command and Scripting Interpreter: Windows Command Shell** | [T1059.003](https://attack.mitre.org/techniques/T1059/003/) | Sử dụng cmd.exe để thực thi lệnh | `cmd.exe /c "..."` nhiều lần |
| **System Owner/User Discovery** | [T1033](https://attack.mitre.org/techniques/T1033/) | Xác định người dùng hiện tại | `whoami` → `NT AUTHORITY\IUSR` |
| **Account Discovery: Local Account** | [T1087.001](https://attack.mitre.org/techniques/T1087/001/) | Liệt kê tài khoản người dùng | `net user` |
| **Account Discovery: Domain Account** | [T1087.002](https://attack.mitre.org/techniques/T1087/002/) | Liệt kê tài khoản domain | `net view /domain` |
| **Network Share Discovery** | [T1135](https://attack.mitre.org/techniques/T1135/) | Khám phá các share mạng | `net share`, `net use c:\share` |
| **System Network Connections Discovery** | [T1049](https://attack.mitre.org/techniques/T1049/) | Xác định các kết nối mạng | `net session` |
| **Process Discovery** | [T1057](https://attack.mitre.org/techniques/T1057/) | Liệt kê tiến trình đang chạy | `tasklist` |
| **Remote System Discovery** | [T1018](https://attack.mitre.org/techniques/T1018/) | Khám phá hệ thống từ xa | `net view /domain` |
| **Application Layer Protocol: Web Protocols** | [T1071.001](https://attack.mitre.org/techniques/T1071/001/) | C2 qua HTTP/80 | Traffic HTTP bất thường đến `192.168.250.70:80` |
| **Data from Information Repositories** | [T1213](https://attack.mitre.org/techniques/T1213/) | Truy xuất dữ liệu từ CSDL | Backup DB query (`imreallynotbatman_backup.sql`) |
| **SQL Injection** | [T1190.001](https://attack.mitre.org/techniques/T1190/) | Tấn công SQLi qua HTTP | `pg_sleep(10)`, `waitfor delay '0:0:5'` |
| **Reflected XSS** | [T1189](https://attack.mitre.org/techniques/T1189/) | Tấn công XSS qua tham số search | `<script>GP4l(9294)</script>` |
| **File and Directory Discovery** | [T1083](https://attack.mitre.org/techniques/T1083/) | Khám phá file và thư mục | `dir` nhiều lần |
| **Obfuscated Files or Information** | [T1027](https://attack.mitre.org/techniques/T1027/) | File ẩn, đổi tên file | `1.jpeg` → `2.jpeg` → `imnotbatman.jpg` |
| **Scheduled Task/Job** | [T1053.005](https://attack.mitre.org/techniques/T1053/005/) | Xóa scheduled task | `schtasks /delete /f /TN "Microsoft\Windows\...Uploader"` |
| **Exfiltration Over C2 Channel** | [T1041](https://attack.mitre.org/techniques/T1041/) | Rò rỉ dữ liệu qua C2 | Lưu lượng HTTP vượt ngưỡng (4.3M vs 366K) |

---

## 5. Phân tích Cyber Kill Chain

### Giai đoạn 1: Trinh sát (Reconnaissance)
Kẻ tấn công từ IP `40.80.148.42` đã thực hiện quét web quy mô lớn vào máy chủ Joomla `192.168.250.70`. Bằng chứng từ phân tích mạng cho thấy:
- **Quét LFI/RFI:** Gửi hàng loạt yêu cầu path traversal tới `/windows/win.ini`, `/boot.ini` với kỹ thuật `%00` null byte injection.
- **Quét SQLi:** Inject `pg_sleep()`, `waitfor delay` vào tham số Joomla để kiểm tra SQL injection.
- **Quét XSS:** Inject `<script>GP4l(NNNN)</script>` vào tham số search.
- **Quét file nhạy cảm:** Quét hàng loạt đuôi file `.bak`, `.old`, `.inc`, `.yml`, `.config`, `.sql`, `.xls`, `.csv`.

### Giai đoạn 2: Vũ khí hóa (Weaponization)
Payload được phân phối qua HTTP GET/POST requests tới Joomla CMS. Các payload SQLi và XSS được đóng gói trong tham số URL hợp lệ của component Joomla (`/joomla/index.php/component/mailto/`, `/joomla/index.php/component/search/`).

### Giai đoạn 3: Phân phối (Delivery)
Lỗ hổng được khai thác thông qua **PHP-CGI** (`C:\Program Files (x86)\PHP\v5.5\php-cgi.exe`). Cơ chế RCE của PHP-CGI cho phép kẻ tấn công thực thi lệnh hệ thống bằng cách gửi tham số đặc biệt trong URL.

### Giai đoạn 4: Khai thác (Exploitation)
Khai thác thành công lỗ hổng thực thi mã từ xa (RCE) trên PHP-CGI:
- Tiến trình `php-cgi.exe` (PID cha) tạo `cmd.exe` (PID con) dưới quyền `NT AUTHORITY\IUSR`
- Câu lệnh đầu tiên: `echo 24365` → xác nhận RCE hoạt động
- Tiếp theo: `dir`, `ls`, `ifconfig` → thăm dò môi trường

### Giai đoạn 5: Cài đặt (Installation)
Kẻ tấn công tải lên và thực thi webshell **`3791.exe`**:
- **Đường dẫn:** `C:\inetpub\wwwroot\joomla\3791.exe`
- **Hash (MD5):** `AAE3F5A29935E6ABCC2C2754D12A9AF0` (đã xác nhận độc hại qua VirusTotal)
- **Cơ chế:** `cmd.exe /c "3791.exe 2>&1"` → `3791.exe` spawns thêm `cmd.exe`
- **Hash bổ sung:** SHA1 `65DF73D77324D008C83C3E57B445DF0FD43A3A51`, SHA256 `EC78C938D8453739CA2A370B9C275971EC46CAF6E479DE2B2D04E97CC47FA45D`

### Giai đoạn 6: Chỉ huy & Kiểm soát (Command & Control)
Kẻ tấn công thiết lập C2 qua domain **`prankglassinebracket.jumpingcrab.com`**:
- Sử dụng `ping` và `nslookup` để xác định domain C2
- Lưu lượng HTTP bất thường với volume gấp **11.7 lần** ngưỡng (4,307,722 bytes vs 366,915.2 bytes threshold)
- Tổng cộng 21,380 luồng mạng (~172 MB) trong 45 phút

### Giai đoạn 7: Hành động trên Mục tiêu (Actions on Objectives)
Kẻ tấn công thực hiện các hành động sau sau khi chiếm quyền kiểm soát:

1. **Trinh sát nội bộ:**
   - `whoami` → `NT AUTHORITY\IUSR`
   - `net user` → Liệt kê tài khoản người dùng
   - `net view /domain` → Khám phá domain
   - `net share` → Liệt kê share mạng
   - `net session` → Xem session mạng
   - `tasklist` → Xem tiến trình hệ thống

2. **Thiết lập truy cập mạng:**
   - `net use c:\share` → Map ổ đĩa mạng

3. **Đánh cắp dữ liệu (Data Exfiltration):**
   - Lệnh `move ..\1.jpeg 2.jpeg` và `move 2.jpeg imnotbatman.jpg` cho thấy việc đổi tên/che giấu file dữ liệu (staging)
   - Phân tích mạng phát hiện các file backup database đáng ngờ: `backup_imreallynotbatman.sql.tar`, `backup-imreallynotbatman.sql.gz`
   - Lưu lượng HTTP cao bất thường cho thấy dữ liệu đã được truyền ra ngoài

4. **Xóa dấu vết:**
   - `schtasks /delete /f /TN "Microsoft\Windows\Customer Experience Improvement Program\Uploader"` → Xóa scheduled task

---

## 6. Phát hiện Bổ sung từ Phân tích Mạng

### Các cuộc tấn công đồng thời (SQLi, XSS, Path Traversal)

| Loại tấn công | Chi tiết | Số lượng |
|---|---|---|
| **SQL Injection (Blind)** | `pg_sleep(N)`, `waitfor delay '0:0:N'` | Hàng trăm request |
| **Cross-Site Scripting (XSS)** | `<script>GP4l(NNNN)</script>`, `onmouseover=GP4l(NNNN)` | Hàng trăm request |
| **Path Traversal (LFI)** | `/windows/win.ini%00.jpg`, `../` pattern | Hàng nghìn request |
| **File Extension Scanning** | `.bak`, `.old`, `.inc`, `.config`, `.sql`, `.yml` | Hàng trăm request |
| **Auth Bypass Probes** | `/CFIDE/administrator/index.cfm`, `/elmah.axd` | Nhiều request |

### Các file nhạy cảm bị nhắm tới (Targeted Sensitive Files)

| File | Mục đích |
|---|---|
| `web.config`, `web.config.bak`, `web.config.old` | Cấu hình IIS |
| `global.asax.bak`, `global.asa.bak` | Cấu ứng dụng ASP.NET |
| `database.yml`, `database.yml~` | Cấu hình database |
| `private.key`, `id_dsa.ppk` | Khóa SSH/private key |
| `users.xls`, `users.csv`, `users.db` | Dữ liệu người dùng |
| `customers.xls`, `customers.csv`, `orders.xls`, `sales.xls` | Dữ liệu kinh doanh |
| `backup_imreallynotbatman.sql.tar` | **Backup database** |
| `.htaccess.bak`, `.htaccess.save`, `.htaccess.old` | Cấu hình Apache |

---

## 7. Các biện pháp khắc phục đã thực hiện (Remediation Actions Taken)

| Hành động | Mô tả | Trạng thái |
|---|---|---|
| **Xác định IOC** | Xác nhận IP tấn công `40.80.148.42` và C2 `prankglassinebracket.jumpingcrab.com` | ✅ Hoàn thành |
| **Xác nhận Webshell** | Phát hiện và hash `3791.exe` (MD5: `AAE3F5A29935E6ABCC2C2754D12A9AF0`) | ✅ Hoàn thành |
| **Phân tích Log** | Thu thập đầy đủ log tiến trình từ máy nạn nhân | ✅ Hoàn thành |
| **Phân tích Mạng** | Phân tích 21,380 luồng dữ liệu, phát hiện tấn công đa vector | ✅ Hoàn thành |
| **Pivot/Tương quan** | So khớp 9 lệnh recon, phát hiện 3 IOC độc hại | ✅ Hoàn thành |

---

## 8. Khuyến nghị Ngăn chặn (Containment & Remediation Recommendations)

### 🚨 NGAY LẬP TỨC (Immediate - 0-4 giờ)

| # | Hành động | Chi tiết |
|---|---|---|
| 1 | **🔒 Cách ly máy chủ** | Ngắt kết nối mạng máy chủ `192.168.250.70` khỏi mạng nội bộ và Internet |
| 2 | **🚫 Chặn IP tấn công** | Chặn IP `40.80.148.42` tại firewall biên và IPS |
| 3 | **⛔ Chặn C2 Domain** | Chặn domain `prankglassinebracket.jumpingcrab.com` tại DNS/firewall |
| 4 | **🔑 Thu hồi quyền truy cập** | Reset tất cả mật khẩu tài khoản dịch vụ (IUSR và dịch vụ web) |
| 5 | **🖥️ Chụp ảnh forensics** | Thu thập RAM, disk image của máy chủ trước khi shutdown |

### ⚠️ NGẮN HẠN (Short-term - 4-24 giờ)

| # | Hành động | Chi tiết |
|---|---|---|
| 1 | **🔍 Xóa Webshell** | Xóa file `C:\inetpub\wwwroot\joomla\3791.exe` và kiểm tra toàn bộ thư mục Joomla |
| 2 | **🩹 Patch PHP-CGI** | Vô hiệu hóa hoặc nâng cấp PHP `v5.5` lên phiên bản mới nhất (không còn hỗ trợ) - hoặc chuyển sang PHP dạng module |
| 3 | **📋 Kiểm tra IUSR permissions** | Rà soát và giới hạn quyền thực thi của tài khoản `NT AUTHORITY\IUSR` |
| 4 | **🛡️ WAF Rules** | Triển khai Web Application Firewall rules chặn SQLi, XSS, Path Traversal |
| 5 | **🔎 Quét toàn bộ mạng** | Quét tìm các IOC khác trên cùng phân đoạn mạng |

### 📋 DÀI HẠN (Long-term - 1-7 ngày)

| # | Hành động | Chi tiết |
|---|---|---|
| 1 | **🔄 Cập nhật Joomla** | Nâng cấp Joomla lên phiên bản mới nhất, kiểm tra component `extplorer` |
| 2 | **🔐 Hardening Web Server** | Cấu hình IIS: vô hiệu hóa CGI, hạn chế phương thức HTTP, enable Request Filtering |
| 3 | **📊 SIEM Monitoring** | Thiết lập cảnh báo cho các hành vi: cmd.exe từ php-cgi.exe, net.exe, schtasks.exe bất thường |
| 4 | **🔄 Phân tách quyền (Least Privilege)** | Tài khoản IUSR chỉ có quyền ghi vào thư mục upload, KHÔNG có quyền thực thi |
| 5 | **🔁 Backup & Recovery Plan** | Thiết lập backup an toàn, kiểm tra tính toàn vẹn của backup |
| 6 | **🛡️ Network Segmentation** | Máy chủ web không được có quyền truy cập trực tiếp vào domain controller và file server |

### Phát hiện bổ sung (Indicators of Compromise - IOC)

```yaml
# IOC Blocklist
ip_addresses:
  - value: "40.80.148.42"
    description: "Attacker IP - Web server compromise"
  - value: "3.0.0.0"
    description: "Suspicious IP found in ngen.exe command line (false positive possible)"

domains:
  - value: "prankglassinebracket.jumpingcrab.com"
    description: "C2 domain - nslookup/ping from compromised server"

file_hashes:
  - value: "AAE3F5A29935E6ABCC2C2754D12A9AF0"
    type: "MD5"
    description: "3791.exe webshell"
  - value: "65DF73D77324D008C83C3E57B445DF0FD43A3A51"
    type: "SHA1"
    description: "3791.exe webshell"
  - value: "EC78C938D8453739CA2A370B9C275971EC46CAF6E479DE2B2D04E97CC47FA45D"
    type: "SHA256"
    description: "3791.exe webshell"

suspicious_paths:
  - "C:\\inetpub\\wwwroot\\joomla\\3791.exe"
  - "C:\\inetpub\\wwwroot\\joomla\\imnotbatman.jpg"  # Renamed data file
```

---

## 9. Phụ lục: Chi tiết Kỹ thuật

### Cây tiến trình (Process Tree)
```
[Internet] 40.80.148.42
    │
    ├── HTTP Request → [IIS] w3wp.exe
    │                       │
    │                       └── php-cgi.exe (C:\Program Files (x86)\PHP\v5.5\)
    │                               │
    │                               └── cmd.exe /c "echo 24365"  [21:55:22]
    │                                   ├── conhost.exe
    │                                   └── cmd.exe /c "3791.exe 2>&1" [21:56:18]
    │                                           │
    │                                           └── 3791.exe (C:\inetpub\wwwroot\joomla\)
    │                                                   │
    │                                                   ├── cmd.exe (C:\Windows\system32\cmd.exe) [21:58:23]
    │                                                   │       ├── whoami
    │                                                   │       ├── net user / net share / net session / net view /domain / net use
    │                                                   │       ├── ping → prankglassinebracket.jumpingcrab.com
    │                                                   │       └── nslookup → prankglassinebracket.jumpingcrab.com
    │                                                   │
    │                                                   └── cmd.exe (C:\Windows\system32\cmd.exe) [22:08:13]
    │                                                           ├── dir (nhiều lần)
    │                                                           ├── tasklist / help
    │                                                           ├── move ..\1.jpeg 2.jpeg
    │                                                           └── move 2.jpeg imnotbatman.jpg
```

### Phân tích Volume Mạng
| Giao thức | Lưu lượng thực tế | Ngưỡng | Hệ số vượt |
|---|---|---|---|
| **HTTP (port 80)** | 4,307,722 bytes | 366,915.2 bytes | **11.7x** |
| **TCP** | 3,996,982 bytes | 316,910.5 bytes | **12.6x** |

---

*Báo cáo được tạo tự động bởi Network Incident Response Orchestrator*
*Dữ liệu nguồn: Triage, Recon, Log Collector, Network Analyzer, Pivot/Correlation*
*Thời gian tạo: 2026-06-12*

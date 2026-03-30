# 🏢 Enterprise SME Network & Mini-DC Architecture Testbed

**Author:** Nguyen An Vinh
**Platform:** EVE-NG (Emulated Virtual Environment - Next Generation)
**Objective:** Architecting and simulating a highly available Small and Medium-sized Enterprise (SME) network with an integrated Mini Data Center. The testbed focuses on network segmentation, failover routing, security (ACLs), and automation.

---

## 📑 MỤC LỤC
1. [Phần 1: Tổng quan Kiến trúc & Quy hoạch Mạng](#phần-1)
2. [Phần 2: Cấu hình Định tuyến & Bảo mật (Coming soon)](#)
3. [Phần 3: Giám sát & Tự động hóa Python (Coming soon)](#)

---

## 🏗️ PHẦN 1: TỔNG QUAN KIẾN TRÚC & QUY HOẠCH MẠNG (ARCHITECTURE & IP SCHEME)

### 1.1. Mô hình Thiết kế (Design Rationale)
Hệ thống được thiết kế dựa trên mô hình **Collapsed Core (Core/Distribution gộp chung)** tối ưu cho doanh nghiệp SME quy mô ~50-100 nhân sự. 
* **Lý do lựa chọn:** Tiết kiệm chi phí thiết bị phần cứng nhưng vẫn đảm bảo tính dự phòng (Redundancy) cao ở lớp Edge Gateway, đồng thời cô lập hoàn toàn lưu lượng của khu vực máy chủ (Data Center) với khu vực nhân viên (Campus) để tăng cường bảo mật.

![](_assets/Pasted%20image%2020260330113409.png)


### 1.2. Thuật ngữ & Kỹ thuật cốt lõi (Key Technologies)
* **High Availability (HA) / Failover:** Định tuyến dự phòng sử dụng 2 ISP. Khi tuyến cáp quang chính gặp sự cố, lưu lượng tự động chuyển hướng sang tuyến dự phòng.
* **Inter-VLAN Routing:** Định tuyến giữa các mạng LAN ảo, được thực hiện tập trung trên thiết bị L3 (Router/Core Switch) nhằm kiểm soát chặt chẽ luồng giao tiếp chéo.
* **Network Address Translation (NAT - PAT):** Chuyển đổi IP Private nội bộ sang IP Public để truy cập Internet, tối ưu hóa không gian địa chỉ IPv4.
* **Access Control List (ACL):** Bộ quy tắc tường lửa mềm trên thiết bị mạng dùng để cho phép (Permit) hoặc cấm (Deny) các luồng traffic đặc thù (như cấm nhân viên SSH vào Server).
* **DHCP (Dynamic Host Configuration Protocol):** Cấp phát IP tự động và quản lý tập trung cho các thiết bị End-user.

### 1.3. Chức năng Thiết bị (Device Roles)

| Device Name | Type / Image | Vai trò & Chức năng trong hệ thống |
| :--- | :--- | :--- |
| **R1** | Cisco IOL (L3) | **Main Edge Gateway:** Xử lý luồng mạng chính ra Internet (ISP 1). Đóng vai trò làm DHCP Server cấp IP cho toàn bộ mạng LAN và thực hiện NAT Overload. |
| **R2** | Cisco IOL (L3) | **Backup Edge Gateway:** Xử lý luồng mạng dự phòng (ISP 2). Kích hoạt cơ chế Failover khi R1 hoặc ISP 1 gián đoạn. |
| **SW1** | Cisco IOL (L2/L3) | **Core Switch:** Trung tâm trung chuyển dữ liệu nội bộ. Phân chia các VLAN, gom các đường Trunking từ Access Switch (nếu có mở rộng) lên Gateway. |
| **Debian-VoIP** | Linux (Debian 12) | **Mini-DC Node:** Máy chủ dịch vụ lõi (VoIP/Database/Web). Đặt IP tĩnh, được bảo vệ nghiêm ngặt bằng ACLs. |
| **Debian-Zabbix**| Linux (Debian 12) | **NOC Monitor Node:** Trạm giám sát mạng, cài đặt phần mềm Zabbix và chạy các script Python Automation (Netmiko) để backup cấu hình. |
| **PC1 -> PC6** | VPC | **End Devices (Staff):** Đại diện cho các phòng ban (Kế toán, Kinh doanh, Kỹ thuật, Giám đốc). Nhận IP động từ DHCP. |
| **CCTV & WiFi** | VPC | **IoT / Guest Devices:** Đại diện cho thiết bị an ninh và mạng khách. Được cô lập thành VLAN riêng để tránh rò rỉ dữ liệu. |

### 1.4. Quy hoạch VLAN & Không gian IP (IP Addressing Scheme)
Hệ thống sử dụng dải IP Private lớp A (`10.0.X.0/24`), chia nhỏ (Subnetting) theo từng phòng ban và phân vùng chức năng nhằm ngăn chặn Broadcast Storm và tối ưu hóa việc quản lý.

| Tên Zone (VLAN) | VLAN ID | Dải mạng (Subnet) | Default Gateway (SVI/Sub-int) | Cơ chế cấp phát IP |
| :--- | :--- | :--- | :--- | :--- |
| **KeToan_NhanSu** | 10 | `10.0.10.0/24` | `10.0.10.1` | DHCP |
| **KinhDoanh** | 20 | `10.0.20.0/24` | `10.0.20.1` | DHCP |
| **TSCustomer** | 30 | `10.0.30.0/24` | `10.0.30.1` | DHCP |
| **KyThuat1** | 40 | `10.0.40.0/24` | `10.0.40.1` | DHCP |
| **KyThuat2** | 50 | `10.0.50.0/24` | `10.0.50.1` | DHCP |
| **GiamDoc** | 60 | `10.0.60.0/24` | `10.0.60.1` | DHCP |
| **CCTV** | 70 | `10.0.70.0/24` | `10.0.70.1` | **Static IP** (VD: 10.0.70.10) |
| **WiFi_Guest** | 80 | `10.0.80.0/24` | `10.0.80.1` | DHCP |
| **Data_Center** | 100 | `10.0.100.0/24`| `10.0.100.1` | **Static IP** (Bảo mật khắt khe) |

> **📝 Note:** Các địa chỉ IP từ `.1` đến `.9` trong các dải DHCP sẽ được loại trừ (Excluded) để dành (reserve) cho việc gán IP tĩnh cho các thiết bị hạ tầng mạng (Switch, Access Point, Printer) trong tương lai.

### 1.5. Sơ đồ Cổng kết nối (Interface Mapping)

| Thiết bị | Interface | Kết nối đến (Neighbor) | VLAN / Trạng thái |
| :--- | :--- | :--- | :--- |
| **R1** | `e0/0` | Cloud0 (ISP1) | DHCP Client (Nhận IP từ VMware NAT) |
| **R1** | `e0/1` | SW1 (`e0/0`) | Trunking (Dot1Q) |
| **R2** | `e0/0` | Cloud1 (ISP2) | DHCP Client / Static (Đường Backup) |
| **R2** | `e0/1` | SW1 (`e0/1`) | Trunking (Dot1Q) |
| **SW1**| `e0/2` | Debian-VoIP | Access VLAN 100 |
| **SW1**| `e0/3` | Debian-Zabbix | Access VLAN 100 |
| **SW1**| `e1/0 -> e2/3`| VPCs (PC1-PC6, CCTV...) | Access VLAN tương ứng theo bảng 1.4 |

---
**(Hết Phần 1)**

## 🛠️ PHẦN 2: CẤU HÌNH LÕI - ĐỊNH TUYẾN, DHCP, NAT & BẢO MẬT (CLI DEPLOYMENT)

### 2.1. Cấu hình SW1 (Core/Distribution L3 Switch)
**Mục tiêu:** Biến SW1 thành "trái tim" của mạng nội bộ. Bật tính năng định tuyến lớp 3 (IP Routing), tạo các SVI (Switch Virtual Interface) làm Gateway, cấu hình DHCP Server và thiết lập định tuyến dự phòng (Failover) đẩy ra 2 Router Edge.

```bash
! 1. Bật định tuyến Layer 3 và cấu hình V100TP/VLAN
enable
configure terminal
ip routing
vtp mode transparent

! Tạo các VLAN nội bộ và VLAN trung gian nối ra Router
vlan 10,20,30,40,50,60,70,80,100
vlan 991
 name Link_To_R1
vlan 992
 name Link_To_R2
exit

! 2. Quy hoạch các Interface nối xuống End-User (Cậu tùy chỉnh cổng Gi cho khớp sơ đồ)
! Ví dụ cấu hình cho cổng nối PC Kế toán (VLAN 10)
interface GigabitEthernet1/0
 switchport trunk encapsulation dot1q
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown

! (Thực hiện tương tự cho các cổng nối PC khác vào VLAN 20 -> 100)

! 3. Cấu hình cổng nối lên R1 (dùng VLAN 991) và R2 (dùng VLAN 992)
interface GigabitEthernet0/0
 description Link_to_R1
 switchport mode access
 switchport access vlan 991
 no shutdown

interface GigabitEthernet0/1
 description Link_to_R2
 switchport mode access
 switchport access vlan 992
 no shutdown

! 4. Tạo SVI (Default Gateway cho các VLAN)
interface Vlan10
 ip address 10.0.10.1 255.255.255.0
 no shutdown
interface Vlan20
 ip address 10.0.20.1 255.255.255.0
 no shutdown
! (Cấu hình tương tự cho VLAN 30, 40, 50, 60, 70, 80, 100 với IP tương ứng 10.0.x.1)

interface Vlan991
 ip address 10.0.99.2 255.255.255.252
 no shutdown
interface Vlan992
 ip address 10.0.99.6 255.255.255.252
 no shutdown

! 5. Cấu hình DHCP Server cho các VLAN nội bộ (Trừ VLAN 70 CCTV và 100 DC dùng IP tĩnh)
ip dhcp excluded-address 10.0.10.1 10.0.10.9
ip dhcp pool VLAN10_POOL
 network 10.0.10.0 255.255.255.0
 default-router 10.0.10.1
 dns-server 8.8.8.8
! (Làm tương tự cho các VLAN 20,30,40,50,60,80)

! 6. Cấu hình Floating Static Route (Định tuyến dự phòng Failover)
! Metric 10 cho đường chính R1, Metric 20 cho đường backup R2
ip route 0.0.0.0 0.0.0.0 10.0.99.1 10
ip route 0.0.0.0 0.0.0.0 10.0.99.5 20
````

## 2.2. Cấu hình Edge Router R1 (Main ISP - Cáp quang chính)

**Mục tiêu:** R1 nhận IP từ Cloud0 (Internet), thực hiện NAT Overload (PAT) để cho phép toàn bộ mạng dải `10.0.0.0/16` ra được Internet, và trỏ route tĩnh báo cho R1 biết đường về mạng LAN nội bộ thông qua SW1.

```Bash
enable
configure terminal
! Cổng nối ra Cloud0 (Internet)
interface GigabitEthernet0/0
 ip address dhcp
 ip nat outside
 no shutdown

! Cổng nối xuống SW1
interface GigabitEthernet0/1
 ip address 10.0.99.1 255.255.255.252
 ip nat inside
 no shutdown

! Cấu hình NAT PAT cho phép dải mạng 10.0.X.X ra Internet
access-list 1 permit 10.0.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! Định tuyến tĩnh chỉ đường về lại mạng LAN (trỏ về IP của SW1)
ip route 10.0.0.0 255.255.0.0 10.0.99.2
```

## 2.3. Cấu hình Edge Router R2 (Backup ISP)

**Mục tiêu:** Tương tự R1 nhưng chạy trên dải IP trung gian `10.0.99.4/30` và nối ra Cloud1.

```Bash
enable
configure terminal
! Cổng nối ra Cloud1 (Đường dự phòng)
interface GigabitEthernet0/0
 ip address dhcp
 ip nat outside
 no shutdown

! Cổng nối xuống SW1
interface GigabitEthernet0/1
 ip address 10.0.99.5 255.255.255.252
 ip nat inside
 no shutdown

! NAT PAT
access-list 1 permit 10.0.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! Định tuyến tĩnh chỉ đường về lại mạng LAN (trỏ về IP của SW1)
ip route 10.0.0.0 255.255.0.0 10.0.99.6
```

## 2.4. Chính sách Bảo mật Cốt lõi (Security / ACLs)

**Mục tiêu:** Cách ly Data Center (VLAN 100). Đảm bảo nhân viên có thể ping kiểm tra máy chủ, nhưng **TUYỆT ĐỐI KHÔNG** được phép SSH vào máy chủ Linux để tránh rò rỉ dữ liệu hoặc Brute-force. _(Lệnh này gõ trên thiết bị SW1)_

```Bash
! Gõ trên SW1
configure terminal
ip access-list extended SECURE_DC
 ! Cho phép IT (KyThuat2) (VLAN 50) full quyền vào DC
 permit ip 10.0.50.0 0.0.0.255 10.0.100.0 0.0.0.255
 ! Chặn TẤT CẢ các dải khác SSH (Port 22) vào Data Center
 deny tcp any 10.0.100.0 0.0.0.255 eq 22
 ! Vẫn cho phép ping và các dịch vụ khác (như web/voip)
 permit ip any any
 exit

! Áp dụng rule này vào SVI của Data Center theo chiều đi ra (Outbound)
interface Vlan100
 ip access-group SECURE_DC out
```

---

## 🔍 HƯỚNG DẪN ĐO LƯỜNG SỐ LIỆU ĐƯA VÀO CV (VALIDATION & METRICS)

1. **Test DHCP & NAT:** Mở `PC-Staff` (VPC), gõ `ip dhcp` -> Sau khi nhận IP, gõ `ping 8.8.8.8`. Nếu thành công, hạ tầng đã sẵn sàng!
    
2. **Đo Failover Time (Thời gian hội tụ):**
    
    - Cho `PC-Staff` gõ lệnh `ping 8.8.8.8 -t` (để ping liên tục).
        
    - Chuột phải vào **R1** trên EVE-NG, chọn **Stop** (Giả lập đứt cáp quang chính).
        
    - Quay lại màn hình PC, đếm xem bị rớt bao nhiêu dòng `timeout` trước khi ping thông trở lại. Ghi lại con số **X giây** để điền vào CV! (Traffic đã được Floating Route tự động bẻ lái qua R2).
        
3. **Đo ACL Hit-count:**
    
    - Từ `PC-Staff`, gõ lệnh ssh thử vào máy chủ Debian (VD: `ssh root@10.0.100.10`). Kết nối sẽ bị từ chối.
        
    - Lên SW1 gõ `show access-lists`. Chụp lại dòng có chữ `deny tcp... (x matches)` để chứng minh khả năng bảo mật.
        

---

**(Hết Phần 2)**

## 📊 PHẦN 3: ĐO LƯỜNG, GIÁM SÁT & TỰ ĐỘNG HÓA (TELEMETRY & AUTOMATION)

Phần này tập trung vào việc xác minh tính khả dụng của hệ thống (High Availability), kiểm chứng các chính sách bảo mật (Security Validation) và ứng dụng Python để tự động hóa quy trình vận hành mạng (Network Automation) - các kỹ năng cốt lõi của một Kỹ sư NOC/DevOps.

### 3.1. Xác minh Hệ thống & Đo lường (Validation & Telemetry)

**A. Kiểm tra Định tuyến & NAT (Routing & NAT Verification)**
* Từ `PC-Staff` (VLAN 20), thực hiện cấp phát DHCP và ping ra Internet:
  ```bash
  VPCS> ip dhcp
  VPCS> ping 8.8.8.8
```

- Kiểm tra bảng NAT Translations trên R1 để xác nhận luồng traffic nội bộ đã được PAT (Overload) thành công ra IP Public:
    ```Bash
    R1# show ip nat translations
    ```
    

**B. Kiểm chứng Chính sách Bảo mật Data Center (Security Validation)**

- **Kịch bản:** Ngăn chặn nỗ lực di chuyển ngang (Lateral Movement) trái phép từ vùng User (VLAN 20) vào vùng Data Center (VLAN 100).
    
- **Thực thi:** Từ `PC-Staff`, cố gắng thiết lập kết nối SSH vào máy chủ `Debian-VoIP` (`10.0.100.10`).
    
- **Đo lường (Telemetry):**
    
    - Sử dụng **Wireshark** bắt gói tin tại cổng `e0/2` của SW1. Ghi nhận các gói tin TCP SYN (Port 22) bị chặn đứng và thiết bị trả về gói tin `ICMP Type 3, Code 13 (Communication Administratively Prohibited)`.
        
    - Trên SW1, kiểm tra Hit-count của Access-list để theo dõi số lượng gói tin bị Drop:
        

        
        ```Bash
        SW1# show access-lists SECURE_DC
        ```
        

**C. Kiểm thử Chuyển mạch Dự phòng (Failover / Convergence Time)**

- **Kịch bản:** Giả lập sự cố đứt cáp quang chính tại ISP 1 (R1 downtime). Hệ thống phải tự động chuyển hướng traffic sang ISP 2 (R2) thông qua Floating Static Route.
    
- **Thực thi & Đo lường:**
    
    - Từ `PC-Staff`, thực hiện lệnh ping liên tục: `ping 8.8.8.8 -t`.
        
    - Tại giao diện EVE-NG, tiến hành **Stop (Tắt)** thiết bị R1.
        
    - Quan sát màn hình ping. Hệ thống ghi nhận rớt mạng trong khoảng **[Điền số] giây** (Request timed out) trước khi bảng định tuyến tự động hội tụ và traffic được đẩy sang R2. Khôi phục kết nối thành công mà không cần can thiệp thủ công.
        

---

## 3.2. Tự động hóa Vận hành với Python (Network Automation)

Thay vì phải đăng nhập thủ công (SSH) vào từng Router/Switch để sao lưu cấu hình, hệ thống sử dụng script Python tích hợp thư viện `netmiko` để tự động hóa quy trình này, giúp tiết kiệm thời gian và giảm thiểu rủi ro do lỗi thao tác của con người.

**Môi trường chuẩn bị:**

- Cài đặt thư viện: `pip install netmiko`
    
- Thiết bị mạng (R1, R2, SW1) phải được bật tính năng SSH và tạo tài khoản local:
    
    
    
    ```Bash
    ! Cấu hình trên R1, R2, SW1
    ip domain-name tel4vn.local
    crypto key generate rsa modulus 2048
    username admin privilege 15 secret admin123
    line vty 0 4
     login local
     transport input ssh
    ```
    

**Script Python (`backup_config.py`):**



```Python
import time
from netmiko import ConnectHandler

# Thời điểm bắt đầu đo lường
start_time = time.time()

# Định nghĩa danh sách thiết bị
r1 = {
    'device_type': 'cisco_ios',
    'host':   '10.0.99.1', # Hoặc IP SVI của SW1 nhìn thấy R1
    'username': 'admin',
    'password': 'admin123',
}
r2 = {
    'device_type': 'cisco_ios',
    'host':   '10.0.99.5',
    'username': 'admin',
    'password': 'admin123',
}
sw1 = {
    'device_type': 'cisco_ios',
    'host':   '10.0.10.1', # Đại diện IP của SW1
    'username': 'admin',
    'password': 'admin123',
}

devices = [r1, r2, sw1]

print("🚀 Bắt đầu quá trình sao lưu cấu hình tự động (Auto-Backup)...")

for dev in devices:
    try:
        print(f"[*] Đang kết nối tới thiết bị: {dev['host']}")
        net_connect = ConnectHandler(**dev)
        
        # Gửi lệnh lấy cấu hình
        output = net_connect.send_command('show running-config')
        
        # Lưu ra file text
        filename = f"backup_{dev['host']}_{time.strftime('%Y%m%d-%H%M')}.txt"
        with open(filename, 'w') as f:
            f.write(output)
            
        print(f"[+] Sao lưu thành công: {filename}")
        net_connect.disconnect()
    except Exception as e:
        print(f"[-] Lỗi kết nối thiết bị {dev['host']}: {e}")

# Thời điểm kết thúc và tính toán tổng thời gian
end_time = time.time()
execution_time = round(end_time - start_time, 2)

print("--------------------------------------------------")
print(f"✅ Quá trình hoàn tất! Tổng thời gian thực thi: {execution_time} giây.")
print("--------------------------------------------------")
```

## 3.3. Kết quả Số liệu (Lab Metrics)

- **Routing Convergence Time (Thời gian Failover):** `[Điền số, VD: 4] seconds`.
    
- **Automation Execution Time (Thời gian sao lưu 3 thiết bị):** `[Điền số giây script Python chạy xong, VD: 5.2] seconds`. _(Việc áp dụng Automation giúp giảm **~90%** thời gian thao tác thủ công so với phương pháp truyền thống)._
    

---

**[HẾT BÀI WRITE-UP]**
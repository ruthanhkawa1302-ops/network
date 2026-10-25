# Lab 1: FortiGate ⇄ FortiGate Site-to-Site IPsec VPN (Route-based)

## 1. Mục tiêu
Dựng VPN IPsec **route-based** (IKEv2) giữa 2 FortiGate qua WAN, để các VLAN hai site liên lạc được với nhau.

## 2. Topology
```
 VLAN10 192.168.10.0/24                                  VLAN30 192.168.30.0/24
 VLAN20 192.168.20.0/24                                  VLAN40 192.168.40.0/24
        │                                                        │
   [ FGT1 ]  port1 192.168.119.11 ═══ IPsec tunnel ═══ 192.168.119.12 port1 [ FGT2 ]
   port2 (trunk VLAN 10,20)                                       port2 (trunk VLAN 30,40)
   .1 mỗi VLAN                                                    .1 mỗi VLAN
```

| | FGT1 | FGT2 |
|---|---|---|
| WAN (port1) | 192.168.119.11/24 | 192.168.119.12/24 |
| LAN (port2, VLAN) | vlan10 192.168.10.1/24, vlan20 192.168.20.1/24 | vlan30 192.168.30.1/24, vlan40 192.168.40.1/24 |
| Tunnel interface | `to-FGT2` | `to-FGT1` |
| Tunnel IP (tuỳ chọn) | 10.255.255.1 ↔ 10.255.255.2 | 10.255.255.2 ↔ 10.255.255.1 |

> Tên port (`port1`, `port2`) là giả định — đổi cho khớp model của bạn (`get system interface physical`). PSK trong lab: `Str0ngKey-Example` (đổi khi dùng thật).

## 3. Logic cần nhớ (route-based trên FortiGate)
Cần **4 thành phần** trên mỗi FGT:
1. **Phase1-interface** (IKE) → tạo ra interface tunnel.
2. **Phase2-interface** (IPsec SA; selector mặc định `0.0.0.0/0 ↔ 0.0.0.0/0`).
3. **Static route** trỏ subnet đầu xa vào interface tunnel.
4. **Firewall policy** hai chiều giữa VLAN ↔ tunnel (FortiGate **chặn mặc định**, thiếu policy là không đi được).

## 4. Các bước thực hiện

### Bước 0 – Chuẩn bị
- Hai FGT ping được nhau qua WAN: `execute ping 192.168.119.12` (từ FGT1).
- Port LAN nối switch ở chế độ **trunk**, cho phép VLAN 10, 20 (FGT1) và 30, 40 (FGT2).
- Nếu có NAT/firewall giữa hai FGT: mở UDP 500, UDP 4500, ESP (proto 50).

### Bước 1 – Cấu hình interface (VLAN + WAN)
**GUI:** Network > Interfaces > Create New > Interface (Type VLAN).

**FGT1**
```
config system interface
    edit "port1"
        set mode static
        set ip 192.168.119.11 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "vlan10"
        set vdom "root"
        set interface "port2"
        set vlanid 10
        set ip 192.168.10.1 255.255.255.0
        set allowaccess ping
        set role lan
    next
    edit "vlan20"
        set vdom "root"
        set interface "port2"
        set vlanid 20
        set ip 192.168.20.1 255.255.255.0
        set allowaccess ping
        set role lan
    next
end
```
**FGT2** (tương tự)
```
config system interface
    edit "port1"
        set mode static
        set ip 192.168.119.12 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "vlan30"
        set vdom "root"
        set interface "port2"
        set vlanid 30
        set ip 192.168.30.1 255.255.255.0
        set allowaccess ping
        set role lan
    next
    edit "vlan40"
        set vdom "root"
        set interface "port2"
        set vlanid 40
        set ip 192.168.40.1 255.255.255.0
        set allowaccess ping
        set role lan
    next
end
```

### Bước 2 – Phase 1 (IKE)
**GUI:** VPN > IPsec Tunnels > Create New > Custom (đặt tên, Remote Gateway = Static IP, chọn Interface, IKEv2, PSK; Proposal AES256-SHA256, DH group 14).

**FGT1**
```
config vpn ipsec phase1-interface
    edit "to-FGT2"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal aes256-sha256
        set dhgrp 14
        set remote-gw 192.168.119.12
        set psksecret Str0ngKey-Example
        set dpd on-idle
        set dpd-retrycount 3
        set dpd-retryinterval 10
    next
end
```
**FGT2**
```
config vpn ipsec phase1-interface
    edit "to-FGT1"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal aes256-sha256
        set dhgrp 14
        set remote-gw 192.168.119.11
        set psksecret Str0ngKey-Example
        set dpd on-idle
        set dpd-retrycount 3
        set dpd-retryinterval 10
    next
end
```
> Hai đầu **phải khớp**: IKE version, proposal, DH group, PSK. `net-device disable` = một interface tunnel dùng cho cả tunnel (kiểu chuẩn cho site-to-site cố định).

### Bước 3 – Phase 2 (IPsec SA)
**FGT1**
```
config vpn ipsec phase2-interface
    edit "to-FGT2"
        set phase1name "to-FGT2"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set auto-negotiate enable
    next
end
```
**FGT2**
```
config vpn ipsec phase2-interface
    edit "to-FGT1"
        set phase1name "to-FGT1"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set auto-negotiate enable
    next
end
```
Selector không đặt → mặc định `0.0.0.0/0` hai phía (đúng bản chất route-based). `auto-negotiate enable` giúp tunnel tự dựng không cần chờ traffic.

### Bước 4 – (Tuỳ chọn) IP cho interface tunnel
Giúp ping/monitor/routing động tiện hơn.

**FGT1**
```
config system interface
    edit "to-FGT2"
        set ip 10.255.255.1 255.255.255.255
        set remote-ip 10.255.255.2 255.255.255.255
        set allowaccess ping
    next
end
```
**FGT2**
```
config system interface
    edit "to-FGT1"
        set ip 10.255.255.2 255.255.255.255
        set remote-ip 10.255.255.1 255.255.255.255
        set allowaccess ping
    next
end
```

### Bước 5 – Address object
**FGT1**
```
config firewall address
    edit "LAN-10"
        set subnet 192.168.10.0 255.255.255.0
    next
    edit "LAN-20"
        set subnet 192.168.20.0 255.255.255.0
    next
    edit "REMOTE-30"
        set subnet 192.168.30.0 255.255.255.0
    next
    edit "REMOTE-40"
        set subnet 192.168.40.0 255.255.255.0
    next
end
config firewall addrgrp
    edit "FGT1-LANS"
        set member "LAN-10" "LAN-20"
    next
    edit "FGT2-LANS"
        set member "REMOTE-30" "REMOTE-40"
    next
end
```
**FGT2**
```
config firewall address
    edit "LAN-30"
        set subnet 192.168.30.0 255.255.255.0
    next
    edit "LAN-40"
        set subnet 192.168.40.0 255.255.255.0
    next
    edit "REMOTE-10"
        set subnet 192.168.10.0 255.255.255.0
    next
    edit "REMOTE-20"
        set subnet 192.168.20.0 255.255.255.0
    next
end
config firewall addrgrp
    edit "FGT2-LANS"
        set member "LAN-30" "LAN-40"
    next
    edit "FGT1-LANS"
        set member "REMOTE-10" "REMOTE-20"
    next
end
```

### Bước 6 – Static route (+ blackhole)
**FGT1**
```
config router static
    edit 10
        set dst 192.168.30.0 255.255.255.0
        set device "to-FGT2"
    next
    edit 11
        set dst 192.168.40.0 255.255.255.0
        set device "to-FGT2"
    next
    edit 12
        set dst 192.168.30.0 255.255.255.0
        set blackhole enable
        set distance 254
    next
    edit 13
        set dst 192.168.40.0 255.255.255.0
        set blackhole enable
        set distance 254
    next
end
```
**FGT2**
```
config router static
    edit 10
        set dst 192.168.10.0 255.255.255.0
        set device "to-FGT1"
    next
    edit 11
        set dst 192.168.20.0 255.255.255.0
        set device "to-FGT1"
    next
    edit 12
        set dst 192.168.10.0 255.255.255.0
        set blackhole enable
        set distance 254
    next
    edit 13
        set dst 192.168.20.0 255.255.255.0
        set blackhole enable
        set distance 254
    next
end
```
> Route blackhole (distance 254) là dự phòng: khi tunnel down, traffic bị drop thay vì thoát ra Internet dạng plaintext qua default route.

### Bước 7 – Firewall policy (hai chiều)
Cần dùng ID chưa tồn tại (kiểm tra `show firewall policy`). NAT **disable** (không NAT traffic VPN).

**FGT1**
```
config firewall policy
    edit 101
        set name "LAN-to-FGT2"
        set srcintf "vlan10" "vlan20"
        set dstintf "to-FGT2"
        set srcaddr "FGT1-LANS"
        set dstaddr "FGT2-LANS"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
    edit 102
        set name "FGT2-to-LAN"
        set srcintf "to-FGT2"
        set dstintf "vlan10" "vlan20"
        set srcaddr "FGT2-LANS"
        set dstaddr "FGT1-LANS"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
end
```
**FGT2**
```
config firewall policy
    edit 101
        set name "LAN-to-FGT1"
        set srcintf "vlan30" "vlan40"
        set dstintf "to-FGT1"
        set srcaddr "FGT2-LANS"
        set dstaddr "FGT1-LANS"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
    edit 102
        set name "FGT1-to-LAN"
        set srcintf "to-FGT1"
        set dstintf "vlan30" "vlan40"
        set srcaddr "FGT1-LANS"
        set dstaddr "FGT2-LANS"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
end
```
> Policy `ALL` để lab chạy trước; môi trường thật nên giới hạn service/UTM.

### Bước 8 – Kiểm tra
Đặt PC mỗi VLAN với gateway `.1` tương ứng (ví dụ PC1 192.168.10.10, PC3 192.168.30.10).

1. **Trạng thái tunnel**
```
get vpn ipsec tunnel summary
diagnose vpn ike gateway list
diagnose vpn tunnel list name to-FGT2
get router info routing-table static
```
   Kỳ vọng: tunnel `up`, `selectors(total,up): 1/1`, route 192.168.30.0/24 & 192.168.40.0/24 → `to-FGT2`.
   GUI: Dashboard > Network > IPsec hoặc Monitor > IPsec Monitor.
2. **Ping qua tunnel**: PC VLAN10 → 192.168.30.10 và 192.168.40.10; PC VLAN20 → cả hai; chiều ngược lại.
   Hoặc từ FGT1: `execute ping-options source 192.168.10.1` rồi `execute ping 192.168.30.1`.
3. **Counter** `diagnose vpn tunnel list name to-FGT2` → `npkts` / `enc:` / `dec:` tăng ở cả hai đầu.
4. **Log**: Log & Report > Forward Traffic, lọc theo policy 101/102.

### Bước 9 – Troubleshoot
| Triệu chứng | Kiểm tra / cách xử lý |
|---|---|
| Phase1 không lên | `execute ping` WAN; mở UDP 500/4500; PSK, IKE version, proposal, DH khớp? |
| Debug IKE | `diagnose vpn ike log filter rem-addr4 192.168.119.12` → `diagnose debug application ike -1` → `diagnose debug enable` ; tắt: `diagnose debug disable` + `diagnose debug reset` |
| Phase1 up, phase2 không | Proposal/PFS/DH phase2 lệch; selector không còn `0.0.0.0/0` ở một đầu |
| Tunnel up, ping không qua | Thiếu route / policy; sai chiều policy; nat bật nhầm |
| Phân tích gói | `diagnose debug flow filter addr 192.168.30.10` → `diagnose debug flow trace start 10` → `diagnose debug enable` |
| Bắt gói IKE | `diagnose sniffer packet any 'host 192.168.119.12 and (udp port 500 or udp port 4500)' 4` |
| Ping nhỏ OK, file lớn treo | MTU/MSS: giảm MTU tunnel hoặc dùng `tcp-mss-sender/receiver` trên policy |

## 5. Cấu hình bằng GUI
Đăng nhập `https://192.168.119.11` (FGT1) và `https://192.168.119.12` (FGT2). Làm lần lượt trên **từng** FGT, thay giá trị theo bảng ở mục 2. Tên menu dựa trên FortiOS 7.x, bản khác có thể lệch nhẹ.

### G1 – Interface
**Network > Interfaces**
- Sửa `port1`: Role = WAN, Addressing mode = Manual, IP/Netmask (`192.168.119.11/24` hoặc `.12/24`), Administrative Access = PING + HTTPS + SSH → OK.
- **Create New > Interface** cho từng VLAN:
  - Name: `vlan10`, Type: **VLAN**, Interface: `port2`, VLAN ID: `10`
  - Role: LAN, Addressing mode: Manual, IP/Netmask: `192.168.10.1/255.255.255.0`
  - Administrative Access: PING → OK
  - Lặp lại cho `vlan20` (FGT1) hoặc `vlan30`, `vlan40` (FGT2).

### G2 – Address object
**Policy & Objects > Addresses > Create New > Address**
- Name `LAN-10`, Type Subnet, Subnet `192.168.10.0/24`, Interface `any` → OK. Tương tự cho các subnet còn lại (4 object mỗi FGT: 2 local + 2 remote).
- **Create New > Address Group**: `FGT1-LANS` (gồm các subnet ở FGT1) và `FGT2-LANS` (các subnet ở FGT2). Trên mỗi FGT cần cả hai group.

### G3 – Tạo tunnel (Phase 1 + Phase 2)
**VPN > IPsec Tunnels > Create New > IPsec Tunnel**
1. **VPN Setup:** Name `to-FGT2` (FGT2: `to-FGT1`), Template type = **Custom** → Next.
2. **Network:**
   - IP version: IPv4, Remote Gateway: **Static IP Address**
   - IP Address: `192.168.119.12` (FGT2: `192.168.119.11`)
   - Interface: `port1`
   - NAT traversal: Enable; Dead Peer Detection: **On Idle** (Retry count 3, interval 10)
3. **Authentication:** Method = **Pre-shared Key**, nhập `Str0ngKey-Example`, IKE Version = **2**, Peer options = Accept any peer ID.
4. **Phase 1 Proposal:** Encryption **AES256**, Authentication **SHA256**; Diffie-Hellman Groups: chỉ tick **14** (bỏ các nhóm khác); Key Lifetime 86400.
5. **Phase 2 Selectors:** Name `to-FGT2`; Local Address / Remote Address = `0.0.0.0/0.0.0.0` (route-based nên để mặc định).
   - Mở **Advanced**: Encryption **AES256** + Authentication **SHA256**; bật **Perfect Forward Secrecy** với DH **14**; bật **Auto-negotiate**; Key Lifetime 43200.
6. **OK**. Interface tunnel `to-FGT2` được tạo, nằm lồng dưới `port1` trong Network > Interfaces.

> Cách nhanh: dùng **IPsec Wizard** (Template: Site to Site → Remote device type FortiGate). Wizard tự tạo luôn address, route và policy, nhưng thiết lập mặc định (IKE version, proposal, DH, selector) khác lab này, nên sau đó phải vào chỉnh lại tunnel và xoá phần thừa. Làm tay (Custom) dễ kiểm soát hơn.

### G4 – (Tuỳ chọn) IP cho interface tunnel
**Network > Interfaces** → mở rộng `port1` → sửa `to-FGT2`:
- IP: `10.255.255.1` (FGT2: `10.255.255.2`), Remote IP/Netmask: `10.255.255.2/255.255.255.255` (FGT2: `10.255.255.1`)
- Administrative Access: PING → OK.

### G5 – Static route
**Network > Static Routes > Create New**
- Destination: Subnet `192.168.30.0/255.255.255.0`, Interface: `to-FGT2`, Administrative Distance `10` → OK. Lặp lại cho `192.168.40.0/24`.
- Route blackhole (dự phòng khi tunnel down): Destination `192.168.30.0/24`, Interface = **Blackhole**, Administrative Distance `254` → OK; lặp lại cho `192.168.40.0/24`. Nếu GUI không hiển thị lựa chọn Blackhole ở bản của bạn, cấu hình phần này bằng CLI (bước 6 mục 4).
- FGT2: đảo lại, route `192.168.10.0/24`, `192.168.20.0/24` qua `to-FGT1`.

### G6 – Firewall policy
**Policy & Objects > Firewall Policy > Create New** (tạo 2 policy mỗi FGT)

| Trường | Policy outbound (FGT1) | Policy inbound (FGT1) |
|---|---|---|
| Name | `LAN-to-FGT2` | `FGT2-to-LAN` |
| Incoming Interface | `vlan10`, `vlan20` | `to-FGT2` |
| Outgoing Interface | `to-FGT2` | `vlan10`, `vlan20` |
| Source | `FGT1-LANS` | `FGT2-LANS` |
| Destination | `FGT2-LANS` | `FGT1-LANS` |
| Schedule / Service | always / ALL | always / ALL |
| Action | ACCEPT | ACCEPT |
| NAT | **Tắt** | **Tắt** |
| Log Allowed Traffic | All Sessions | All Sessions |

FGT2 làm tương tự: `vlan30`, `vlan40` ⇄ `to-FGT1`, source/destination đảo theo mục 4 – Bước 7.

### G7 – Kiểm tra bằng GUI
- **VPN > IPsec Tunnels** hoặc **Dashboard > Network > IPsec**: trạng thái tunnel phải **Up** (xanh). Nếu chưa lên: chuột phải vào tunnel > **Bring Up**.
- **Network > Static Routes** hoặc **Dashboard > Network > Routing**: route đến subnet đầu xa trỏ vào tunnel.
- **Log & Report > Forward Traffic**: lọc theo policy, thấy phiên `accept` khi ping giữa các VLAN.
- Dùng CLI Console (biểu tượng `>_` góc trên phải) chạy `get vpn ipsec tunnel summary`, `diagnose vpn tunnel list` khi cần xem sâu hơn.

## 6. Kết quả mong đợi
- `get vpn ipsec tunnel summary` hiển thị `to-FGT2: ... selectors(total,up): 1/1`.
- VLAN10/20 ⇄ VLAN30/40 ping được hai chiều.
- Tắt WAN của một đầu → ping fail (blackhole route chặn leak), bật lại → tunnel tự lên.

## 7. Mở rộng
- Thêm WAN thứ hai + tunnel thứ hai, chạy **BGP/OSPF** qua interface tunnel để failover.
- Đổi PSK sang certificate.
- Thêm VLAN mới chỉ cần: address object + route + cập nhật policy (không đụng IPsec).

> Cú pháp CLI có thể khác chút giữa các bản FortiOS (6.4 / 7.0 / 7.2 / 7.4 / 7.6). Dùng `?` hoặc tài liệu Fortinet để xác nhận trên phiên bản của bạn.
Lưu í ko bật nat policy khi tạo policy cho thông qua VPN
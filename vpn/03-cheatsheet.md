# VPN / IPsec Cheatsheet

Mặc định Cisco IOS-XE (IKEv2). Cú pháp có thể khác theo phiên bản/nền tảng; dùng `?` để xác nhận.

**Topology ví dụ:** R1 (203.0.113.1, LAN 10.1.0.0/16) ⇄ R2 (198.51.100.2, LAN 10.2.0.0/16). PSK: `Str0ngKey-Example`.

## 1. Cấu hình chung (IKEv2 + IPsec) – R1
```
crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy IKEV2-POL
 proposal IKEV2-PROP
!
crypto ikev2 keyring KR
 peer R2
  address 198.51.100.2
  pre-shared-key Str0ngKey-Example
!
crypto ikev2 profile IKEV2-PROF
 match identity remote address 198.51.100.2 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR
!
crypto ipsec transform-set TSET esp-aes 256 esp-sha256-hmac
 mode tunnel
```
R2: tương tự, đổi `address` thành 203.0.113.1 và `match identity remote address 203.0.113.1`.

## 2. Policy-based (crypto map) – R1
```
ip access-list extended VPN-TRAFFIC
 permit ip 10.1.0.0 0.0.255.255 10.2.0.0 0.0.255.255
!
crypto map CMAP 10 ipsec-isakmp
 set peer 198.51.100.2
 set transform-set TSET
 set ikev2-profile IKEV2-PROF
 match address VPN-TRAFFIC
!
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.0
 crypto map CMAP
```
R2: ACL **mirror** (`permit ip 10.2.0.0 0.0.255.255 10.1.0.0 0.0.255.255`), `set peer 203.0.113.1`.

**NAT exemption** (nếu có NAT overload): chặn traffic VPN khỏi NAT
```
ip access-list extended NAT-ACL
 deny   ip 10.1.0.0 0.0.255.255 10.2.0.0 0.0.255.255
 permit ip 10.1.0.0 0.0.255.255 any
```

## 3. Route-based (SVTI) – R1
```
crypto ipsec profile IPSEC-PROF
 set transform-set TSET
 set ikev2-profile IKEV2-PROF
!
interface Tunnel0
 ip address 172.16.0.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROF
!
ip route 10.2.0.0 255.255.0.0 Tunnel0
```
R2: `ip address 172.16.0.2`, `tunnel destination 203.0.113.1`, route `10.1.0.0/16 → Tunnel0`.

### Thêm BGP qua tunnel (thay cho static route)
```
router bgp 65001
 neighbor 172.16.0.2 remote-as 65002
 neighbor 172.16.0.2 timers 10 30
 address-family ipv4
  network 10.1.0.0 mask 255.255.0.0
  neighbor 172.16.0.2 activate
```

## 4. Kiểm tra
| Mục đích | Lệnh |
|---|---|
| Trạng thái IKE SA | `show crypto ikev2 sa` / `show crypto ikev2 sa detail` |
| IPsec SA, counter encaps/decaps | `show crypto ipsec sa` |
| Tổng quan session | `show crypto session` / `show crypto session detail` |
| Crypto map (policy-based) | `show crypto map` |
| Tunnel (route-based) | `show interface Tunnel0`, `show ip route`, `show ip interface brief` |
| Profile / proposal | `show crypto ikev2 profile`, `show crypto ikev2 proposal`, `show crypto ipsec profile` |
| Reset | `clear crypto ikev2 sa` / `clear crypto session` (gây gián đoạn!) |
| Test MTU | `ping 10.2.0.1 source 10.1.0.1 size 1400 df-bit` |

Đọc `show crypto ipsec sa`: `#pkts encaps` tăng mà `#pkts decaps` = 0 → peer không trả/ESP bị chặn/routing ngược. `#pkts encrypt` = 0 → traffic không khớp selector (policy) hoặc không route vào tunnel (route-based).

## 5. Troubleshoot theo triệu chứng
| Triệu chứng | Nguyên nhân hay gặp | Kiểm tra |
|---|---|---|
| Không có IKE SA | Firewall chặn UDP 500/4500; sai peer IP; proposal mismatch | `debug crypto ikev2`, `show crypto ikev2 sa`, capture UDP 500 |
| `AUTHENTICATION_FAILED` | PSK/identity/cert sai | keyring, `match identity`, FQDN vs IP |
| `NO_PROPOSAL_CHOSEN` | Mismatch encryption/hash/DH | so sánh proposal hai đầu |
| IKE lên, không có IPsec SA, `TS_UNACCEPTABLE` | Selector (ACL/proxy-ID) không mirror | ACL hai đầu; đặt selector cụ thể trên đầu route-based |
| SA lên, ping không qua | Policy: thiếu NAT exemption / ACL; Route: thiếu route | `show crypto ipsec sa` counters; `show ip route` |
| Ping OK, app lớn treo | MTU/MSS, ICMP bị chặn | `df-bit` ping, `ip tcp adjust-mss` |
| Tunnel flap `RECURDOWN` | Recursive routing | route tới tunnel destination qua underlay, dùng VRF |
| Một chiều | Asymmetric route / NAT-T | capture ESP/UDP 4500, `show crypto ipsec sa` hai đầu |
| Rekey rớt | Lifetime lệch, PFS lệch | so sánh lifetime, `set pfs` |

Debug an toàn: dùng `debug condition peer ipv4 <ip>` / `debug crypto ikev2` ngắn hạn, tắt bằng `undebug all`.

## 6. Vendor tương đương (tham khảo)
**Junos (route-based):**
```
set interfaces st0 unit 0 family inet address 172.16.0.1/30
set security ipsec vpn VPN-R2 bind-interface st0.0
set security ipsec vpn VPN-R2 ike gateway GW-R2
set security ipsec vpn VPN-R2 ike ipsec-policy IPSEC-POL
set routing-options static route 10.2.0.0/16 next-hop st0.0
```
Policy-based: bỏ `bind-interface`, trong security policy dùng `then permit tunnel ipsec-vpn VPN-R2`. Kiểm tra: `show security ike security-associations`, `show security ipsec security-associations`.

**ASA:** `show crypto ikev2 sa`, `show crypto ipsec sa`, `show vpn-sessiondb l2l`. VTI: `interface Tunnel1` + `tunnel mode ipsec ipv4` + `tunnel protection ipsec profile`.

**strongSwan (Linux):** `swanctl --list-sas`, `ip xfrm state`, `ip xfrm policy`. Route-based: interface xfrm với `if_id`.

## 7. Mẹo nhớ nhanh
- **Policy** = *ACL quyết định* "mã hoá cái gì"; **Route** = *routing table quyết định*.
- Policy-based lỗi → nhìn **ACL mirror + NAT exemption**.
- Route-based lỗi → nhìn **route vào tunnel + MTU + recursive**.
- Interop khác kiểu → ép **selector khớp** ở đầu route-based.

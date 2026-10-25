# VPN / IPsec cơ bản

## 1. VPN là gì?
**Định nghĩa:** VPN (Virtual Private Network) tạo một "đường hầm" logic qua mạng không tin cậy (Internet) để hai đầu giao tiếp như trong mạng riêng.

**Vì sao cần:** thuê leased line/MPLS giữa các chi nhánh đắt; Internet rẻ nhưng công khai → cần **bảo mật** (mã hoá, xác thực, chống replay) trên đường truyền.

| Loại | Dùng cho | Công nghệ điển hình |
|---|---|---|
| Site-to-Site | Nối 2 mạng (chi nhánh ↔ HQ, on-prem ↔ cloud) | IPsec, GRE over IPsec, DMVPN |
| Remote Access | Người dùng đơn lẻ ↔ mạng công ty | SSL/TLS VPN, IKEv2/IPsec, WireGuard |
| Overlay WAN | Nhiều site, tự động hoá | DMVPN, FlexVPN, SD-WAN |

File này tập trung **Site-to-Site IPsec**.

## 2. IPsec: các thành phần
IPsec không phải một giao thức mà là một bộ khung (RFC 4301):

| Thành phần | Vai trò |
|---|---|
| **IKE** (IKEv1 / IKEv2) | Thương lượng thuật toán, xác thực peer, trao đổi khoá (Diffie-Hellman) |
| **ESP** (IP proto 50) | Mã hoá + xác thực + chống replay. Gần như luôn dùng ESP |
| **AH** (IP proto 51) | Chỉ xác thực, không mã hoá, và không đi qua NAT → hiếm dùng |
| **SA** (Security Association) | "Hợp đồng" một chiều: thuật toán, khoá, SPI, lifetime. Mỗi tunnel cần ít nhất 2 IPsec SA (in/out) |

### Hai pha
```
Phase 1 (IKE SA / IKEv2 IKE_SA_INIT + IKE_AUTH)
  → peer xác thực nhau, dựng kênh điều khiển an toàn (UDP 500)
Phase 2 (IPsec SA / IKEv2 CHILD_SA)
  → dựng SA thực sự mang dữ liệu người dùng (ESP)
```
- IKEv1: Phase 1 (Main mode 6 gói hoặc Aggressive 3 gói) + Phase 2 (Quick mode 3 gói).
- IKEv2: gọn hơn (4 gói cho cả IKE SA + CHILD_SA đầu tiên), có DPD/MOBIKE/EAP chuẩn sẵn. **Nên dùng IKEv2** khi cả hai đầu hỗ trợ.

### Cổng/giao thức cần mở trên firewall
UDP 500 (IKE), UDP 4500 (NAT-T), IP proto 50 (ESP).

### Tunnel mode vs Transport mode
| | Tunnel mode | Transport mode |
|---|---|---|
| Gói gốc | Được bọc toàn bộ trong IP header mới | Giữ IP header gốc, chèn ESP |
| Dùng cho | Site-to-site (gateway ↔ gateway) | Host ↔ host, hoặc GRE/L2TP over IPsec |

## 3. Traffic selector – chìa khoá để hiểu policy vs route
**Selector** (IKEv1: *proxy-ID / interesting traffic ACL*; IKEv2: *traffic selector TSi/TSr*) là cặp (src, dst, protocol, port) quyết định **gói nào được đưa vào SA**.

- Hai đầu phải có selector **khớp nhau (đối xứng, mirror)**, nếu không Phase 2 / CHILD_SA fail.
- Cách selector được xác định chính là điểm khác biệt giữa policy-based và route-based.

## 4. Policy-based VPN
**Định nghĩa:** traffic được chọn vào tunnel bằng **ACL/policy** khớp (src → dst). Không có interface tunnel; crypto map gắn trực tiếp lên interface ra Internet.

```
LAN A 10.1.0.0/16 ──[R1: crypto map + ACL]── Internet ──[R2]── LAN B 10.2.0.0/16
ACL: permit ip 10.1.0.0/16 10.2.0.0/16   ← ACL này = selector
```
Luồng xử lý trên R1: gói đến → **routing lookup chọn interface ra** → ra interface có crypto map → so ACL → khớp thì mã hoá, không khớp thì gửi **plaintext**.

**Ưu:** đơn giản cho 1–2 subnet; tương thích rộng (thiết bị cũ, IKEv1); có thể tách SA theo từng cặp subnet (kiểm soát chi tiết).
**Nhược:** không chạy được routing protocol (OSPF/BGP) trực tiếp qua tunnel; mỗi cặp subnet = 1 SA (số lượng tăng theo cấp số nhân); khó mở rộng, khó failover; ACL sai/lệch → mất kết nối hoặc rò rỉ plaintext.

## 5. Route-based VPN
**Định nghĩa:** tạo **interface tunnel ảo** (VTI – Virtual Tunnel Interface). Mọi gói **được route vào interface đó** sẽ được mã hoá. Selector thường là `any any` (0.0.0.0/0 ↔ 0.0.0.0/0).

```
LAN A ── R1 [Tunnel0 172.16.0.1] ═ IPsec ═ [Tunnel0 172.16.0.2] R2 ── LAN B
R1: ip route 10.2.0.0 255.255.0.0 Tunnel0   (hoặc học qua OSPF/BGP)
```
Luồng xử lý: gói đến → **routing lookup trỏ vào Tunnel0** → mã hoá → gói ESP được route ra interface vật lý.

**Ưu:** chạy được routing động (BGP/OSPF/EIGRP) qua tunnel → failover tự động; thêm subnet chỉ cần thêm route; hỗ trợ multicast (với GRE), QoS/PBR/ACL/firewall zone gắn trên interface tunnel; chuẩn cho cloud & SD-WAN.
**Nhược:** selector `any-any` rộng nên **phải kiểm soát bằng routing/firewall**; một số thiết bị cũ không hỗ trợ; khi làm việc với vendor chỉ policy-based phải xử lý khác biệt selector (xem `02-advanced.md`).

## 6. So sánh nhanh
| Tiêu chí | Policy-based | Route-based |
|---|---|---|
| Cách chọn traffic | ACL / policy (src,dst) | Routing table (next-hop = tunnel) |
| Interface tunnel | Không | Có (VTI / GRE) |
| Selector | Cụ thể, theo ACL | Thường `any-any` |
| Routing động qua tunnel | Không | Có (OSPF, BGP, EIGRP) |
| Số SA | Mỗi cặp subnet 1 SA | Thường 1 SA cho cả tunnel |
| Mở rộng khi thêm subnet | Sửa ACL **hai đầu** | Thêm route/advertise |
| Multicast | Không | Có (qua GRE; VTI IPv4 hỗ trợ hạn chế – kiểm tra nền tảng) |
| Failover nhiều tunnel/ISP | Khó | Dễ (ECMP, metric, BGP) |
| Kiểm soát chi tiết trên tunnel | Tốt (ACL ngay trên selector) | Cần firewall/ACL trên interface tunnel |
| Interop thiết bị cũ | Rộng nhất | Phụ thuộc hỗ trợ VTI |
| Cloud (AWS/Azure/GCP) | Hạn chế | Chuẩn, ưu tiên |

### Khi nào chọn gì?
- **Policy-based:** site nhỏ, 1 subnet ↔ 1 subnet, peer là thiết bị cũ/bên thứ ba chỉ hỗ trợ policy-based, yêu cầu compliance phải giới hạn rõ từng cặp subnet.
- **Route-based:** nhiều subnet, cần routing động/HA, kết nối cloud, hub-and-spoke, mọi thiết kế mới.

## 7. Lỗi thường gặp
1. **Selector/ACL không mirror** → IKE SA lên nhưng không có IPsec SA (`NO_PROPOSAL_CHOSEN` / `TS_UNACCEPTABLE`).
2. **Mismatch** proposal: encryption/hash/DH group/lifetime/PSK/identity.
3. **NAT** giữa hai đầu mà không bật NAT-T, hoặc firewall chặn UDP 4500/ESP.
4. **Policy-based:** quên **NAT exemption** (no-NAT / identity NAT) → traffic bị NAT trước khi vào tunnel, không khớp ACL.
5. **Route-based:** thiếu route trỏ vào tunnel, hoặc tunnel source/destination sai → tunnel up nhưng không có traffic.
6. **MTU/MSS:** ESP + (GRE) overhead làm phân mảnh → ping nhỏ OK, ứng dụng lớn treo. Xem `02-advanced.md`.
7. **Asymmetric routing** qua nhiều ISP với firewall stateful.

# VXLAN & EVPN cơ bản

## 1. Vì sao cần VXLAN?
Mạng Data Center truyền thống dùng VLAN + STP, có 3 hạn chế:
1. **VLAN ID chỉ 12-bit = 4094 segment** → không đủ cho multi-tenant/cloud.
2. **STP** block link, lãng phí băng thông, hội tụ chậm; L2 domain lớn → broadcast storm.
3. **VM mobility** cần L2 kéo dài giữa rack/DC.

**VXLAN (Virtual eXtensible LAN, RFC 7348)** đóng gói frame Ethernet L2 vào gói **UDP/IP** (overlay) chạy trên mạng L3 (underlay). Dùng **VNI 24-bit = ~16 triệu segment**. Underlay L3 + ECMP → dùng hết mọi đường, không STP.

**Ẩn dụ:** Underlay là hệ thống đường bưu điện (chỉ biết địa chỉ nhà ga). Overlay là bức thư (frame của VM) đặt trong phong bì có địa chỉ ga đến. Nhà ga (VTEP) mở phong bì và giao thư.

## 2. Thuật ngữ
| Thuật ngữ | Giải nghĩa |
|---|---|
| **Underlay** | Mạng IP vật lý (OSPF/IS-IS/eBGP) chở gói VXLAN giữa các VTEP |
| **Overlay** | Mạng logic của tenant (L2/L3) chạy trên underlay |
| **VTEP** | VXLAN Tunnel End Point: thiết bị đóng/mở gói (leaf switch, hypervisor vSwitch). Có IP loopback (NVE source) |
| **VNI** | VXLAN Network Identifier (24-bit). **L2VNI** map 1 VLAN/bridge domain; **L3VNI** cho routing inter-subnet theo VRF |
| **NVE** | Network Virtualization Edge: interface logic của VTEP (`nve1`) |
| **BUM** | Broadcast, Unknown unicast, Multicast |
| **Leaf / Spine** | Kiến trúc Clos: leaf nối server, spine nối các leaf. Mọi leaf cách nhau 2 hop |
| **Border leaf** | Leaf nối ra ngoài (WAN, firewall) |
| **VRF** | Tenant L3 isolation |

## 3. Cấu trúc gói VXLAN
```
Outer Ethernet | Outer IP (src=VTEP-A, dst=VTEP-B) | UDP (dst 4789, src=hash) | VXLAN hdr (8B, VNI) | Inner Ethernet frame (VM)
```
- Overhead ≈ **50 byte** (14 + 20 + 8 + 8) → MTU underlay phải ≥ 1550 (khuyến nghị jumbo **9216**).
- UDP source port = hash của inner flow → underlay ECMP chia tải tốt.
- Flag I=1 trong VXLAN header = VNI hợp lệ.

## 4. Cơ chế hoạt động: data plane (flood-and-learn)
Mô hình gốc (không có control plane):
- Mỗi VNI map tới **multicast group** trong underlay (PIM). VTEP join group.
- BUM từ VM → đóng gói gửi tới multicast group → mọi VTEP trong VNI nhận.
- VTEP học MAC nguồn → (VTEP IP nguồn) khi nhận gói (giống switch học MAC).
Hoặc **ingress replication / head-end replication (unicast)**: VTEP nhân bản BUM thành unicast tới từng VTEP đối tác (danh sách tĩnh).

**Ví dụ:** VM-A (MAC aaaa) sau VTEP1, VM-B (MAC bbbb) sau VTEP2, cùng VNI 10010.
1. VM-A ARP broadcast tìm IP VM-B → VTEP1 đóng gói gửi multicast 239.1.1.10.
2. VTEP2 mở gói, flood tới port VLAN 10, đồng thời học `aaaa → VTEP1`.
3. VM-B reply unicast → VTEP2 tra `aaaa → VTEP1`, đóng gói unicast; VTEP1 học `bbbb → VTEP2`.

Nhược điểm: phụ thuộc multicast, flood nhiều, khó scale, không có thông tin IP/MAC trước → **dẫn tới BGP EVPN**.

## 5. BGP EVPN, control plane cho VXLAN (RFC 7432 / 8365)
**Ý tưởng:** thay vì học MAC bằng flood, các VTEP dùng **MP-BGP address-family L2VPN EVPN** để *quảng bá* MAC/IP và thông tin VTEP cho nhau.
Lợi ích: giảm flood, **ARP suppression**, hỗ trợ mobility, multi-tenancy chuẩn L3VPN, anycast gateway, multi-homing, không cần multicast (dùng ingress replication).

### Các Route Type chính
| RT | Tên | Dùng cho |
|---|---|---|
| **1** | Ethernet Auto-Discovery | Multi-homing (ESI), fast convergence |
| **2** | **MAC/IP Advertisement** | Quảng bá MAC (+IP) của host, kèm L2VNI (và L3VNI cho IP) |
| **3** | **Inclusive Multicast Ethernet Tag** | Quảng bá VTEP tham gia VNI → dựng BUM replication list |
| **4** | Ethernet Segment | DF election cho multi-homing |
| **5** | **IP Prefix** | Quảng bá prefix L3 (ngoài, summary, redistribute) |

### Thuật ngữ EVPN
- **RD** (Route Distinguisher): làm route duy nhất; **RT** (Route Target) import/export: điều khiển ai nhận route.
- **MAC-VRF** (per L2VNI) và **IP-VRF** (per L3VNI/VRF).
- **Symmetric IRB** (khuyên dùng): routing inter-subnet qua L3VNI: ingress leaf route → VXLAN L3VNI → egress leaf route. **Asymmetric IRB**: ingress route + bridge, cần mọi VNI trên mọi leaf.
- **Anycast Gateway**: tất cả leaf dùng *cùng IP + MAC gateway* cho một subnet → VM luôn có default-gateway gần nhất, di chuyển không đổi.
- **Underlay + overlay BGP**: thường underlay = eBGP hoặc OSPF/IS-IS; overlay = iBGP (spine làm RR) hoặc eBGP multihop.

## 6. Topology minh họa
```
         [Spine1]      [Spine2]       (RR cho EVPN nếu iBGP, underlay P2P /31)
          /  |  \      /  |  \
    [Leaf1] [Leaf2] [Leaf3]  [BorderLeaf]
      |        |        |
    VM-A     VM-B     VM-C         VLAN10 -> VNI 10010 ; VRF TENANT1 -> L3VNI 50001
```
- Underlay: Leaf Loopback0 (router-id), Loopback1 (VTEP/NVE source), link /31 đến spine, OSPF/BGP.
- Overlay: `nve1` source `loopback1`, `host-reachability protocol bgp`, member vni.

## 7. Cấu hình tối thiểu Cisco NX-OS (Leaf, iBGP EVPN, ingress replication)
```
nv overlay evpn
feature ospf
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
fabric forwarding anycast-gateway-mac 0000.2222.3333

! Underlay (rút gọn)
interface Ethernet1/1
  no switchport
  mtu 9216
  ip address 10.0.1.1/31
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0
interface loopback1
  ip address 10.255.1.1/32          ! VTEP IP
  ip router ospf UNDERLAY area 0

! L2 VNI
vlan 10
  vn-segment 10010
vlan 3001
  vn-segment 50001                  ! L3VNI cho VRF
vrf context TENANT1
  vni 50001
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn
interface Vlan3001
  no shutdown
  mtu 9216
  vrf member TENANT1
  ip forward
interface Vlan10
  no shutdown
  vrf member TENANT1
  ip address 10.10.10.1/24
  fabric forwarding mode anycast-gateway

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    ingress-replication protocol bgp
    suppress-arp
  member vni 50001 associate-vrf

evpn
  vni 10010 l2
    rd auto
    route-target import auto
    route-target export auto

router bgp 65000
  router-id 10.255.0.1
  neighbor 10.255.100.1 remote-as 65000    ! Spine (RR)
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
  vrf TENANT1
    address-family ipv4 unicast
      advertise l2vpn evpn
```
Kiểm tra nhanh: `show nve peers`, `show nve vni`, `show bgp l2vpn evpn summary`, `show l2route evpn mac all`.

## 8. Luồng traffic ví dụ (Symmetric IRB)
VM-A (10.10.10.10, VLAN10, Leaf1) → VM-C (10.20.20.20, VLAN20, Leaf3), cùng VRF TENANT1.
1. VM-A gửi tới gateway anycast 10.10.10.1 (Leaf1).
2. Leaf1 route trong VRF, tìm `10.20.20.20/32` (EVPN RT-2/5): next-hop VTEP Leaf3, L3VNI 50001, router-MAC Leaf3.
3. Leaf1 đóng gói VXLAN VNI 50001, dst-MAC = router-MAC của Leaf3.
4. Leaf3 mở gói, VRF lookup, route vào VLAN20 → VM-C.
Chiều ngược lại đối xứng (cùng VNI) → tên "symmetric".

## 9. Lỗi thường gặp
| Triệu chứng | Nguyên nhân |
|---|---|
| NVE peer không lên | Underlay không ping được loopback VTEP; thiếu `advertise` loopback1; MTU |
| Có peer, không có MAC/IP | Thiếu `send-community extended`; RT/RD không khớp; VNI chưa member |
| Ping L2 được, L3 không | Thiếu L3VNI/`ip forward`/`advertise l2vpn evpn` trong VRF; anycast-gateway MAC không đồng nhất |
| Packet lớn rớt | MTU underlay < 1550; fragmentation |
| BUM loop/flood | Multicast/PIM sai; ingress replication thiếu RT-3 |

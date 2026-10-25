# VXLAN & EVPN nâng cao

## 1. Thiết kế Underlay
| Lựa chọn | Ưu / Nhược |
|---|---|
| **OSPF/IS-IS** | Đơn giản, hội tụ nhanh, nhưng 2 giao thức (IGP + BGP) |
| **eBGP (RFC 7938)** | 1 giao thức, scale lớn, policy tốt; mỗi leaf ASN riêng (hoặc ASN theo tầng), spine ASN chung |
| **Unnumbered** (`ip unnumbered lo0`/ BGP unnumbered) | Tiết kiệm IP, dễ tự động hóa |

Yêu cầu: P2P /31, MTU 9216, ECMP (`maximum-paths`), BFD, loopback0 (router-id/peering) + loopback1 (VTEP) quảng bá vào underlay. Dùng **anycast VTEP** (secondary IP trên lo1) cho vPC pair.

**eBGP overlay & underlay (ví dụ):** Leaf AS 65101..., Spine AS 65000. Overlay `neighbor <spine-lo> ebgp-multihop 3`, spine: `retain route-target all` (vì spine không có VRF/VNI mà phải giữ route để chuyển tiếp), `next-hop-unchanged` (để leaf thấy VTEP thật là next-hop):
```
route-map NH-UNCHANGED permit 10
  set ip next-hop unchanged
router bgp 65000
  address-family l2vpn evpn
    retain route-target all
  neighbor 10.255.1.1
    address-family l2vpn evpn
      send-community extended
      route-map NH-UNCHANGED out
```
Với iBGP + spine RR: next-hop mặc định không đổi, không cần route-map.

## 2. Multi-homing
### vPC (Cisco) / MLAG (Arista) 
Hai leaf thành cặp, server dual-attach LACP. Dùng chung **anycast VTEP IP** (secondary IP loopback) để cả cặp giống 1 VTEP. Cần peer-link, keepalive; `advertise-pip` / `advertise virtual-rmac` để tối ưu route-type 5 theo PIP.
### EVPN Multihoming (ESI-LAG, RFC 7432)
Standards-based, nhiều hơn 2 leaf: ESI (Ethernet Segment Identifier, 10 byte) cho mỗi LAG. Route type 1 (A-D per ES / per EVI) dùng cho aliasing và mass-withdraw; type 4 cho **DF election**. Tránh duplicate BUM bằng split-horizon label/ **local-bias**.
```
interface port-channel10
  ethernet-segment 10          ! ESI tự sinh qua system-mac
  system-mac 0000.0000.0010
```

## 3. Bên trong EVPN Route-Types
**Route-type 2 (MAC/IP)**: `[2]:[ESI]:[EthTag]:[MACLen]:[MAC]:[IPLen]:[IP]` + L2VNI, (+ L3VNI + Router-MAC extended community nếu có IP). Có thể 2 route riêng cho MAC và MAC+IP.
**Route-type 3**: `[3]:[EthTag]:[IPLen]:[OrigIP]` + PMSI Tunnel attribute (ingress replication / PIM / ...). Mọi VTEP của VNI quảng bá.
**Route-type 5**: `[5]:[EthTag]:[PfxLen]:[Prefix]` + L3VNI + Router-MAC + gateway IP (0 hoặc overlay index).
Extended communities: RT, Encapsulation (VXLAN type 8), Router MAC, **MAC Mobility** (sequence number), **Default Gateway**, ESI-label, DF election.

### MAC mobility (VM move)
Khi VM di chuyển, leaf mới quảng bá RT-2 cùng MAC với *sequence number + 1*; nơi cao hơn thắng. Chống MAC duplicate/loop: **duplicate detection** (mặc định 5 moves/180s → freeze MAC).
```
l2rib dup-host-addr-detect 5 180     ! NX-OS (tên có thể khác theo version)
```
Kiểm tra: `show l2route evpn mac all`, `show bgp l2vpn evpn <mac>` (xem MAC Mobility seq).

### ARP suppression & ND
Leaf trả lời ARP từ bảng MAC-IP EVPN → giảm flood; cần `suppress-arp`, TCAM region `arp-ether` (NX-OS). **Unknown unicast suppression** tùy chọn.

## 4. Routing ra ngoài fabric (Border / External)
- **VRF-lite** hoặc **MPLS L3VPN / SR / EVPN-to-VPN stitching** ở border.
- Border leaf chạy eBGP/OSPF per-VRF với firewall/router (`vrf TENANT1 neighbor ...`), redistribute vào EVPN RT-5.
- **Route leaking giữa VRF**: dùng import/export RT khác nhau (`route-target import 65000:50002 evpn`) – shared services VRF.
- **Default route**: border quảng bá `0.0.0.0/0` RT-5; leaf chỉ cần default thay vì full routes.
- **Firewall insertion**: service-chaining bằng VRF kép (inside/outside) hoặc PBR.

## 5. Multi-Site / DCI
| Mô hình | Đặc điểm |
|---|---|
| **Multi-Pod** | Một fabric logic, nhiều pod, underlay nối; chung 1 BGP overlay |
| **Multi-Site (BGW)** | Mỗi site fabric riêng, **Border Gateway (BGW)** nối site qua VXLAN. Giới hạn blast radius, tách failure domain; BGW re-originate RT-2/5 |
| **DCI VXLAN-to-VXLAN / OTV / EVPN-over-MPLS/SR** | Tùy hạ tầng WAN |

BGW dùng 2 giao diện: **fabric-facing** và **DCI-facing**; loopback **multisite VIP** dùng chung cả site (anycast BGW).
```
evpn multisite border-gateway 100
interface nve1
  multisite border-gateway interface loopback100
interface Ethernet1/10
  evpn multisite dci-tracking
interface Ethernet1/1
  evpn multisite fabric-tracking
```
**Stretching VLAN**: cần cân nhắc (L2 stretch tăng rủi ro). Dùng ARP suppression, storm control, host mobility chặt chẽ. Ưu tiên L3 giữa site khi có thể (/32 host routes).

## 6. QoS, MTU, ECMP, Load balancing
- **DSCP/CoS copy**: outer IP DSCP copy từ inner (uniform) hoặc đặt cố định (pipe).
- **ECMP hash**: UDP source port từ inner 5-tuple → cần underlay hash dùng L4 port.
- **ECN/PFC** cho RoCE (lossless Ethernet) – cấu hình QoS theo fabric.
- **MTU**: underlay ≥ overlay + 50 (+4 nếu 802.1Q inner, +20 nếu IPv6 outer).
- **Anycast gateway MAC** thống nhất toàn fabric.

## 7. Bảo mật & Micro-segmentation
- **VRF** = tenant isolation; **VNI** = L2 isolation.
- **Micro-segmentation**: private VLAN, group-based policy (SGT / Group Policy Option VXLAN-GPO), EPG (ACI), host-based firewall (NSX).
- VXLAN không mã hóa → dùng **MACsec** trên link hoặc **IPsec/GRE** cho DCI.
- Chống spoofing: DHCP snooping / ARP inspection / **IP source guard**; storm-control; bpdu-guard (vì STP vẫn tồn tại ở edge).

## 8. Triển khai & kiến trúc vendor
| Hãng | Sản phẩm | Ghi chú |
|---|---|---|
| Cisco | NX-OS VXLAN EVPN, **ACI** (controller APIC, VXLAN thuộc nội bộ), Catalyst 9K (SDA dùng VXLAN-GPO + LISP) | |
| Arista | EOS VXLAN EVPN, CloudVision | cấu hình `interface Vxlan1` |
| Juniper | Apstra, QFX/EX, MX | `set switch-options vtep-source-interface` |
| NVIDIA | Cumulus Linux (NVUE/FRR) | BGP unnumbered chuẩn |
| VMware | NSX (Geneve, không phải VXLAN) | |
| Linux | `ip link add vxlan10 type vxlan id 10 ...`, FRR EVPN | |

### Arista EOS mẫu
```
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf TENANT1 vni 50001
router bgp 65101
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE ebgp-multihop 3
   neighbor SPINE send-community extended
   address-family evpn
      neighbor SPINE activate
   vlan 10
      rd auto
      route-target both 10010:10010
      redistribute learned
   vrf TENANT1
      rd 10.255.1.1:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      redistribute connected
```
### Linux/FRR mẫu
```
ip link add br10 type bridge
ip link add vxlan10 type vxlan id 10010 dstport 4789 local 10.255.1.1 nolearning
ip link set vxlan10 master br10 ; ip link set vxlan10 up ; ip link set br10 up
# FRR: router bgp 65101 / address-family l2vpn evpn / neighbor SPINE activate / advertise-all-vni
```

## 9. Giám sát & vận hành
- Telemetry (gNMI/streaming), BMP, syslog; theo dõi: NVE peer up/down, MAC/IP count, BGP EVPN prefix, TCAM utilization, ARP/ND, MAC move, duplicate-detect events, packet drop.
- Công cụ: **Nexus Dashboard Insights**, Arista CloudVision, Juniper Apstra, Grafana+Prometheus.
- **Automation**: Ansible, Terraform, NetBox + Jinja2 templates, **Nexus-as-Code**, Containerlab cho lab CI.
- OAM: `ping nve`, `traceroute nve`, `show vxlan oam`/`ping vxlan` (tùy platform), hoặc `ping vrf T1 10.20.20.20 source 10.10.10.1`.

## 10. Giới hạn & lưu ý
- **TCAM / scale**: số VNI, MAC, host routes, ECMP, ARP suppression, SVI limit tùy ASIC/dòng switch.
- **Recirculation** (một số ASIC): VXLAN routing cần feature/template phù hợp.
- Hạn chế nhỏ về **IPv6 underlay**; mixing vendor khả thi nhưng test (RT format, ESI, `type 5` gateway IP).
- Nếu chỉ cần ít VLAN trong 1 DC nhỏ → không nhất thiết VXLAN. Cân nhắc độ phức tạp.

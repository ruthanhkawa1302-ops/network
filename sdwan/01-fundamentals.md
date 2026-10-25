# SD-WAN cơ bản

## 1. SD-WAN là gì?
**SD-WAN (Software-Defined WAN)** là kiến trúc WAN trong đó **control plane và management tách khỏi thiết bị chuyển tiếp**, được điều phối tập trung bằng phần mềm. Các site nối với nhau bằng **overlay** (tunnel mã hóa) chạy trên **nhiều loại đường truyền (transport)**: MPLS, Internet broadband, 4G/5G... SD-WAN chọn đường theo **ứng dụng** và **chất lượng đo thực tế**.

**Vấn đề WAN truyền thống:**
- MPLS đắt, giao hàng chậm (tuần/tháng), băng thông thấp.
- Cấu hình từng router bằng CLI, dễ sai, khó đồng bộ policy cho hàng trăm chi nhánh.
- Truy cập SaaS/cloud (Microsoft 365, AWS) phải "backhaul" về HQ rồi ra Internet → trễ.
- Kết nối dự phòng thường để idle (active/standby).

**Ẩn dụ:** WAN cũ giống công ty vận tải chỉ có một đường cao tốc trả phí, tài xế (router) tự nhớ lộ trình. SD-WAN giống ứng dụng bản đồ trung tâm biết tình trạng mọi đường (cao tốc, quốc lộ, đường làng), chọn lộ trình theo loại hàng (voice nhanh, backup file chậm cũng được), đổi đường ngay khi kẹt xe.

## 2. Lợi ích chính
1. **Dùng nhiều transport cùng lúc** (active/active), giảm chi phí.
2. **Application-aware routing**: chọn đường theo SLA của từng app.
3. **Zero-touch provisioning (ZTP)**: cắm điện, thiết bị tự tải cấu hình.
4. **Quản lý tập trung**, policy/template nhất quán.
5. **Bảo mật tích hợp**: mã hóa IPsec, segmentation, firewall/SSE.
6. **Cloud onramp**: kết nối tối ưu đến SaaS/IaaS.
7. **Quan sát**: dashboard hiệu năng theo app/site/tunnel.

## 3. Kiến trúc chung (3 mặt phẳng + orchestration)
| Mặt phẳng | Vai trò |
|---|---|
| **Data plane** | Edge router chuyển tiếp traffic qua overlay tunnel (IPsec/GRE) |
| **Control plane** | Phân phối route, khóa mã hóa, policy; xác thực thiết bị |
| **Management plane** | UI/API, template, giám sát |
| **Orchestration** | Onboarding thiết bị, kết nối ban đầu |

## 4. Cisco Catalyst SD-WAN (Viptela), các thành phần
| Thành phần | Tên cũ | Chức năng |
|---|---|---|
| **SD-WAN Manager** | vManage | GUI/API, template, giám sát, cấu hình |
| **SD-WAN Controller** | vSmart | Control plane: chạy **OMP**, phân phối route + policy + key cho edge |
| **SD-WAN Validator** | vBond | Orchestrator: xác thực, thông báo địa chỉ controller/manager, hỗ trợ **NAT traversal** (STUN-like). Phải có IP public |
| **WAN Edge** | vEdge / cEdge | Router tại site: IOS-XE SD-WAN (cEdge) hoặc Viptela OS (vEdge) |
| **SD-WAN Analytics** | vAnalytics | Phân tích |

```
              [Manager]   [Controller]   [Validator]
                  \          |           /
          (DTLS/TLS control connections qua Internet/MPLS)
                  /          |           \
   [Edge HQ]===== IPsec data plane tunnels =====[Edge Branch1]
       |                                              |
   MPLS + INET                                    MPLS + INET
```
Control connection dùng **DTLS** (UDP 12346 mặc định) hoặc TLS (TCP). Data plane dùng **IPsec** (ESP) hoặc GRE; BFD chạy trong tunnel để đo chất lượng.

## 5. Khái niệm cốt lõi
| Khái niệm | Giải thích |
|---|---|
| **System-IP** | ID duy nhất của thiết bị trong overlay (giống router-id, không nhất thiết route được) |
| **Site-ID** | Số định danh site (1 site có thể nhiều edge); dùng để lọc/định hướng policy |
| **Organization Name** | Tên tổ chức, phải giống nhau trên mọi thiết bị trong overlay |
| **VPN / VRF (service-side)** | VPN 0 = transport (WAN), VPN 512 = management; VPN 1-511 = dịch vụ người dùng |
| **TLOC** | Transport Locator: điểm kết thúc tunnel = {System-IP, Color, Encapsulation} |
| **Color** | Nhãn loại transport: `mpls`, `biz-internet`, `public-internet`, `lte`, `3g`, `metro-ethernet`, `private1..6`, `red`, `blue`, `gold`... **Public color** (đằng sau NAT) vs **private color** quyết định cách dựng tunnel |
| **OMP** | Overlay Management Protocol: TCP-based, giống BGP, chạy giữa edge ↔ controller, quảng bá route, TLOC, service, key, policy |
| **Template** | Device template (feature templates / CLI template / Configuration Group mới) |
| **BFD** | Đo loss/latency/jitter trong từng tunnel (hello 1s, multiplier 7) |
| **Colors restrict** | Giới hạn chỉ dựng tunnel giữa các TLOC cùng color |

### OMP route types
- **OMP routes**: prefix service-side (từ VPN) – giống VPNv4.
- **TLOC routes**: danh sách TLOC có sẵn + attribute (preference, weight, public/private IP).
- **Service routes**: dịch vụ (firewall, IPS) gắn vào VPN.

## 6. Luồng khởi động thiết bị (Onboarding / ZTP)
1. Edge bật nguồn, lấy IP (DHCP) trên cổng WAN, có DNS.
2. Liên hệ **ZTP/PnP server** (Cisco PnP Connect) hoặc cấu hình tay `vbond` address → biết Validator.
3. Edge ↔ **Validator** (DTLS): xác thực bằng certificate/serial-number whitelist; Validator trả danh sách Manager + Controller.
4. Edge ↔ **Manager**: tải config (template), edge ↔ **Controller**: thiết lập OMP.
5. Edge nhận route/TLOC/key từ Controller → dựng **IPsec tunnel + BFD** trực tiếp tới edge khác (full mesh động).

## 7. Cấu hình tối thiểu cEdge (CLI, IOS-XE SD-WAN)
```
system
 host-name BR1
 system-ip 10.255.0.11
 site-id 11
 organization-name "LAB-ORG"
 sp-organization-name "LAB-ORG"
 vbond 203.0.113.10
!
vrf definition 1
 rd 1:1
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
!
interface GigabitEthernet1           ! transport (VPN 0, global)
 ip address dhcp
 no shutdown
!
interface Tunnel1
 ip unnumbered GigabitEthernet1
 tunnel source GigabitEthernet1
 tunnel mode sdwan
!
sdwan
 interface GigabitEthernet1
  tunnel-interface
   encapsulation ipsec
   color biz-internet
   allow-service dhcp
   allow-service dns
   allow-service icmp
 omp
  no shutdown
  address-family ipv4
   advertise connected
   advertise static
```
Thực tế mô hình quản lý bằng **template từ Manager**; CLI chỉ để bootstrap/gỡ lỗi.

## 8. Application-Aware Routing (AAR), ví dụ
Yêu cầu: Voice cần loss <1%, latency <150 ms, jitter <30 ms. Có 2 đường: MPLS và Internet.
- Mỗi tunnel BFD đo mỗi giây, tính trung bình theo `poll-interval` (mặc định 10 phút) × `multiplier` (6).
- Policy: app = voice → ưu tiên **MPLS**; nếu MPLS vi phạm SLA → chuyển **biz-internet**; hết cả hai → chọn tốt nhất (`fallback-to-best-path`).
Kết quả: không phải đổi route thủ công; voice luôn trên đường tốt.

## 9. Mô hình topology
- **Full mesh** (mặc định): tunnel trực tiếp giữa mọi edge cần thiết (dựng khi có route).
- **Hub-and-spoke**: dùng **Centralized Control Policy** để buộc spoke đi qua hub.
- **Regional hub / partial mesh**: spoke ↔ regional hub; hub ↔ hub.
- **Cloud**: edge ảo (CSR/Cat8000V) trong AWS/Azure/GCP.

## 10. So sánh vendor nhanh
| Vendor | Thành phần | Ghi chú |
|---|---|---|
| Cisco Catalyst SD-WAN | Manager/Controller/Validator/Edge | OMP, TLOC |
| Fortinet Secure SD-WAN | FortiGate + FortiManager | SD-WAN trong FortiOS, SLA rules; BGP/IPsec ADVPN |
| VMware (Broadcom) VeloCloud | Orchestrator/Gateway/Edge | DMPC, Gateways cloud |
| Palo Alto Prisma SD-WAN (CloudGenix) | Controller/ION | App-defined |
| Versa | Director/Controller/Analytics/FlexVNF | Mạnh về multi-tenant |
| HPE Aruba EdgeConnect (Silver Peak) | Orchestrator/EdgeConnect | WAN optimization |
| Juniper | Session Smart Router (128T) | Session-based routing |

## 11. Khái niệm liên quan
- **SASE / SSE**: SD-WAN + bảo mật cloud (SWG, CASB, ZTNA, FWaaS).
- **DIA** (Direct Internet Access) / **Local Internet Breakout**: ra Internet tại chi nhánh.
- **Underlay** (đường vật lý: MPLS, Internet) vs **Overlay** (tunnel SD-WAN).
- **ZTP**: Zero Touch Provisioning.
- **Cloud OnRamp**: tối ưu SaaS/IaaS.

## 12. Lỗi thường gặp
| Triệu chứng | Nguyên nhân |
|---|---|
| Edge không lên control (DTLS connect) | Không tới được vBond; sai org-name; chứng chỉ hết hạn/sai; đồng hồ lệch NTP; chặn UDP 12346-12446; DNS |
| Control up, không có data tunnel | Thiếu `color` phù hợp; NAT chặn IPsec (UDP 12346+); firewall chặn ESP/UDP 4500; `restrict` |
| Có tunnel, không có route | OMP không `advertise`; VPN khác; control policy lọc; site-id trùng |
| Thiết bị không onboard | Serial number chưa có trong whitelist, sai Smart Account, license/certificate |

# SD-WAN nâng cao (Cisco Catalyst SD-WAN trọng tâm)

## 1. Policy framework
Hai họ policy × hai phạm vi áp dụng:

| | Centralized (trên Controller) | Localized (trên Edge) |
|---|---|---|
| **Control policy** | Điều khiển route/TLOC OMP (topology, hub-spoke, service chaining) | Route-policy (BGP/OSPF cục bộ) |
| **Data policy** | Điều khiển data traffic theo 5-tuple/app (AAR, QoS, NAT/DIA, firewall, PBR) | ACL, QoS, mirror |

Thành phần policy: **Lists** (site, VPN, prefix, TLOC, app, data-prefix, color) → **Policy definitions** (control, app-route, data, cflowd) → **Apply policy** (`apply-policy site-list ... control-policy ... in/out`, `data-policy ... from-service / from-tunnel / all`, `app-route-policy`).

### Ví dụ 1: Hub-and-spoke bằng Control policy
Mục tiêu: spoke chỉ đi qua Hub, không dựng tunnel trực tiếp spoke↔spoke.
```
policy
 lists
  site-list SPOKES  site-id 11-100
  site-list HUB     site-id 1
  tloc-list HUB-TLOC  tloc 10.255.0.1 color mpls encap ipsec preference 100
 control-policy HUB-SPOKE
  sequence 10
   match route
    site-list SPOKES
   action accept
    set tloc-list HUB-TLOC       ! ép next-hop = TLOC của hub
  default-action accept
apply-policy
 site-list SPOKES
  control-policy HUB-SPOKE out
```
Kết quả: controller viết lại next-hop của route spoke → hub; spoke nhìn thấy mọi destination qua hub. (Hub phải re-advertise/summary.)

### Ví dụ 2: Application-Aware Routing (SLA)
```
policy
 sla-class VOICE
  loss 1
  latency 150
  jitter 30
 app-route-policy AAR-VOICE
  vpn-list CORP-VPN
   sequence 10
    match app-list VOICE-APPS
    action sla-class VOICE preferred-color mpls backup-preferred-color biz-internet
   default-action sla-class DEFAULT
 apply-policy site-list ALL app-route-policy AAR-VOICE
```
`sla-class ... strict` = drop nếu không đạt. Điều chỉnh BFD: `app-route poll-interval 120000 poll-multiplier 5` cho phản ứng nhanh hơn (đánh đổi: tốn băng thông probe, flap).

### Ví dụ 3: Data policy cho DIA (Internet breakout)
```
data-policy DIA
 vpn-list CORP-VPN
  sequence 10
   match app-list O365
   action accept nat use-vpn 0       ! thoát VPN 0 trực tiếp (NAT tại cổng WAN)
  default-action accept
```
Cần `nat` trên interface transport (`ip nat outside`), cho phép firewall/zone-based, hoặc dùng **SIG tunnel** (Zscaler/Umbrella) thay vì DIA thuần.

## 2. Dịch vụ định tuyến service-side (LAN)
Edge hỗ trợ OSPF, BGP (eBGP với LAN/MPLS PE), EIGRP (cEdge), static, connected. Quảng bá vào OMP bằng `omp / advertise ospf|bgp|connected|static`.
**Ví dụ BGP với MPLS CE (VPN 0 tới PE):**
```
router bgp 65100
 bgp log-neighbor-changes
 neighbor 192.0.2.1 remote-as 7018
 address-family ipv4 unicast
  neighbor 192.0.2.1 activate
 !  (transport side: advertise loopback/system prefix)
sdwan
 omp
  address-family ipv4
   advertise bgp
```
Tránh vòng lặp khi site có cả MPLS và SD-WAN: dùng **route-policy, OMP tag/ site-type, `route-tag`, `ignore` OSPF sham** ; thiết kế **transit site** cẩn thận (bgp as-override, community).
Tính toán ưu tiên OMP: **OMP route preference** (path-selection): valid → AD/OMP-preference → TLOC pref → origin protocol → ... Mặc định **ECMP** tối đa 4–16 path (`omp ecmp-limit`).

## 3. Segmentation & multi-tenancy
- **VPN** = VRF, cách ly hoàn toàn. VPN 0 = transport, VPN 512 = OOB management.
- **Route leaking** qua control policy hoặc qua firewall/hub (service chain).
- **Service chaining**: chèn firewall/IPS vào path: edge của dịch vụ advertise **service route** `service FW vpn 1`, control policy chỉ dẫn `set service FW vpn 1`.
- **Multi-tenancy** (Manager multitenant: tenants, per-tenant Controller).

## 4. Bảo mật
- **Mã hóa IPsec AES-256-GCM** mặc định; khóa phân phối qua OMP (không IKE giữa edge, giảm CPU, có thể rekey 24 giờ).
- **Enterprise firewall zone-based, IPS/IDS (Snort), URL filtering, AMP, DNS security (Umbrella)** tích hợp trên cEdge.
- **SIG** (Secure Internet Gateway): tunnel IPsec/GRE tới Zscaler/Umbrella/Prisma Access; policy chuyển traffic Internet vào tunnel.
- **AAA, RBAC** trên Manager; **certificate** (enterprise CA hoặc Cisco PKI); **Allow-list serial**.
- **Control-plane protection**: `allow-service` per-interface (chỉ mở dịch vụ cần), ACL.
- **Hardening**: tắt SSH/HTTP từ Internet, đồng hồ NTP, TLS 1.2+.

## 5. NAT, Cloud, Colocation
- **NAT traversal**: edge sau NAT dùng STUN-like thông qua Validator để biết public IP:port; tunnel dùng UDP 12346 + offset khi nhiều tunnel (12366, ...). Symmetric NAT ↔ symmetric NAT khó (cần đi qua relay/hub).
- **Cloud OnRamp for SaaS**: probe nhiều đường tới Microsoft 365/Salesforce..., chọn path tốt nhất (DIA, qua gateway).
- **Cloud OnRamp for IaaS/Multicloud**: dựng edge ảo trong VPC/VNet, transit gateway; **Cloud Interconnect Fabric** (Equinix, Megaport) cho kết nối chuyên dụng.
- **Colo / SD-Cloud Interconnect**: Hub tại carrier-neutral facility.

## 6. QoS
- 8 queue trên cEdge (class-map/policy-map IOS-XE). Chuẩn hóa DSCP (EF voice, AF41 video, AF21 critical).
- **Hierarchical QoS** (shaper trên interface WAN + queue con), **per-tunnel QoS** (nhiều mức bandwidth khác nhau).
- Markings: `set dscp` trong data policy; ECN; LLQ (priority) ≤ ~33% băng thông.
- **Adaptive QoS** (điều chỉnh shaper theo băng thông thực), **FEC** (Forward Error Correction) và **packet duplication** cho voice/video trên đường lossy; **TCP optimization** (SD-WAN WAN opt, TCP proxy) tuỳ platform.

## 7. Scale & High Availability
- **Controller**: ≥2 (mỗi controller phục vụ ~ vài nghìn–hàng chục nghìn edge tuỳ phiên bản); edge kết nối tới tối đa N controller (`max-controllers`).
- **Manager cluster** (3 node, cluster DB); **Validator** ≥2 (DNS round-robin).
- **Site HA**: 2 edge/site, VRRP ở LAN, OMP/OSPF cân bằng; **tracker** để failover (`tracker` HTTP/ICMP).
- **Transport**: ≥2 color; **TLOC preference** & **weight** điều khiển ECMP/active-standby; `tloc-extension` giữa 2 edge cùng site (một edge dùng transport của edge kia).
- **Region/MRF (Multi-Region Fabric)**: chia mạng thành regions với Border Router; giảm số tunnel, tối ưu đường backbone (ví dụ qua Equinix).
- **Graceful restart** OMP (mặc định 12 giờ); edge giữ route khi mất controller.

## 8. Cấu hình hiện đại: Configuration Groups, Policy Groups
Từ 20.12+ (UX 2.0): **Configuration Groups** thay feature template, **Policy Groups** hợp nhất centralized/localized, **Sites & Devices**, **Workflows** (Quick Connect, Cloud). Python SDK `catalystwan`, REST API `/dataservice/...`, Ansible, Terraform provider.
```
# Ví dụ API: liệt kê device
GET https://<manager>/dataservice/device
Header: X-XSRF-TOKEN, Cookie: JSESSIONID
```

## 9. Troubleshoot flow nâng cao
1. **Control plane**: `show sdwan control connections` — mỗi controller state `up`. Nếu `connect`/`challenge`/`tear_down` → xem *local-error* và *remote-error* (CRTREJSER, DISTLOC, ORGNMMIS, VM_TMO, HTBMISS...).
2. **Certificates**: `show sdwan certificate installed/serial/validity`.
3. **OMP**: `show sdwan omp summary`, `... routes`, `... tlocs`, `... peers`; kiểm tra policy với `show sdwan policy from-vsmart`.
4. **Data plane**: `show sdwan bfd sessions`, `show sdwan ipsec inbound-connections`; `show sdwan tunnel statistics`.
5. **AAR**: `show sdwan app-route stats`, `show sdwan app-route sla-class`; kiểm tra tunnel SLA vi phạm.
6. **Forwarding**: `show ip route vrf 1`, `show sdwan omp routes <prefix> detail`, `show platform packet-trace` (Trace packet) / `show sdwan policy service-path vpn 1 interface Gi2 source-ip ... dest-ip ... protocol 6 app ...` (**service-path** & **tunnel-path** cho dự đoán nexthop).
7. **Packet capture**: Manager UI → Tools → Packet Capture, hoặc `monitor capture`.

## 10. Ví dụ thiết kế đầy đủ
Công ty 200 chi nhánh, 2 DC (Hà Nội, HCM), AWS, SaaS (O365).
- **Controller + Validator + Manager** đặt tại cloud/DC; dự phòng.
- **Transports**: MPLS (color mpls) + Internet (biz-internet) tại DC & chi nhánh lớn; chi nhánh nhỏ 1 Internet + 5G (lte).
- **Topology**: Region VN-North (HN hub) / VN-South (HCM hub), MRF; spoke → regional hub; hub↔hub qua MPLS.
- **Segments**: VPN 10 corp, VPN 20 guest, VPN 30 IoT (cách ly), VPN 40 server.
- **Policy**: AAR voice → MPLS pref; video → biz-internet với FEC; O365 → DIA (Cloud OnRamp SaaS); guest → SIG (Umbrella).
- **Security**: zone firewall tại DIA, SIG cho guest, MACsec ở DC, RBAC cho NOC.
- **Cloud**: edge ảo ở AWS Transit Gateway cho VPC prod; route advertise.
- **Vận hành**: configuration groups theo role (branch-small/branch-large/hub), ZTP/PnP, CI/CD cho policy.

## 11. So sánh Fortinet/VeloCloud/… về nâng cao
- **Fortinet**: SD-WAN rule (Manual, Best Quality, Lowest Cost (SLA), Maximize Bandwidth), health-check (ping/http/dns) per member, **ADVPN** (tunnel động spoke↔spoke), BGP on loopback / IPsec overlay, `diagnose sys sdwan ...`.
- **VeloCloud**: DMPC (Dynamic Multipath Optimization), Gateway cloud-hosted, business policy (QoE).
- **Versa**: SD-WAN + SASE tích hợp, VOS (Versa Operating System).
- Điểm chung: tunnel overlay, SLA monitoring, app-ID, central policy.

## 12. Best practices
- Chuẩn hóa **system-ip/site-id/colors** từ đầu (khó đổi sau).
- Giữ **VPN 0** tối giản; mở `allow-service` tối thiểu.
- Đồng bộ NTP, theo dõi hạn certificate.
- Thử policy ở site pilot; dùng `show sdwan policy` để verify.
- Tham số BFD/SLA không quá hung hăng (tránh flap, chi phí probe).
- Monitoring: SNMP/Syslog/Netflow (cflowd), ThousandEyes cho visibility underlay.

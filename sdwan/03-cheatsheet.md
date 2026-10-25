# SD-WAN Cheatsheet (Cisco Catalyst SD-WAN, cEdge IOS-XE)

> Lệnh theo IOS-XE SD-WAN 17.x. Trên vEdge (Viptela OS) bỏ `sdwan` ở giữa (vd `show control connections`, `show omp routes`, `show bfd sessions`). Tên lệnh có thể khác nhẹ theo phiên bản, dùng `?` để xác nhận.

## A. Kiểm tra control plane
```
show sdwan control connections                  # controller/manager/validator: state up?
show sdwan control connections detail
show sdwan control connection-history           # lịch sử + local/remote error
show sdwan control local-properties             # system-ip, site-id, org, cert, vbond, TLOC IP
show sdwan control affinity config|status
show sdwan certificate installed | serial | validity | root-ca-cert
show sdwan running-config | show sdwan version
show sdwan system status                        # uptime, CPU, mem, personality
show sdwan ntp associations / show clock
```
Mã lỗi thường gặp (connection-history):
| Code | Ý nghĩa |
|---|---|
| `DCONFAIL` | Không kết nối được (mạng/firewall/NAT/DNS) |
| `CRTREJSER` | Serial không có/bị chặn trong whitelist |
| `ORGNMMIS` | Organization name khác nhau |
| `SERNTPRES` | Serial number chưa hợp lệ/ NTP |
| `VM_TMO` / `VP_TMO` | Timeout tới Manager/Controller |
| `HTBMISS` | Mất heartbeat |
| `BIDNTVRFD` | Board ID chưa xác minh |
| `CTORGNMMIS` | Org-name cert không khớp |
| `LISFD` | Hết license/ chứng chỉ |
| `NOVMCFG` | Chưa có config |
| `TMRALC` | Hết bộ nhớ/tài nguyên |

## B. Kiểm tra OMP
```
show sdwan omp summary                          # peers up, routes/TLOC received-installed-sent
show sdwan omp peers
show sdwan omp routes [vpn 1] [<prefix>] [detail]
show sdwan omp received-routes | advertised-routes
show sdwan omp tlocs [detail]
show sdwan omp services
show sdwan omp tloc-paths
show sdwan omp multicast-routes
show sdwan omp cloudexpress
```
Cờ trạng thái route: `C` chosen, `I` installed, `Red` redistributed, `Rej` rejected, `L` looped, `R` resolved, `S` stale, `Ext` extranet, `Inv` invalid, `Stg` staged.
Route Rej/Inv → thường do policy hoặc TLOC unresolved (không có BFD/tunnel tới TLOC đó).

## C. Kiểm tra Data plane
```
show sdwan bfd sessions                          # state up, color pair, tx/rx, uptime
show sdwan bfd summary
show sdwan bfd history
show sdwan ipsec inbound-connections | outbound-connections
show sdwan ipsec pwk (pairwise keys)
show sdwan tunnel statistics [bfd | ipsec | gre]
show sdwan tunnel sla-class                      # trạng thái SLA mỗi tunnel
show sdwan tunnel path-mtu
show sdwan app-route stats                       # loss/latency/jitter từng tunnel
show sdwan app-route sla-class
show sdwan app-fwd cflowd flows
show sdwan policy service-path vpn 1 interface GigabitEthernet3 source-ip 10.1.1.10 dest-ip 10.2.2.20 protocol 6 dest-port 443 all   # dự đoán path
show sdwan policy tunnel-path vpn 1 ...
show sdwan policy from-vsmart                    # policy nhận từ controller
show sdwan policy data-policy-filter
show sdwan appqoe ...
show sdwan reverse-proxy ...
```
### Đọc `show sdwan bfd sessions`
```
SYSTEM IP    SITE ID STATE  SOURCE TLOC COLOR  REMOTE TLOC COLOR   SOURCE IP     DST PUBLIC IP   PORT  ENCAP  DETECT-MULT TX-INTVL UPTIME
10.255.0.1   1       up     biz-internet       biz-internet        10.0.0.2      198.51.100.9    12346 ipsec  7           1000     0:05:11:22
```
`down` → xem NAT, firewall (UDP 12346-12446, 4500), `color` + `restrict`, certificate, clock.

## D. Interface & routing (VPN)
```
show ip interface brief
show sdwan interface [vpn 0]
show ip route vrf 1  |  show ip route (VPN 0)
show ip bgp summary | show ip ospf neighbor
show sdwan omp routes vpn 1
show sdwan running-config | section sdwan
show sdwan policy ...
ping vrf 1 10.2.2.20 source 10.1.1.1
traceroute vrf 1 10.2.2.20
show platform hardware qfp active ... (data path)
show platform packet-trace ...
```
### Packet trace
```
debug platform condition ipv4 10.1.1.10/32 both
debug platform packet-trace packet 128 fia-trace data-size 2048
debug platform condition start
show platform packet-trace summary
show platform packet-trace packet 0 decode
debug platform condition stop ; clear platform condition all
```

## E. Cấu hình mẫu
### System/Transport
```
system
 system-ip 10.255.0.11
 site-id 11
 organization-name "ORG"
 vbond vbond.example.com
!
sdwan
 interface GigabitEthernet1
  tunnel-interface
   encapsulation ipsec weight 1
   color mpls restrict
   max-control-connections 1
   allow-service all|dhcp|dns|icmp|sshd|netconf|ntp|stun|https
   nat-refresh-interval 5
   hello-interval 1000
   hello-tolerance 12
  tloc-extension GigabitEthernet2
  bandwidth-upstream 50000 ; bandwidth-downstream 100000
 omp
  graceful-restart ; ecmp-limit 4 ; send-path-limit 4
  advertise connected|static|bgp|ospf|ospfv3
```
### BFD / SLA
```
sdwan
 app-route poll-interval 600000
 app-route poll-multiplier 6
 bfd color biz-internet hello-interval 1000 multiplier 7 pmtu-discovery
```
### Template CLI nhanh (Device CLI template, ví dụ)
```
vrf definition 10
 rd 1:10
 address-family ipv4
  route-target export 1:10
  route-target import 1:10
interface GigabitEthernet5
 vrf forwarding 10
 ip address 10.1.10.1 255.255.255.0
 no shutdown
```
### Control policy: hub-spoke / tag
```
policy
 control-policy NAME
  sequence 1 ; match route ; prefix-list ... ; site-list ... ; action accept ; set tloc-list HUB ; (set service ...)
  default-action accept
apply-policy
 site-list SPOKES ; control-policy NAME out
```
### Data policy / DIA
```
policy
 data-policy DP
  vpn-list V1
   sequence 1 ; match app-list O365 ; action accept ; nat use-vpn 0 ; (set local-tloc color biz-internet)
   default-action accept
apply-policy
 site-list ALL ; data-policy DP from-service
```
### Localized QoS
```
class-map match-any VOICE ; match dscp ef
policy-map WAN-QOS
 class VOICE ; priority percent 30
 class class-default ; fair-queue
policy-map SHAPER ; class class-default ; shape average 50000000 ; service-policy WAN-QOS
interface GigabitEthernet1 ; service-policy output SHAPER
```
### Reset / Quản trị
```
request platform software sdwan config reset              # factory-reset cấu hình SD-WAN
request platform software sdwan vedge_cloud activate chassis-number X token Y
request platform software sdwan port_hop color biz-internet
request platform software sdwan control-connection clear  # (tên có thể khác)
clear sdwan control connections / clear sdwan omp all
request security ... 
show sdwan software
```
Từ Manager: Tools → SSH Terminal, Ping/Traceroute, Packet Capture, Rediscover Network, Device Bring-Up; Monitor → Devices → Real-Time (các lệnh show qua UI).

## F. Quy trình troubleshoot
**1. Edge không lên control (Manager UI: Offline / không DTLS)**
1. `show ip interface brief`, default route trong VPN 0 / DHCP? `ping 8.8.8.8` & `ping vbond`. 
2. DNS resolve vbond: `ping vbond.example.com`; `show running-config | include vbond`.
3. `show sdwan control connections` + `connection-history`: đọc `local-error`/`remote-error`.
4. Kiểm tra thời gian: `show clock` / NTP; chứng chỉ: `show sdwan certificate validity`.
5. Org name, serial/whitelist, license; vBond hoạt động: Manager > Configuration > Devices.
6. Firewall: UDP 12346 → 12446 (DTLS), TCP 23456+ nếu TLS, HTTPS 443 tới Manager.
7. `allow-service` trên tunnel-interface (dhcp, dns, icmp).

**2. Control up, BFD down**
1. `show sdwan bfd sessions` – color cặp? có `restrict`? 
2. `show sdwan omp tlocs` – TLOC remote hiện không?
3. NAT: loại NAT? `show sdwan control local-properties` xem private/public IP-port, `nat-type`.
4. Firewall ISP/MPLS chặn UDP 12346+/4500/ESP.
5. `show sdwan ipsec outbound-connections` có SPI? Key rekey kẹt: `request platform software sdwan ... rekey`.

**3. Tunnel up, không thấy route**
1. `show sdwan omp summary`: peers up, `routes received/installed`.
2. VPN đúng? `advertise connected/ospf/bgp` có?
3. Control policy lọc: `show sdwan policy from-vsmart`, `show sdwan omp routes detail` (status Rej/Inv).
4. Site-ID/system-ip trùng, TLOC unresolved.
5. `show ip route vrf N` – route có AD thấp hơn?

**4. Traffic đi sai đường**
1. `show sdwan policy service-path` để dự đoán path.
2. `show sdwan app-route stats` & `sla-class`: tunnel nào vi phạm SLA.
3. Kiểm tra `preferred-color`, `fallback`, TLOC `preference`, `weight`, ECMP.
4. App-ID chưa nhận dạng (DPI/NBAR) – `show sdwan app-fwd dpi flows`.

**5. Chậm / mất gói**
1. `show sdwan tunnel statistics`, `app-route stats` (loss/latency/jitter), `show sdwan tunnel path-mtu` (MTU/MSS: set `tcp-mss-adjust 1350`).
2. QoS shaper đúng với băng thông ISP? `show policy-map interface`.
3. Underlay: traceroute/ThousandEyes, speed test.
4. CPU/Mem: `show processes cpu platform sorted`, throughput license.

## G. Port & thông số
| Item | Giá trị |
|---|---|
| DTLS control | UDP **12346** (+offset theo core, 12366, 12386, ...) |
| TLS control | TCP 23456+ |
| IPsec data | UDP 12346+ (NAT-T 4500 cũng dùng) / ESP 50 |
| Manager UI/API | HTTPS 443 / 8443; NETCONF 830 |
| BFD hello / multiplier | 1s / 7 |
| App-route poll / multiplier | 600 s / 6 |
| OMP graceful-restart | 43200 s |
| Rekey | 24 giờ (mặc định) |
| ECMP | mặc định 4 path |
| VPN | 0 transport · 512 management · 1–511 service |

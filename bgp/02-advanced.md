# BGP nâng cao

## 1. Route Reflector (RR)
**Vấn đề:** iBGP split-horizon bắt buộc full-mesh: n router → n(n-1)/2 session (100 router = 4950).
**Giải pháp:** RR nhận route từ client và *phản chiếu* cho các client khác.

Quy tắc reflect:
| Route học từ | Gửi tới |
|---|---|
| Client | Tất cả client (trừ nguồn) + non-client |
| Non-client iBGP | Chỉ client |
| eBGP | Client + non-client |

Chống loop: **ORIGINATOR_ID** (router-id của nguồn) và **CLUSTER_LIST** (danh sách cluster-id đã đi qua).
```
router bgp 65000
 neighbor 10.0.0.1 remote-as 65000
 address-family ipv4
  neighbor 10.0.0.1 route-reflector-client
```
Thiết kế: dùng ≥2 RR dư phòng, cùng `cluster-id` hoặc khác (khác = có thêm đường dự phòng, nhưng nhiều route hơn); RR nên nằm trên đường chuyển tiếp hoặc đặt dedicated; hierarchical RR cho mạng lớn.

## 2. Confederation
Chia AS lớn thành các **sub-AS** (private ASN), bên ngoài vẫn thấy một AS.
```
router bgp 65001
 bgp confederation identifier 100
 bgp confederation peers 65002
```
Giữa sub-AS dùng "eBGP confederation" (giữ LOCAL_PREF, NEXT_HOP). AS_PATH có AS_CONFED_SEQUENCE. Dùng ít hơn RR.

## 3. Communities
Nhãn gắn vào route để áp chính sách. Dạng `ASN:giá trị` (32-bit); **Extended** (RT/SoO), **Large** (RFC 8092, 12 byte `ASN:x:y`, hỗ trợ ASN 32-bit).

Well-known:
- `no-export`: không gửi ra ngoài AS (eBGP). 
- `no-advertise`: không gửi cho bất kỳ peer.
- `local-AS` (no-export-subconfed): không ra ngoài sub-AS.
- `internet`: mặc định.
- `graceful-shutdown` (65535:0), `blackhole` (65535:666, RFC 7999).

```
ip community-list standard CL-CUST permit 65001:100
route-map FROM-ISP permit 10
 match community CL-CUST
 set local-preference 200
route-map TO-ISP permit 10
 set community 65001:100 additive
!
neighbor 10.1.1.1 send-community both   ! bắt buộc để community được gửi
```
**Ví dụ thực tế:** ISP công bố "65000:80 = set LP 80 (backup), 65000:3 = prepend 3 lần tới Châu Á, 65000:666 = blackhole" → khách hàng gắn community để điều khiển chính sách của ISP.

## 4. Điều khiển đường vào / đường ra (Traffic Engineering)
| Muốn | Dùng | Chiều |
|---|---|---|
| Chọn đường **ra** | LOCAL_PREF, Weight | Outbound |
| Gợi ý đường **vào** | AS-PATH prepend, MED, communities ISP, advertise more-specific | Inbound |

**Prepend:** `set as-path prepend 65001 65001 65001` trên route-map outbound tới ISP-backup → đường dài hơn nên ít được chọn.
**MED:** `set metric 50` outbound/ inbound (peer cùng AS).
**More-specific:** advertise /24 qua link chính, /23 qua link phụ → longest-match luôn thắng, dù AS_PATH.
**Load-balancing:** `maximum-paths 4` (eBGP) / `maximum-paths ibgp 4`; cần các attribute bằng nhau; `bgp bestpath as-path multipath-relax` cho AS khác nhau cùng độ dài; DMZ link bandwidth để cân bằng tỷ lệ.

## 5. Filtering & Policy
Công cụ: **prefix-list** (match prefix/length), **as-path access-list** (regex), **community-list**, **route-map** (match/set, permit/deny, sequence), **distribute-list** (legacy), **filter-list**.

```
ip prefix-list CUST-IN seq 5 permit 203.0.113.0/24 le 24
ip prefix-list BOGON seq 5 deny 0.0.0.0/8 le 32
ip as-path access-list 10 permit ^65010(_65010)*$      ! chỉ route do AS65010 sinh ra
route-map CUST-IN permit 10
 match ip address prefix-list CUST-IN
 set local-preference 150
! route-map cuối ngầm deny all
```
Regex AS-path: `^$` (route nội bộ), `^65010_` (từ láng giềng 65010), `_65010$` (xuất xứ 65010), `_65010_` (đi qua 65010), `.*` (tất cả).
Prefix-list: `ge/le`: `10.0.0.0/8 ge 24 le 24` khớp mọi /24 trong 10/8.

**Best practice an toàn (peer/ISP):** 
- Chỉ nhận prefix của khách hàng (prefix-list/IRR), chặn bogon + default (trừ khi cần), chặn ASN private trong AS_PATH.
- `maximum-prefix 1000 80 warning-only` / restart.
- `ttl-security hops 1` (GTSM, RFC 5082), MD5 hoặc TCP-AO (RFC 5925).
- RPKI ROV, BGPsec chưa phổ biến, MANRS.

## 6. Route Origin Validation (RPKI)
ROA = ({prefix, max-length, origin ASN}) ký bởi RIR. Router hỏi RPKI cache (RTR protocol, port 323/8282) → trạng thái **valid / invalid / not-found**. Chặn invalid.
```
router bgp 65001
 bgp rpki server tcp 192.0.2.50 port 3323 refresh 600
 address-family ipv4
  bgp bestpath prefix-validate disable   ! hoặc allow-invalid / chọn theo policy
route-map RPKI permit 10
 match rpki invalid
 set local-preference 0 / hoặc deny
```

## 7. Tối ưu hội tụ & độ ổn định
- **BFD**: phát hiện lỗi link <1s. `neighbor X fall-over bfd`.
- **Timers**: `neighbor X timers 10 30`; Fast external fallover (eBGP link down).
- **Graceful Restart / NSR**: giữ forwarding khi control plane restart; **LLGR** (long-lived GR).
- **Route dampening**: phạt route flap (ít dùng ở IPv4 hiện đại, RIPE khuyến cáo hạn chế).
- **Add-Paths** (RFC 7911): advertise nhiều path, giúp RR gửi đường dự phòng → hội tụ nhanh.
- **PIC (Prefix Independent Convergence)**: pre-install backup path.
- **Best-external**: advertise best eBGP dù best là iBGP.
- **Soft reconfiguration / Route Refresh**: áp lại policy không reset. Ưu tiên route-refresh (capability) thay vì `soft-reconfiguration inbound` (tốn RAM).
- **Update groups / peer-groups / peer-templates**: giảm CPU.
- **Dynamic neighbor**: `bgp listen range 10.0.0.0/24 peer-group P`.

## 8. Address Family & MP-BGP (RFC 4760)
MP-BGP mang nhiều loại NLRI qua MP_REACH_NLRI / MP_UNREACH_NLRI:
- IPv4/IPv6 unicast, multicast
- **VPNv4/VPNv6** (MPLS L3VPN), **L2VPN EVPN** (xem phần VXLAN), **BGP-LS** (link-state cho SDN/PCE), **IPv4 labeled-unicast** (Seamless MPLS/Inter-AS option C), **FlowSpec** (lọc DDoS qua BGP), **MVPN**, **SR-TE policy**.
Mỗi AF có thể yêu cầu `activate` riêng; capability thương lượng trong OPEN.

### MPLS L3VPN (tóm tắt)
- **VRF** tách bảng route; **RD** (Route Distinguisher) làm prefix duy nhất (8 byte, `ASN:nn`); **RT** (Route Target, extended community) điều khiển import/export.
- PE-CE: eBGP/OSPF/static; PE-PE: MP-iBGP VPNv4 (qua RR).
```
vrf definition CUST-A
 rd 65000:1
 address-family ipv4
  route-target export 65000:1
  route-target import 65000:1
router bgp 65000
 address-family vpnv4
  neighbor 10.255.0.2 activate
  neighbor 10.255.0.2 send-community extended
 address-family ipv4 vrf CUST-A
  neighbor 172.16.1.2 remote-as 65100
  neighbor 172.16.1.2 activate
```
Lưu ý: CE cùng ASN ở nhiều site → `allowas-in` (CE) hoặc `as-override` (PE).

## 9. Các tính năng khác
- **Aggregation:** `aggregate-address 10.0.0.0 255.255.0.0 summary-only [as-set]`; ATOMIC_AGGREGATE/AGGREGATOR; `as-set` giữ thông tin AS_PATH.
- **Conditional advertisement:** `advertise-map`/`non-exist-map` (chỉ gửi route A nếu route B không có - dự phòng).
- **Default originate:** `neighbor X default-originate [route-map]`.
- **Remove-private-as / local-as / replace-as / dual-as:** khi migrate ASN.
- **Allowas-in / as-override:** vượt cơ chế chống loop AS_PATH có chủ đích.
- **Backdoor:** `network x mask y backdoor` (ưu tiên IGP).
- **Next-hop-unchanged**, **next-hop recursion policy** (`bgp nexthop trigger delay`).
- **Peer session templates, TCP-AO, TCP MSS / path-mtu-discovery**.
- **Anycast/ECMP với BGP trong Data Center (RFC 7938):** eBGP-only Clos, mỗi leaf 1 ASN riêng hoặc ASN chung theo tầng, `allowas-in`, `bestpath as-path multipath-relax`, **BGP unnumbered** (RFC 5549, IPv6 LLA làm next-hop).
- **BGP Egress/Ingress Peer Engineering (EPE)** với Segment Routing, **BMP** (RFC 7854) để giám sát, **BGP Monitoring**, telemetry.

## 10. BGP & Internet: sự cố điển hình
- **Route leak**: AS A học route từ transit X rồi quảng bá cho transit Y → kéo traffic. Phòng: prefix-list, communities, max-prefix, peer-lock, RFC 9234 (Role / OTC attribute).
- **Hijack**: AS khác quảng bá prefix của bạn (hoặc more-specific). Phòng: RPKI ROA, monitoring (BGPStream, RIPE RIS), more-specific.
- **Blackhole (RTBH)**: community 65535:666 hoặc của ISP để drop traffic DDoS tại biên ISP.

## 11. Ví dụ tổng hợp: Dual-homed multi-ISP
```
        ISP-A (AS 64500)      ISP-B (AS 64501)
              \                /
   10.1.1.0/30 \              / 10.2.2.0/30
                [ R-EDGE AS 65001 ]
                      |  192.0.2.0/24
```
Yêu cầu: A là chính (outbound & inbound), B là backup.
```
route-map FROM-A permit 10
 set local-preference 200
route-map FROM-B permit 10
 set local-preference 100
route-map TO-B permit 10
 set as-path prepend 65001 65001 65001
router bgp 65001
 neighbor 10.1.1.1 remote-as 64500
 neighbor 10.2.2.1 remote-as 64501
 address-family ipv4
  network 192.0.2.0 mask 255.255.255.0
  neighbor 10.1.1.1 route-map FROM-A in
  neighbor 10.2.2.1 route-map FROM-B in
  neighbor 10.2.2.1 route-map TO-B out
```
Kết quả: traffic ra ưu tiên A (LP 200), vào ưu tiên A (path B dài hơn). Nếu A rớt, B gánh. Lưu ý: **ISP có thể ghi đè** bằng LP riêng → dùng community của ISP cho hiệu quả chắc chắn.

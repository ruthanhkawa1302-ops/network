# BGP Cheatsheet

## A. Kiểm tra (Cisco IOS-XE)
```
show ip bgp summary                      # trạng thái neighbor, PfxRcd, Up/Down
show bgp all summary                     # mọi address-family
show ip bgp neighbors 10.12.0.2          # chi tiết: state, timers, capabilities, lý do reset
show ip bgp neighbors 10.12.0.2 advertised-routes
show ip bgp neighbors 10.12.0.2 received-routes     # cần soft-reconfig inbound
show ip bgp neighbors 10.12.0.2 routes              # route nhận được và chấp nhận
show ip bgp                              # toàn bảng BGP
show ip bgp 192.0.2.0/24                 # chi tiết prefix: attribute, best path, multipath, lý do
show ip bgp 192.0.2.0/24 bestpath
show ip bgp regexp _65002$               # lọc theo AS-path
show ip bgp community 65001:100          # lọc theo community
show ip bgp cidr-only | longer-prefixes 10.0.0.0/8
show ip bgp filter-list 10
show ip bgp rib-failure                  # route BGP không vào được RIB (vì AD thấp hơn)
show ip bgp update-group summary         # update group
show ip bgp peer-group
show ip route bgp
show ip bgp ipv6 unicast summary
show bgp vpnv4 unicast all summary / show bgp vpnv4 unicast vrf CUST-A
show bgp l2vpn evpn summary
show ip bgp template peer-session
show ip bgp attr nexthop
show ip bgp scan                         # next-hop scan timer
```
### Đọc `show ip bgp`
```
     Network          Next Hop     Metric LocPrf Weight Path
 *>  192.0.2.0/24     10.12.0.1         0            0 65001 i
 * i 198.51.100.0/24  10.0.0.9          0    100     0 i
```
- `*` valid, `>` best, `i` học từ iBGP, `r` RIB-failure, `s` suppressed, `d` damped, `h` history, `S` stale.
- Cuối dòng: `i` IGP, `e` EGP, `?` incomplete.

### Trạng thái neighbor
| show summary | Ý nghĩa |
|---|---|
| Idle | Chưa khởi tạo / admin down / không có route tới peer |
| Idle (Admin) | `shutdown` neighbor |
| Active | TCP chưa thành công |
| Idle (PfxCt) | Vượt max-prefix |
| Số | Established, số prefix nhận |

## B. Cấu hình
```
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 no bgp default ipv4-unicast                    # bắt buộc activate thủ công theo AF
 neighbor X remote-as N
 neighbor X description ...
 neighbor X update-source Loopback0
 neighbor X ebgp-multihop 2
 neighbor X password SECRET
 neighbor X ttl-security hops 1
 neighbor X timers 10 30
 neighbor X fall-over bfd
 neighbor X shutdown
 address-family ipv4 unicast
  network A.B.C.D mask M
  neighbor X activate
  neighbor X next-hop-self [all]
  neighbor X route-reflector-client
  neighbor X send-community both
  neighbor X soft-reconfiguration inbound
  neighbor X maximum-prefix 1000 80
  neighbor X prefix-list PL in|out
  neighbor X route-map RM in|out
  neighbor X filter-list 10 in|out
  neighbor X default-originate
  neighbor X remove-private-as [all]
  neighbor X allowas-in 1
  neighbor X as-override
  neighbor X weight 100
  maximum-paths 4 / maximum-paths ibgp 4
  aggregate-address 10.0.0.0 255.255.0.0 summary-only
  redistribute connected|static|ospf 1 route-map RM
  bgp additional-paths install / bestpath ...
 bgp bestpath as-path multipath-relax
 bgp always-compare-med
 bgp deterministic-med
 bgp bestpath med missing-as-worst
 bgp listen range 10.0.0.0/24 peer-group DYN
 bgp graceful-restart
 bgp confederation identifier 100 / peers 65002
 bgp cluster-id 1.1.1.1
```
### Route-map / Filter
```
ip prefix-list NAME seq 5 permit 10.0.0.0/8 ge 16 le 24
ip as-path access-list 1 permit ^65002_
ip community-list standard CL permit 65001:100
ip community-list expanded CLX permit 65001:1[0-9]+
route-map RM permit 10
 match ip address prefix-list NAME
 match as-path 1
 match community CL [exact-match]
 set local-preference 200
 set metric 50
 set weight 100
 set as-path prepend 65001 65001
 set community 65001:100 additive
 set ip next-hop 10.0.0.1
 set origin igp
```

## C. Reset / Debug
```
clear ip bgp *                       # hard reset (cẩn thận!)
clear ip bgp 10.12.0.2 soft in       # áp policy inbound (route-refresh)
clear ip bgp 10.12.0.2 soft out
clear ip bgp 10.12.0.2 in prefix-filter
debug ip bgp events | updates | keepalives | ipv4 unicast
debug ip packet detail <acl>          # luôn dùng ACL giới hạn!
terminal monitor
undebug all
```

## D. Quy trình troubleshoot từng bước
1. **Neighbor không lên (Idle/Active)**
   - `ping <neighbor> source <local-ip>`; với loopback: có route tới loopback đối phương? 
   - `telnet <neighbor> 179 /source-interface ...` kiểm tra TCP 179.
   - Đối chiếu `remote-as`, router-id không trùng, `update-source`, `ebgp-multihop`/`disable-connected-check`, `password`, `ttl-security`.
   - Kiểm tra ACL/CoPP/firewall chặn TCP 179 (chiều nào chủ động mở kết nối: IP cao hơn thường là active).
   - `show ip bgp neighbors X | include BGP state|Last reset|error` và log `%BGP-3-NOTIFICATION` (mã/subcode: 2/2 bad peer AS, 2/6 unacceptable hold time, 6/? cease: max-prefix...).
2. **Established nhưng không nhận route**
   - `show ip bgp summary` PfxRcd = 0? → phía kia có `network`/route trong RIB? `advertised-routes` ở đối phương.
   - Có `activate` trong AF? `send-community`? Kiểm tra route-map/prefix-list/filter-list inbound (`show ip bgp neighbors X policy`).
3. **Có route nhưng không best (không có `>`)**
   - Next-hop inaccessible: `show ip route <next-hop>`; iBGP cần `next-hop-self`.
   - AS-path loop (chứa ASN mình) → `allowas-in` nếu có chủ đích.
   - RIB-failure: có route AD thấp hơn (IGP/static) → `show ip bgp rib-failure`.
   - Route bị dampen/suppress: `show ip bgp dampening flap-statistics`.
4. **Chọn sai path**: `show ip bgp <prefix>` xem từng path và "best" theo thứ tự attribute; áp dụng thứ tự Weight→LP→Origin local→AS-path→Origin→MED→eBGP→IGP metric→age→RID.
5. **Route có trong BGP, không forwarding**: `show ip cef <prefix>`, `show ip route <prefix>`, kiểm tra recursion, `traceroute`, MTU/MSS, ACL/uRPF, return path (asymmetric).
6. **Flap**: xem `Last reset`, hold timer expired → CPU/MTU/loss; BFD; `debug ip bgp events`; kiểm tra IGP flap kéo theo next-hop.
7. **Hội tụ chậm**: `bgp scan-time` / nexthop tracking (`bgp nexthop trigger delay 0`), BFD, PIC, timer MRAI (advertisement-interval).

## E. NX-OS khác biệt
```
feature bgp
router bgp 65001
  router-id 1.1.1.1
  address-family ipv4 unicast
    network 10.0.0.0/24
  neighbor 10.0.0.2
    remote-as 65002
    update-source loopback0
    address-family ipv4 unicast
      send-community both
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors 10.0.0.2 advertised-routes
show bgp ipv4 unicast 10.0.0.0/24
show ip bgp neighbors ...
```
## F. Junos / Arista EOS / FRR nhanh
| Việc | Junos | Arista EOS | FRR |
|---|---|---|---|
| Summary | `show bgp summary` | `show ip bgp summary` | `show bgp summary` |
| Prefix | `show route 192.0.2.0/24 protocol bgp detail` | `show ip bgp 192.0.2.0/24` | `show bgp ipv4 unicast 192.0.2.0/24` |
| Advertised | `show route advertising-protocol bgp X` | `show ip bgp neighbors X advertised-routes` | `show bgp neighbor X advertised-routes` |
| Received | `show route receive-protocol bgp X` | `... received-routes` | `... received-routes` |
| Reset soft | `clear bgp neighbor X soft` | `clear ip bgp X soft in` | `clear bgp X soft in` |
Junos mẫu: `set protocols bgp group EXT type external peer-as 65002 neighbor 10.12.0.2 import IMPORT-POL export EXPORT-POL`. EBGP mặc định cần policy (RFC 8212 trên FRR/XR/Junos mới).

## G. Hằng số cần nhớ
TCP 179 · Keepalive 60 / Hold 180 · eBGP AD 20 / iBGP AD 200 · LP mặc định 100 · Weight local 0 (32768 cho route tự sinh) · MRAI eBGP 30s / iBGP 5s (Cisco) · eBGP TTL 1 · Max-prefix thường 80% warning · Well-known community: no-export, no-advertise, local-AS.

# BGP cơ bản

## 1. BGP là gì?
**BGP (Border Gateway Protocol, RFC 4271)** là giao thức định tuyến *path-vector* dùng để trao đổi thông tin đến được (reachability) giữa các **Autonomous System (AS)**. Đây là giao thức "keo dán" của Internet.

**Ẩn dụ:** OSPF/EIGRP giống bản đồ chi tiết trong *một thành phố* (một tổ chức). BGP giống thỏa thuận giữa *các quốc gia*: "tôi đến được những vùng này, đường đi qua các nước này", và mỗi nước tự quyết định chính sách (cho ai đi qua, ưu tiên đường nào).

| Đặc điểm | Ý nghĩa |
|---|---|
| Path-vector | Mỗi route mang danh sách AS đã đi qua (AS_PATH) → chống loop |
| Chạy trên TCP/179 | Tin cậy, không tự discovery; neighbor phải cấu hình tay |
| Policy-based | Chọn đường theo chính sách (attribute), không theo metric thuần |
| Hội tụ chậm, ổn định | Thiết kế cho quy mô khổng lồ (>1 triệu route IPv4 trên Internet) |
| Incremental update | Chỉ gửi thay đổi, không flood định kỳ |

## 2. Autonomous System (AS) & ASN
- **AS**: tập hợp mạng dưới cùng một quản trị, chính sách định tuyến thống nhất.
- **ASN**: số định danh AS. 16-bit (1–65535), 32-bit (RFC 6793, ghi asplain `4200000001` hoặc asdot `64086.59904`).
- Private ASN: `64512–65534` và `4200000000–4294967294`. Documentation: `64496–64511`, `65536–65551`.
- Public ASN do RIR cấp (APNIC, ARIN, RIPE...).

## 3. eBGP vs iBGP
| | eBGP | iBGP |
|---|---|---|
| Giữa | 2 AS khác nhau | Cùng AS |
| AD (Cisco) | 20 | 200 |
| TTL mặc định | 1 (phải kề nhau) | 255 |
| Next-hop khi advertise | Đổi thành địa chỉ mình | **Giữ nguyên** (cần `next-hop-self`) |
| AS_PATH | Thêm ASN mình | Không thêm |
| Chống loop | AS_PATH chứa ASN mình → drop | **Split-horizon**: route học từ iBGP không gửi cho iBGP khác → cần full-mesh/RR |

**Ví dụ topology**
```
 AS 65001            AS 65002
 [R1]----eBGP----[R2]====iBGP====[R3]
 1.1.1.1  10.12.0.0/30   2.2.2.2     3.3.3.3
```
R1 advertise `192.0.2.0/24`. R2 nhận (next-hop 10.12.0.1), gửi tiếp cho R3 qua iBGP với next-hop **vẫn là 10.12.0.1**. R3 không đến được 10.12.0.1 (không có trong IGP) → route "inaccessible". Sửa: trên R2 `neighbor 3.3.3.3 next-hop-self` hoặc đưa subnet 10.12.0.0/30 vào IGP.

## 4. Thiết lập neighbor, Finite State Machine
Các message BGP (5 loại): **OPEN** (ASN, hold time, router-id, capabilities), **KEEPALIVE** (mặc định 60s), **UPDATE** (route mới/withdraw + attributes), **NOTIFICATION** (lỗi, đóng session), **ROUTE-REFRESH**.

Trạng thái:
1. **Idle**: chưa làm gì / chờ retry.
2. **Connect**: đang mở TCP.
3. **Active**: TCP chưa thành công, đang thử lại (*kẹt ở Active = TCP không thông, sai IP/ASN, ACL chặn 179*).
4. **OpenSent**: đã gửi OPEN.
5. **OpenConfirm**: nhận OPEN hợp lệ, chờ KEEPALIVE.
6. **Established**: trao đổi UPDATE.

Hold time thỏa thuận = số nhỏ hơn của hai bên (mặc định 180s, keepalive 60s).
Router-ID: 32-bit, chọn theo `bgp router-id` > loopback IP cao nhất > interface IP cao nhất. Hai router cùng router-id → session không lên.

## 5. Cấu hình cơ bản (Cisco IOS-XE)
Topology: R1 (AS 65001) — R2 (AS 65002), link 10.12.0.0/30.
```
! R1
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 10.12.0.2 remote-as 65002
 neighbor 10.12.0.2 description to-R2
 address-family ipv4 unicast
  network 192.0.2.0 mask 255.255.255.0
  neighbor 10.12.0.2 activate
! Cần tồn tại route 192.0.2.0/24 trong RIB (vd ip route 192.0.2.0 255.255.255.0 Null0)

! R2
router bgp 65002
 bgp router-id 2.2.2.2
 neighbor 10.12.0.1 remote-as 65001
 address-family ipv4 unicast
  neighbor 10.12.0.1 activate
```
Kiểm tra: `show ip bgp summary` → cột State/PfxRcd hiển thị số prefix (số) = Established.

**Quy tắc `network`**: prefix + mask phải **khớp chính xác** một route trong bảng định tuyến (bất kể giao thức), nếu không BGP không advertise.

**iBGP qua loopback** (khuyến nghị để chịu lỗi link):
```
neighbor 3.3.3.3 remote-as 65002
neighbor 3.3.3.3 update-source Loopback0
neighbor 3.3.3.3 next-hop-self
```
eBGP qua loopback cần thêm `ebgp-multihop 2` (hoặc `disable-connected-check`).

## 6. Path Attributes
Bốn nhóm: 
- **Well-known mandatory**: ORIGIN, AS_PATH, NEXT_HOP (luôn có).
- **Well-known discretionary**: LOCAL_PREF, ATOMIC_AGGREGATE.
- **Optional transitive**: COMMUNITY, AGGREGATOR.
- **Optional non-transitive**: MED, ORIGINATOR_ID, CLUSTER_LIST.

| Attribute | Ý nghĩa | Ưu tiên |
|---|---|---|
| Weight (Cisco, local) | Chỉ trên router, không quảng bá | Cao hơn tốt hơn |
| LOCAL_PREF | Trong AS, chọn **đường ra** (mặc định 100) | Cao hơn tốt |
| Locally originated | network/redistribute/aggregate | Ưu tiên route do mình tạo |
| AS_PATH | Danh sách AS | Ngắn hơn tốt |
| ORIGIN | i (IGP) < e (EGP) < ? (incomplete) | Thấp tốt |
| MED | Gợi ý cho AS láng giềng chọn **đường vào** | Thấp tốt |
| eBGP > iBGP | | eBGP thắng |
| IGP metric tới next-hop | | Thấp tốt |
| Router-ID | Tie-break cuối | Thấp tốt |

## 7. Thuật toán chọn best path (Cisco)
Thứ tự: (0) next-hop phải reachable → 1. **Weight** cao → 2. **Local Pref** cao → 3. Locally originated → 4. **AS_PATH** ngắn → 5. **Origin** thấp → 6. **MED** thấp (chỉ so sánh giữa route từ cùng AS láng giềng) → 7. eBGP hơn iBGP → 8. IGP metric tới next-hop thấp → 9. (Multipath nếu bật) → 10. Route cũ nhất (eBGP) → 11. Router-ID thấp → 12. Cluster-list ngắn → 13. Neighbor IP thấp.

Mẹo nhớ: **W**eight → **L**ocal pref → **O**riginate (local) → **A**S-path → **O**rigin → **M**ED → **e**BGP → **I**GP metric → **R**outer-ID. Cụm "We Love Oranges As Oranges Mean Early Id Rotten" là cách nhớ phổ biến.

**Ví dụ**: AS 65001 có 2 đường ra Internet: ISP-A (LP 100) và ISP-B (LP 200). Prefix 8.8.8.0/24 học từ cả hai → chọn ISP-B vì LP 200 thắng, dù AS_PATH của B dài hơn (LP đứng trước AS_PATH).

## 8. Advertise & nhận route
- Một router chỉ advertise **best path** của mình.
- `network`, `redistribute` (ORIGIN: network=i, redistribute=?), `aggregate-address`.
- Mặc định eBGP trên IOS-XE không cần policy (nhưng IOS-XR/Junos nhận/gửi phải có policy → RFC 8212).
- Quy tắc split-horizon iBGP: giải quyết bằng **full mesh**, **Route Reflector**, hoặc **Confederation** (xem nâng cao).

## 9. Tóm tắt khái niệm hay nhầm
- **BGP không tự tìm neighbor** → cấu hình tay (trừ unnumbered/dynamic neighbor).
- **Established không đồng nghĩa có route**: kiểm tra `PfxRcd`, policy, next-hop.
- **Clear session**: `clear ip bgp * soft in/out` (không ngắt TCP) thay vì hard reset.
- **MED so sánh** chỉ khi cùng AS láng giềng (trừ `bgp always-compare-med`).
- **Synchronization** đã bỏ từ IOS 12.2(8)T, mặc định tắt.

## 10. Lỗi thường gặp (cơ bản)
| Triệu chứng | Nguyên nhân hay gặp |
|---|---|
| Kẹt Active/Idle | Không ping được neighbor, sai IP/remote-as, ACL chặn TCP 179, thiếu update-source, eBGP multihop |
| OpenSent ↔ Idle | Sai ASN, router-id trùng, MD5 password lệch, capabilities lệch |
| Established nhưng 0 prefix | Chưa `activate` trong address-family, thiếu `network` match, route-map/prefix-list chặn |
| Route có nhưng không best (`*` không `>`) | Next-hop unreachable (iBGP), AS_PATH loop, kém attribute |
| Flapping | Hold timer hết (CPU/link), MTU/TCP MSS, loopback không route |

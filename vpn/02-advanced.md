# VPN / IPsec nâng cao

## 1. Các biến thể route-based
| Công nghệ | Mô tả | Khi dùng |
|---|---|---|
| **Static VTI (SVTI)** | `tunnel mode ipsec ipv4` + `tunnel protection`; đóng gói IPsec trực tiếp, không GRE | Site-to-site point-to-point, IP unicast |
| **GRE over IPsec** | Tunnel GRE, IPsec bảo vệ GRE (thường transport mode) | Cần multicast, non-IP protocol, routing protocol dùng multicast (OSPF/EIGRP) |
| **DVTI** (Dynamic VTI) | Virtual-template, interface tạo động cho mỗi spoke | Hub nhận nhiều spoke, spoke dùng IP động |
| **DMVPN** | mGRE + NHRP + IPsec; spoke-to-spoke động | Nhiều site, full-mesh theo nhu cầu |
| **FlexVPN** | Khung IKEv2 hợp nhất (site-to-site, hub-spoke, RA) | Thiết kế mới trên Cisco, thay thế dần DMVPN IKEv1 |

> Với BGP: có thể chạy eBGP/iBGP qua SVTI hoặc GRE tunnel; peer bằng IP của tunnel interface.

## 2. Cơ chế selector chi tiết

### Policy-based
- IKEv1: **một IPsec SA pair cho mỗi dòng ACL** (mỗi proxy-ID). ACL 3 dòng → 3 cặp SA.
- IKEv2: TS có thể chứa **dải/nhiều range** trong một CHILD_SA, tuỳ vendor.
- ACL phải mirror: A→B ở R1 ⇔ B→A ở R2. ACL rộng hơn ở một đầu có thể fail hoặc tuỳ implementation (Cisco IOS có thể chấp nhận subset trong một số trường hợp) → luôn kiểm tra thực tế.

### Route-based
- Selector `0.0.0.0/0 ↔ 0.0.0.0/0`; routing quyết định traffic.
- **Hệ quả bảo mật:** bất cứ gì route vào tunnel đều được mã hoá và peer sẽ chấp nhận → cần lọc bằng route-map/prefix-list (BGP), firewall zone, ACL trên Tunnel.

### Interop: route-based ↔ policy-based
Hai đầu khác kiểu vẫn nói chuyện được nếu **selector khớp**:
- Đầu route-based phải cấu hình selector cụ thể thay vì `any-any` (Palo Alto: *Proxy ID*; Junos: `proxy-identity` trong `ike gateway/ipsec vpn`; Fortinet: phase2 selector; Azure: `UsePolicyBasedTrafficSelectors`; Cisco VTI: thường cần dùng thiết bị/profile hỗ trợ — kiểm tra tài liệu nền tảng).
- Nếu đầu policy-based có **nhiều cặp subnet** mà đầu route-based chỉ hỗ trợ một selector → phải dùng nhiều tunnel hoặc cấu hình nhiều proxy-ID/phase2.
- Thất bại thường thấy: `TS_UNACCEPTABLE` (IKEv2), `no matching proxy ID` / `invalid ID info` (IKEv1).

## 3. Thiết kế HA & routing
- **Hai tunnel / hai ISP + BGP:** dùng AS-PATH prepend, local-pref, hoặc MED để chọn primary; BFD hoặc IKE DPD để phát hiện chết nhanh.
- **ECMP qua nhiều VTI:** cần cân bằng theo flow, chú ý asymmetric với firewall stateful.
- **Policy-based HA:** thường active/standby cứng (crypto map với nhiều peer, *reverse route injection* – RRI để đẩy route động khi SA lên).
- **Recursive routing:** route đến địa chỉ underlay (tunnel destination) **không được** học qua chính tunnel → tunnel flap `%TUN-5-RECURDOWN`. Giải pháp: dùng VRF riêng cho underlay (front-door VRF / `tunnel vrf`) hoặc route cụ thể hơn.

## 4. MTU, MSS, phân mảnh
Overhead ước lượng (ESP tunnel mode, AES-CBC + SHA, IPv4):
- ESP tunnel: khoảng 50–73 byte tuỳ thuật toán/padding (AES-GCM nhỏ hơn).
- GRE over IPsec: +24 byte GRE/IP, tổng thường 70–100 byte.
- NAT-T: +8 byte UDP.

Thực hành:
- Đặt `ip mtu 1400` trên Tunnel (Cisco thường dùng 1400 khi tunnel protection IPsec).
- Đặt `ip tcp adjust-mss 1360` để TCP tự giảm segment.
- Kiểm tra bằng `ping <dst> size 1400 df-bit`.
- Chú ý ICMP type 3 code 4 bị firewall chặn → PMTUD hỏng.

## 5. NAT-T, DPD, rekey, PFS
- **NAT-T:** phát hiện NAT trong IKE (NAT-D payload); nếu có NAT, chuyển sang UDP 4500 và bọc ESP trong UDP. Phải mở UDP 4500.
- **DPD** (Dead Peer Detection): IKEv2 dùng INFORMATIONAL message; cần để phát hiện peer chết và xoá SA. Cisco: `dpd 10 3 on-demand|periodic`.
- **Lifetime:** IKE SA thường 86400s, IPsec SA 3600s (hoặc theo KB); hai đầu lệch lifetime thì thường lấy giá trị nhỏ hơn nhưng nên đặt giống nhau.
- **Rekey:** nên có make-before-break; collision rekey có thể gây flap, kiểm tra log.
- **PFS** (Perfect Forward Secrecy): thêm DH exchange khi rekey CHILD_SA (`set pfs group14` trên IPsec profile/crypto map). Hai đầu phải cùng bật/tắt.

## 6. Thuật toán khuyến nghị (tham khảo)
| Mục | Nên dùng | Tránh |
|---|---|---|
| Mã hoá | AES-GCM-256 / AES-CBC-256 | DES, 3DES |
| Integrity | SHA-256/384 (hoặc AEAD của GCM) | MD5, SHA-1 |
| DH group | 14 (2048-bit), 19/20 (ECP), 21 | Group 1, 2, 5 |
| Auth | Certificate (PKI) cho quy mô lớn; PSK mạnh, riêng từng peer | PSK ngắn, dùng chung |
| IKE | IKEv2 | IKEv1 Aggressive mode + PSK |

> Hỗ trợ thuật toán và cú pháp khác nhau theo phiên bản/nền tảng; xác nhận bằng `?` trong CLI hoặc tài liệu vendor.

## 7. Cloud
- **AWS Site-to-Site VPN:** route-based, mỗi connection có 2 tunnel, hỗ trợ static hoặc BGP (nên BGP). Selector thực tế `any-any`.
- **Azure VPN Gateway:** route-based là mặc định (IKEv2); policy-based (IKEv1) có với một số SKU/cấu hình cũ (Basic) và tuỳ chọn *policy-based traffic selectors* trên gateway route-based. Giới hạn thay đổi theo thời gian → kiểm tra tài liệu Azure.
- **GCP Cloud VPN:** HA VPN dùng route-based + BGP (Cloud Router); Classic VPN hỗ trợ policy-based/route-based nhưng đang bị thay thế.
- Khi nối cloud ↔ thiết bị on-prem chỉ policy-based: dùng selector cụ thể ở phía cloud (nếu hỗ trợ) hoặc đặt thiết bị route-based ở giữa.

## 8. Vendor notes
- **Cisco IOS-XE:** crypto map (policy) vs `Tunnel` + `tunnel protection ipsec profile` (route). IKEv2 profile/keyring/proposal.
- **Cisco ASA/FTD:** crypto map (policy-based) là truyền thống; **VTI** hỗ trợ từ ASA 9.8 / FTD 6.7 (kiểm tra phiên bản cụ thể).
- **Junos (SRX):** route-based dùng `st0.x` và `bind-interface`; policy-based gắn `then permit tunnel ipsec-vpn` trong security policy.
- **Palo Alto:** tunnel interface (route-based), mặc định proxy-ID `0.0.0.0/0`; peer policy-based cần Proxy ID.
- **Fortinet:** IPsec interface-based (route-based) khuyến nghị; policy-based có ở các bản cũ — tuỳ phiên bản, kiểm tra tài liệu FortiOS.
- **FRR/Linux:** strongSwan + VTI/`xfrm` interface (route-based) hoặc policy mặc định theo `leftsubnet/rightsubnet`.

## 9. Bảo mật & vận hành
- Không dùng chung PSK cho nhiều peer; xoay PSK/cert định kỳ.
- Route-based: dùng **prefix-list/route-map** để chỉ chấp nhận prefix hợp lệ từ peer; gắn firewall zone cho interface tunnel.
- Logging IKE (không bật debug lâu trên production), theo dõi SA counters, DPD, rekey.
- Giám sát: SNMP/telemetry cho trạng thái tunnel, BFD/BGP session.

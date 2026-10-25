# VXLAN / EVPN Cheatsheet

## A. Kiểm tra theo tầng (NX-OS)
```
# 1. Underlay
show ip ospf neighbors | show bgp ipv4 unicast summary
show ip route <remote-VTEP-IP>
ping <remote-VTEP-IP> source <local-VTEP-IP>     # source = loopback VTEP!
ping <remote-VTEP> source-interface loopback1 df-bit packet-size 9100   # test MTU
show interface Ethernet1/1 | include MTU
show ip pim neighbor / show ip mroute               # nếu dùng multicast BUM

# 2. NVE / VNI
show nve interface nve1 detail                      # source IP, state, host-reach
show nve peers                                      # peer VTEP, state Up, learn type CP
show nve vni                                        # VNI, mcast/IR, state, type L2/L3
show nve vni ingress-replication
show nve peers detail

# 3. BGP EVPN control plane
show bgp l2vpn evpn summary
show bgp l2vpn evpn neighbors 10.255.100.1 advertised-routes
show bgp l2vpn evpn route-type 2 | 3 | 5
show bgp l2vpn evpn vni-id 10010
show bgp l2vpn evpn 0050.56aa.bbbb                  # theo MAC
show bgp evi 10010 / show bgp l2vpn evpn rd all     # tùy version
show bgp ipv4 unicast vrf TENANT1

# 4. Layer 2 / Layer 3 tables
show l2route evpn mac all | evi 10010
show l2route evpn mac-ip all
show mac address-table vlan 10 | include nve
show ip arp vrf TENANT1 | show ip arp suppression-cache detail
show ip route vrf TENANT1
show vrf TENANT1 | show vxlan interface
show vlan id 10 | show running-config | section evpn
show fabric forwarding ip local-host-db vrf TENANT1 / anycast-gateway-mac

# 5. Multi-homing / multi-site
show vpc | show vpc consistency-parameters global
show nve ethernet-segment
show nve multisite fabric-links|dci-links
show nve multisite dci-links

# 6. Hardware
show system internal l2fwd info / show hardware capacity forwarding
show platform software ...  (tuỳ model)
```

### Output mong đợi (`show nve peers`)
```
Interface Peer-IP        State LearnType Uptime   Router-Mac
nve1      10.255.1.3     Up    CP        00:42:11 5254.00aa.bb08
```
`CP` = control-plane learned (EVPN). `DP` = data-plane learned (flood-and-learn). State `Down` → underlay/BGP lỗi.

## B. Cấu hình tham khảo nhanh
### Bật feature (NX-OS)
`nv overlay evpn` · `feature nv overlay` · `feature vn-segment-vlan-based` · `feature bgp` · `feature interface-vlan` · `feature ospf` · `fabric forwarding anycast-gateway-mac <mac>` · `feature vpc` (nếu dùng)

### Mapping VLAN–VNI
```
vlan 10
  vn-segment 10010
interface nve1
  member vni 10010
    suppress-arp
    ingress-replication protocol bgp        # hoặc: mcast-group 239.1.1.10
evpn
  vni 10010 l2
    rd auto
    route-target import auto
    route-target export auto
```
### L3VNI
```
vlan 3001 ; vn-segment 50001
vrf context T1 ; vni 50001 ; rd auto ; address-family ipv4 unicast ; route-target both auto evpn
interface Vlan3001 ; vrf member T1 ; ip forward ; no shutdown
interface nve1 ; member vni 50001 associate-vrf
router bgp 65000 ; vrf T1 ; address-family ipv4 unicast ; advertise l2vpn evpn
```
### Route-type 5 (redistribute từ ngoài)
```
router bgp 65000
  vrf T1
    address-family ipv4 unicast
      advertise l2vpn evpn
      redistribute static route-map RM-STATIC
      network 10.50.0.0/16
```
### Spine (route reflector overlay)
```
router bgp 65000
  neighbor 10.255.1.1 remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
      route-reflector-client
```
### Underlay OSPF + PIM
```
router ospf UNDERLAY
interface loopback0/1 ; ip router ospf UNDERLAY area 0 ; ip pim sparse-mode
ip pim rp-address 10.255.0.100 group-list 224.0.0.0/4 | anycast RP
```
### VPC cặp leaf (tóm tắt)
```
vpc domain 1 ; peer-switch ; peer-gateway ; ip arp synchronize ; peer-keepalive destination <ip> vrf mgmt
interface port-channel1 ; vpc peer-link
interface nve1 ; source-interface loopback1 ; (loopback1: secondary ip <anycast> )
```

## C. Arista EOS
```
show vxlan vtep | show vxlan vni | show vxlan address-table
show vxlan flood vtep
show bgp evpn summary | show bgp evpn route-type mac-ip|imet|ip-prefix
show interfaces vxlan 1
show vxlan config-sanity detail
show ip route vrf T1
show ip arp vrf T1
```
## D. Linux / FRR
```
bridge fdb show dev vxlan10
bridge vlan show ; ip -d link show vxlan10
vtysh -c "show bgp l2vpn evpn summary"
vtysh -c "show evpn vni" ; "show evpn mac vni 10010" ; "show evpn arp-cache vni 10010"
vtysh -c "show bgp l2vpn evpn route"
```
## E. Junos
```
show evpn database | show ethernet-switching vxlan-tunnel-end-point remote
show route table bgp.evpn.0 | show route table T1.inet.0
show ethernet-switching table
```

## F. Quy trình troubleshoot
**Tình huống: VM-A (Leaf1) ping VM-B (Leaf2) cùng VNI không được.**
1. VLAN/VNI đúng 2 đầu? `show vlan id 10`, `show nve vni`, VNI state Up.
2. Underlay: ping loopback VTEP ↔ VTEP source = lo1, test MTU với DF-bit.
3. `show nve peers` Up? Nếu không: kiểm tra BGP EVPN (`send-community extended`, RR, next-hop-unchanged, peer ASN).
4. Có RT-3 của peer cho VNI? `show bgp l2vpn evpn route-type 3`. Nếu không: `member vni` thiếu, RT import không khớp, `ingress-replication` chưa bật.
5. Có RT-2 của MAC VM-B? `show bgp l2vpn evpn <mac>`; thiếu → MAC chưa được học local ở Leaf2 (VM im lặng, port-channel/VLAN sai): `show mac address-table`.
6. MAC có trong `show l2route evpn mac all` với next-hop đúng? 
7. Duplicate MAC / mobility freeze? `show l2route evpn mac all | include dup`.
8. Forwarding: capture/SPAN, kiểm tra ACL, TCAM, hardware programming.

**Tình huống: L2 ok, L3 (inter-VLAN) fail**
1. Anycast gateway MAC đồng nhất, `fabric forwarding mode anycast-gateway` trên SVI.
2. L3VNI: `show vrf T1`, `show nve vni` (L3 state Up), SVI vlan3001 up với `ip forward`.
3. `show bgp l2vpn evpn route-type 2/5` có prefix/host cần? `advertise l2vpn evpn` trong VRF BGP?
4. RT import/export của VRF: `route-target both auto evpn`.
5. Router-MAC của peer: `show nve peers detail`. 
6. Asymmetric vs symmetric config lệch giữa 2 leaf.

**Tình huống: Host không bị ARP suppress / ARP không trả**
`show ip arp suppression-cache`, TCAM region `arp-ether`, `suppress-arp` đã bật?

**Tình huống: Mất kết nối khi VM di chuyển**
Kiểm tra MAC mobility sequence, GARP/RARP gửi sau migration, route cũ chưa withdraw, duplicate detection.

## G. Bảng tra nhanh
| Item | Giá trị |
|---|---|
| UDP port | **4789** (IANA); Linux cũ 8472 |
| Overhead | 50B (IPv4 outer) |
| VNI | 24-bit (1–16,777,215; 0 reserved ở một số nền tảng) |
| Route type | RT-2 = MAC/IP · RT-3 = IMET (BUM) · RT-5 = IP prefix |
| AFI/SAFI EVPN | 25 / 70 |
| ES-Import RT | tự động từ ESI |
| Default Underlay MTU khuyến nghị | 9216 |
| Encapsulation ext-community | 0x030c type 8 = VXLAN |

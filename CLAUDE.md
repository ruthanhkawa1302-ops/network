# Network Knowledge Base: BGP, SD-WAN, VXLAN

Project này là kho kiến thức (tiếng Việt) về 3 chủ đề mạng: **BGP**, **SD-WAN**, **VXLAN/EVPN**, từ cơ bản đến nâng cao, kèm cheatsheet lệnh.

## Cấu trúc
```
README.md                  Mục lục + lộ trình học
bgp/01-fundamentals.md     BGP cơ bản
bgp/02-advanced.md         BGP nâng cao
bgp/03-cheatsheet.md       Lệnh cấu hình / kiểm tra / troubleshoot
sdwan/01-fundamentals.md   SD-WAN cơ bản
sdwan/02-advanced.md       SD-WAN nâng cao
sdwan/03-cheatsheet.md
vxlan/01-fundamentals.md   VXLAN + EVPN cơ bản
vxlan/02-advanced.md       VXLAN/EVPN nâng cao
vxlan/03-cheatsheet.md
vpn/01-fundamentals.md     VPN/IPsec cơ bản, policy-based vs route-based
vpn/02-advanced.md         VTI/DMVPN/FlexVPN, interop, MTU, cloud
vpn/03-cheatsheet.md
.claude/commands/          Slash command: /explain, /quiz, /lab, /troubleshoot
```

## Quy ước khi trả lời / chỉnh sửa
- Ngôn ngữ: tiếng Việt; giữ nguyên thuật ngữ kỹ thuật tiếng Anh (route reflector, underlay, TLOC...).
- Mỗi khái niệm: **định nghĩa dễ hiểu → vì sao cần → ví dụ cụ thể (topology/config/output) → lỗi thường gặp**.
- Lệnh mặc định theo Cisco IOS-XE / NX-OS; có ghi chú khi khác biệt với Junos, Arista EOS, FRR. SD-WAN mặc định Cisco Catalyst SD-WAN (Viptela), có so sánh vendor khác.
- Chính xác là ưu tiên: không bịa lệnh/giá trị. Nếu không chắc về phiên bản/nền tảng, nói rõ và khuyên kiểm tra `?` hoặc tài liệu vendor.
- Khi thêm kiến thức mới: đặt vào đúng file theo cấp độ, cập nhật mục lục trong README.md.
- Ví dụ IP dùng dải tài liệu/private (RFC 5737, 10.0.0.0/8, 192.0.2.0/24), ASN private 64512–65534 / 4200000000–4294967294.

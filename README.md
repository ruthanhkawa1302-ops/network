# Network Knowledge Base: BGP · SD-WAN · VXLAN

## Mục lục
| Chủ đề | Cơ bản | Nâng cao | Cheatsheet |
|---|---|---|---|
| BGP | [bgp/01](bgp/01-fundamentals.md) | [bgp/02](bgp/02-advanced.md) | [bgp/03](bgp/03-cheatsheet.md) |
| SD-WAN | [sdwan/01](sdwan/01-fundamentals.md) | [sdwan/02](sdwan/02-advanced.md) | [sdwan/03](sdwan/03-cheatsheet.md) |
| VXLAN/EVPN | [vxlan/01](vxlan/01-fundamentals.md) | [vxlan/02](vxlan/02-advanced.md) | [vxlan/03](vxlan/03-cheatsheet.md) |

## Lộ trình học đề xuất
1. **BGP cơ bản** → nắm eBGP/iBGP, path attributes, best-path. (BGP là nền cho cả EVPN lẫn nhiều thiết kế SD-WAN.)
2. **VXLAN cơ bản** → hiểu overlay/underlay, VTEP, VNI, flood-and-learn.
3. **BGP nâng cao** → route reflector, communities, policy, MP-BGP.
4. **VXLAN nâng cao** → BGP EVPN, route type 2/3/5, anycast gateway, multi-site.
5. **SD-WAN** → control/data/management plane, TLOC, OMP, policy, rồi nâng cao.

## Slash commands (Claude Code)
- `/explain <chủ đề>`: giải thích một khái niệm kèm ví dụ.
- `/quiz <chủ đề>`: tạo câu hỏi ôn tập.
- `/lab <chủ đề>`: tạo bài lab (topology + config + kiểm tra).
- `/troubleshoot <triệu chứng>`: quy trình debug từng bước.

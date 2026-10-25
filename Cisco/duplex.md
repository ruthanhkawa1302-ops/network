Duplex nói về việc thiết bị có thể gửi và nhận dữ liệu cùng lúc hay không.

1. Half-duplex = 1 chiều tại một thời điểm

Giống như bộ đàm 📻:

A  ─────────→  B
      SEND

A  ←─────────  B
      RECEIVE

A và B không thể truyền cùng lúc.

Ví dụ:

A đang gửi → B phải chờ.
B đang gửi → A phải chờ.

Nếu cả hai cùng nói:

"Hello!" + "Hello!"

→ dễ xảy ra collision.

Thiết bị mạng ngày xưa dùng hub thường hoạt động theo kiểu half-duplex.

2. Full-duplex = gửi và nhận cùng lúc

Giống như điện thoại ☎️:

A  ─────────→  B
A  ←─────────  B
     SAME TIME

A có thể gửi cho B đồng thời B gửi cho A.

Ví dụ:

PC ─────────────── Switch
     TX  →→→
     RX  ←←←

Hai chiều hoạt động đồng thời.

Switch Ethernet hiện đại + Ethernet point-to-point thường chạy full-duplex.

3. Tại sao Network Engineer phải quan tâm?

Ví dụ interface:

Speed: 1000 Mbps
Duplex: Full

→ 1 Gbps theo mỗi chiều về mặt link capacity.

Nếu xảy ra duplex mismatch:

PC              Switch
Full-duplex     Half-duplex

có thể xuất hiện:

packet loss
retransmission
CRC/FCS errors
collisions (ở phía half-duplex)
performance rất tệ dù interface vẫn báo up

Đây là một lỗi troubleshooting khá kinh điển.

Mẹo nhớ:

Half = một nửa → một thời điểm chỉ truyền một hướng.
Full = đầy đủ → hai hướng cùng lúc.

Và có một điểm rất quan trọng khi học Cisco/Fortigate: duplex không giống với bandwidth/speed. Ví dụ 1 Gbps full-duplex không có nghĩa là “tổng cộng chỉ 1 Gbps”; nó mô tả khả năng truyền hai chiều đồng thời trên link.
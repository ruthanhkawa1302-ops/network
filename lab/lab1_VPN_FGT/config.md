FGT config lacp:
config system interface
    edit "EtherChannel1"
        set lacp-mode static
    next
end

Lưu í ko bật nat policy khi tạo policy cho thông qua VPN
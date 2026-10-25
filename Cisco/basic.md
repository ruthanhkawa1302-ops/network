# Enable
enable
configure terminal

# Rename
enable
configure terminal
hostname SW1

# Set banner when login
enable
configure terminal
banner motd "this is switch"

# Set password when using console log
enable
configure terminal
line console 0
password cisco
login

# Save configure
write memory
or 
copy running-config startup-config
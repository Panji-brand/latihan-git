B.1 Alamat dan connected route
Di tiap router, login admin dengan password kosong. Tekan Ctrl-C untuk melewati permintaan ganti password.
```
# R1
/ip dhcp-client remove [find]
/system identity set name=R1
/ip address add address=10.239.20.129/28 interface=ether1 comment="LAN Ruang Dosen"
/ip address add address=10.239.20.161/30 interface=ether2 comment="link ke R2"

# R2
/ip dhcp-client remove [find]
/system identity set name=R2
/ip address add address=10.239.20.65/26 interface=ether1 comment="LAN IF-105"
/ip address add address=10.239.20.162/30 interface=ether2 comment="link ke R1"
/ip address add address=10.239.20.165/30 interface=ether3 comment="link ke R3"

# R3
/ip dhcp-client remove [find]
/system identity set name=R3
/ip address add address=10.239.20.1/26 interface=ether1 comment="LAN IF-106"
/ip address add address=10.239.20.166/30 interface=ether2 comment="link ke R2"
```

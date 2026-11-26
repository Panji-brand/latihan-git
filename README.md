## B.1 Alamat dan connected route
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

Host (di Laterano, paste dengan klik tengah mouse atau Shift+Insert):
```
# Lungmen
ip addr add 10.239.20.130/28 dev eth0
ip route add default via 10.239.20.129
# Laterano
ip addr add 10.239.20.131/28 dev eth0
ip route add default via 10.239.20.129
# Yan
ip addr add 10.239.20.70/26 dev eth0
ip route add default via 10.239.20.65
# Kazdel
ip addr add 10.239.20.10/26 dev eth0
ip route add default via 10.239.20.1
```
- [SS] `/ip address print` dan `/ip route print` di R1, R2, dan R3. Routing table hanya berisi route DAc, jadi ambil sebelum static route dipasang.
- [Prediksi] Laterano ke Yan gagal, karena R1 tidak punya route ke .64/26.
- [SS] `ping -c 3 10.239.20.162` dari Laterano. Hasilnya gagal karena R2 tidak punya return route.



## B.2 Static route dan Wireshark
Pasang bertahap. Comment harus persis seperti ini karena dipakai lagi nanti.
```
# R1
/ip route add dst-address=10.239.20.64/26 gateway=10.239.20.162 comment="IF-105 via R2"
/ip route add dst-address=10.239.20.0/26 gateway=10.239.20.162 comment="IF-106 via R2"

# R2
/ip route add dst-address=10.239.20.128/28 gateway=10.239.20.161 comment="Ruang Dosen via R1"
/ip route add dst-address=10.239.20.0/26 gateway=10.239.20.166 comment="IF-106 via R3"
```
- [Prediksi] Ping Laterano ke Kazdel timeout, dan reply berhenti di R3.
- [SS] `ping -c 3 10.239.20.10` dari Laterano (100% loss) dan `/tool traceroute 10.239.20.131` di R3 (timeout).


```
# R3
/ip route add dst-address=10.239.20.128/28 gateway=10.239.20.165 comment="Ruang Dosen via R2"
/ip route add dst-address=10.239.20.64/26 gateway=10.239.20.165 comment="IF-105 via R2"
```
- [SS] /ip route print di R1, R2, dan R3.
- [SS] `mtr -r -c 5 10.239.20.70` dan `mtr -r -c 5 10.239.20.10` dari Laterano, lalu `mtr -r -c 5 10.239.20.131` dari Kazdel. TTL ping ke Kazdel adalah 61 dan ke Yan 62.
- Wireshark: Start capture di link Switch1–R1, R1–R2, dan R2–R3 dengan filter icmp, lalu `ping -c 3 10.239.20.10` dari Laterano.
- [SS] Buka Internet Protocol Version 4 pada Echo request yang sama di tiap capture. Hasil yang kami dapat: TTL 64 → 63 → 62, Protocol ICMP (1), dan Header Checksum 0x215e → 0x225e → 0x235e.



## B.3 Manipulasi route

(a) Summarization (di R1):
```
/ip route remove [find where comment~"via R2"]
/ip route add dst-address=10.239.20.0/25 gateway=10.239.20.162 comment="IF-105 dan IF-106 via R2 (summary)"
```
- [SS] `/ip route print` dan ping ke Yan dan Kazdel. Summary ini dipertahankan sampai Bagian C.

(b) Longest prefix match:
[Prediksi] .10 gagal, .1 dan .70 berhasil.
```
# R1
/ip route add dst-address=10.239.20.10/32 blackhole comment="uji LPM"
```
- [SS] Ping `10.239.20.10`, `10.239.20.1`, dan `10.239.20.70` dari Laterano.
-  Hapus route uji: `/ip route remove [find where comment="uji LPM"]`

(c) Routing loop:
[Prediksi] Paket bolak-balik R1 ↔ R2 sampai TTL habis, lalu muncul Time-to-live exceeded.
```
# R2
/ip route set [find where comment="IF-106 via R3"] gateway=10.239.20.161
```
- [SS] `ping -c 2 10.239.20.10` dari Laterano (pesan Time to live exceeded), serta capture R1–R2 dengan kolom TTL (filter icmp.type == 8).
- Kembalikan: `/ip route set [find where comment="IF-106 via R3"] gateway=10.239.20.166`, lalu [SS] ping Kazdel yang berhasil.

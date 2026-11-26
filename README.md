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

## C.1 Redundant link static
Sambungkan kabel R1 ether3 ↔ R3 ether3.
```
# R1
/ip address add address=10.239.20.169/30 interface=ether3 comment="link ke R3"

# R3
/ip address add address=10.239.20.170/30 interface=ether3 comment="link ke R1"


# R1
/ip route add dst-address=10.239.20.0/26 gateway=10.239.20.170 comment="IF-106 langsung via R3"

# R3
/ip route set [find where comment="Ruang Dosen via R2"] distance=10
/ip route add dst-address=10.239.20.128/28 gateway=10.239.20.169 comment="Ruang Dosen langsung via R1"
```
- [SS] `/ip route print` di R1 dan R3, dan `mtr -r -c 5 10.239.20.10` dari Laterano (hanya R1 dan R3, jadi lebih pendek dari B.2).

  
Asymmetric path. [Prediksi] Arah pergi lewat R3, arah pulang lewat R2. Lalu:
```
# R3
/ip route disable [find where comment="Ruang Dosen langsung via R1"]

$ mtr -r -c 5 10.239.20.10         # dari Laterano
$ mtr -r -c 5 10.239.20.131        # dari Kazdel
 
# R3
/ip route enable [find where comment="Ruang Dosen langsung via R1"]

```
- [SS] mtr dari Laterano ke .10 dan dari Kazdel ke .131. Hasil kami: pergi lewat 2 router (R1, R3), pulang lewat 3 router (R3, R2, R1). Alamat hop yang tampil adalah interface yang dipakai router untuk membalas, jadi bisa berbeda dari prediksi. Aktifkan lagi dengan /ip route enable [find where comment="Ruang Dosen langsung via R1"].


Link failure. [Prediksi] Traffic tidak pindah ke R2.
Jalankan `ping -O 10.239.20.10` dari Laterano, lalu `Suspend` link R1–R3. 
Hasil kami: traffic justru pindah sendiri (TTL 62 → 61, 2 paket hilang), karena suspend menurunkan status interface dan route langsung otomatis inactive.
- [SS] Ping saat suspend, serta `/interface print where name=ether3` dan `/ip route print where dst-address=10.239.20.0/26` di R1 (route berflag Is).
- Resume link. Kalau jadi `no-link` (cek dengan `/interface ethernet monitor ether3 once)`, hapus kabel lalu sambung ulang.
- Aktifkan check-gateway, lalu ulangi uji. Hasil kami: 1 paket hilang, hampir sama dengan uji pertama.
```
# R1
/ip route set [find where comment="IF-106 langsung via R3"] check-gateway=ping

# R3
/ip route set [find where comment="Ruang Dosen langsung via R1"] check-gateway=ping
```


## C.2 OSPF
pada R1,2,3
```
/ip route remove [find where static]
/ip route print
```
- [SS] Routing table di R1, R2, dan R3 hanya berisi DAc

```
# R1
/routing ospf instance add name=ospf1 version=2 router-id=1.1.1.1
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template add area=backbone interfaces=ether2,ether3 type=ptp comment="link inter-router"
/routing ospf interface-template add area=backbone interfaces=ether1 passive comment="LAN"

# R2
/routing ospf instance add name=ospf1 version=2 router-id=2.2.2.2
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template add area=backbone interfaces=ether2,ether3 type=ptp comment="link inter-router"
/routing ospf interface-template add area=backbone interfaces=ether1 passive comment="LAN"

# R3
/routing ospf instance add name=ospf1 version=2 router-id=3.3.3.3
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template add area=backbone interfaces=ether2,ether3 type=ptp comment="link inter-router"
/routing ospf interface-template add area=backbone interfaces=ether1 passive comment="LAN"
```

Verifikasi neighbor, interface, dan route yang dibentuk OSPF
```
/routing ospf neighbor print
/routing ospf interface print
/ip route print where ospf
```
- [Prediksi] Traffic pulih otomatis dalam beberapa detik.
- Jalankan `ping -O 10.239.20.10`, lalu Suspend link R1–R3. Hasil kami: 3 paket hilang, lalu TTL 62 → 61.
- [SS] Ping saat suspend, serta neighbor dan route OSPF R1 saat suspend (neighbor R3 hilang).
- Resume link (hapus dan sambung ulang kabel kalau no-link), lalu [SS] neighbor dan route R1 yang sudah pulih.

## D.1 - Migrasi ke Backbone Ganda
Urutan yang aman adalah alamat dan OSPF dulu, baru matikan link p2p
```
# R1
/interface disable ether2,ether3
/ip address add address=10.239.20.145/29 interface=ether4 comment="Backbone A"
/ip address add address=10.239.20.153/29 interface=ether5 comment="Backbone B"
/routing ospf interface-template add area=backbone interfaces=ether4,ether5 comment="backbone ganda"

# R2
/interface disable ether2,ether3
/ip address add address=10.239.20.146/29 interface=ether4 comment="Backbone A"
/ip address add address=10.239.20.154/29 interface=ether5 comment="Backbone B"
/routing ospf interface-template add area=backbone interfaces=ether4,ether5 comment="backbone ganda"

# R3/interface disable ether2,ether3
/ip address add address=10.239.20.147/29 interface=ether4 comment="Backbone A"
/ip address add address=10.239.20.155/29 interface=ether5 comment="Backbone B"
/routing ospf interface-template add area=backbone interfaces=ether4,ether5 comment="backbone ganda"
```
- [Prediksi] Awalnya kami kira R3 jadi DR karena router-id terbesar.
- [SS] neighbor print dan interface print.
```
/routing ospf neighbor print
/routing ospf interface print
```
Hasil kami: R1 DR, R2 BDR, R3 DROther. OSPF tidak preemptive, jadi router yang aktif lebih dulu menjadi DR.

- [Prediksi] R1 punya 2 gateway ECMP ke IF-106 (.147 dan .155).
- [SS] di R1 (dua entri DAo+).
```
# R1
/ip route print detail where dst-address=10.239.20.0/26
```
 
- Jalankan `ping -O 10.239.20.10`, lalu `Suspend` link R3 ↔ SW-BackboneA. Hasil kami: 2 paket hilang dan TTL tetap 62.
- [SS] Ping saat suspend, dan route R1 yang tinggal satu gateway (.155). Resume link, lalu cek gateway kembali dua.
- [Prediksi] Setelah priority 0, R1 jadi DROther, R2 DR, R3 BDR.
- [SS] interface print di R1, R2, dan R3 sebagai kondisi sebelum, lalu jalankan:

Pada R1, ubah DR priority menjadi 0, kemudian nonaktifkan dan aktifkan kembali ether4 dan ether5 agar re-election terjadi. Amati perubahan peran R1
```
# R1
/routing ospf interface-template set [find where comment="backbone ganda"] priority=0
/interface disable ether4,ether5
/interface enable ether4,ether5
/routing ospf interface print
```
- [SS] interface print di R1, R2, dan R3 sebagai kondisi sesudah.


##  D.2 - Penambahan Ruang Baca melalui R4
Tambahkan R4 (MikroTik CHR, 5 adapter), switch Ruang Baca, dan Higashi sesuai Gambar D.1. Sambungkan ether1 ke switch Ruang Baca, ether4 ke Backbone A, dan ether5 ke Backbone B
```
# R4
/ip dhcp-client remove [find]
/system identity set name=R4
/ip address add address=10.239.20.193/26 interface=ether1 comment="LAN Ruang Baca"
/ip address add address=10.239.20.148/29 interface=ether4 comment="Backbone A"
/ip address add address=10.239.20.156/29 interface=ether5 comment="Backbone B"
/routing ospf instance add name=ospf1 version=2 router-id=4.4.4.4
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template add area=backbone interfaces=ether4,ether5 comment="backbone ganda"
/routing ospf interface-template add area=backbone interfaces=ether1 passive comment="LAN"
```
```
# Higashi
ip addr add 10.239.20.200/26 dev eth0
ip route add default via 10.239.20.193
```
- Jangan menyentuh R1, R2, dan R3.
- [Prediksi] R1 sudah mengenal Ruang Baca dengan 2 gateway ECMP (.148 dan .156).
- [SS] `/routing ospf neighbor print` di R4, `/ip route print where dst-address=10.239.20.192/26` di R1, dan `mtr -r -c 5 10.239.20.200` dari Laterano.
- Export di R1 sampai R4. Jalankan `/export file=case4-R1` (dan seterusnya), lalu ambil isinya. Nyalakan satu router, jalankan `/export` tanpa file=, klik kanan judul PuTTY → Copy All to Clipboard, tempel ke Notepad, lalu simpan sebagai case4-R1.txt. File .rsc berada di dalam router, bukan di laptopmu.

# Failover dalam Jaringan

## 1. Cara Kerja Failover

Failover adalah mekanisme otomatis yang memindahkan beban kerja dari sistem utama (primary) ke sistem cadangan (secondary) ketika terjadi kegagalan. Prosesnya bekerja seperti ini:

**Monitoring dan Deteksi**
Sistem failover terus-menerus memantau kondisi server atau perangkat utama melalui heartbeat atau health check. Biasanya menggunakan ping, protokol khusus, atau pengecekan service berkala setiap beberapa detik.

**Deteksi Kegagalan**
Ketika sistem utama tidak merespon dalam waktu tertentu (timeout threshold), sistem backup menganggap primary sudah down. Biasanya ada mekanisme retry beberapa kali untuk menghindari false positive.

**Proses Switchover**
Setelah kegagalan terdeteksi, sistem backup mengambil alih dengan cara:
- Mengaktifkan IP address yang sama dengan primary (IP takeover)
- Memulai service yang tadinya berjalan di primary
- Mengambil alih koneksi dan session yang masih aktif
- Melakukan sinkronisasi data jika diperlukan

**Recovery dan Fallback**
Setelah sistem utama kembali normal, ada dua skenario: tetap menggunakan backup (manual fallback) atau otomatis kembali ke primary (automatic failback).

## 2. Jenis-jenis Failover

**Active-Passive (Cold Standby)**
Hanya sistem primary yang aktif melayani traffic. Sistem backup dalam keadaan standby atau bahkan mati, baru dinyalakan ketika primary down. Metode ini hemat resource tapi waktu switchover lebih lama (bisa beberapa menit).

**Active-Passive (Hot Standby)**
Primary aktif melayani traffic, sementara backup sudah menyala dan siap mengambil alih kapan saja. Data biasanya sudah tersinkronisasi real-time. Waktu switchover sangat cepat (detik hingga menit).

**Active-Active (Load Balancing)**
Kedua sistem aktif bersamaan dan melayani traffic secara bersamaan. Ketika salah satu down, yang lain langsung menangani semua beban. Tidak ada downtime karena traffic sudah terdistribusi sejak awal.

**Active-Active-Passive**
Kombinasi dimana dua server aktif melayani traffic dengan load balancing, dan satu server lagi standby sebagai backup terakhir jika kedua server aktif mengalami masalah.

## 3. Contoh Konfigurasi Failover

### Dual WAN Failover di MikroTik RouterOS

**Topologi:**

```
           Internet
              |
     ┌────────┴────────┐
     |                 |
ISP 1 (Primary)    ISP 2 (Backup)
10.10.1.1          10.20.1.1
     |                 |
  Ether1           Ether2
     |                 |
     └────────┬────────┘
              |
      MikroTik Router
              |
          Ether3
              |
         Client
     192.168.1.10
```

**Konfigurasi Failover:**

**1. Set IP Address**
```
/ip address
add address=10.10.1.2/24 interface=ether1 comment="ISP1"
add address=10.20.1.2/24 interface=ether2 comment="ISP2"
add address=192.168.1.1/24 interface=ether3 comment="LAN"
```

**2. Set NAT**
```
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade
add chain=srcnat out-interface=ether2 action=masquerade
```

**3. Set Route Failover (Inti Konfigurasi)**
```
/ip route
add dst-address=0.0.0.0/0 gateway=10.10.1.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=10.20.1.1 distance=2 check-gateway=ping
```

**Penjelasan:**
- Route pertama (distance=1) adalah jalur primary melalui ISP 1
- Route kedua (distance=2) adalah jalur backup melalui ISP 2
- `check-gateway=ping` membuat router otomatis ping gateway setiap 10 detik
- Jika ISP 1 down, route otomatis beralih ke ISP 2
- Jika ISP 1 hidup lagi, route otomatis kembali ke ISP 1

**Cara Kerja:**
Dalam kondisi normal, semua traffic client keluar melalui ISP 1. Router terus melakukan ping ke gateway 10.10.1.1. Jika ping gagal 2 kali berturut-turut (sekitar 20 detik), route ISP 1 otomatis disabled dan traffic langsung beralih ke ISP 2. Ketika ISP 1 kembali online, traffic otomatis failback ke ISP 1 karena distance-nya lebih kecil (prioritas lebih tinggi).

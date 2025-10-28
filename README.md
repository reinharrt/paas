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

### Contoh 1: Failover Load Balancing ISP di MikroTik

**Topologi:**

```
                    Internet
                       |
          ┌────────────┴────────────┐
          |                         |
      ISP 1 (Primary)           ISP 2 (Backup)
    Gateway: 10.10.1.1        Gateway: 10.20.1.1
          |                         |
          |                         |
      Ether1                    Ether2
    (10.10.1.2/24)           (10.20.1.2/24)
          |                         |
          └────────────┬────────────┘
                       |
                 MikroTik Router
                       |
                   Ether3
                (192.168.1.1/24)
                       |
                   Switch
                       |
              ┌────────┴────────┐
              |                 |
          Client 1          Client 2
       (192.168.1.10)   (192.168.1.11)
```

**Keterangan:**
- ISP 1 sebagai koneksi utama dengan bandwidth lebih besar
- ISP 2 sebagai backup yang akan aktif otomatis jika ISP 1 down
- Client menggunakan gateway 192.168.1.1 (MikroTik)
- MikroTik melakukan monitoring kedua ISP secara berkala

**Konfigurasi:**

Langkah 1: Setting IP Address pada interface
- Ether1 terhubung ke ISP 1: 10.10.1.2/24
- Ether2 terhubung ke ISP 2: 10.20.1.2/24
- Ether3 untuk LAN: 192.168.1.1/24

Langkah 2: Membuat routing table dengan distance berbeda
- Route ISP 1: dst-address 0.0.0.0/0, gateway 10.10.1.1, distance 1, check-gateway ping
- Route ISP 2: dst-address 0.0.0.0/0, gateway 10.20.1.1, distance 2, check-gateway ping

Distance yang lebih kecil memiliki prioritas lebih tinggi. ISP 1 dengan distance 1 akan digunakan secara default.

Langkah 3: Enable check-gateway
Check-gateway akan melakukan ping ke gateway secara berkala. Jika gateway ISP 1 tidak merespon, route tersebut akan dianggap down dan otomatis beralih ke ISP 2.

Langkah 4: (Optional) Netwatch untuk monitoring lebih detail
Buat netwatch yang melakukan ping ke host eksternal seperti 8.8.8.8 melalui masing-masing ISP untuk memastikan koneksi benar-benar berfungsi, bukan hanya gateway yang hidup.

**Cara Kerja:**
Dalam kondisi normal, semua traffic client akan keluar melalui ISP 1. MikroTik terus melakukan ping ke gateway 10.10.1.1. Jika gateway ISP 1 tidak merespon selama beberapa kali (biasanya 3-5 detik), route dengan distance 1 akan disabled otomatis. Traffic kemudian beralih ke route dengan distance 2 yaitu ISP 2. Ketika ISP 1 kembali online dan merespon ping, route distance 1 akan aktif kembali dan traffic akan failback ke ISP 1 secara otomatis.

### Contoh 2: Failover Router dengan VRRP

**Topologi:**

```
         Internet
              |
    ┌─────────┴──────────┐
    |                    |
Router A            Router B
(Master)            (Backup)
192.168.1.1        192.168.1.2
    |                    |
    └─────────┬──────────┘
              |
        Virtual IP
      192.168.1.254
              |
         Switch
              |
    ┌─────────┼─────────┐
    |         |         |
Client 1  Client 2  Client 3
```

**Konfigurasi:**
- Protocol: VRRP (Virtual Router Redundancy Protocol)
- Virtual IP: 192.168.1.254 (gateway yang dipakai semua client)
- Router A (Master): Priority 100
- Router B (Backup): Priority 90
- Advertisement interval: 1 detik

Client menggunakan 192.168.1.254 sebagai gateway. Router A yang aktif mengirim advertisement terus-menerus. Jika Router B tidak menerima advertisement selama 3 detik, ia mengambil alih sebagai master dan menghandle semua traffic. Client tidak perlu mengubah konfigurasi apapun karena tetap menggunakan IP 192.168.1.254 yang sama.

### Contoh 3: Failover Database dengan Hot Standby

**Topologi:**

```
              Internet
                  |
            Load Balancer
                  |
        ┌─────────┴─────────┐
        |                   |
   Web Server 1         Web Server 2
   192.168.10.10       192.168.10.11
        |                   |
        └─────────┬─────────┘
                  |
            Virtual IP
          192.168.10.100
                  |
        ┌─────────┴─────────┐
        |                   |
   DB Primary           DB Secondary
   192.168.10.20       192.168.10.21
   (Active)            (Hot Standby)
```

**Konfigurasi:**
- Virtual IP (VIP): 192.168.10.100 (digunakan aplikasi untuk koneksi database)
- Primary DB: 192.168.10.20 (memiliki VIP saat aktif)
- Secondary DB: 192.168.10.21 (standby, data sudah tersinkronisasi)
- Heartbeat interval: 2 detik
- Timeout threshold: 3 kali miss (6 detik)
- Replikasi: Streaming replication real-time

Ketika primary down, secondary mengambil VIP 192.168.10.100 dan menjadi master baru dalam hitungan detik. Aplikasi tetap konek ke IP yang sama tanpa perlu konfigurasi ulang. Data yang sudah direplikasi sebelumnya langsung tersedia.

---

**Catatan Penting:**
- Selalu test failover secara berkala untuk memastikan mekanisme berjalan dengan baik
- Monitor log failover untuk mendeteksi false positive atau masalah jaringan
- Pastikan data tersinkronisasi antara primary dan backup untuk menghindari data loss
- Dokumentasikan prosedur manual failback jika automatic failback gagal

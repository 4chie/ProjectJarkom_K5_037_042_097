# ProjectJarkom_K5_037_042_097



## 6. Pengalamatan Dinamis (DHCP Server dan DHCP Relay)

Pada Bagian 2.3 draf Anggota 1, perangkat pengguna (PC kerja, laptop WiFi, dan kamera CCTV) sengaja belum diberi alamat IP statis. Pengalamatan seluruh perangkat tersebut diotomatisasi menggunakan protokol DHCP dengan dua metode:
1. **DHCP Server Lokal pada Router**: Melayani host pada LAN Pegawai Balai Kota, WiFi Publik Barat, dan Puskesmas.
2. **DHCP Relay pada Router & Server Terpusat**: Melayani kamera pengawas pada LAN CCTV Kota yang dialokasikan dari server Data Center (`SRV-DC`).

---

### 6.1 Konfigurasi DHCP Server Lokal pada Router

Router Cisco menyediakan layanan DHCP terintegrasi. Setiap pool dipetakan ke network LAN masing-masing, dengan reservasi 10 alamat IP pertama (`excluded-address`) untuk antarmuka gateway router, switch, maupun kebutuhan perangkat statis lain agar tidak terjadi konflik alamat IP (*IP conflict*).

#### A. Router R-PUSAT (LAN Pegawai Balai Kota)

Router `R-PUSAT` melayani pemberian alamat IP otomatis untuk komputer pegawai di Balai Kota pada segmen `10.40.224.0/21`:

```cisco
! Reservasi 10 IP pertama
ip dhcp excluded-address 10.40.224.1 10.40.224.10

! Konfigurasi pool pegawai Balai Kota
ip dhcp pool POOL-PEGAWAI
 network 10.40.224.0 255.255.248.0
 default-router 10.40.224.1
 dns-server 10.40.239.2
```

- `network 10.40.224.0 255.255.248.0`: Menyediakan rentang alamat IP subnet `/21` (kapasitas 2.046 host).
- `default-router 10.40.224.1`: Memberikan parameter Option 3 (*Default Gateway*) mengarah ke port `Gi0/1` R-PUSAT.
- `dns-server 10.40.239.2`: Mengarahkan resolusi nama domain ke server `SRV-DC`.

---

#### B. Router R-BARAT (WiFi Publik Barat dan Puskesmas)

Router `R-BARAT` mengelola dua pool terpisah untuk membedakan segmen publik dan instansi kesehatan:

```cisco
! Reservasi IP infrastruktur
ip dhcp excluded-address 10.40.192.1 10.40.192.10
ip dhcp excluded-address 10.40.238.1 10.40.238.10

! Pool layanan nirkabel WiFi Publik Barat
ip dhcp pool POOL-WIFI-BARAT
 network 10.40.192.0 255.255.224.0
 default-router 10.40.192.1
 dns-server 10.40.239.2

! Pool fasilitas kesehatan Puskesmas
ip dhcp pool POOL-PUSKESMAS
 network 10.40.238.0 255.255.255.0
 default-router 10.40.238.1
 dns-server 10.40.239.2
```

- Permintaan DHCP yang masuk melalui port `Gi0/1` (`10.40.192.1`) otomatis dialokasikan dari `POOL-WIFI-BARAT` (`/19`).
- Permintaan DHCP yang masuk melalui port `Gi0/2` (`10.40.238.1`) otomatis dialokasikan dari `POOL-PUSKESMAS` (`/24`).

---

### 6.2 Konfigurasi DHCP Relay Khusus CCTV (R-DC dan SRV-DC)

Kamera pengawas perkotaan (`PC-CCTV-KOTA`) berada di LAN `10.40.96.0/19` yang terhubung ke port `Gi0/1` router `R-DC`. Sesuai pembagian tugas, pengelolaan sewa IP CCTV dipusatkan pada server Data Center **`SRV-DC`** (`10.40.239.2`) di port `Gi0/2`.

Karena pesan *DHCP Discover* dikirimkan secara broadcast layer 2/3 (`255.255.255.255`), paket tersebut dibuang (*dropped*) oleh router `R-DC`. Router `R-DC` harus dikonfigurasi sebagai **DHCP Relay Agent** menggunakan perintah `ip helper-address`.

#### A. Konfigurasi Relay pada Router R-DC

```cisco
interface GigabitEthernet0/1
 ip address 10.40.96.1 255.255.224.0
 ip helper-address 10.40.239.2
 duplex auto
 speed auto
!
interface GigabitEthernet0/2
 ip address 10.40.239.1 255.255.255.0
 duplex auto
 speed auto
```

**Cara Kerja Relay:**
1. Klien CCTV memancarkan broadcast *DHCP Discover*.
2. Port `Gi0/1` mencegat paket broadcast dan mengubahnya menjadi paket **unicast** dengan tujuan `10.40.239.2` (`SRV-DC`).
3. `R-DC` menyisipkan alamat IP antarmuka penerimanya (`10.40.96.1`) ke dalam field **GIADDR** (*Gateway IP Address*).
4. `SRV-DC` membaca nilai GIADDR tersebut, mencocokkannya dengan pool `CCTV` (`10.40.96.0/19`), dan mengirimkan penawaran IP (*DHCP Offer*) kembali secara unicast ke `R-DC`.

#### B. Konfigurasi Layanan DHCP pada Server SRV-DC

Pada menu GUI **Services > DHCP** di `SRV-DC`:
- Service: **On**
- Pool Name: `CCTV`
- Default Gateway: `10.40.96.1`
- DNS Server: `10.40.239.2`
- Start IP Address: `10.40.96.0` (penyewaan host dimulai dari `10.40.96.11`)
- Subnet Mask: `255.255.224.0`
- Maximum Number of Users: `250`

---

### 6.3 Hasil Pengujian DHCP

Seluruh komputer pengguna berhasil memperoleh konfigurasi IP dinamis secara tepat tanpa adanya tumpang tindih.

| Perangkat Klien | Subnet LAN | IP Diperoleh | Gateway | DHCP Server | Keterangan Hasil |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PC-PEGAWAI-BALAIKOTA** | `10.40.224.0/21` | `10.40.224.11` | `10.40.224.1` | `10.40.224.1` (R-PUSAT) | Berhasil disewakan via DHCP lokal |
| **PC-PUSKESMAS** | `10.40.238.0/24` | `10.40.238.11` | `10.40.238.1` | `10.40.238.1` (R-BARAT) | Berhasil disewakan via DHCP lokal |
| **PC-CCTV-KOTA** | `10.40.96.0/19` | `10.40.96.11` | `10.40.96.1` | `10.40.239.2` (SRV-DC) | Berhasil disewakan via DHCP Relay |

#### Bukti Pengujian:

1. **Konfigurasi & Binding DHCP pada Router R-BARAT**:
   Menampilkan konfigurasi pool `POOL-WIFI-BARAT` dan `POOL-PUSKESMAS`. Tabel `show ip dhcp binding` mencatat IP `10.40.192.11` dan `10.40.238.11` terikat secara otomatis (*Automatic*).
   
   ![Konfigurasi dan Binding DHCP R-BARAT](screenshots/01_dhcp_server_r-barat_config_binding.png)

2. **Konfigurasi Pool Pegawai pada Router R-PUSAT**:
   Konfigurasi `POOL-PEGAWAI` serta tabel sewa IP awal.

   ![Konfigurasi DHCP Pool Pegawai R-PUSAT](screenshots/02_dhcp_server_r-pusat_pool_pegawai.png)

3. **Detail Status Binding pada Router R-PUSAT**:
   Memastikan MAC Address komputer pegawai (`00E0.A3B4.8363`) terikat aktif pada alamat `10.40.224.11`.

   ![Detail Binding DHCP R-PUSAT](screenshots/03_dhcp_server_r-pusat_binding_detail.png)

4. **Verifikasi `ipconfig /all` pada PC-PEGAWAI-BALAIKOTA**:
   Klien menerima IP `10.40.224.11`, netmask `255.255.248.0`, gateway `10.40.224.1`, dan DNS `10.40.239.2`.

   ![IPConfig PC Pegawai Balai Kota](screenshots/05_client_pc_pegawai_dhcp_ipconfig.png)

5. **Verifikasi `ipconfig /all` pada PC-PUSKESMAS**:
   Klien menerima IP `10.40.238.11`, netmask `255.255.255.0`, gateway `10.40.238.1`, dan DNS `10.40.239.2`.

   ![IPConfig PC Puskesmas](screenshots/07_client_pc_puskesmas_dhcp_ipconfig.png)

6. **Konfigurasi Service DHCP Pool CCTV pada SRV-DC**:
   Tampilan parameter service DHCP untuk subnet CCTV pada server Data Center.

   ![Konfigurasi Service DHCP Pool CCTV di SRV-DC](screenshots/08_dhcp_server_srv-dc_pool_cctv.png)

7. **Konfigurasi `ip helper-address` pada Interface R-DC**:
   Potongan running-config antarmuka `Gi0/1` R-DC yang menjalankan peran DHCP Relay ke `10.40.239.2`.

  ![Konfigurasi IP Helper-Address R-DC](screenshots/09_dhcp_relay_r-dc_ip_helper_config.png)

8. **Verifikasi `ipconfig /all` pada PC-CCTV-KOTA (Pembuktian Relay)**:
   Klien CCTV memperoleh IP `10.40.96.11` dengan gateway `10.40.96.1`, dan field **DHCP Server** tertulis **`10.40.239.2`**. Ini membuktikan secara langsung bahwa paket DHCP berhasil diseberangkan lintas router dari Data Center.

  ![IPConfig PC CCTV Kota](screenshots/06_client_pc_cctv_relay_dhcp_ipconfig.png)

---

## 7. Network Address Translation (NAT) untuk Akses 8.8.8.8

### 7.1 Alasan dan Konfigurasi NAT di R-PUSAT

Pada pengujian Bagian 4.4 draf Anggota 1, ping dari router `R-PUSAT` ke server internet `8.8.8.8` telah berhasil, namun pengujian dari PC pengguna belum bisa dilakukan. Hal ini disebabkan oleh:
- Seluruh host internal memakai alamat IPv4 privat RFC 1918 (`10.40.0.0/16`).
- Router `R-ISP` sengaja tidak memiliki tabel routing ke blok alamat privat `10.40.0.0/16`.

Agar paket balasan dari internet dapat kembali ke host pengirim, router `R-PUSAT` dikonfigurasi dengan **NAT Overload / PAT (Port Address Translation)** pada antarmuka uplink WAN:

```cisco
! 1. Interface luar (Outside) menghadap ke ISP
interface GigabitEthernet0/0
 ip address 203.0.113.21 255.255.255.252
 ip nat outside

! 2. Interface dalam (Inside) menghadap ke LAN Balai Kota dan Backbone
interface GigabitEthernet0/1
 ip address 10.40.224.1 255.255.248.0
 ip nat inside

interface GigabitEthernet0/2
 ip address 10.40.241.37 255.255.255.252
 ip nat inside

! 3. ACL mengidentifikasi seluruh subnet privat internal
access-list 1 permit 10.40.0.0 0.0.255.255

! 4. Translasi dinamis overload ke IP publik port Gi0/0
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! 5. Default route keluar menuju ISP (melanjutkan Bagian 3.1 Anggota 1)
ip route 0.0.0.0 0.0.0.0 203.0.113.22
```

**Penjelasan Perintah:**
- `ip nat outside`: Menetapkan `Gi0/0` sebagai gerbang keluar publik menuju `R-ISP`.
- `ip nat inside`: Menetapkan antarmuka LAN lokal dan interkoneksi transit sebagai zona asal paket privat.
- `overload`: Mengizinkan seluruh perangkat privat berbagi satu alamat IP publik `203.0.113.21` dengan membedakan sesi berdasarkan port layer 4 dan identifier ICMP.

---

### 7.2 Hasil Pengujian NAT

Akses internet dari perangkat pengguna diuji dari `PC-PEGAWAI-BALAIKOTA` menuju `8.8.8.8`:

| Pengujian | Sumber | Tujuan | Hasil yang Diharapkan | Hasil Pengujian |
| :--- | :--- | :--- | :--- | :--- |
| `ping 8.8.8.8` | PC-PEGAWAI-BALAIKOTA (`10.40.224.11`) | `8.8.8.8` | Reply 4/4 (*0% loss*) | **Berhasil** |
| `show ip nat translations` | R-PUSAT CLI | N/A | Entri translasi `10.40.224.11` -> `203.0.113.21` tercatat | **Berhasil** |

#### Bukti Pengujian:

1. **Konfigurasi Interface NAT R-PUSAT**:
   Potongan running-config `R-PUSAT` membuktikan status `ip nat outside` pada `Gi0/0` serta `ip nat inside` pada `Gi0/1` dan `Gi0/2`.
   
   ![Konfigurasi Antarmuka NAT R-PUSAT](screenshots/04_r-pusat_running_config_interfaces_nat_gre.png)

2. **Uji Ping ke 8.8.8.8 dari PC Pegawai Balai Kota**:
   Empat paket ICMP dijawab lengkap dengan waktu tempuh `<1ms` dan `TTL=126`.
   
   ![Ping ke 8.8.8.8 dari PC Pegawai](screenshots/10_nat_test_ping_google_8888_pc_pegawai.png)

3. **Verifikasi Tabel `show ip nat translations` pada R-PUSAT**:
   Tabel membuktikan alamat **Inside Local** `10.40.224.11` ditranslasikan secara dinamis menjadi alamat **Inside Global** `203.0.113.21` untuk menjangkau destinasi **Outside Global** `8.8.8.8`.
   
   ![Tabel NAT Translations R-PUSAT](screenshots/11_nat_show_ip_nat_translations_r-pusat.png)

---

## 8. GRE Tunnel antara Balai Kota dan Pelabuhan

### 8.1 Desain dan Konfigurasi Interface Tunnel0

Sesuai catatan Bagian 5 draf Anggota 1, penggantian `R-PUSAT` ke router Cisco 2911 (IOS 15.1) memungkinkan pembuatan antarmuka virtual `interface Tunnel0`. Jalur ini menghubungkan Balai Kota (`R-PUSAT`) dan Pelabuhan (`R-PELABUHAN`) secara *point-to-point* privat di atas infrastruktur transit yang ada.

#### Parameter Konfigurasi Tunnel0:

| Parameter | Router R-PUSAT (Balai Kota) | Router R-PELABUHAN (Pelabuhan) | Alasan Teknis |
| :--- | :--- | :--- | :--- |
| **Interface Virtual** | `Tunnel0` | `Tunnel0` | Port logis terowongan virtual GRE |
| **Alamat IP Tunnel** | `192.168.5.1 255.255.255.252` | `192.168.5.2 255.255.255.252` | Subnet `/30` Kelompok 5 ($K = 5$) |
| **MTU** | `1476` | `1476` | Mencegah fragmentasi akibat overhead GRE (24 byte) |
| **Tunnel Source** | `GigabitEthernet0/2` (atau IP `10.40.241.37`) | `GigabitEthernet0/0` (atau IP `10.40.241.54`) | Titik awal pengiriman paket underlay |
| **Tunnel Destination** | `10.40.241.54` (IP fisik R-PELABUHAN) | `10.40.241.37` (IP fisik R-PUSAT) | Titik akhir tujuan pengiriman paket underlay |

```cisco
! Konfigurasi di R-PUSAT
interface Tunnel0
 ip address 192.168.5.1 255.255.255.252
 mtu 1476
 tunnel source GigabitEthernet0/2
 tunnel destination 10.40.241.54

! Konfigurasi di R-PELABUHAN
interface Tunnel0
 ip address 192.168.5.2 255.255.255.252
 mtu 1476
 tunnel source GigabitEthernet0/0
 tunnel destination 10.40.241.37
```

**Alasan Penetapan MTU 1476 Byte:**
Ukuran MTU Ethernet standar adalah 1500 byte. GRE menambahkan header pembungkus sebesar 24 byte (20 byte IPv4 outer header + 4 byte GRE header). Mengatur MTU menjadi $1500 - 24 = 1476$ byte memastikan paket berukuran maksimal tidak mengalami pemotongan/fragmentasi (*IP fragmentation*) di sepanjang jalur transit.

---

### 8.2 Pembaruan Rute Statis Melintasi Tunnel (Penggantian Rute Anggota 1)

Pada Bagian 3.1 draf Anggota 1 tercatat:
> *"route R-PUSAT ke LAN Pelabuhan lewat R-UTARA (10.40.241.38) akan diganti ke tunnel GRE pada soal 6 (Anggota 2)."*

Sesuai instruksi tersebut, tabel routing statis diperbarui:

1. **Pembaruan di R-PUSAT**:
   Rute lama ke LAN Pelabuhan via next-hop R-UTARA (`10.40.241.38`) dialihkan ke next-hop IP Tunnel `192.168.5.2`:
   ```cisco
   ip route 10.40.160.0 255.255.224.0 192.168.5.2
   ip route 10.40.240.0 255.255.255.128 192.168.5.2
   ```

2. **Penambahan di R-PELABUHAN**:
   Rute spesifik menuju LAN Balai Kota diarahkan melalui next-hop IP Tunnel `192.168.5.1`:
   ```cisco
   ip route 10.40.224.0 255.255.248.0 192.168.5.1
   ip route 10.40.241.0 255.255.255.224 192.168.5.1
   ```

Dengan konfigurasi ini, seluruh paket data antara Balai Kota dan Pelabuhan langsung masuk ke terowongan enkapsulasi GRE tanpa melalui pemrosesan routing biasa di router perantara `R-UTARA`.

---

### 8.3 Hasil Pengujian GRE Tunnel

| No | Pengujian | Titik Uji | Target | Hasil yang Diharapkan | Hasil |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Status Antarmuka Tunnel0 | R-PUSAT & R-PELABUHAN | `Tunnel0` | Status `up/up` | **Berhasil** |
| 2 | Ping Antar-Endpoint Tunnel | R-PUSAT CLI | `192.168.5.2` | Reply 5/5 (`!!!!!`) | **Berhasil** |
| 3 | Ping Antar-Endpoint Tunnel | R-PELABUHAN CLI | `192.168.5.1` | Reply 5/5 (`!!!!!`) | **Berhasil** |
| 4 | Verifikasi Rute Overlay | R-PUSAT & R-PELABUHAN | `show ip route` | Prefix diarahkan via IP Tunnel | **Berhasil** |
| 5 | Ping End-to-End Antar-LAN | PC Balai Kota (`10.40.224.11`) | PC Pelabuhan (`10.40.160.11`) | Reply 4/4 (*0% loss*) | **Berhasil** |
| 6 | Traceroute Enkapsulasi | PC Balai Kota (`10.40.224.11`) | PC Pelabuhan (`10.40.160.11`) | Hop 2 langsung IP Tunnel `192.168.5.2` | **Berhasil** |

#### Bukti Pengujian:

1. **Konfigurasi Interface Tunnel0 pada R-PUSAT**:
   Running-config antarmuka Tunnel0 dengan IP `192.168.5.1/30` dan MTU `1476`.

   ![Konfigurasi Tunnel0 pada R-PUSAT](screenshots/12_gre_tunnel0_config_r-pusat.png)

2. **Konfigurasi Interface Tunnel0 pada R-PELABUHAN**:
   Running-config antarmuka Tunnel0 dengan IP `192.168.5.2/30` dan MTU `1476`.

   ![Konfigurasi Tunnel0 pada R-PELABUHAN](screenshots/13_gre_tunnel0_config_r-pelabuhan.png)

3. **Status Antarmuka Brief pada R-PUSAT**:
   Menunjukkan `Tunnel0` berstatus aktif normal (**up/up**).

   ![Interface Brief R-PUSAT](screenshots/15_gre_r-pusat_show_ip_int_brief.png)

4. **Status Antarmuka Brief pada R-PELABUHAN**:
   Menunjukkan `Tunnel0` berstatus aktif normal (**up/up**).

   ![Interface Brief R-PELABUHAN](screenshots/14_gre_r-pelabuhan_show_ip_int_brief.png)

5. **Tabel Routing R-PUSAT Membuktikan Rute via Tunnel**:
   Output `show ip route` membuktikan rute `10.40.160.0/19` dan `10.40.240.0/25` diarahkan via `192.168.5.2`.

   ![Tabel Routing R-PUSAT via Tunnel](screenshots/16_routing_table_r-pusat_via_tunnel.png)

6. **Tabel Routing R-PELABUHAN Membuktikan Rute via Tunnel**:
   Output `show ip route` membuktikan rute `10.40.224.0/21` diarahkan via `192.168.5.1`.
   
   ![Tabel Routing R-PELABUHAN via Tunnel](screenshots/17_routing_table_r-pelabuhan_via_tunnel.png)

7. **Ping Tunnel dari R-PUSAT ke R-PELABUHAN (`192.168.5.2`)**:
   Tingkat keberhasilan 100% (5/5 paket balasan diterima).
   
   ![Ping Tunnel dari R-PUSAT ke R-PELABUHAN](screenshots/21_gre_ping_tunnel_pusat_to_pelabuhan.png)

8. **Ping Tunnel dari R-PELABUHAN ke R-PUSAT (`192.168.5.1`)**:
   Tingkat keberhasilan 100% (5/5 paket balasan diterima).
   
   ![Ping Tunnel dari R-PELABUHAN ke R-PUSAT](screenshots/20_gre_ping_tunnel_pelabuhan_to_pusat.png)

9. **Ping End-to-End dari PC Balai Kota ke PC Pelabuhan (`10.40.160.11`)**:
   Komunikasi antar komputer pengguna lintas pulau/kantor berhasil tanpa packet loss.

   ![Ping End-to-End PC Balai Kota ke PC Pelabuhan](screenshots/18_gre_test_ping_balaikota_to_pelabuhan.png)

10. **Traceroute Membuktikan Jalur GRE Tunnel**:
    Eksekusi `tracert 10.40.160.11` dari `PC-PEGAWAI-BALAIKOTA`:
    
    ![Traceroute Membuktikan Jalur GRE Tunnel](screenshots/19_gre_test_traceroute_balaikota_to_pelabuhan.png)

    ```text
    C:\>tracert 10.40.160.11

    Tracing route to 10.40.160.11 over a maximum of 30 hops:

      1   0 ms      0 ms      0 ms      10.40.224.1
      2   0 ms      0 ms      0 ms      192.168.5.2
      3   0 ms      0 ms      1 ms      10.40.160.11

    Trace complete.
    ```
# Laporan Mid-Project Jarkom - Kelompok 5

> Draf bagian **Anggota 3** (Verifikasi CIDR/VLSM, Server Aplikasi, dan Dynamic Routing OSPF).

## 1. Verifikasi Perhitungan CIDR & VLSM

### 1.1 Ringkasan Verifikasi & Perbandingan Efisiensi

Berdasarkan hasil verifikasi perhitungan pada spreadsheet kelompok (blok utama `10.40.0.0/16`, dengan nilai $K = 5$), dipilih skema **VLSM (Variable Length Subnet Mask)** dibandingkan alokasi blok tetap per router (CIDR).

* **Total Kebutuhan Host (14 LAN):** `42.080` host.
* **Skema VLSM (`10.40.0.0/16`):** Menyediakan kapasitas total `65.536` alamat ($2^{16}$). Seluruh 14 LAN dan 7 link *point-to-point* (`/30`) muat dalam satu blok `/16` tanpa adanya *overlapping* IP, menghasilkan tingkat efisiensi pemakaian sebesar **64,21%** ($\frac{42.080}{65.536} \times 100\%$).
* **Pembanding Skema CIDR (Blok Pangkat 2 per Router):** Apabila setiap router dialokasikan satu blok tunggal berbasis pangkat 2 tertinggi, total jaringan membutuhkan ruang sebesar blok `/14` (`262.144` alamat) dengan tingkat efisiensi hanya **16,05%** ($\frac{42.080}{262.144} \times 100\%$).

---

### 1.2 Tabel Verifikasi Alokasi Subnet LAN (VLSM)

Pengurutan alokasi dilakukan dari kebutuhan host terbesar hingga terkecil untuk mencegah tumpang tindih (*overlapping*) subnet:

| No | Router | Jaringan / LAN | Kebutuhan Host | Kapasitas Host ($2^n - 2$) | Network ID | Prefix | Subnet Mask | Gateway (Host 1) | Usable Host Range | Broadcast |
| :-: | :--- | :--- | :---: | :---: | :--- | :---: | :--- | :--- | :--- | :--- |
| 1 | R-UTARA | WiFi Publik Utara | 11.000 | 16.382 | 10.40.0.0 | `/18` | 255.255.192.0 | 10.40.0.1 | 10.40.0.1 - 10.40.63.254 | 10.40.63.255 |
| 2 | R-SELATAN | WiFi Publik Selatan | 6.500 | 8.190 | 10.40.64.0 | `/19` | 255.255.224.0 | 10.40.64.1 | 10.40.64.1 - 10.40.95.254 | 10.40.95.255 |
| 3 | R-DC | CCTV Kota | 6.000 | 8.190 | 10.40.96.0 | `/19` | 255.255.224.0 | 10.40.96.1 | 10.40.96.1 - 10.40.127.254 | 10.40.127.255 |
| 4 | R-TIMUR | WiFi Publik Timur | 5.500 | 8.190 | 10.40.128.0 | `/19` | 255.255.224.0 | 10.40.128.1 | 10.40.128.1 - 10.40.159.254 | 10.40.159.255 |
| 5 | R-PELABUHAN | Sensor dan SCADA Pelabuhan | 5.000 | 8.190 | 10.40.160.0 | `/19` | 255.255.224.0 | 10.40.160.1 | 10.40.160.1 - 10.40.191.254 | 10.40.191.255 |
| 6 | R-BARAT | WiFi Publik Barat | 4.500 | 8.190 | 10.40.192.0 | `/19` | 255.255.224.0 | 10.40.192.1 | 10.40.192.1 - 10.40.223.254 | 10.40.223.255 |
| 7 | R-PUSAT | Pegawai Balai Kota | 1.800 | 2.046 | 10.40.224.0 | `/21` | 255.255.248.0 | 10.40.224.1 | 10.40.224.1 - 10.40.231.254 | 10.40.231.255 |
| 8 | R-TIMUR | Sensor Banjir Timur | 700 | 1.022 | 10.40.232.0 | `/22` | 255.255.252.0 | 10.40.232.1 | 10.40.232.1 - 10.40.235.254 | 10.40.235.255 |
| 9 | R-UTARA | Sensor Lalu Lintas Utara | 450 | 510 | 10.40.236.0 | `/23` | 255.255.254.0 | 10.40.236.1 | 10.40.236.1 - 10.40.237.254 | 10.40.237.255 |
| 10 | R-BARAT | Puskesmas dan Sekolah Barat | 220 | 254 | 10.40.238.0 | `/24` | 255.255.255.0 | 10.40.238.1 | 10.40.238.1 - 10.40.238.254 | 10.40.238.255 |
| 11 | R-DC | Server dan Storage DC | 180 | 254 | 10.40.239.0 | `/24` | 255.255.255.0 | 10.40.239.1 | 10.40.239.1 - 10.40.239.254 | 10.40.239.255 |
| 12 | R-PELABUHAN | Kantor Pelabuhan | 120 | 126 | 10.40.240.0 | `/25` | 255.255.255.128 | 10.40.240.1 | 10.40.240.1 - 10.40.240.126 | 10.40.240.127 |
| 13 | R-SELATAN | PJU Pintar Selatan | 90 | 126 | 10.40.240.128 | `/25` | 255.255.255.128 | 10.40.240.129 | 10.40.240.129 - 10.40.240.254 | 10.40.240.255 |
| 14 | R-PUSAT | Server Balai Kota | 20 | 30 | 10.40.241.0 | `/27` | 255.255.255.224 | 10.40.241.1 | 10.40.241.1 - 10.40.241.30 | 10.40.241.31 |
| **Total** | | **14 Segmen LAN** | **42.080** | **61.700** | | | | | | |

> **Catatan Alokasi Server & DHCP:**
> * Alamat `.1` pada setiap LAN dialokasikan sebagai **Default Gateway** di interface router.
> * Alamat `.2` pada LAN Server dialokasikan secara statis untuk server (`SRV-DC`: `10.40.239.2` dan `SRV-BALAIKOTA`: `10.40.241.2`).
> * Rentang `.1` hingga `.10` pada setiap LAN dikecualikan (`excluded-address`) dari DHCP pool untuk mencegah konflik IP.

---

### 1.3 Tabel Verifikasi Alokasi Link Backbone, ISP, Internet, & Tunnel GRE

Alokasi *point-to-point* (`/30`) untuk koneksi antar-router mengambil blok kosong setelah subnet LAN terakhir (`10.40.241.32` ke atas):

| Link / Segmen | Koneksi Perangkat | Network ID | Prefix | Subnet Mask | Usable Host Range | Broadcast | Pembagian IP Interface |
| :--- | :--- | :--- | :---: | :--- | :--- | :--- | :--- |
| **L1** | R-PUSAT $\leftrightarrow$ R-DC | 10.40.241.32 | `/30` | 255.255.255.252 | 10.40.241.33 - 10.40.241.34 | 10.40.241.35 | R-PUSAT (`.33`), R-DC (`.34`) |
| **L2** | R-PUSAT $\leftrightarrow$ R-UTARA | 10.40.241.36 | `/30` | 255.255.255.252 | 10.40.241.37 - 10.40.241.38 | 10.40.241.39 | R-PUSAT (`.37`), R-UTARA (`.38`) |
| **L3** | R-PUSAT $\leftrightarrow$ R-SELATAN | 10.40.241.40 | `/30` | 255.255.255.252 | 10.40.241.41 - 10.40.241.42 | 10.40.241.43 | R-PUSAT (`.41`), R-SELATAN (`.42`) |
| **L4** | R-PUSAT $\leftrightarrow$ R-TIMUR | 10.40.241.44 | `/30` | 255.255.255.252 | 10.40.241.45 - 10.40.241.46 | 10.40.241.47 | R-PUSAT (`.45`), R-TIMUR (`.46`) |
| **L5** | R-PUSAT $\leftrightarrow$ R-BARAT | 10.40.241.48 | `/30` | 255.255.255.252 | 10.40.241.49 - 10.40.241.50 | 10.40.241.51 | R-PUSAT (`.49`), R-BARAT (`.50`) |
| **L6** | R-UTARA $\leftrightarrow$ R-PELABUHAN | 10.40.241.52 | `/30` | 255.255.255.252 | 10.40.241.53 - 10.40.241.54 | 10.40.241.55 | R-UTARA (`.53`), R-PELABUHAN (`.54`) |
| **L7 (Cadangan OSPF)** | R-DC $\leftrightarrow$ R-UTARA | 10.40.241.56 | `/30` | 255.255.255.252 | 10.40.241.57 - 10.40.241.58 | 10.40.241.59 | R-DC (`.57`), R-UTARA (`.58`) |
| **Link ISP** | R-PUSAT $\leftrightarrow$ R-ISP | 203.0.113.20 | `/30` | 255.255.255.252 | 203.0.113.21 - 203.0.113.22 | 203.0.113.23 | R-PUSAT (`.21`), R-ISP (`.22`) |
| **Internet Publik** | R-ISP $\leftrightarrow$ SRV-INTERNET | 8.8.8.0 | `/24` | 255.255.255.0 | 8.8.8.1 - 8.8.8.254 | 8.8.8.255 | R-ISP (`.1`), SRV-INTERNET (`.8`) |
| **Tunnel GRE** | R-PUSAT $\leftrightarrow$ R-PELABUHAN | 192.168.5.0 | `/30` | 255.255.255.252 | 192.168.5.1 - 192.168.5.2 | 192.168.5.3 | R-PUSAT (`.1`), R-PELABUHAN (`.2`) |

---

## 2. Konfigurasi Layanan Server Aplikasi

Konfigurasi layanan dilakukan pada dua server utama, yaitu **SRV-DC** (`10.40.239.2`) dan **SRV-BALAIKOTA** (`10.40.241.2`).

### 2.1 DNS Server (SRV-DC)

Layanan DNS diaktifkan pada SRV-DC untuk menerjemahkan nama domain `k5.smartcity.lab` beserta subdomainnya ke alamat IP Server terkait.

| Nama Domain | Tipe Record | IP Address Tujuan | Server Tujuan |
| :--- | :---: | :--- | :--- |
| `k5.smartcity.lab` | A Record | `10.40.241.2` | SRV-BALAIKOTA |
| `www.k5.smartcity.lab` | A Record | `10.40.241.2` | SRV-BALAIKOTA |
| `ftp.k5.smartcity.lab` | A Record | `10.40.239.2` | SRV-DC |
| `mail.k5.smartcity.lab` | A Record | `10.40.241.2` | SRV-BALAIKOTA |

### 2.2 Web Smart City (SRV-BALAIKOTA)

Layanan HTTP/HTTPS diaktifkan, dan file `index.html` diedit untuk menampilkan Portal Smart City milik Kelompok 5.

### 2.3 FTP Server (SRV-DC)

Layanan FTP diaktifkan untuk penyimpanan data CCTV. Dibuat akun dengan *username* `cctv` yang memiliki hak akses penuh (*Write, Read, Delete, Rename, List*).

### 2.4 Email Server (SRV-BALAIKOTA)

Layanan SMTP dan POP3 diaktifkan dengan nama domain `k5.smartcity.lab`. Dibuat akun pengguna `pegawai` (`pegawai@k5.smartcity.lab`) dan `pelabuhan` (`pelabuhan@k5.smartcity.lab`) untuk komunikasi antar-divisi.

---

## 3. Dynamic Routing OSPF & Link Cadangan L7

Bagian ini dikerjakan pada file terpisah (`K5_VLSM_OSPF.pkt`) untuk mensimulasikan redundansi jaringan. Seluruh *static route* antar-router dihapus dan digantikan oleh protokol *dynamic routing* OSPF.

### 3.1 Pemasangan Link Cadangan (L7)

Kabel cadangan L7 dipasang menggunakan port Gigabit Ethernet tambahan antara router **R-DC** dan **R-UTARA** dengan alokasi IP `10.40.241.56/30`.

* **R-DC (Fa0/0/0 via VLAN 17):** `10.40.241.57`
* **R-UTARA (Gi10/0):** `10.40.241.58`

### 3.2 Konfigurasi OSPF

OSPF diaktifkan (Process ID `1`) pada semua router. Jaringan didaftarkan menggunakan *wildcard mask* di `area 0`. Antarmuka yang mengarah ke jaringan pengguna (LAN) diset sebagai `passive-interface` agar pembaruan OSPF tidak dikirim ke perangkat pengguna.

![Verifikasi OSPF Neighbor](screenshots/ospf_neighbor.png)
*Gambar: Verifikasi `show ip ospf neighbor` menunjukkan R-UTARA telah membentuk hubungan tetangga yang FULL dengan R-DC melalui link cadangan.*

![Tabel Routing OSPF](screenshots/ospf_route.png)
*Gambar: Tabel routing menunjukkan rute-rute dinamis yang ditandai dengan kode "O" (OSPF).*

---

## 4. Hasil Pengujian Layanan

Berikut adalah bukti bahwa layanan di *Application Layer* berjalan dengan baik dan dapat diakses dari PC Klien menggunakan DNS.

### 4.1 Pengujian Web Server

Pengujian dilakukan dari `PC-PEGAWAI-BALAIKOTA` dengan mengakses URL `www.k5.smartcity.lab` melalui Web Browser.

![Pengujian Web Server](screenshots/test_web.png)

### 4.2 Pengujian FTP

Pengujian dilakukan dari Command Prompt di `PC-CCTV-KOTA`. Berhasil melakukan *login* dengan *username* `cctv` ke alamat `ftp.k5.smartcity.lab`.

![Pengujian FTP](screenshots/test_ftp.png)

### 4.3 Pengujian Email

Pengujian dilakukan dengan mengirimkan pesan dari `PC-PEGAWAI-BALAIKOTA` (`pegawai@k5.smartcity.lab`) kepada `PC-KANTOR-PELABUHAN` (`pelabuhan@k5.smartcity.lab`) dan sebaliknya.

![Pengujian Email](screenshots/test_email.png)

### 4.4 Pengujian Redundansi Jalur OSPF (Failover)

Saat link utama (`L1` atau `L2`) dimatikan secara paksa, OSPF secara otomatis mengalihkan jalur *routing* melewati kabel cadangan `L7`, sehingga layanan PING dan komunikasi antar-LAN tetap berjalan setelah sempat terputus sesaat (*Request Timed Out*).

![Pengujian Failover OSPF](screenshots/test_ospf_failover.png)

# Tugas-1-Jarkom-Fika-Arka-5027241071
Dokumentasi ini menjelaskan proses perancangan, alokasi IP, dan konfigurasi jaringan untuk memenuhi studi kasus Yayasan Pendidikan ARA, yang mencakup Kantor Pusat (HQ) dan Kantor Cabang (Branch).

* **Nama:** Fika Arka Nuriyah 
* **NRP:** 5027241071  
* **Base Network:**  
  * 5027241071 mod 256 \= 111  
  * **10.111.0.0**

## **1\. Desain Topologi**

Topologi yang digunakan adalah model **Hub-and-Spoke** untuk Kantor Pusat dan satu link WAN ke Kantor Cabang.

* **Kantor Pusat (Hub-and-Spoke):**  
  * **1 Router Sentral (Router0):** Bertindak sebagai *hub* atau "otak" utama jaringan. Router ini menangani semua rute antar departemen dan koneksi ke cabang.  
  * **5 Router Departemen (Spoke):** Masing-masing router (Sekretariat, Kurikulum, Guru, Sarpras, Server) mengelola LAN departemennya sendiri dan terhubung ke Router0.  
  * **1 Jaringan "Interlink":** Sebuah subnet /29 (10.111.3.224/29) dibuat khusus sebagai "lem" untuk menghubungkan Router0 dengan kelima router departemen melalui satu switch sentral.  
* **Kantor Cabang (Branch):**  
  * **1 Router Cabang (Router1):** Mengelola LAN Bidang Pengawas di lokasi cabang.  
* **Koneksi WAN (Pusat-Cabang):**  
  * **1 Jaringan "WAN Link":** Sebuah subnet /30 (10.111.3.240/30) digunakan untuk koneksi *point-to-point* antara Router0 (Pusat) dan Router1 (Cabang).

![Uploading image.png…]()


## **2\. Tabel Perhitungan VLSM**

Alokasi IP dihitung menggunakan VLSM, diurutkan dari kebutuhan host terbesar ke terkecil.

| Jaringan / Kegunaan | Kebutuhan Host | Jml IP Dibutuhkan | Network Address | Prefix | Subnet Mask | Range Host (Awal \- Akhir) | Broadcast | Gateway (Contoh) |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **Sekretariat** | 380 host | 512 | **10.111.0.0** | **/23** | 255.255.254.0 | 10.111.0.1 \- 10.111.1.254 | 10.111.1.255 | **10.111.0.1** |
| **Bidang Kurikulum** | 220 host | 256 | **10.111.2.0** | **/24** | 255.255.255.0 | 10.111.2.1 \- 10.111.2.254 | 10.111.2.255 | **10.111.2.1** |
| **Bidang Guru & Tendik** | 95 host | 128 | **10.111.3.0** | **/25** | 255.255.255.128 | 10.111.3.1 \- 10.111.3.126 | 10.111.3.127 | **10.111.3.1** |
| **Bidang Sarana Prasarana** | 45 host | 64 | **10.111.3.128** | **/26** | 255.255.255.192 | 10.111.3.129 \- 10.111.3.190 | 10.111.3.191 | **10.111.3.129** |
| **Bidang Pengawas (Cabang)** | 18 host | 32 | **10.111.3.192** | **/27** | 255.255.255.224 | 10.111.3.193 \- 10.111.3.222 | 10.111.3.223 | **10.111.3.193** |
| **"Interlink" Pusat** (R0-R6) | 6 host | 8 | **10.111.3.224** | **/29** | 255.255.255.248 | 10.111.3.225 \- 10.111.3.230 | 10.111.3.231 | (IP di router) |
| **Server & Admin** | 6 host | 8 | **10.111.3.232** | **/29** | 255.255.255.248 | 10.111.3.233 \- 10.111.3.238 | 10.111.3.239 | **10.111.3.233** |
| **"WAN Link"** (Pusat-Cabang) | 2 host | 4 | **10.111.3.240** | **/30** | 255.255.255.252 | 10.111.3.241 \- 10.111.3.242 | 10.111.3.243 | (IP di router) |

## **3\. CIDR (Supernetting) & Strategi Routing**

Untuk membuat *routing table* menjadi efisien, diterapkan *static routing* dengan strategi berikut:

### **Agregasi Rute Kantor Pusat (Supernetting)**

Untuk Router Cabang (Router1), tidak perlu mengetahui rincian 5+ subnet di Kantor Pusat. Semua subnet di Kantor Pusat (dari 10.111.0.0 sampai 10.111.3.239) dapat diagregasi menjadi satu *supernet*.

* **Subnet yang diagregasi:** 10.111.0.0/23, 10.111.2.0/24, 10.111.3.0/25, 10.111.3.128/26, 10.111.3.232/29.  
* **Hasil Supernet:** **10.111.0.0/22**  
* **Mask:** 255.255.252.0  
* **Range:** Mencakup 10.111.0.0 s/d 10.111.3.255

### **Konfigurasi Routing**

* **Router Departemen (Spoke):**  
  * Menggunakan **1 Default Route (0.0.0.0 0.0.0.0)** yang menunjuk ke IP Router0 (Pusat) di jaringan Interlink (10.111.3.225).  
  * Perintah: ip route 0.0.0.0 0.0.0.0 10.111.3.225  
* **Router Pusat (Hub):**  
  * Menggunakan **Static Route** spesifik untuk setiap subnet, termasuk semua LAN departemen, LAN Server, dan LAN Cabang.  
  * *Contoh Rute ke Guru:* ip route 10.111.3.0 255.255.255.128 10.111.3.228 (lewat R\_Guru)  
  * *Contoh Rute ke Cabang:* ip route 10.111.3.192 255.255.255.224 10.111.3.242 (lewat R\_Cabang)  
* **Router Cabang (Branch):**  
  * Menggunakan **1 Supernet Route** (hasil CIDR) untuk menjangkau *semua* jaringan di Kantor Pusat.  
  * Perintah: ip route 10.111.0.0 255.255.252.0 10.111.3.241

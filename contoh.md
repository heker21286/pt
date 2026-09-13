# LAPORAN HASIL PENGUJIAN KEAMANAN

*Penetration Testing Report*

> **KLASIFIKASI: RAHASIA**  
> Laporan Pengujian Keamanan — Internal Lab Targets

---

# 1. Executive Summary

Pengujian penetrasi model black-box dilakukan terhadap dua target aplikasi web pada jaringan internal lab, yaitu TeknoBantu Helpdesk (192.168.56.110:8083) dan KlaimKu, aplikasi klaim reimburse PT Cahaya Abadi Sejahtera (192.168.56.109:8084). Tujuan pengujian adalah mengidentifikasi kerentanan pada validasi input, kontrol otorisasi, dan konfigurasi sistem yang berpotensi dieksploitasi oleh pihak yang tidak berwenang. Metodologi mengikuti alur reconnaissance (port/service scanning), enumerasi direktori dan endpoint, hingga eksploitasi kerentanan yang ditemukan sampai tercapainya akses tingkat root pada kedua sistem, dengan area fokus pada fitur unggah berkas dan local file inclusion/arbitrary file read.

Selama periode pengujian, teridentifikasi 2 (dua) temuan dengan keparahan Critical pada kedua target,  masing-masing berujung pada Remote Code Execution dan eskalasi privilese penuh ke root.

| Tingkat Keparahan | Jumlah Temuan |
|---|---:|
| **Critical** | **2** |
| High | 0 |
| Medium | 0 |
| Low | 0 |
| Informational | 0 |

## Rekomendasi Utama

Akar masalah utama pada kedua target adalah lemahnya validasi berkas unggahan/parameter path di sisi server serta konfigurasi sistem operasi yang memberikan hak istimewa berlebih (excessive capability/SUID) pada binary yang tidak semestinya, yang pada akhirnya memungkinkan penyerang mengeksekusi kode arbitrer dan meningkatkan privilese hingga root pada kedua server target.

- Terapkan validasi berkas unggahan berbasis konten aktual (magic byte/MIME sniffing), gunakan whitelist ekstensi yang ketat termasuk varian eksekusi alternatif (.phtml, .phar, .pht), dan nonaktifkan eksekusi skrip pada direktori unggahan.

- Audit seluruh binary dengan Linux capability (getcap -r
/) dan SUID bit (find / -perm -4000) secara berkala; cabut hak istimewa yang tidak diperlukan seperti cap_setuid pada interpreter (python3) dan SUID pada utilitas umum (find).

- Batasi endpoint debug/diagnostik (mis. `debug_viewer.php`) agar tidak dapat diakses di lingkungan produksi, dan terapkan whitelist path/realpath check bila fungsi baca berkas dinamis benar-benar diperlukan.

- Terapkan prinsip least privilege pada akun database aplikasi dan jangan mengekspos service database (MySQL) ke jaringan yang lebih luas dari yang diperlukan.

- Perbaiki filter command injection dengan `escapeshellarg()`/`escapeshellcmd()` atau hindari pemanggilan shell sepenuhnya; blacklist karakter tunggal (;, |, &) tidak cukup karena newline dapat memisahkan perintah.

# 2. Tabel Daftar Temuan

Berikut adalah ringkasan seluruh temuan yang diidentifikasi selama pengujian, diurutkan berdasarkan target.

| No | Judul Temuan | Severity |
|---:|---|---|
| 1 | Unrestricted File Upload pada Fitur Lampiran Tiket (TeknoBantu Helpdesk) Menyebabkan RCE dan Privilege Escalation ke Root via Capability Abuse | **Critical** |
| 2 | Arbitrary File Read (LFI) pada `debug_viewer.php` (KlaimKu) Berujung Command Injection dan Privilege Escalation ke Root via SUID `find` | **Critical** |

# 3. Detail Temuan

## Temuan 1: Unrestricted File Upload pada Fitur Lampiran Tiket (TeknoBantu Helpdesk) Menyebabkan RCE dan Privilege Escalation ke Root via Capability Abuse

**Severity:** Critical

### Deskripsi

Aplikasi helpdesk TeknoBantu (http://192.168.56.110:8083/) menyediakan fitur unggah lampiran pada form pembuatan tiket. Validasi tipe berkas di sisi server hanya memblokir ekstensi ".php" secara eksplisit dan memeriksa header Content-Type yang dikirim klien; ekstensi eksekusi alternatif yang tetap ditangani oleh handler PHP Apache — yaitu ".phtml" dan ".phar" — tidak masuk dalam daftar blokir sehingga lolos validasi. Berkas yang diunggah disimpan pada path yang dapat diakses dan dieksekusi langsung oleh web server, yaitu /uploads/tickets/<id>_<nama_file>.<ext>, tanpa autentikasi pada endpoint unggah. Dari titik ini penyerang memperoleh Remote Code Execution dengan konteks pengguna `www-data`. Enumerasi lanjutan pada sistem menemukan Linux capability `cap_setuid=ep` yang melekat pada binary `/usr/bin/python3.11`, sebuah hak istimewa yang mengizinkan proses Python memanggil `setuid(0)` tanpa memerlukan hak akses root maupun sudo, sehingga penyerang dapat meningkatkan privilese menjadi root sepenuhnya.

### Endpoint

- `POST /submit.php` — unggah lampiran tiket
- `GET /uploads/tickets/<id>_<nama_file>.<ext>` — eksekusi berkas

### URL PoC

`http://192.168.56.110:8083/submit.php`

### Langkah Proof of Concept (PoC)

1. Melakukan port scanning terhadap target dengan perintah nmap -sV -p- 192.168.56.110. Hasil scanning menunjukkan port 8083 terbuka dengan service Apache/2.4.68 (Debian).

> **Bukti 1:** Screenshot PoC dari dokumen sumber.

2. Memeriksa aplikasi dengam mengakses aplikasi secara langsung melalui browser dan melakukan brute-force direktori menggunakan dirsearch (dirsearch -u [http://192.168.56.110:8083](http://192.168.56.110:8083) -i 200,301). Ditemukan direktori /assets/, /includes/, /staff/, dan /uploads/, serta halaman aplikasi index.php (form tiket), submit.php (proses unggah), confirmation.php, dan staff/login.php.

> **Bukti 2:** Screenshot PoC dari dokumen sumber.

3. Menguji validasi ekstensi pada fitur unggah lampiran. Unggah berkas .txt ditolak dengan pesan "Lampiran harus berupa gambar (screenshot)". Unggah berkas .php dengan Content-Type image/png juga ditolak dengan pesan "Tipe file lampiran tidak diizinkan", mengindikasikan validasi berbasis whitelist ekstensi gambar dan/atau Content-Type.

> **Bukti 3:** Screenshot PoC dari dokumen sumber.

4. Menguji kerentanan dengan memodifikasi ekstensi pada filename. Hasil pengujian menunjukkan ekstensi .php, .php5, pht, .phtml, .phar berhasil diunggah dengan. Berikut hasil uji cmd.php5 berada pada id tiket 62 dan cmd.phtml berada pada id tiket 63.

> **Bukti 4:** Screenshot PoC dari dokumen sumber.

> **Bukti 5:** Screenshot PoC dari dokumen sumber.

5. Melakukan fuzzing pada direktori /uploads/ menggunakan ffuf (ffuf -u [http://192.168.56.110:8083/uploads/FUZZ -w /usr/share/wordlists/dirb/common.txt](http://192.168.56.110:8083/uploads/FUZZ%20-w%20/usr/share/wordlists/dirb/common.txt)) untuk mencari tahu dimana lokasi berkas diungah, ditemukan subdirektori /uploads/tickets/

> **Bukti 6:** Screenshot PoC dari dokumen sumber.

6. Melakukan beberapa kemungkinan format penamaan berkas melalui browser, didapatkan bawah format berkas adalah <nomor_tiket>_<nama_file>.<ext>. Akses berkas cmd.php5 dengan id tiket 62, sehingga berkas berada pada url [http://192.168.56.110:8083/uploads/tickets/62_cmd.php5](http://192.168.56.110:8083/uploads/tickets/62_cmd.php5), namun tidak dieksekusi sebagai PHP oleh Apache.

> **Bukti 7:** Screenshot PoC dari dokumen sumber.

Sementara itu berkas cmd.phmtl dengan id tiket 63 dieksekusi sebagai PHP oleh Apache. Penguji berhasil mendapatkan shell target sebagai user `www-data`.

> **Bukti 8:** Screenshot PoC dari dokumen sumber.

7. Melakukan enumerasi dari shell `www-data` untuk mengumpulkan informasi sistem operasi, memeriksa konfigurasi perintah sudo tanpa kata sandi, mengidentifikasi berkas dengan bit SUID aktif, serta memetakan Linux capabilities yang tertanam pada binary sistem guna menemukan jalur eskalasi hak akses (privilege escalation) dengan perintah sbb.

```text
http://192.168.56.110:8083/uploads/tickets/63_cmd.phtml?cmd=uname -a; cat /etc/os-release; sudo -n -l 2>/dev/null; find / -perm -4000 -type f 2>/dev/null; getcap -r / 2>/dev/null
```

Dari hasil enumerasi didapatkan bahwa terdapat sistem operasi Debian GNU/Linux 12 (bookworm) dengan kernel 6.1.0-53-amd64. Pemeriksaan terhadap hak akses sudo (`sudo -n -l`) tidak menghasilkan perintah yang dapat dieksekusi tanpa kata sandi oleh user `www-data`. Sementara itu, hasil pemindaian berkas SUID hanya menampilkan binary standar bawaan sistem operasi (seperti /usr/bin/passwd, /usr/bin/su, /usr/bin/mount, dll.) yang telah diperkeras (hardened) sehingga tidak dapat dieksploitasi. Namun, pada hasil pemindaian Linux capabilities (`getcap -r /`), ditemukan sebuah anomali berbahaya di mana interpreter bahasa pemrograman Python memiliki hak istimewa khusus, yaitu: `/usr/bin/python3.11` `cap_setuid=ep`.

Sehingga dipilih capability `cap_setuid=ep` pada `/usr/bin/python3.11` sebagai vektor eskalasi hak akses. Sebagai interpreter kode bebas, atribut ini memungkinkan user `www-data` memanipulasi proses Python untuk mengubah User ID (UID) secara langsung menjadi 0 (root) dan memicu privilege escalation.

> **Bukti 9:** Screenshot PoC dari dokumen sumber.

8. Memeriksa dokumen gtfobins.org untuk mengetahui eksploitasi binary menggunakan Linux capability Python. Perintah yang dapat digunakan untuk memicu shell interaktif (via reverse shell) adalah: python -c 'import os; os.`setuid(0)`; os.execl("/bin/sh", "sh")

> **Bukti 10:** Screenshot PoC dari dokumen sumber.

Sedangkan perintah yang digunakan untuk membaca isi berkas sensitif langsung dari browser (non-interaktif) adalah:

```bash
python -c 'import os; os.setuid(0); print(open("/path/to/input-file").read())'
```

> **Bukti 11:** Screenshot PoC dari dokumen sumber.

9. Mengeksploitasi capability tersebut untuk meningkatkan privilese menjadi root dan mengambil bukti akses (flag) dengan membaca file `/root/flag.txt` dengan perintah pada url sbb.

```text
http://192.168.56.110:8083/uploads/tickets/63_cmd.phtml?cmd=/usr/bin/python3.11 -c "import os; os.setuid(0); print(open('/root/flag.txt').read())"
```

> **Bukti 12:** Screenshot PoC dari dokumen sumber.

### Dampak

Penyerang tanpa autentikasi apa pun dapat mengeksekusi kode arbitrer pada server target melalui fitur unggah lampiran tiket, kemudian meningkatkan privilese menjadi root sepenuhnya melalui kesalahan konfigurasi capability pada interpreter Python. Dengan akses root, penyerang memiliki kendali penuh atas sistem — termasuk pembacaan/perubahan seluruh data, penanaman backdoor persisten, dan pergerakan lateral ke sistem lain pada jaringan yang sama.

### Skor CVSS (sangat dianjurkan)

`CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N`

**Estimasi Skor Dasar (CVSS-B): 9.3 — Critical**

### Rekomendasi Perbaikan

- Gunakan whitelist ekstensi berkas yang ketat dan verifikasi tipe berkas berdasarkan konten aktual (mis. finfo_file() / getimagesize()), bukan hanya nama ekstensi dan header Content-Type dari klien.

- Simpan berkas hasil unggahan di luar document root, atau nonaktifkan handler eksekusi PHP pada direktori unggahan (mis. php_admin_flag engine off pada konfigurasi Apache untuk direktori /uploads/).

- Cabut capability yang tidak diperlukan pada interpreter: setcap -r `/usr/bin/python3.11`, dan lakukan audit capability secara berkala dengan `getcap -r /`.

- Tambahkan autentikasi dan rate limiting pada endpoint pembuatan tiket/unggahan untuk mempersempit permukaan serangan.

## Temuan 2: Arbitrary File Read (LFI) pada `debug_viewer.php` (KlaimKu) Berujung Command Injection dan Privilege Escalation ke Root via SUID find

**Severity:** Critical

### Deskripsi

Aplikasi KlaimKu, sistem klaim reimburse milik PT Cahaya Abadi Sejahtera (http://192.168.56.109:8084/)  memiliki endpoint `debug_viewer.php` yang menerima parameter path dan menggunakannya langsung pada fungsi include() tanpa validasi maupun pembatasan direktori (path whitelist/realpath check). Kondisi ini memungkinkan pembacaan berkas arbitrer pada sistem, dan dengan memanfaatkan PHP filter wrapper (php://filter/convert.base64-encode/resource=), source code aplikasi PHP dapat diekstraksi dalam bentuk base64 tanpa dieksekusi. Dari hasil pembacaan source code, ditemukan kredensial database di `includes/db.php`. Akun database tersebut ternyata memiliki hak akses penuh (INSERT/UPDATE/DELETE) pada seluruh tabel di database klaimku, sehingga dimanfaatkan untuk menyisipkan akun pengguna baru dengan role "hr" secara langsung melalui query SQL. Dengan sesi hr yang sah, ditemukan celah command injection pada `admin/verify_rekening.php` dimana parameter rekening diteruskan ke `shell_exec()` dengan blacklist karakter yang tidak lengkap (hanya ;, |, dan &), sehingga karakter newline dapat digunakan untuk menyisipkan perintah tambahan. Dari RCE sebagai `www-data`, enumerasi SUID menemukan binary `/usr/bin/find` memiliki SUID bit yang aktif, sebuah GTFOBins klasik yang memungkinkan eksekusi perintah sebagai root.

### Endpoint

- `GET /debug_viewer.php?path=` — arbitrary file read / LFI
- `POST /admin/verify_rekening.php` — command injection, memerlukan role `hr`

### URL PoC

`http://192.168.56.109:8084/debug_viewer.php?path=`

### Langkah Proof of Concept (PoC)

1. Melakukan reconnaissance terhadap target menggunakan nmap, didapatkan port 8084 (Apache 2.4.68/Debian) dan 3306 (MySQL 8.0.46).

> **Bukti 13:** Screenshot PoC dari dokumen sumber.

2. Reconnaissance lanjutan menggunakan dirsearch, didapatkan direktori /assets/, /includes/, /uploads/ dan berkas login.php.

> **Bukti 14:** Screenshot PoC dari dokumen sumber.

3. Directory listing ditemukan terbuka pada /includes/ dan /uploads/. Berkas robots.txt mengungkap keberadaan endpoint `/debug_viewer.php` dan /admin/ melalui aturan Disallow.

> **Bukti 15:** Screenshot PoC dari dokumen sumber.

> **Bukti 16:** Screenshot PoC dari dokumen sumber.

> **Bukti 17:** Screenshot PoC dari dokumen sumber.

> **Bukti 18:** Screenshot PoC dari dokumen sumber.

4. Menguji parameter path pada `debug_viewer.php` dengan nilai `/etc/passwd`, berkas berhasil terbaca. Namun percobaan dengan path menuju berkas .php lain (mis. /var/www/html/login.php) justru mengeksekusi berkas tersebut, mengindikasikan penggunaan fungsi include() dan bukan readfile().

> **Bukti 19:** Screenshot PoC dari dokumen sumber.

> **Bukti 20:** Screenshot PoC dari dokumen sumber.

5. Melakukan bypass eksekusi menggunakan PHP filter wrapper agar source code terbaca sebagai teks (base64) alih-alih dieksekusi:

> **Bukti 21:** Screenshot PoC dari dokumen sumber.

6. Hasil decode base64 mengungkap beberapa berkas kunci: `includes/db.php` (kredensial database klaimku_app), `includes/auth.php` (helper session, otorisasi berbasis $_SESSION['role']), login.php dan search.php (keduanya menggunakan prepared statement, tidak rentan SQLi.

> **Bukti 22:** Screenshot PoC dari dokumen sumber.

> **Bukti 23:** Screenshot PoC dari dokumen sumber.

7. Menggunakan kredensial dari db.php untuk mengakses MySQL secara langsung, karena port 3306 terbuka ke jaringan:

> **Bukti 24:** Screenshot PoC dari dokumen sumber.

8. Alih-alih memecahkan hash bcrypt milik akun hr yang ada, menyisipkan akun baru dengan role hr langsung melalui query INSERT:

$ php -r "echo password_hash('password123', PASSWORD_BCRYPT), PHP_EOL;"

$2y$12$LvkQbFxnLkOqr3jXU3VEhu1ucg2qubw8PvQws3nbsqEL4YvDgZm8G

> **Bukti 25:** Screenshot PoC dari dokumen sumber.

MySQL [klaimku]> INSERT INTO users (username, password_hash, nama, role, created_at) VALUES ('hunter', '$2y$12$LvkQbFxnLkOqr3jXU3VEhu1ucg2qubw8PvQws3nbsqEL4YvDgZm8G', 'Hunter', 'hr', NOW());

Query OK, 1 row affected (0.030 sec)

> **Bukti 26:** Screenshot PoC dari dokumen sumber.

9.   Login
pada /login.php dengan akun baru berhasil dan sesi tercatat dengan role hr, memberikan akses ke panel `admin/verify_rekening.php`.

> **Bukti 27:** Screenshot PoC dari dokumen sumber.

10. Mengeksploitasi command injection pada `admin/verify_rekening.php`. Filter blacklist hanya memblokir karakter ;, |, dan & — karakter newline tidak difilter, sehingga dapat digunakan untuk menyisipkan perintah baru pada baris terpisah:

> **Bukti 28:** Screenshot PoC dari dokumen sumber.Output perintah muncul pada

halaman hasil, membuktikan Remote Code Execution dengan konteks pengguna `www-data`.

> **Bukti 29:** Screenshot PoC dari dokumen sumber.

11. Melakukan enumerasi privilege escalation dari shell `www-data`:

uname -a     # container Docker, kernel 6.1.0-53-amd64

ls -la /home/     # kosong

find / -perm -4000 -type f     # SUID enumeration

Temuan kunci: `/usr/bin/find` memiliki SUID bit aktif

> **Bukti 30:** Screenshot PoC dari dokumen sumber.

12. Mengeksploitasi SUID pada `/usr/bin/find` (GTFOBins) untuk membaca berkas milik root dan mengambil bukti akses (flag):

> **Bukti 31:** Screenshot PoC dari dokumen sumber.

> **Bukti 32:** Screenshot PoC dari dokumen sumber.

Karaktaer %2b adalah URL Encode dari karakter plus (+). Penulisan ini wajib dilakukan karena pada protokol HTTP, karakter + mentah (tanpa encode) akan otomatis diterjemahkan oleh server web sebagai karakter spasi (whitespace). Jika dikirim tanpa encode, perintah find akan kehilangan argumen penutupnya dan menghasilkan error "missing argument to -exec". Dengan menggunakan %2b, server akan melakukan decode kembali menjadi karakter + yang valid, sehingga perintah -exec cat {} + dapat berjalan sempurna di sisi backend untuk membaca berkas flag.

### Dampak

Rantai kerentanan ini memungkinkan penyerang tanpa kredensial sah untuk membaca source code sensitif, mengambil alih database dengan hak akses penuh, membuat akun dengan privilese administratif (hr), mengeksekusi kode arbitrer pada server, dan pada akhirnya memperoleh akses root penuh. Dampaknya mencakup pencurian dan perubahan seluruh data klaim reimburse karyawan, penanaman backdoor persisten, serta kompromi total terhadap server dan seluruh layanan yang berjalan di atasnya.

### Skor CVSS (sangat dianjurkan)

`CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N`

**Estimasi Skor Dasar (CVSS-B): 9.3 — Critical**

### Rekomendasi Perbaikan

- Hapus atau nonaktifkan `debug_viewer.php` pada environment produksi; apabila fungsi baca berkas dinamis diperlukan, terapkan whitelist path absolut beserta pengecekan `realpath()` dan prefix direktori.

- Terapkan prinsip least privilege pada akun database aplikasi — pisahkan hak akses per operasi (read-only untuk fitur yang hanya membaca data) dan jangan mengekspos port MySQL (3306) ke jaringan yang lebih luas dari yang diperlukan.

- Perbaiki filter command injection dengan `escapeshellarg()`/`escapeshellcmd()`, atau hindari pemanggilan `shell_exec()` sama sekali dengan mengganti pendekatan menjadi pemanggilan API/pustaka native.

- Cabut SUID bit pada `/usr/bin/find` — tidak ada kebutuhan operasional bagi utilitas find untuk berjalan dengan hak akses root, dan lakukan audit SUID/capability secara berkala.

- Validasi role pengguna di sisi server melalui lookup database pada setiap request sensitif, jangan hanya mengandalkan nilai pada $_SESSION.

# 4. Lampiran

## A. Ruang Lingkup & Metodologi Pengujian

Pengujian dilakukan terhadap dua target pada jaringan lab internal: 192.168.56.110:8083 (TeknoBantu Helpdesk) dan 192.168.56.109:8084 (KlaimKu), dengan metode Black Box Web Application Penetration Testing. Alur pengujian meliputi port/service scanning, enumerasi direktori dan endpoint, identifikasi kerentanan validasi input dan kontrol akses, hingga eksploitasi sampai tercapainya akses tingkat root pada kedua target, mengikuti Rules of Engagement (RoE) yang disepakati sebelum pengujian dimulai.

## B. Tools yang Digunakan

- Browser

- Burp Suite Community Edition – Entercept proxy

- nmap — port dan service scanning

- dirsearch / ffuf — enumerasi direktori dan endpoint tersembunyi

- curl — pengiriman request HTTP

- MySQL client — akses langsung ke database aplikasi menggunakan kredensial yang ditemukan

- getcap / find (perm -4000) — enumerasi Linux capability dan SUID bit untuk privilege escalation

## C. Batasan Pengujian

Pengujian difokuskan pada eksploitasi kerentanan pada fitur unggah berkas dan local file inclusion/arbitrary file read pada endpoint-endpoint yang disebutkan di atas hingga tercapainya akses root pada kedua target. Aktivitas post-exploitation lanjutan setelah akses root tercapai (mis. pivoting ke sistem lain di luar target, eksfiltrasi data dalam jumlah besar), pengujian terhadap komponen/endpoint lain di luar cakupan di atas, serta pengujian non-teknis (mis. social engineering) tidak termasuk dalam ruang lingkup pengujian ini.

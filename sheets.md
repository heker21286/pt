 1. Executive Summary

Pengujian penetrasi dengan metode black-box dilakukan terhadap aplikasi web `target.com` untuk mengidentifikasi potensi kelemahan keamanan yang dapat dimanfaatkan oleh pihak yang tidak berwenang.

Hasil pengujian menemukan beberapa kelemahan keamanan pada aplikasi, yaitu Cross-Site Scripting (XSS), Local File Inclusion (LFI), Unrestricted File Upload/Upload Bypass yang dapat dieksploitasi hingga Remote Code Execution (RCE), serta Password Reuse.

Temuan yang memiliki dampak paling signifikan adalah mekanisme Unrestricted File Upload / Upload Bypass yang memungkinkan file script melewati validasi upload, tersimpan pada lokasi yang dapat diakses melalui web, dan kemudian dieksekusi oleh server. Kondisi tersebut memungkinkan attacker memperoleh kemampuan untuk menjalankan perintah pada sistem server melalui aplikasi web.

Ditemukan 4 temuan keamanan:

| No | Temuan                                                                 | Severity |
| -- | ---------------------------------------------------------------------- | -------- |
| 1  | Cross-Site Scripting (XSS)                                             | Medium   |
| 2  | Local File Inclusion (LFI)                                             | High     |
| 3  | Unrestricted File Upload / Upload Bypass → Remote Code Execution (RCE) | Critical |
| 4  | Password Reuse                                                         | High     |

Temuan RCE memiliki risiko paling tinggi karena kompromi tidak lagi terbatas pada aplikasi, tetapi dapat berkembang menjadi kompromi terhadap server yang menjalankan aplikasi. Dampak aktual bergantung pada privilege akun service yang menjalankan aplikasi dan kontrol keamanan pada server.

Temuan lain seperti LFI dan kelemahan upload juga berpotensi menjadi bagian dari attack chain yang dapat meningkatkan dampak kompromi secara keseluruhan.

Perbaikan terhadap temuan RCE harus menjadi prioritas utama, kemudian dilanjutkan dengan remediation terhadap LFI, Password Reuse, dan XSS serta pengujian ulang untuk memastikan seluruh vulnerability telah tertutup.

 2. Rekomendasi Utama

Prioritas perbaikan yang direkomendasikan:

1. Segera menutup celah Unrestricted File Upload / Upload Bypass yang memungkinkan eksekusi kode.
2. Nonaktifkan eksekusi script pada seluruh direktori yang digunakan untuk menyimpan upload.
3. Simpan file upload di luar web root apabila file tersebut tidak perlu diakses secara langsung melalui web.
4. Terapkan allowlist ekstensi dan validasi MIME type serta magic bytes.
5. Generate nama file secara server-side dan jangan menggunakan filename yang dikontrol pengguna.
6. Lakukan pemeriksaan terhadap file yang telah di-upload untuk memastikan tidak terdapat webshell atau file berbahaya.
7. Periksa server untuk indikasi kompromi akibat kemungkinan eksploitasi RCE.
8. Rotasi credential apabila terdapat indikasi bahwa credential aplikasi/server telah terekspos.
9. Perbaiki LFI dengan menerapkan allowlist dan pembatasan path.
10. Terapkan output encoding untuk mencegah XSS.
11. Gunakan password unik dan MFA untuk akun penting.
12. Lakukan retest setelah remediation.

 3. Detail Temuan

 Temuan 1: Cross-Site Scripting (XSS)

 Deskripsi

Ditemukan indikasi Cross-Site Scripting (XSS) pada parameter input tertentu pada `target.com`.

Input yang diberikan pengguna diproses dan ditampilkan kembali oleh aplikasi tanpa encoding/output escaping yang memadai sehingga memungkinkan browser menginterpretasikan input sebagai kode HTML/JavaScript.

 Endpoint

```text
https://target.com/<endpoint>
```

 Parameter

```text
<parameter>
```

 Langkah Proof of Concept (PoC)

1. Akses endpoint yang terdampak.
2. Masukkan payload XSS sederhana pada parameter yang diuji.
3. Kirim request.
4. Aplikasi mengembalikan input tersebut ke halaman.
5. Browser menginterpretasikan input sebagai JavaScript.

Contoh payload pengujian:

```html
<script>alert(document.domain)</script>
```

Jika JavaScript berhasil dieksekusi dan menampilkan domain `target.com`, maka XSS dapat dikonfirmasi.

 CVSS 4.0

Severity: Medium

```text
CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:P/VC:L/VI:L/VA:N/SC:N/SI:N/SA:N
```

Score: 5.1 Medium

 Dampak

 Eksekusi JavaScript pada browser korban.
 Manipulasi tampilan halaman.
 Phishing melalui halaman aplikasi.
 Pencurian informasi yang tersedia dalam konteks browser.
 Pada Stored XSS, payload dapat dieksekusi terhadap banyak pengguna.

 Rekomendasi Perbaikan

 Terapkan context-aware output encoding.
 Validasi dan sanitasi input.
 Gunakan framework escaping bawaan.
 Hindari penggunaan `innerHTML` terhadap input pengguna.
 Terapkan Content Security Policy (CSP).
 Lakukan pengujian terhadap seluruh parameter yang menerima input pengguna.

---

 Temuan 2: Local File Inclusion (LFI)

 Deskripsi

Ditemukan kelemahan Local File Inclusion (LFI) pada parameter yang digunakan untuk menentukan file yang akan diproses oleh aplikasi.

Aplikasi menerima path dari pengguna tanpa validasi atau pembatasan lokasi file yang memadai.

 Endpoint

```text
https://target.com/<endpoint>?file=<value>
```

 Parameter

```text
file
```

 Langkah Proof of Concept (PoC)

1. Akses endpoint yang menggunakan parameter `file`.
2. Ubah nilai parameter dengan path file lokal yang digunakan untuk pengujian.
3. Amati respons aplikasi.
4. Jika aplikasi mengembalikan isi file lokal di luar direktori yang seharusnya dapat diakses, LFI terkonfirmasi.

Contoh pengujian:

```text
GET /<endpoint>?file=../<test-file>
```

Hasil pengujian menunjukkan aplikasi memproses path yang dikontrol pengguna tanpa pembatasan direktori yang memadai.

 CVSS 4.0

Severity: High

```text
CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:L/VA:N/SC:N/SI:N/SA:N
```

Score: 8.2 High

 Dampak

 Pembacaan file lokal.
 Pengungkapan source code aplikasi.
 Pengungkapan konfigurasi aplikasi.
 Potensi disclosure credential.
 Pengungkapan informasi sensitif pada server.
 Dapat digunakan sebagai bagian dari attack chain menuju compromise lebih lanjut.

 Rekomendasi Perbaikan

 Jangan menerima path file secara langsung dari pengguna.
 Gunakan allowlist identifier file.
 Validasi menggunakan `realpath()` dan pastikan hasil berada dalam direktori yang diizinkan.
 Hindari penggunaan `include()`, `require()`, atau fungsi file lainnya terhadap input pengguna secara langsung.
 Nonaktifkan endpoint debug yang tidak diperlukan pada production.

---

 Temuan 3: Unrestricted File Upload / Upload Bypass → Remote Code Execution (RCE)

 Deskripsi

Ditemukan kelemahan pada mekanisme file upload yang memungkinkan validasi tipe file dilewati.

Validasi upload tidak melakukan pembatasan yang memadai terhadap jenis file yang dapat diterima. Setelah file berhasil di-upload, file tersebut tersimpan pada lokasi yang dapat diakses melalui web dan server memproses file tersebut sebagai executable script.

Kondisi tersebut menyebabkan kelemahan upload berkembang menjadi Remote Code Execution (RCE).

Dengan memanfaatkan vulnerability tersebut, attacker yang tidak memiliki akses administratif dapat mengunggah file yang mengandung kode server-side dan kemudian mengakses file tersebut melalui web sehingga kode dijalankan oleh server.

Dengan demikian, temuan ini bukan hanya merupakan Upload Bypass, tetapi merupakan confirmed Remote Code Execution melalui unrestricted file upload.

 Attack Chain

```text
Attacker
   │
   ▼
Upload Endpoint
   │
   ▼
Upload Validation Bypass
   │
   ▼
Malicious Script Stored
   │
   ▼
Web-Accessible Upload Directory
   │
   ▼
Server Executes Script
   │
   ▼
Remote Code Execution (RCE)
   │
   ▼
Potential Server Compromise
```

 Endpoint

```text
https://target.com/<upload-endpoint>
```

 Method

```text
POST
```

 Langkah Proof of Concept (PoC)

1. Akses fitur upload yang tersedia pada aplikasi.
2. Lakukan pengujian terhadap validasi tipe file.
3. Identifikasi kondisi yang memungkinkan file script melewati validasi upload.
4. Upload file pengujian yang telah disiapkan khusus untuk verifikasi code execution.
5. Aplikasi menerima dan menyimpan file tersebut.
6. Identifikasi URL/lokasi file hasil upload.
7. Akses file tersebut melalui endpoint web.
8. Server memproses file sebagai script.
9. Eksekusi kode berhasil dikonfirmasi melalui output yang dikendalikan oleh penguji.

Untuk menjaga keamanan sistem, PoC tidak perlu menggunakan perintah destruktif atau melakukan perubahan terhadap sistem. Konfirmasi RCE dapat dilakukan menggunakan operasi harmless seperti menghasilkan output statis atau informasi runtime non-sensitif.

 Bukti Konfirmasi

```text
Upload:
<uploaded-test-file>

Upload Result:
File berhasil diterima dan tersimpan pada server.

Accessible URL:
https://target.com/<upload-path>/<test-file>

Execution Result:
Server berhasil mengeksekusi kode server-side dan menghasilkan
output yang dikendalikan oleh penguji.
```

Berdasarkan hasil tersebut, vulnerability dikategorikan sebagai Remote Code Execution (RCE).

 CVSS 4.0

Severity: Critical

Contoh vector:

```text
CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H
```

Score: Sesuaikan dengan kondisi aktual berdasarkan privilege aplikasi, kebutuhan autentikasi, dan tingkat dampak terhadap sistem.

> Karena RCE telah berhasil dikonfirmasi, severity tidak lagi dinilai hanya berdasarkan kemampuan melakukan upload. Penilaian harus mempertimbangkan kemampuan attacker menjalankan kode pada server dan privilege proses aplikasi.

 Dampak

Dampak yang mungkin timbul antara lain:

 Remote Code Execution pada server.
 Pengambilalihan aplikasi.
 Instalasi webshell/backdoor.
 Modifikasi atau penghapusan file aplikasi.
 Pengungkapan credential dan konfigurasi.
 Akses terhadap database yang dapat dijangkau oleh server.
 Penggunaan server sebagai titik pivot menuju sistem internal.
 Manipulasi data aplikasi.
 Gangguan terhadap availability layanan.
 Potensi compromise lebih lanjut apabila proses aplikasi memiliki privilege tinggi.

 Rekomendasi Perbaikan

 1. Nonaktifkan Eksekusi Script pada Upload Directory

Direktori upload harus dikonfigurasi agar file yang diunggah tidak dapat dieksekusi sebagai script.

 2. Gunakan Allowlist

Hanya izinkan ekstensi file yang benar-benar diperlukan oleh aplikasi.

 3. Validasi File

Validasi harus mencakup:

 Extension.
 MIME type.
 Magic bytes/file signature.
 Struktur dan isi file.
 Ukuran file.

Jangan menjadikan filename atau `Content-Type` dari client sebagai satu-satunya mekanisme validasi.

 4. Simpan File di Luar Web Root

Apabila file tidak perlu diakses langsung melalui URL, simpan file di luar web root dan berikan akses melalui mekanisme download yang terkontrol.

 5. Generate Filename Server-Side

Jangan menggunakan nama file yang diberikan pengguna secara langsung.

 6. Terapkan Least Privilege

Pastikan user/service account yang menjalankan web application tidak memiliki permission filesystem yang berlebihan.

 7. Audit Existing Uploads

Lakukan pemeriksaan terhadap direktori upload untuk mencari:

 Script yang tidak seharusnya berada di sana.
 Webshell.
 File executable.
 File dengan timestamp mencurigakan.
 File yang tidak sesuai dengan fungsi aplikasi.

 8. Investigasi Potensi Kompromi

Karena RCE telah berhasil dikonfirmasi, lakukan pemeriksaan log aplikasi dan server untuk mengetahui apakah vulnerability tersebut pernah dieksploitasi sebelum pengujian.

Periksa antara lain:

 Web server access log.
 Error log.
 Application log.
 Authentication log.
 File creation/modification.
 Proses yang mencurigakan.
 Outbound connection.
 Persistence mechanism.

---

 Temuan 4: Password Reuse

 Deskripsi

Ditemukan indikasi password reuse, yaitu penggunaan credential/password yang sama pada lebih dari satu akun atau layanan.

Temuan ini menunjukkan bahwa compromise terhadap satu credential berpotensi digunakan untuk memperoleh akses ke resource atau layanan lain.

 Langkah Proof of Concept (PoC)

1. Identifikasi akun yang digunakan dalam scope pengujian.
2. Berdasarkan credential yang diperoleh secara sah dalam pengujian, lakukan verifikasi terhadap layanan lain yang termasuk scope.
3. Gunakan akun uji atau credential yang telah diotorisasi.
4. Password yang sama berhasil digunakan pada lebih dari satu layanan/akun.

 CVSS 4.0

Severity: High

```text
CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:L/SC:N/SI:N/SA:N
```

Score: 8.9 High

 Dampak

 Credential stuffing atau password spraying menjadi lebih efektif.
 Compromise satu akun dapat menyebabkan akses ke layanan lain.
 Potensi privilege escalation.
 Memperbesar blast radius apabila satu password berhasil diketahui.
 Meningkatkan risiko account takeover.

 Rekomendasi Perbaikan

 Gunakan password unik untuk setiap akun dan layanan.
 Terapkan password policy yang memadai.
 Gunakan password manager.
 Terapkan MFA terutama pada akun privileged.
 Lakukan rotasi credential yang terindikasi reused.
 Gunakan breached-password screening.
 Hindari penggunaan credential default atau credential yang sama antar sistem.
 Audit akun privileged dan service account secara berkala.

 4. Kesimpulan

Pengujian terhadap `target.com` menunjukkan adanya beberapa kelemahan keamanan pada area input validation, file handling, dan credential management.

Temuan paling kritis adalah Unrestricted File Upload / Upload Bypass yang berhasil dieksploitasi hingga Remote Code Execution (RCE).

Attack chain menunjukkan bahwa kelemahan validasi upload memungkinkan attacker memasukkan file script ke server, kemudian file tersebut dapat diakses melalui web dan dieksekusi oleh server. Kondisi tersebut memberikan kemampuan kepada attacker untuk menjalankan kode pada server aplikasi.

Prioritas remediation:

1. Upload Bypass → RCE — Critical

    Tutup mekanisme bypass.
    Nonaktifkan script execution pada upload directory.
    Audit file yang telah tersimpan.
    Investigasi kemungkinan compromise.

2. LFI — High

    Hilangkan penggunaan path yang dikontrol pengguna.
    Terapkan allowlist dan path validation.

3. Password Reuse — High

    Rotasi credential.
    Gunakan password unik.
    Implementasikan MFA.

4. XSS — Medium

    Terapkan output encoding.
    Perbaiki input validation dan sanitization.

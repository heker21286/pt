# LAPORAN HASIL PENGUJIAN KEAMANAN

**Target:** `target.com`
**Metodologi:** Black Box Penetration Testing
**Scope:** Aplikasi Web
**Klasifikasi:** Confidential

## 1. Executive Summary

Pengujian keamanan dilakukan terhadap aplikasi web `target.com` untuk mengidentifikasi kelemahan pada sisi input validation, file handling, autentikasi, dan kontrol akses.

Ditemukan **4 temuan keamanan**:

| No | Temuan                                   | Severity |
| -- | ---------------------------------------- | -------- |
| 1  | Cross-Site Scripting (XSS)               | Medium   |
| 2  | Local File Inclusion (LFI)               | High     |
| 3  | Unrestricted File Upload / Upload Bypass | High     |
| 4  | Password Reuse                           | High     |

Beberapa temuan dapat saling dikombinasikan dan berpotensi meningkatkan dampak terhadap kerahasiaan, integritas, dan ketersediaan sistem.

## 2. Rekomendasi Utama

Prioritas perbaikan yang direkomendasikan:

1. Terapkan validasi dan sanitasi input secara konsisten pada seluruh endpoint.
2. Hindari penggunaan input pengguna secara langsung pada fungsi file handling.
3. Perketat validasi upload berdasarkan ekstensi, MIME type, magic bytes, dan isi file.
4. Nonaktifkan eksekusi script pada direktori upload.
5. Terapkan password unik untuk setiap akun dan sistem.
6. Gunakan MFA untuk akun dengan hak akses penting.
7. Lakukan audit credential dan rotasi password yang berpotensi digunakan ulang.
8. Implementasikan security logging dan monitoring terhadap aktivitas mencurigakan.

# 3. Detail Temuan

## Temuan 1: Cross-Site Scripting (XSS)

### Deskripsi

Ditemukan indikasi **Cross-Site Scripting (XSS)** pada parameter input tertentu pada `target.com`.

Input yang diberikan pengguna diproses dan ditampilkan kembali oleh aplikasi tanpa encoding/output escaping yang memadai sehingga memungkinkan browser menginterpretasikan input sebagai kode HTML/JavaScript.

**Endpoint:**

```text
https://target.com/<endpoint>
```

**Parameter:**

```text
<parameter>
```

### Langkah Proof of Concept (PoC)

1. Akses endpoint yang terdampak.
2. Masukkan payload XSS sederhana pada parameter yang diuji.
3. Kirim request.
4. Aplikasi mengembalikan input tersebut ke halaman.
5. Browser menginterpretasikan input sebagai script.

Contoh payload pengujian:

```html
<script>alert(document.domain)</script>
```

Jika JavaScript dieksekusi dan menampilkan domain `target.com`, maka XSS dapat dikonfirmasi.

### CVSS 4.0

**Severity:** Medium

**Contoh Vector:**

```text
CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:P/VC:L/VI:L/VA:N/SC:N/SI:N/SA:N
```

**Score:** 5.1 Medium

> Nilai akhir perlu disesuaikan dengan konteks aktual, jenis XSS (Reflected/Stored/DOM), authentication requirement, dan dampak terhadap pengguna.

### Dampak

* Eksekusi JavaScript pada browser korban.
* Manipulasi tampilan halaman.
* Phishing melalui halaman aplikasi.
* Pencurian informasi yang tersedia dalam konteks browser.
* Pada Stored XSS, payload dapat dieksekusi terhadap banyak pengguna.

### Rekomendasi Perbaikan

* Terapkan context-aware output encoding.
* Validasi dan sanitasi input.
* Gunakan framework escaping bawaan.
* Hindari penggunaan `innerHTML` terhadap input pengguna.
* Terapkan Content Security Policy (CSP).
* Lakukan pengujian terhadap seluruh parameter yang menerima input pengguna.

## Temuan 2: Local File Inclusion (LFI)

### Deskripsi

Ditemukan kelemahan **Local File Inclusion (LFI)** pada parameter yang digunakan untuk menentukan file yang akan diproses oleh aplikasi.

Aplikasi menerima path dari pengguna tanpa validasi atau pembatasan lokasi file yang memadai.

**Endpoint:**

```text
https://target.com/<endpoint>?file=<value>
```

**Parameter:**

```text
file
```

### Langkah Proof of Concept (PoC)

1. Akses endpoint yang menggunakan parameter `file`.
2. Ubah nilai parameter dengan path file lokal yang tidak sensitif untuk pengujian.
3. Amati respons aplikasi.
4. Jika aplikasi mengembalikan isi file lokal di luar direktori yang seharusnya dapat diakses, LFI terkonfirmasi.

Contoh pengujian terkontrol:

```text
GET /<endpoint>?file=../<test-file>
```

Hasil pengujian menunjukkan aplikasi memproses path yang dikontrol pengguna tanpa pembatasan direktori yang memadai.

### CVSS 4.0

**Severity:** High

**Contoh Vector:**

```text
CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:L/VA:N/SC:N/SI:N/SA:N
```

**Score:** 8.2 High

> Score harus disesuaikan berdasarkan kebutuhan autentikasi dan jenis file yang benar-benar dapat dibaca.

### Dampak

* Pembacaan file lokal.
* Pengungkapan source code aplikasi.
* Pengungkapan konfigurasi.
* Potensi disclosure credential aplikasi.
* Dapat menjadi bagian dari attack chain menuju compromise lebih lanjut.

### Rekomendasi Perbaikan

* Jangan menerima path file secara langsung dari pengguna.
* Gunakan allowlist identifier file.
* Validasi menggunakan `realpath()` dan pastikan hasil berada dalam direktori yang diizinkan.
* Hindari penggunaan `include()`, `require()`, atau fungsi file lainnya terhadap input pengguna secara langsung.
* Nonaktifkan endpoint debug yang tidak diperlukan pada production.

## Temuan 3: Upload Bypass

### Deskripsi

Ditemukan kelemahan pada mekanisme **file upload** yang memungkinkan validasi tipe file dilewati.

Validasi upload hanya mengandalkan satu atau beberapa atribut yang dapat dimanipulasi, seperti ekstensi, MIME type, atau nama file.

**Endpoint:**

```text
https://target.com/<upload-endpoint>
```

**Method:**

```text
POST
```

### Langkah Proof of Concept (PoC)

1. Akses fitur upload yang tersedia.
2. Upload file yang secara fungsional tidak sesuai dengan tipe file yang diizinkan.
3. Ubah metadata upload seperti filename atau MIME type.
4. Amati apakah server menerima file tersebut.
5. Verifikasi apakah file tersimpan pada lokasi yang dapat diakses.

Contoh request konseptual:

```http
POST /<upload-endpoint> HTTP/1.1
Host: target.com
Content-Type: multipart/form-data; boundary=----boundary

------boundary
Content-Disposition: form-data; name="file"; filename="test.<bypass-extension>"
Content-Type: <allowed-mime-type>

<test-content>
------boundary--
```

Pengujian dilakukan menggunakan file harmless dan tidak digunakan untuk menjalankan kode pada sistem target.

### CVSS 4.0

**Severity:** High

**Contoh Vector apabila upload dapat menghasilkan eksekusi kode:**

```text
CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N
```

**Score:** 9.3 Critical

> Jika bypass hanya menyebabkan penyimpanan file tanpa code execution, severity harus diturunkan sesuai dampak aktual.

### Dampak

* Upload file yang tidak seharusnya diperbolehkan.
* Penyimpanan konten berbahaya pada server.
* Potensi phishing atau malware hosting.
* Jika direktori upload mengizinkan eksekusi script, dapat berkembang menjadi **Remote Code Execution (RCE)**.

### Rekomendasi Perbaikan

* Gunakan allowlist ekstensi file.
* Validasi MIME type dan magic bytes.
* Validasi isi file, bukan hanya nama file.
* Generate filename secara server-side.
* Simpan file di luar web root.
* Nonaktifkan eksekusi script pada direktori upload.
* Terapkan permission filesystem minimum.
* Batasi ukuran dan jumlah upload.
* Lakukan malware/content scanning jika relevan.

## Temuan 4: Password Reuse

### Deskripsi

Ditemukan indikasi **password reuse**, yaitu penggunaan credential/password yang sama pada lebih dari satu akun atau layanan.

Temuan ini menunjukkan bahwa compromise terhadap satu credential berpotensi digunakan untuk memperoleh akses ke resource atau layanan lain.

### Langkah Proof of Concept (PoC)

1. Identifikasi akun yang digunakan dalam scope pengujian.
2. Berdasarkan credential yang telah diperoleh secara sah dalam pengujian, lakukan verifikasi terhadap layanan lain yang termasuk scope.
3. Gunakan akun uji atau credential yang telah diotorisasi.
4. Password yang sama berhasil digunakan pada lebih dari satu layanan/akun.

Contoh dokumentasi:

```text
Account A
Service : target.com
Result  : Authentication berhasil

Account A
Service : <service-2>
Result  : Authentication berhasil menggunakan password yang sama
```

**Catatan:** Credential aktual tidak dicantumkan dalam laporan.

### CVSS 4.0

**Severity:** High

**Contoh Vector:**

```text
CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:L/SC:N/SI:N/SA:N
```

**Score:** 8.9 High

> CVSS harus disesuaikan dengan privilege akun, layanan yang dapat diakses, dan dampak aktual dari credential reuse.

### Dampak

* Credential stuffing atau password spraying menjadi lebih efektif.
* Compromise satu akun dapat menyebabkan akses ke layanan lain.
* Potensi privilege escalation.
* Memperbesar blast radius apabila satu password berhasil diketahui.
* Meningkatkan risiko account takeover.

### Rekomendasi Perbaikan

* Gunakan password unik untuk setiap akun dan layanan.
* Terapkan password policy yang memadai.
* Gunakan password manager.
* Terapkan MFA terutama pada akun privileged.
* Lakukan rotasi credential yang terindikasi reused.
* Gunakan breached-password screening.
* Hindari penggunaan credential default atau credential yang sama antar sistem.
* Audit akun privileged dan service account secara berkala.

# 4. Kesimpulan

Pengujian terhadap `target.com` menunjukkan adanya kelemahan pada beberapa area keamanan aplikasi, terutama **input validation, file handling, dan credential management**.

Prioritas remediation:

1. **Upload Bypass** — segera perbaiki validasi upload dan pastikan direktori upload tidak dapat mengeksekusi script.
2. **LFI** — hilangkan penggunaan path yang dikontrol pengguna dan terapkan allowlist.
3. **Password Reuse** — lakukan rotasi credential dan implementasikan MFA.
4. **XSS** — terapkan output encoding dan input validation secara konsisten.

Selain memperbaiki masing-masing temuan, pengujian ulang (**retest**) disarankan setelah remediation untuk memastikan vulnerability telah tertutup dan tidak dapat dieksploitasi kembali.

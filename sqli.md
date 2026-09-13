 Deskripsi

Ditemukan kerentanan SQL Injection (SQLi) pada aplikasi yang memungkinkan input pengguna diproses secara tidak aman dalam query SQL pada sisi server.

Kerentanan terjadi karena aplikasi tidak melakukan validasi dan sanitasi input secara memadai sebelum nilai input digunakan dalam proses query ke database. Kondisi tersebut memungkinkan input yang dimanipulasi memengaruhi struktur atau logika query SQL yang dijalankan oleh aplikasi.

Berdasarkan pengujian, parameter yang terindikasi rentan dapat menghasilkan respons aplikasi yang berbeda ketika diberikan input tertentu, yang menunjukkan bahwa input pengguna berpotensi memengaruhi query database.

 Langkah Proof of Concept (PoC)

1. Mengidentifikasi parameter aplikasi yang menerima input dari pengguna.
2. Mengirimkan input normal sebagai baseline dan mencatat respons aplikasi.
3. Melakukan pengujian terhadap parameter dengan karakter/sintaks SQL yang relevan.
4. Membandingkan respons aplikasi antara input normal dan input yang telah dimanipulasi.
5. Ditemukan adanya perubahan respons yang konsisten, sehingga mengindikasikan bahwa input pengguna diproses sebagai bagian dari query SQL.
6. Pengujian lebih lanjut dapat dilakukan secara terkontrol untuk memastikan jenis DBMS dan tingkat akses yang dapat dipengaruhi oleh kerentanan tersebut.

Contoh PoC generik:

```text
Request normal:
GET /[endpoint]?[parameter]=[nilai_normal]

Request pengujian:
GET /[endpoint]?[parameter]=[input_uji]

Hasil:
Respons aplikasi mengalami perubahan dibandingkan dengan baseline,
yang mengindikasikan bahwa parameter tersebut berpotensi dipengaruhi
oleh manipulasi sintaks SQL.
```

> Bukti aktual berupa request/response, parameter, dan payload pengujian dapat dilampirkan pada bagian lampiran laporan.

 CVSS 4.0

Severity: High

CVSS 4.0: Dapat disesuaikan berdasarkan hasil eksploitasi aktual.

Sebagai gambaran, SQL Injection yang dapat dieksploitasi secara remote tanpa autentikasi dan memungkinkan akses atau manipulasi data database dapat memiliki tingkat risiko High hingga Critical, tergantung pada hak akses database, kebutuhan autentikasi, serta dampak terhadap kerahasiaan, integritas, dan ketersediaan data.

 Dampak

Apabila berhasil dieksploitasi, kerentanan SQL Injection berpotensi menyebabkan:

 Akses tidak sah terhadap data pada database.
 Pengungkapan informasi sensitif.
 Bypass terhadap mekanisme aplikasi tertentu.
 Manipulasi atau perubahan data.
 Penghapusan data apabila hak akses database memungkinkan.
 Pengungkapan struktur database dan informasi internal aplikasi.
 Akses terhadap akun atau kredensial yang tersimpan secara tidak aman.
 Dalam kondisi tertentu, SQL Injection dapat menjadi tahap awal untuk eskalasi serangan lebih lanjut terhadap server/aplikasi.

Dampak aktual bergantung pada hak akses akun database yang digunakan oleh aplikasi dan konfigurasi lingkungan server.

 Rekomendasi Perbaikan

1. Gunakan parameterized query / prepared statement untuk seluruh query yang menggunakan input pengguna.
2. Hindari membangun query SQL menggunakan konkatenasi string secara langsung.
3. Terapkan server-side input validation berdasarkan tipe dan format data yang diharapkan.
4. Gunakan ORM atau database abstraction layer secara aman apabila tersedia.
5. Terapkan prinsip least privilege pada akun database yang digunakan aplikasi.
6. Pastikan akun database aplikasi tidak memiliki hak akses administratif yang tidak diperlukan.
7. Jangan menampilkan error database secara langsung kepada pengguna. Gunakan pesan error generik dan simpan detail error pada log internal.
8. Lakukan pengujian keamanan terhadap seluruh parameter aplikasi yang berinteraksi dengan database.
9. Pertimbangkan penggunaan mekanisme tambahan seperti WAF sebagai lapisan pertahanan tambahan, bukan sebagai pengganti perbaikan pada kode aplikasi.
10. Lakukan regression testing setelah perbaikan untuk memastikan kerentanan tidak dapat dieksploitasi kembali.

IDOR (Insecure Direct Object Reference)

Deskripsi

Aplikasi memiliki kerentanan Insecure Direct Object Reference (IDOR) pada fitur pengelolaan data. Parameter ID objek dapat dimodifikasi secara langsung oleh pengguna untuk mengakses, mengubah, atau menghapus data yang bukan miliknya.

Server hanya memvalidasi bahwa pengguna telah login, tetapi tidak melakukan pemeriksaan otorisasi untuk memastikan bahwa pengguna tersebut berhak melakukan tindakan terhadap objek yang diminta.

Step to Reproduce

1. Login menggunakan akun Pengguna A.
2. Akses salah satu data yang dimiliki Pengguna A.
3. Tangkap request menggunakan Burp Suite, misalnya:

DELETE /api/berita/15 HTTP/1.1
Host: target.example
Authorization: Bearer [TOKEN_PENGGUNA_A]

4. Ubah nilai ID "15" menjadi ID objek milik pengguna atau unit lain, misalnya "16":

DELETE /api/berita/16 HTTP/1.1
Host: target.example
Authorization: Bearer [TOKEN_PENGGUNA_A]

5. Kirim ulang request tersebut.
6. Server memberikan respons berhasil dan data dengan ID "16" dapat dihapus, meskipun Pengguna A tidak memiliki hak terhadap data tersebut.

Dampak

Kerentanan ini memungkinkan pengguna dengan hak akses terbatas untuk:

- Melihat data milik pengguna atau unit lain.
- Mengubah informasi tanpa izin.
- Menghapus data yang bukan menjadi kewenangannya.
- Menyalahgunakan hak akses antarperan atau antarunit.
- Mengganggu integritas dan ketersediaan data aplikasi.

Jika objek mengandung informasi sensitif, kerentanan ini juga dapat menyebabkan kebocoran data dan pelanggaran kerahasiaan.

Mitigasi

- Terapkan pemeriksaan otorisasi pada setiap request di sisi server.
- Pastikan pengguna memiliki hak terhadap objek sebelum operasi baca, ubah, atau hapus dilakukan.
- Jangan hanya mengandalkan validasi pada antarmuka pengguna atau penyembunyian tombol.
- Terapkan kontrol akses berdasarkan peran dan kepemilikan objek, misalnya:

Izinkan tindakan hanya jika:
objek.owner_id == pengguna.id
atau
pengguna memiliki role/permission yang sesuai

- Gunakan middleware atau fungsi otorisasi terpusat agar pemeriksaan diterapkan secara konsisten.
- Kembalikan respons "403 Forbidden" ketika pengguna tidak memiliki hak terhadap objek.
- Gunakan identifier yang sulit ditebak seperti UUID sebagai pertahanan tambahan, bukan sebagai pengganti pemeriksaan otorisasi.
- Lakukan pengujian untuk memastikan akses lintas akun, peran, dan unit organisasi telah ditolak.
- Catat aktivitas sensitif seperti perubahan dan penghapusan data dalam audit log.

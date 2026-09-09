# JaoHongKhai
Ini repositori untuk proyek Jao Hong Khai
yang sedang mengembangkan Aplikasi "Administrasi HISADA"

# Ringkasan Fungsi Sistem Hisada
### Himpunan Santri Daarul Uluum Lido

---

## 1. Autentikasi & RBAC (Role-Based Access Control)

- **Hanya santri yang sedang menjabat** (punya `riwayat_jabatan` aktif dengan `punya_akses_sistem = true`) yang bisa login — santri biasa tanpa jabatan **tidak mendapat akun sistem sama sekali**.
- Login menggunakan **email format `(NIS)@daarululumlido.com` + password** (bukan NIS polos) — email ini otomatis dibuat sistem saat admin mengaktifkan akses sistem untuk santri bersangkutan (lihat §12).
- Role ditentukan **per periode jabatan aktif** (lihat bagian rekomendasi §11), bukan hardcode statis — sehingga hak akses otomatis berubah saat periode kepengurusan berganti.
- Password wajib di-hash (Bcrypt), disimpan di kolom `password` tabel `users`.

## 2. Dashboard

- Statistik utama, ditampilkan dalam satu baris ringkasan: **jumlah santri laki-laki, perempuan, dan total keseluruhan**.
- Statistik tambahan: jumlah santri sakit (1 bulan terakhir), jumlah yang berstatus pulang saat ini, dan jumlah alpha.
- Ringkasan kalender: daftar kegiatan **upcoming** (agenda terdekat yang belum lewat tanggalnya).
- Shortcut Google Drive dinamis (diambil dari database, bukan hardcode).
- Kalender akademik dengan dua mode tampilan: **publik** (wali santri) dan **internal** (pengurus).

## 3. Modul Absensi

- Halaman awal berupa **pilihan card**: Absensi Harian (Kamar), Halaqah Qur'an, Muhadhoroh, Olahraga, Kesenian.
- **Absensi Harian (Kamar)**: cascading dropdown Gender → Gedung → Kamar (untuk 75+ kamar), bulk-insert per kamar.
- **Absensi selain kamar** (Halaqah Qur'an, Muhadhoroh, Olahraga, Kesenian): menggunakan **grup tersendiri per kegiatan** (bukan cascading kamar) — tiap kegiatan punya daftar anggota grupnya sendiri, terutama untuk Olahraga & Kesenian yang mengikuti keanggotaan ekskul (§13).
- Validasi unique index (`santri_id` + `tanggal` + `jenis_kegiatan`) untuk mencegah duplikasi.
- Status absensi harian sebagian besar **terisi otomatis** lewat integrasi dari modul lain:
  - Poskestren → status "Sakit" (lihat logika terbaru di §4)
  - Perizinan → status "Izin"/"Pulang"/"Alpha" (jika overdue)

## 4. Modul Poskestren (Klinik)

- Asisten mencatat kunjungan santri dengan dua status berbeda:
  - **Rawat Jalan**: hanya konsultasi, **tidak** mengubah status absensi (santri tetap dianggap hadir).
  - **Perawatan**: benar-benar sakit → status absensi hari itu otomatis berubah jadi "Sakit".
- **Tidak ada lagi mekanisme "pasien aktif"/check-in-check-out**: status "Sakit" **hanya berlaku untuk tanggal itu saja**. Besoknya otomatis kembali "Hadir", **kecuali** asisten Poskestren menginput ulang status Perawatan untuk tanggal berikutnya.
- Notifikasi estafet ke dokter → dokter mengisi rekam medis (diagnosa, resep, tindak lanjut).
- **Analisis musim sakit**: dokter dapat melihat analisis tren kunjungan (misal lonjakan kasus tertentu di bulan/musim tertentu) untuk mendeteksi kemungkinan pola penyakit musiman, berdasarkan agregasi data kunjungan historis per kategori keluhan/diagnosa dari waktu ke waktu.

## 5. Modul Mahkamah Santri (Kedisiplinan)

- Brute input pelanggaran oleh sekretaris: Kategori → Keterangan → NIS.
- Antrean sidang terfilter per kategori untuk hakim (asatidz).
- Vonis langsung (Ringan/Sedang/Berat) **tanpa akumulasi poin** (keputusan desain).
- Mekanisme "pemutihan": soft-delete + kolom alasan pembatalan, menjaga audit trail.

## 6. Modul Perizinan & Kamtib

- Membedakan **Izin Keluar Sementara** vs **Izin Pulang**.
- Status absensi otomatis mengikuti rentang tanggal izin.
- Deteksi otomatis **overdue** → status "Alpha" + masuk antrean Mahkamah.
- Halaman konfirmasi kembali untuk menutup status izin.

## 7. Modul Korespondensi

- Surat keluar: nomor otomatis (auto-generated pattern).
- Surat masuk: nomor manual + status disposisi (Belum Dibaca/Diteruskan/Disetujui/Diarsipkan).
- Validasi regex untuk lampiran (hanya menerima URL Google Docs resmi).

## 8. Modul Prestasi Santri

- Form input: NIS, nama kegiatan, lokasi lomba, internal/eksternal, keterangan.
- Galeri prestasi sebagai tampilan rekap.

## 9. Modul Kalender Akademik

- Saat user klik grid tanggal yang **masih kosong**, muncul **popup pilihan** untuk mengisi agenda pada tanggal tersebut.
- Kategori agenda: **Umum, Akademik, Pengasuhan** — dapat memilih **2 kategori sekaligus** untuk satu agenda (misal Umum+Akademik, atau Umum+Pengasuhan), untuk kebutuhan tampilan ke wali santri.
- Setiap kategori punya **warna berbeda** di tampilan kalender agar mudah dibedakan sekilas.
- **Hak akses input**: hanya **sekretaris dan admin** yang bisa menambah/mengubah agenda. Role lain hanya bisa membaca (read-only).

## 10. Modul Audit & Kelola User

- Log sistem dan tombol reset data demo (khusus Super Admin, untuk keperluan testing/beta test).
- Manajemen user & role untuk admin — **admin dapat mengedit seluruh data user, termasuk mereset/mengubah password user lain langsung dari panel Kelola User** (bukan hanya lihat, tapi full edit akses).

## 11. Masa Jabatan / Khidmat (bukan tahun ajaran)

- Tabel `periode_jabatan`: menyimpan periode dengan `status` (`aktif`/`arsip`) — tidak berbasis tahun ajaran akademik.
- Tabel `riwayat_jabatan`: penghubung santri ↔ periode ↔ posisi, dengan kolom `punya_akses_sistem`.
- **Data periode lama tidak dihapus**, hanya diubah status jadi `arsip` — semua tampilan publik query berdasarkan periode `aktif` saja, sehingga histori tetap tersimpan tanpa terekspos ke tampilan umum.

## 12. Penetapan Pengurus Baru (Kelas 5–6, sebelum masa ujian)

- Admin memilih kandidat dari kelas 5 & 6 melalui halaman "Penetapan Pengurus Baru".
- Jabatan bisa disiapkan lebih awal dengan `tanggal_mulai` di masa depan (belum aktif).
- Kolom `punya_akses_sistem` menentukan siapa yang benar-benar dapat akun login — **jadi pengurus tidak otomatis berarti punya akses sistem**.

## 13. Ekstrakurikuler (Olahraga & Kesenian)

- `kategori_ekskul`: Olahraga, Kesenian.
- `ekstrakurikuler`: banyak cabang per kategori, masing-masing punya pelatih (`pelatih_id`) — **pelatih bukan entitas terpisah, cukup FK ke tabel guru/asatidz yang sudah ada** (satu guru bisa merangkap jadi pelatih ekskul tertentu, tanpa perlu tabel `pelatih` sendiri).
- `ekskul_anggota`: keanggotaan santri per cabang, dengan histori keluar-masuk (bukan overwrite).
- `ekskul_agenda`: jadwal kegiatan per cabang.
- Pelatih hanya bisa kelola agenda & anggota untuk cabang yang dia ampu sendiri (scoped access).

## 14. Basis Kamar (bukan Kelas)

- `kamar_id` menjadi filter utama di seluruh modul Hisada (absensi, mahkamah, perizinan, ekskul).
- `kelas_id` tetap ada sebagai referensi akademik, tapi bukan filter utama.
- Wali kamar menjadi role yang di-scope ke kamarnya sendiri, bukan wali kelas.

## 15. Pencarian NIS dengan Filter Kamar

- Satu endpoint pencarian terpadu untuk santri & asatidz.
- Parameter: `q` (NIS/nama) + `kamar_id` (opsional).
- Untuk asatidz, filter kamar berarti "asatidz yang menjabat wali kamar tersebut".

---

## Rekomendasi Tambahan

| # | Rekomendasi | Alasan |
|---|---|---|
| a | Buat halaman khusus **"Tutup Periode & Buka Periode Baru"** yang otomatis mengarsipkan `riwayat_jabatan` lama dan menonaktifkan akun terkait | Mencegah proses ganti periode dilakukan manual satu-satu yang rawan human error |
| b | Beri **masa tenggang (grace period)** beberapa hari setelah periode berakhir sebelum akun benar-benar dinonaktifkan | Menghindari data menggantung jika ada kasus Mahkamah/perizinan yang masih berjalan saat transisi jabatan |
| c | Jadikan pola **status + tanggal_mulai + tanggal_selesai** sebagai standar resmi untuk semua relasi keanggotaan/jabatan (bukan hanya jabatan & ekskul, tapi pola konsisten di seluruh sistem) | Konsistensi arsitektur, memudahkan maintenance jangka panjang |
| d | Ganti pengecekan role saat login dari `role_id` statis menjadi **join ke `riwayat_jabatan` periode aktif** | Hak akses otomatis mengikuti jabatan terkini tanpa admin perlu edit manual setiap pergantian periode |

---

## 16. Portal Wali Santri

- Login terpisah dari akun staff: NIS anak + kode akses unik per keluarga (bukan password sistem internal), atau nomor WA terdaftar + OTP.
- Tabel `wali_akses`: menghubungkan `family_id` ke akun wali (satu keluarga bisa punya lebih dari satu anak, satu akun bisa lihat semua anaknya).
- Scope akses dibatasi query: `WHERE student_id IN (SELECT student_id FROM students WHERE family_id = :family_id_login)` — wali hanya bisa melihat data anak sendiri, tidak bisa melihat santri lain.
- Konten yang ditampilkan (read-only):
  - Riwayat absensi & status kehadiran terkini
  - Riwayat kesehatan (ringkasan Poskestren: tanggal, keluhan, status — tanpa detail diagnosa medis sensitif)
  - Riwayat prestasi
  - Status perizinan/pulang (termasuk info overdue jika ada)
- **Kebijakan privasi perlu diputuskan pihak pesantren**: apakah detail Mahkamah (pelanggaran) ditampilkan penuh ke wali, atau hanya ringkasan status ("ada catatan kedisiplinan, hubungi wali kamar") — direkomendasikan opsi kedua untuk menjaga privasi santri.

## 17. Export Laporan (PDF/Excel)

- **Excel**: gunakan library `PhpSpreadsheet` (server-side) untuk generate file `.xlsx` dari hasil query rekap (absensi bulanan, pelanggaran, prestasi), dengan filter kamar/kelas/rentang tanggal yang sama seperti tampilan di layar.
- **PDF**: gunakan `dompdf` atau `mPDF` untuk laporan cetak resmi (kop surat pesantren, tabel data, kolom tanda tangan pembina).
- Tombol "Export" ditambahkan di setiap halaman rekap yang sudah ada (Absensi, Mahkamah, Prestasi, Korespondensi), memanggil endpoint backend `export.php?modul=...&format=xlsx|pdf&filter=...` — bukan generate di sisi browser, supaya data besar tidak membebani perangkat pengguna.

## 18. Notifikasi Real-time (Web Push)

- Lengkapi tabel `push_subscriptions` yang sudah dirancang sebelumnya dengan alur kerja penuh:
  1. Saat dokter/piket mengizinkan notifikasi di browser, Service Worker generate subscription token → disimpan ke `push_subscriptions` (terhubung ke `user_id`).
  2. Saat event terjadi (santri baru masuk Poskestren, perizinan overdue), backend memanggil library `web-push` (PHP) dengan VAPID key untuk mengirim notifikasi ke semua subscription milik role terkait.
- **Fallback wajib**: jika user belum mengizinkan push notification (umum terjadi, browser sering diblokir), tetap sediakan badge notifikasi di dashboard yang di-refresh via polling AJAX ringan (misal setiap 30 detik) — supaya sistem tidak bergantung 100% pada izin push browser.

## 19. Backup & Disaster Recovery

- Backup otomatis harian via cron job: `mysqldump` seluruh database, disimpan terenkripsi ke storage terpisah dari server utama (Google Drive API atau storage S3-compatible) — jangan simpan backup di server yang sama dengan database produksi.
- Kebijakan retensi: simpan 7 backup harian terakhir + 4 backup mingguan + backup bulanan (hapus otomatis yang lebih lama untuk hemat storage).
- Enkripsi backup wajib mengingat isinya data sensitif (kesehatan, pelanggaran, data pribadi keluarga).
- Uji pemulihan (restore) dilakukan berkala (misal tiap awal periode kepengurusan baru) untuk memastikan file backup benar-benar bisa dipakai saat dibutuhkan, bukan hanya tersimpan.
- **Backup tambahan ke Google Spreadsheet (Google Apps Script)**: sebagai lapisan cadangan kedua yang gratis dan mudah diaudit manual oleh non-teknis:
  1. Buat Google Apps Script Web App (`doPost`) yang menerima data JSON dan menuliskannya sebagai baris baru ke Google Spreadsheet (satu sheet per tabel inti: students, attendances, violations, dsb.).
  2. Backend PHP memanggil URL Web App tersebut secara terjadwal (cron job harian) mengirim snapshot data terbaru per tabel.
  3. Spreadsheet ini berfungsi sebagai **cadangan yang mudah dibaca manual** oleh pengurus/pembina tanpa perlu akses phpMyAdmin — cocok sebagai lapisan kedua di luar backup `mysqldump` di atas, bukan pengganti.

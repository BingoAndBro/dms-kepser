# Bahan Penulisan: Iterasi Pasca-Demonstrasi (DSRM + Modified Waterfall)

> **Untuk Claude Desktop.** Dokumen ini adalah satu-satunya acuan untuk menulis subbab
> "iterasi pengembangan" di Bab IV. Tulis dengan bahasa skripsi yang sederhana dan mengalir.
> Cakupannya **hanya tiga masukan pengguna** di bawah (M1–M3). Jangan menambah perubahan lain dari
> riwayat Git (migrasi data, perbaikan keamanan, temuan D-xx, perubahan login, tema, dan sebagainya),
> karena tidak termasuk iterasi ini.

---

## 1. Konteks metode

- **Metode penelitian:** Design Science Research Methodology (DSRM):
  *problem identification → objectives → design and development → demonstration → evaluation → communication*.
- **Metode pengembangan** pada tahap *design and development*: **Modified Waterfall**:
  *requirement definitions → design → implementation → testing*, dengan umpan balik ke tahap sebelumnya.
- **Posisi iterasi ini:** setelah tahap **demonstration**, masukan pengguna dibawa kembali ke tahap
  **design and development**. Di sana masukan tersebut melewati satu putaran Modified Waterfall
  (kebutuhan → rancangan → implementasi → pengujian), lalu aplikasi hasil iterasi dibawa ke tahap **evaluation**.
- **Aplikasi:** Sistem Informasi Pengelolaan Dokumentasi Kinerja dan Pertanggungjawaban Kegiatan,
  BPS Kabupaten Kepulauan Seribu (TanStack Start + PostgreSQL + Drizzle ORM).
- **Tanggal demonstrasi:** [ISI TANGGAL DEMONSTRASI]. Lihat catatan konsistensi tanggal di bagian 7.

---

## 2. Ringkasan masukan demonstrasi

Pada demonstrasi, pengguna menyampaikan tiga masukan yang semuanya terlihat langsung di antarmuka:

| No | Masukan pengguna | Peran pengusul (sesuaikan) | Bentuk tindak lanjut |
|---|---|---|---|
| M1 | Ketua Tim ingin memantau posisi dokumen anggota timnya (masih di PPK, di PPSPM, atau dikembalikan untuk revisi) tanpa harus menanyakan satu per satu. | Pegawai yang menjadi Ketua Tim | Menu baru **Monitoring Dokumen Tim** |
| M2 | Dokumen yang ditambahkan manual oleh Kepala Sub Bagian Umum (KSBU) juga perlu dihitung dalam rekap nominal realisasi, supaya totalnya sesuai dengan kondisi sebenarnya. | PPK / PPSPM / PJ Kinerja | Dokumen tambahan KSBU masuk perhitungan **Monitoring Nominal Realisasi**, **Laporan Kinerja**, dan **Laporan Kegiatan** |
| M3 | Filter periode di halaman Monitoring Realisasi dinilai praktis dan diminta tersedia juga di halaman laporan lain. | Pegawai / Ketua Tim | Filter periode yang sama dipakai di **Laporan Saya** dan **Laporan Kegiatan**, ditambah mode **Bulanan** |

> Rumusan masukan dan kolom "peran pengusul" disesuaikan penulis dengan kejadian saat demonstrasi.

Catatan: fitur pemilihan tema aplikasi sudah ada saat demonstrasi sebagai inisiatif pengembang, dan
mendapat tanggapan positif dari pengguna. Fitur itu **bukan** bagian iterasi ini.

---

## 3. Rincian per tahap Modified Waterfall

### 3.1 Requirement definitions (pendefinisian kebutuhan)

**Kebutuhan fungsional baru**

| Kode (sesuaikan) | Kebutuhan | Asal |
|---|---|---|
| KF-baru-1 | Sistem menyediakan halaman Monitoring Dokumen Tim bagi Ketua Tim. Halaman ini menampilkan dokumen dari kegiatan yang dipimpinnya beserta **posisi** dokumen: Validasi PPK, Persetujuan PPSPM, Dikembalikan ke Pengaju, Ditolak PPSPM (kembali ke PPK), atau Selesai. | M1 |
| KF-baru-2 | Sistem menyediakan filter kegiatan, posisi, pembuat, lama tertahan (> 3/7/14 hari), dan rentang tanggal. Sistem memberi peringatan untuk dokumen yang tertahan lebih dari 7 hari. | M1 |

**Kebutuhan fungsional yang diubah**

| Kebutuhan lama | Kebutuhan setelah iterasi | Asal |
|---|---|---|
| Rekap nominal realisasi hanya menghitung dokumen material berstatus Selesai dari alur pengajuan. | Rekap nominal realisasi juga menghitung **dokumen tambahan KSBU**. Dokumen tambahan ditandai "Penambahan Dokumen (KSBU)" supaya tidak dikira melewati persetujuan PPK/PPSPM. | M2 |
| Filter periode hanya ada di halaman Monitoring Realisasi dan Laporan Kinerja. | Filter periode menjadi satu komponen bersama yang dipakai juga di Laporan Saya dan Laporan Kegiatan, dengan mode **Bulanan**, Triwulan, Tahunan, Seluruh Periode, dan Kustom. | M3 |

**Aturan pendukung** (cukup disebut singkat): Ketua Tim hanya melihat dokumen dari kegiatan yang ia
pimpin, dan hanya dokumen yang **sudah diajukan**. Draf milik pegawai tetap pribadi.

### 3.2 Design (perancangan)

| Artefak | Perubahan |
|---|---|
| **Use case diagram** | Tambah **UC-25 Memantau Dokumen Tim** (aktor: Pegawai sebagai Ketua Tim). UC laporan (Laporan Saya, Laporan Kegiatan, Monitoring Nominal Realisasi, Laporan Kinerja) diperbarui deskripsinya: filter periode + dokumen tambahan KSBU. |
| **Activity / sequence diagram** | UC-25: Ketua Tim membuka menu → sistem mengambil dokumen dari kegiatan yang dipimpin → sistem menentukan posisi tiap dokumen → menampilkan ringkasan per posisi + daftar dokumen. UC laporan: pengguna memilih periode → sistem menyaring dokumen berdasarkan tanggal dokumen → total dihitung dari dokumen alur dan dokumen tambahan KSBU. |
| **ERD / tabel** | **Tidak ada tabel baru** pada iterasi ini. Ketiga fitur memakai `dokumen_transaksi`, `ketua_tim_assignments`, dan `manual_arsip` yang sudah ada. |
| **State diagram status dokumen** | **Tidak berubah.** "Posisi" pada Monitoring Dokumen Tim hanya pembacaan dari status yang sudah ada: `IN_PPK_VALIDATION` → di PPK; `IN_PPSPM_APPROVAL` → di PPSPM; `NEED_REVISION` + target USER → dikembalikan ke pengaju; `NEED_REVISION` + target PPK → ditolak PPSPM, kembali di PPK; `COMPLETED`/`TERSIMPAN` → selesai. |
| **Rancangan antarmuka** | Halaman Monitoring Dokumen Tim (kartu ringkasan per posisi, filter, tabel, penanda "tertahan"); pemilih periode di Laporan Saya dan Laporan Kegiatan; penanda "Penambahan Dokumen (KSBU)" pada daftar dokumen. |

### 3.3 Implementation (implementasi)

| Masukan | Berkas utama | Commit |
|---|---|---|
| M1 Monitoring Dokumen Tim | `src/routes/pegawai/monitoring-dokumen-tim.tsx` (halaman baru), `src/lib/laporan/kegiatan-scope.ts` (daftar status yang dipantau + fungsi `getPosisiDokumen`), `src/routes/api/laporan/kegiatan.ts` (parameter `scope=monitoring`), `src/config/navigation.ts` (menu baru di grup PJ Kegiatan) | `16f6e1a` |
| M2 Dokumen manual dihitung | `src/routes/api/laporan/kinerja.ts`, `src/lib/laporan/manual-realisasi.ts` (pengambilan dokumen tambahan KSBU yang dipakai bersama), `src/routes/api/laporan/kegiatan.ts`, `src/components/kinerja/MonitoringRealisasiView.tsx` | `37517ff`, `e6ba99c`, `93eeb17` |
| M3 Filter periode dipakai bersama | `src/components/laporan/PeriodeSelector.tsx` (komponen bersama), `src/lib/laporan/periode.ts` (perhitungan rentang tanggal, termasuk mode Bulanan), `src/routes/pegawai/laporan/saya.tsx`, `src/routes/pegawai/laporan/kegiatan.tsx` | `e6ba99c` |

### 3.4 Testing (pengujian)

| Masukan | Pengujian unit (otomatis, Vitest) | Pengujian black-box (disarankan ditulis) |
|---|---|---|
| M1 | `tests/unit/dokumen/dokumen-get-ketua-tim-access.test.ts` (Ketua Tim hanya bisa membuka dokumen kegiatannya dan tidak bisa membuka draf) | Login sebagai Ketua Tim → buka Monitoring Dokumen Tim → posisi dokumen sesuai statusnya; filter "tertahan > 7 hari" berjalan. |
| M2 | `tests/unit/laporan/kinerja-route.test.ts`, `tests/unit/laporan/kegiatan-route.test.ts` (total Laporan Kegiatan sama dengan Nominal Realisasi), `tests/unit/laporan/monitoring-rows.test.ts` | KSBU menambah dokumen manual → dokumen muncul di Monitoring Nominal Realisasi dengan label "Penambahan Dokumen (KSBU)" dan totalnya bertambah. |
| M3 | `tests/unit/laporan/periode.test.ts` (rentang bulanan, triwulan, tahunan, termasuk Februari tahun kabisat) | Pilih periode Bulanan di Laporan Saya → hanya dokumen bulan tersebut yang tampil. |

Hasil: seluruh skenario berhasil, lalu aplikasi hasil iterasi dibawa ke tahap **evaluation** DSRM.

---

## 4. Alur iterasi (untuk gambar/diagram)

```
[Demonstration]
   │  3 masukan pengguna (M1–M3)
   ▼
[Design and Development — Modified Waterfall, putaran ke-2]
   Requirement definitions : +2 KF baru (Monitoring Dokumen Tim), 2 KF diubah (dokumen manual, filter periode)
        ▼
   Design                  : +UC-25, deskripsi UC laporan diperbarui, ERD dan state dokumen tetap
        ▼
   Implementation          : Monitoring Dokumen Tim, perhitungan dokumen manual, filter periode bersama
        ▼
   Testing                 : unit test + black-box
   │
   ▼
[Evaluation]
```

---

## 5. Kerangka paragraf yang disarankan

1. **Pembuka:** demonstrasi dilaksanakan pada [tanggal] kepada [peserta]; pengguna mencoba aplikasi
   sesuai perannya masing-masing. Fitur yang sudah ada, termasuk pilihan tema, mendapat tanggapan positif.
2. **Masukan:** sebutkan M1–M3 dalam satu paragraf atau tabel (tabel di bagian 2).
3. **Kembali ke design and development:** jelaskan bahwa masukan diolah melalui satu putaran Modified
   Waterfall. Uraikan singkat tiap tahap (bagian 3.1 s.d. 3.4), cukup satu paragraf per tahap.
4. **Hasil:** sebutkan bahwa aplikasi hasil iterasi menjadi bahan tahap evaluasi, misalnya kuesioner ISO/IEC 25010.
5. **Gambar yang disarankan:** diagram alur iterasi (bagian 4), tangkapan layar Monitoring Dokumen Tim,
   pemilih periode di Laporan Saya, dan penanda "Penambahan Dokumen (KSBU)" di Monitoring Nominal Realisasi.

---

## 6. Batasan penulisan

- Fokus pada hal yang **terlihat oleh pengguna**. Jangan membahas migrasi lain, perbaikan keamanan, tema, atau temuan validasi kode.
- Jangan menyebut hash commit di badan teks skripsi. Hash di dokumen ini hanya untuk penulis, bila perlu menunjukkan bukti.
- Penomoran UC dan KF disesuaikan dengan Bab IV yang sudah ada. Monitoring Dokumen Tim sudah ditetapkan sebagai **UC-25**.

---

## 7. Catatan konsistensi tanggal (untuk penulis)

Menurut riwayat Git, fitur-fitur iterasi ini dikerjakan pada tanggal berikut:

| Fitur | Tanggal pengerjaan |
|---|---|
| Filter periode di Monitoring Realisasi (yang menjadi acuan M3) | sudah ada sejak 08–09 September 2026 |
| Tema aplikasi (bukan bagian iterasi, tetapi sudah tampil saat demonstrasi) | 13–19 September 2026 |
| Monitoring Dokumen Tim (M1) | 27 September 2026 |
| Dokumen manual dihitung (M2) | 27–29 September 2026 |
| Filter periode di halaman lain (M3) | 28 September 2026 |

Agar narasi konsisten dengan riwayat Git, tanggal demonstrasi sebaiknya **antara 19 dan 26 September 2026**.
Dengan begitu tema sudah selesai dan bisa dilihat pengguna saat demonstrasi, filter periode di Monitoring
Realisasi sudah ada sebagai acuan M3, dan semua pengerjaan M1–M3 terjadi sesudahnya.

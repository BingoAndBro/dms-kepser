# Penjelasan Proyek: Repositori Dokumen Kegiatan — Penyimpanan, Persetujuan, dan Pemberkasan

> **Versi**: disinkronkan dengan kode branch `migration/postgres-local`, commit HEAD `b6c3292` ("Add resubmit validation: kelengkapan lampiran >=1"), 27 September 2026. Versi sebelumnya: `penjelasan-proyek-aplikasi-lama.md` (commit `7927ca5`).
>
> Dokumen ini menjelaskan proyek dari sisi **bisnis lebih dulu**, baru turun ke solusi teknis. Alurnya: *Permasalahan Bisnis Utama* → *masalah spesifik* → *solusi, use case, pengguna, keterkaitan*.
>
> **Pembanding permasalahan = proses kerja manual berbasis kertas yang berjalan di kantor sekarang**, bukan versi kode sebelumnya. Perbandingan dengan versi kode sebelumnya hanya ada di **Bagian A (Changelog)**.
>
> **Otoritas**: satu-satunya otoritas adalah **kode di `src/` + migrasi `drizzle/`**. `AGENTS.md`, `docs/`, dan `docs/planning/` tidak dipakai sebagai sumber fakta. Setiap klaim menunjuk ke `berkas:baris` atau nama fungsi/konstanta. Klaim yang tidak bisa ditunjuk ke kode ditandai **[BELUM TERVERIFIKASI]**. Path relatif terhadap akar repo; `$id` adalah segmen literal di nama berkas rute.

---

## Daftar Isi

1. Gambaran Singkat & kondisi kerja manual sekarang
2. Aktor
3. Peta Permasalahan Bisnis PB-1 s.d. PB-9 (beserta rinciannya)
4. Keterkaitan antar-PB
5. Peta Cepat: Masalah → Lokasi Kode
- **A.** Changelog sejak `7927ca5`
- **B.** Fakta untuk Bab IV (use case, transisi status, state berkas, ERD, sequence)
- **C.** Fakta untuk Bab V (dependensi, struktur, ukuran kode, tes, deploy, halaman, ISO 25010)
- **D.** Ketidakkonsistenan & pertanyaan terbuka
- **Lampiran** teknis (stack, arsitektur, basis data, keamanan, UI, istilah usang)

---

## 1. Gambaran Singkat

Aplikasi ini adalah **aplikasi internal mandiri** milik satuan kerja di lingkungan **BPS**, dengan tiga lapisan yang menyatu:

1. **Repositori dokumen kegiatan.** Pegawai menyimpan semua dokumen kegiatan di satu tempat dan bisa menyaringnya: dokumen bernilai uang (**material**) maupun administratif (**non-material**).
2. **Alur persetujuan berjenjang.** Dokumen material dirutekan **Pegawai → PPK → PPSPM**, lengkap dengan penolakan, revisi, dan pengembalian. Semua perpindahan status dikendalikan satu **modul transisi status terpusat**.
3. **Pemberkasan, unduh, dan pembersihan.** Dokumen yang sudah selesai dikelompokkan Kepala Sub Bagian Umum ke dalam **Berkas**. Satu berkas mewakili satu **Cara Pembayaran** dalam satu **Tahun Anggaran**, dan ditutup dengan satu **Nomor SPM**. Aplikasi menyediakan unduhan ZIP (per berkas, atau banyak dokumen hasil saringan). Karena disk terbatas, soft file berkas lama dan lampiran dokumen non-material yang sudah lewat masa guna bisa dibersihkan, sementara metadatanya tetap tersimpan.

> **Aplikasi ini tidak berintegrasi dengan aplikasi lain** (KipApp, SAKIP, SPIDER, Google Drive, aplikasi arsip nasional, dan lain-lain). Tidak ada kode yang mengirim atau menerima data dari layanan eksternal. `.env.example` hanya memuat `DATABASE_URL`, `DMS_LOCAL_STORAGE_ROOT`, `DMS_FILE_TOKEN_SECRET`, `APP_URL`, dan `DMS_SESSION_COOKIE_SECURE`. Apa yang dilakukan pengguna dengan file unduhan terjadi **manual di luar sistem**.

### Kondisi kerja di kantor sekarang (titik awal permasalahan)

Proses pertanggungjawaban dokumen di satker ini **masih manual dan berbasis kertas**; belum ada sistem informasi yang membantu.

- Pegawai menyusun berkas fisik lalu mengantarnya ke meja PPK. Setelah PPK setuju, berkas diantar lagi ke meja PPSPM.
- Status berkas hanya diketahui dengan bertanya langsung. Berkas bisa "mengendap" di satu meja tanpa ada yang tahu.
- Persetujuan berupa paraf atau tanda tangan basah. Penolakan sering lisan atau berupa coretan, tanpa alasan tertulis.
- Syarat kelengkapan tiap kegiatan hanya diingat staf senior. Nama kegiatan, komponen, dan jenis permintaan ditulis bebas, sehingga tidak seragam.
- Rekap jumlah dokumen selesai dan total realisasi dihitung manual dari banyak orang, lalu dikompilasi ke Excel.
- Dokumen digital pendukung tercecer di PC atau flashdisk masing-masing, dan berkas fisik menumpuk di lemari.
- Tiap akhir tahun, berkas fisik di-scan ulang satu per satu untuk keperluan arsip.

**North Star proyek:** setiap pegawai tahu persis apa yang harus mereka lakukan hari ini, dan setiap dokumen bisa dilacak statusnya secara real-time oleh pihak yang berwenang.

---

## 2. Aktor

| Aktor | Kode internal | Tanggung jawab utama |
|---|---|---|
| **Pegawai** | `PEGAWAI` | Mengajukan dokumen, mengunggah kelengkapan, merevisi dokumen yang ditolak, melihat Laporan Saya. |
| **Ketua Tim** (PJ Kegiatan) | *kapabilitas*, bukan peran login. Sumbernya tabel `master.ketua_tim_assignments`. | Pegawai yang ditunjuk untuk satu atau lebih kegiatan. Mendapat grup menu "PJ Kegiatan": Monitoring Dokumen Tim, Laporan Kegiatan, dan Pembersihan Dokumen. Grup ini disembunyikan bila pegawai tidak punya penugasan (`KETUA_TIM_ONLY_NAV_IDS`, `src/components/layout/AppSidebar.tsx:53`). |
| **PPK** | `PPK` | Validasi tahap pertama: meloloskan atau menolak dokumen. Menindaklanjuti penolakan PPSPM, dengan memperbaiki lalu mengirim ulang, atau mengembalikan ke pegawai. Memantau Nominal Realisasi. |
| **PPSPM** | `PPSPM` | Persetujuan tahap akhir: menyetujui (dokumen selesai) atau menolak kembali ke PPK. Memantau Nominal Realisasi. |
| **Kepala Sub Bagian Umum (KSBU)** | `KEPALA_SUB_BAGIAN_UMUM` (rute `/kasubag`) | Mengklasifikasikan dokumen selesai ke Berkas, menambahkan dokumen tanpa alur persetujuan, menutup berkas dengan Nomor SPM, membersihkan file berkas, dan mengelola Master Klasifikasi Dokumen (Cara Pembayaran). |
| **Penanggung Jawab Kinerja** | `PENANGGUNG_JAWAB_KINERJA` | Melihat Laporan Kinerja (material dan non-material) dan Activity Log seluruh pengguna. |
| **Admin** | `ADMIN` | Mengelola pengguna dan peran, penugasan Ketua Tim, data master, serta tema dan identitas aplikasi. Melihat Activity Log seluruh pengguna. |

Aturan peran:

| Aturan | Bukti |
|---|---|
| Hak akses di server adalah **gabungan seluruh peran** yang dimiliki pengguna. Peran aktif hanya menentukan ruang kerja UI (menu). | `hasLocalRole` / `hasAnyLocalRole` memeriksa `session.roles` (`src/lib/auth/local-server-auth.ts:58-71`). Cookie `dms_active_role` hanya dipakai untuk mengisi `activeRole` (`:48-51`) dan sidebar (`AppSidebar.tsx:83`). |
| Berpindah peran lewat header, bukan untuk Admin. | `src/components/layout/AppLayout.tsx:332` → `POST /api/auth/role-switch`, yang menolak ADMIN dan peran yang tidak dimiliki (`src/routes/api/auth/role-switch.ts:62-78`). |
| Akun operasional **otomatis juga PEGAWAI**. | `normalizeAdminRolePayload` (`src/lib/users/role-assignment.ts:6-16`) menambahkan PEGAWAI bila belum ada. Dipanggil saat membuat dan mengubah pengguna (`src/routes/api/users/index.ts:107`, `src/routes/api/users/$id.ts:119,128`, `src/lib/users/local-user-mutations.ts:64,138`). |
| **ADMIN eksklusif.** | Masukan campuran diringkas menjadi `[ADMIN]` oleh `normalizeAdminRolePayload`. Saat login dan pemuatan sesi, kombinasi campuran ditolak `validateAssignedRoles` (`src/lib/auth/role-resolution.ts:26-28`). |
| **Minimal satu admin aktif.** | `evaluateAdminRoleMutationPolicy` dan `evaluateAdminDeactivationPolicy` (`role-assignment.ts:47-83`), dengan pesan `LAST_ACTIVE_ADMIN_ERROR` = "Minimal harus ada satu akun ADMIN aktif." |

---

## 3. Peta Permasalahan Bisnis (Ringkasan)

| # | Permasalahan Bisnis Utama | Proses manual sekarang → kelemahannya |
|---|---|---|
| **PB-1** | Alur & status dokumen tidak transparan | Berkas diantar dari meja ke meja; pemilik tidak tahu berkasnya di mana, kurang apa, atau harus berbuat apa. |
| **PB-2** | Persetujuan berjenjang lambat & tidak terstandar | PPK lalu PPSPM memaraf tumpukan kertas; tidak ada antrean; penolakan lisan. |
| **PB-3** | Jejak audit & akuntabilitas lemah | Bukti hanya paraf, mudah hilang, dan tidak mencatat kapan maupun alasannya. |
| **PB-4** | Data referensi & kelengkapan tidak seragam | Nama kegiatan dan komponen ditulis bebas; syarat kelengkapan hanya diingat staf senior. |
| **PB-5** | Realisasi anggaran & pembedaan dokumen tidak tertib | Dokumen bernilai uang dan administratif ditumpuk bersama; nominal dicatat manual. |
| **PB-6** | Dokumen selesai tercecer & penyimpanan terbatas | Berkas menumpuk di lemari, file tercecer di PC, dan tiap akhir tahun di-scan ulang. |
| **PB-7** | Pelaporan kinerja & rekap realisasi manual | Rekap dihitung manual dari banyak orang ke Excel. |
| **PB-8** | Kontrol akses & akuntabilitas identitas lemah | Siapa pun yang memegang map bisa membacanya; paraf bisa dipalsukan. |
| **PB-9** | Dokumen kerja non-material menumpuk tanpa batas | Surat tugas, undangan, dan notulen ikut menumpuk; tidak ada yang berwenang memangkasnya. |

> Tidak ada PB baru. Fitur baru sejak versi lama dipetakan ke PB yang ada:
> - **Monitoring Dokumen Tim** masuk PB-7.3 (baru).
> - **Tahun Anggaran pada berkas** masuk PB-6.1/6.2.
> - **Halaman Bantuan** masuk Lampiran F.
> - **Pembersihan unggahan tertunda** masuk PB-6.5 (baru).

---

# PB-1 — Alur & Status Dokumen Tidak Transparan

**Proses sekarang.** Pegawai menyerahkan map ke meja PPK, lalu tidak tahu apakah map itu sudah diterima, sedang ditinjau, ditolak, atau sudah diteruskan ke PPSPM.

## PB-1.1 — Pegawai tidak tahu dokumennya ada di tahap mana

- **Solusi.** Setiap dokumen punya status baku, penanda "sedang di tangan siapa" (PPK/PPSPM), dan penanda "siapa yang harus memperbaiki" (Pegawai/PPK). Semua perpindahan status melewati modul transisi status terpusat (Bagian B.2). Halaman **Dokumen Diajukan** (`/pegawai/dokumen`) menampilkan badge status. Halaman detail (`/pegawai/dokumen/$id`) menampilkan status dan timeline riwayat.
- **Use case.** Pegawai membuka Dokumen Diajukan dan melihat "Sedang Divalidasi PPK" tanpa perlu bertanya.
- **Pengguna.** Pegawai. PPK, PPSPM, dan KSBU melihat status yang sama dari kotak masuk masing-masing.
- **Keterkaitan.** Status mengisi kotak masuk (PB-2), riwayat (PB-3), dan menjadi syarat pemberkasan (PB-6: hanya dokumen "Selesai").
- **Keputusan perancangan.** "Selesai" (`COMPLETED`) adalah status akhir. Dokumen yang sudah masuk berkas **tidak berubah status**; keanggotaannya dibaca dari tabel item berkas.
- **Perubahan dari rancangan lama.** Status "Draf" tidak lagi pernah tersimpan permanen. Pengajuan baru langsung menjadi "Divalidasi PPK" (material) atau "Tersimpan" (non-material) dalam satu transaksi. Lihat PB-5.1.

## PB-1.2 — Tidak jelas dokumen milik pribadi atau atas nama kegiatan

- **Solusi.** Kapabilitas **Ketua Tim** per kegiatan. Satu kegiatan hanya punya satu ketua tim (unique index `ketua_tim_kegiatan_unique` pada `kegiatan_id`, `src/db/schema/master/ketua-tim-assignments.ts:28`). Satu pengguna boleh menjadi ketua di banyak kegiatan. Formulir pengajuan menampilkan penanda Ketua Tim atau Anggota, dan dokumen menyimpan `is_ketua_tim`. Bila pengaju mengaku Ketua Tim, server memeriksa penugasannya (`hasKetuaTimAssignment`, `src/lib/dokumen/local-submit-write-bridge.ts:294-306`).
- **Use case.** Budi memilih kegiatan SAKERNAS dan melihat penanda "Anda Ketua Tim". Dokumennya tampil di Laporan Kegiatan dan Monitoring Dokumen Tim.
- **Pengguna.** Pegawai/Ketua Tim; Admin (menugaskan lewat Master User).
- **Keterkaitan.** Menentukan kelengkapan wajib (PB-4.2), laporan yang tampil (PB-7), dan hak pembersihan non-material (PB-9).

## PB-1.3 — Pegawai tidak tahu "apa yang kurang" dari berkasnya

- **Solusi.** Formulir **Ajukan Dokumen** bertahap (`/pegawai/dokumen/aju`, langkah-langkah di `src/components/dokumen/form/Step*.tsx`) dengan **checklist kelengkapan dinamis** (`KelengkapanChecklist.tsx`). Checklist diambil dari master kelengkapan yang cocok **persis** dengan kombinasi pilihan pengguna (PB-4.2). Server menolak pengajuan bila:
  - tidak ada lampiran sama sekali ("Minimal upload satu lampiran…", `checkLampiranNotEmpty`, `local-submit-write-bridge.ts:515-521`);
  - ada kelengkapan wajib yang belum diunggah (`checkRequiredKelengkapan`, `:537-572`).

  Unggahan yang ditinggalkan di tengah formulir dibuang otomatis (PB-6.5).
- **Use case.** Ada 5 item kelengkapan, 3 di antaranya wajib. Tombol Ajukan terkunci, dan server juga menolak, sampai ketiganya terunggah.
- **Pengguna.** Pegawai.
- **Keterkaitan.** Mengurangi penolakan di PB-2. Bergantung pada master kelengkapan (PB-4.2).
- **Perubahan dari rancangan lama.** Aturan "minimal satu lampiran" dan "kelengkapan wajib" kini dipakai bersama oleh pengajuan pertama, pengajuan ulang oleh pegawai, dan pengiriman ulang oleh PPK (`validateResubmitRequirements`, `src/lib/dokumen/resubmit-validation.ts:31-76`, commit `b6c3292`).

---

# PB-2 — Persetujuan Berjenjang Lambat & Tidak Terstandar

**Proses sekarang.** Setiap pejabat menyortir sendiri tumpukan berkas di mejanya. Tidak ada antrean yang jelas, dan penolakan sering lisan.

## PB-2.1 — Tidak ada kotak masuk kerja per pejabat

- **Solusi.** Kotak masuk per peran yang disaring berdasarkan status:
  - PPK **Validasi Dokumen** (`/ppk/inbox`): dokumen "Divalidasi PPK".
  - PPSPM **Persetujuan Dokumen** (`/ppspm/inbox`): dokumen "Menunggu PPSPM".
  - KSBU **Pengklasifikasian Dokumen** (`/kasubag/inbox`): dokumen "Selesai" yang belum masuk berkas.

  Setiap API memeriksa peran di server.
- **Use case.** PPK login dan langsung melihat 12 dokumen yang menunggu validasi.
- **Pengguna.** PPK, PPSPM, KSBU.
- **Keterkaitan.** Isi kotak masuk ditentukan PB-1.1; setiap aksi tercatat di PB-3.

## PB-2.2 — Alur persetujuan tidak baku dan rawan "loncat tahap"

- **Solusi.** Satu **modul transisi status terpusat** (`src/lib/fsm.ts`) mendefinisikan 8 transisi sah dan aktor yang berhak untuk tiap transisi. Tabel lengkapnya ada di Bagian B.2. Setiap perubahan status dokumen material di rute API memanggil modul ini. Perubahan itu juga memakai **penjaga konflik**: `UPDATE … WHERE status = <status asal>`, dan bila baris tidak berubah, respons konflik dikirim (`DokumenTransitionConflictError`, `src/lib/dokumen/transition-conflict.ts`). Dua pejabat yang menekan tombol bersamaan tidak bisa sama-sama berhasil.
- **Use case.** PPSPM tidak bisa menyetujui dokumen yang belum lolos PPK, karena dokumen itu tidak pernah berstatus "Menunggu PPSPM".
- **Pengguna.** Semua aktor alur kerja.
- **Keterkaitan.** Fondasi PB-1.1 dan PB-6. Perbedaan jalur material dan non-material ada di PB-5.1.
- **Perubahan dari rancangan lama.** Penjaga konflik bersamaan (optimistic guard) ditambahkan di rute approve, reject, kembalikan, dan resubmit (commit `fc59b44`).

## PB-2.3 — Penolakan tanpa alasan yang terekam

- **Solusi.** Penolakan PPK dan PPSPM mewajibkan catatan **10–2000 karakter** (`rejectDokumenSchema`, `src/lib/schemas/dokumen.ts:103-105`). Penolakan menetapkan siapa yang harus memperbaiki:
  - PPK menolak → pegawai yang memperbaiki (`revision_target = USER`).
  - PPSPM menolak → PPK yang memperbaiki (`revision_target = PPK`).
- **Use case.** PPK menolak dengan catatan "Nominal kuitansi tidak sama dengan RAB", dan pegawai membaca catatan itu di halaman revisi.
- **Pengguna.** PPK dan PPSPM menolak; Pegawai dan PPK menindaklanjuti.
- **Keterkaitan.** Catatan masuk riwayat (PB-3) dan memicu revisi (PB-2.4).

## PB-2.4 — Perbaikan dan kirim ulang berbelit

- **Solusi.** Tiga jalur tindak lanjut:
  - **Pegawai merevisi** (`/pegawai/dokumen/$id/revisi` → `POST /api/dokumen/$id/submit`, aksi `RESUBMIT`): dokumen kembali ke "Divalidasi PPK".
  - **PPK memperbaiki lalu mengirim ulang** setelah ditolak PPSPM (`/ppk/dokumen/$id/resubmit` → `POST /api/ppk/resubmit/$id`, aksi `RESUBMIT_PPK`): dokumen **langsung** ke "Menunggu PPSPM" tanpa mengulang antrean PPK.
  - **PPK mengembalikan ke pegawai** (`POST /api/ppk/kembalikan/$id`, aksi `KEMBALIKAN`): status tetap "Perlu Revisi", tetapi penanggung jawab perbaikan pindah dari PPK ke Pegawai. Catatan otomatis berbunyi "Dikembalikan ke pegawai oleh PPK. Alasan penolakan PPSPM: …" (`src/routes/api/ppk/kembalikan/$id.ts:14-23,76,82`).

  Sebelum dikirim ulang, server memeriksa tiga hal (`validateResubmitRequirements`): nominal > 0 untuk material, minimal satu lampiran, dan kelengkapan wajib. Lampiran lama yang diganti baru dihapus **setelah** basis data berhasil diperbarui, dan hanya bila tidak dirujuk baris lain (`cleanupUnreferencedReplacedLocalAttachments`, dipanggil di `src/routes/api/dokumen.$id.ts:615` dan `src/routes/api/ppk/resubmit/$id.ts:357,539`).
- **Use case.** Dokumen ditolak PPSPM karena salah kode akun. PPK membetulkannya sendiri lalu mengirim ulang tanpa membebani pegawai.
- **Pengguna.** Pegawai, PPK.
- **Keterkaitan.** Bergantung pada PB-2.3; tercatat di PB-3.
- **Perubahan dari rancangan lama.** Validasi prasyarat kirim ulang dan penghapusan file lama yang ditunda adalah fitur baru (`fc59b44`, `b6c3292`).

## PB-2.5 — Hasil kerja pejabat tidak terpelihara

- **Solusi.** Halaman turunan per peran:
  - PPK: **Dokumen Tervalidasi** (`/ppk/tervalidasi`), **Dokumen Tidak Valid** (`/ppk/ditolak`), **Revisi Dokumen** (`/ppk/revisi`).
  - PPSPM: **Dokumen Ditolak** (`/ppspm/ditolak`), **Dokumen Selesai** (`/ppspm/selesai`).
- **Pengguna.** PPK, PPSPM.
- **Keterkaitan.** Sumber datanya sama dengan PB-7.

---

# PB-3 — Jejak Audit & Akuntabilitas Lemah

**Proses sekarang.** Bukti hanya paraf, tidak bertanggal jelas, alasan penolakan tidak tertulis, dan tidak bisa ditelusuri.

## PB-3.1 — Tidak ada riwayat aktivitas per dokumen

- **Solusi.** Tabel **riwayat aktivitas dokumen** (`dokumen.log_aktivitas`) yang bersifat append-only berdasarkan kontrak. Setiap pengajuan, validasi, penolakan, pengiriman ulang, dan pengembalian menulis satu baris: aksi, pelaku, waktu, tahap, dan catatan. Contoh nama aksi: `SUBMIT`, `STORE`, `PPK_APPROVE`, `PPK_REJECT`, `PPSPM_APPROVE`, `PPSPM_REJECT`, `RESUBMIT`, `RESUBMIT_PPK`, `PPK_KEMBALIKAN`. Riwayat ditampilkan sebagai timeline di halaman detail. Label berbahasa Indonesia ada di `src/lib/dokumen/aksi-labels.ts`.
- **Pengguna.** Semua aktor membaca; sistem menulis.
- **Keterkaitan.** Menerima kejadian dari PB-2; dilengkapi PB-3.2 dan PB-3.3.

## PB-3.2 — Jejak hilang ketika data dihapus

- **Solusi.** Tabel **audit** terpisah (`audit.audit_log`). Kolom `entity_id` **sengaja tanpa foreign key**, dan isi entitas diawetkan di `metadata_snapshot`. Tiga aksi yang dicatat (`src/db/schema/audit/audit-log.ts:23-27`):

  | Aksi | Penulis |
  |---|---|
  | `DOKUMEN_LAMPIRAN_DIBERSIHKAN` | PB-9 |
  | `DOKUMEN_DIHAPUS_PERMANEN` | Pegawai menghapus dokumen non-material miliknya, `src/routes/api/dokumen.$id.ts:792` |
  | `BERKAS_LAMPIRAN_DIBERSIHKAN` | PB-6.3, `src/lib/archive/berkas-arsip-physical-destruction.ts:713` |

- **Pengguna.** Pemeriksa / pengembang lewat akses basis data. **Tidak ada halaman atau API yang membaca tabel ini**; `auditLog` hanya dipakai oleh tiga penulis di atas.
- **Keterkaitan.** Melengkapi PB-3.1.

## PB-3.3 — Riwayat pemberkasan tidak terlacak

- **Solusi.** Tabel **riwayat berkas** (`arsip.berkas_arsip_activity`). Jenis peristiwanya dibatasi check constraint (`src/db/schema/arsip/berkas-arsip.ts:141-153`):

  | Peristiwa | Keterangan |
  |---|---|
  | Berkas dibuka | |
  | Dokumen persetujuan diklasifikasikan | |
  | Dokumen manual ditambahkan | |
  | Berkas ditutup | |
  | Metadata diperbarui | Juga dipakai untuk "Batalkan Usulan", `berkas-arsip-service.ts:988-999` |
  | Diusulkan dibersihkan | |
  | File dibersihkan | |

  Riwayat ini tampil sebagai timeline di detail berkas.
- **Pengguna.** KSBU.

## PB-3.4 — Tidak ada pandangan aktivitas per pengguna

- **Solusi.** Halaman **Activity Log** untuk enam peran, memakai komponen bersama `ActivityLogView.tsx` dan `GET /api/activity-log`, yang menggabungkan riwayat dokumen dan riwayat berkas (maksimal 500 baris).
  - **Admin dan PJ Kinerja** melihat semua pengguna (`GLOBAL_SCOPE_ROLES`, `src/routes/api/activity-log.ts:15`; `scope=all`, ditolak 403 untuk peran lain, `:65-69`).
  - Empat peran lainnya hanya melihat aktivitas sendiri.
  - Filter per pengguna, filter peran, dan pencarian berjalan **di klien** (`ActivityLogView.tsx:97-109`).
- **Pengguna.** Semua peran untuk aktivitas sendiri; Admin dan PJ Kinerja untuk lintas pengguna.
- **Perubahan dari rancangan lama.** Versi lama menyebut Admin melihat riwayat aktivitas di halaman Detail User. Halaman Master User (`src/routes/admin.master-data.user.tsx`) **tidak memuat riwayat aktivitas**; riwayat lintas pengguna kini ada di Activity Log dengan cakupan `all`, yang juga dibuka untuk PJ Kinerja.

---

# PB-4 — Data Referensi & Kelengkapan Tidak Seragam

**Proses sekarang.** Nama kegiatan, komponen, dan jenis permintaan ditulis bebas. Syarat kelengkapan hanya diingat staf senior.

## PB-4.1 — Entitas referensi tidak terpusat dan tidak saling terhubung

- **Solusi.** Modul **Data Master** dengan dua cabang independen:

  ```
  Cabang A — konteks kegiatan              Cabang B — jenis permintaan (GLOBAL)
  Fungsi                                   Jenis Permintaan   (nama unik di antara yang aktif)
   └ Kegiatan                               └ Kategori Permintaan
      └ Komponen                               └ Detail Permintaan

  Keduanya bertemu di Kelengkapan Dokumen, dicocokkan PERSIS pada:
  (kegiatan, status ketua tim, komponen, jenis, kategori, detail)
  ```

  Hubungan induk-anak ditegakkan dengan foreign key `ON DELETE restrict`. Nama unik per induk berlaku **hanya di antara baris aktif** (partial unique index `… WHERE is_active = true`, misalnya `src/db/schema/master/komponen.ts:32-34`). Penghapusan dari UI bersifat soft-delete (`is_active = false`). Jenis Permintaan sempat punya induk Komponen, tetapi dilepas lagi oleh migrasi `0016_decouple_jenis_permintaan_from_komponen.sql`.
- **Pengguna.** Admin mengelola lewat 7 menu data master (Fungsi, Kegiatan, Komponen, Jenis, Kategori, Detail, Kelengkapan) ditambah Master User. Pegawai, KSBU, dan PJ Kinerja memakainya di dropdown dan filter.
- **Keterkaitan.** Fondasi PB-1.2, PB-5, PB-6 (dokumen manual memakai Fungsi → Kegiatan → Komponen), dan PB-7.
- **Perubahan dari rancangan lama.** **Master Jenis Dokumen dihapus total.** Tabel dan kolom `dokumen_transaksi.jenis_dokumen_id` di-drop oleh migrasi `0019_drop_master_jenis_dokumen.sql`. Rute CRUD dan halaman adminnya dihapus di commit `3834a95`. Dokumen non-material memakai "Nama Dokumen" teks bebas (PB-5.1).

## PB-4.2 — Syarat kelengkapan tidak baku

- **Solusi.** Tabel **master kelengkapan dokumen** mendefinisikan daftar dokumen wajib atau opsional per kombinasi (kegiatan × Ketua Tim/Anggota × komponen × jenis × kategori × detail). Pencocokannya **exact-match**: kolom yang kosong di pengajuan hanya cocok dengan baris yang kolomnya juga kosong. Pencocokan dilakukan oleh:
  - **server**: `buildRequiredKelengkapanCondition`, `src/lib/dokumen/local-submit-drizzle-adapter.ts:159-173`, commit `d45bcd9`;
  - **formulir pengajuan**: `matchesCurrentSelection`, `src/components/dokumen/KelengkapanChecklist.tsx:59-72`.

  Penegakan berlapis:
  - **Basis data**: dua check constraint, yaitu kategori butuh jenis dan detail butuh kategori (`src/db/schema/master/kelengkapan-dokumen.ts:51-58`).
  - **Admin**: validasi rantai saat mengelola master kelengkapan.
  - **Formulir**: tombol lanjut terkunci sampai kedua cabang terisi ke daun.
  - **Server**: pengajuan ditolak bila kelengkapan wajib belum diunggah.
  - **Non-material** tanpa jenis permintaan tidak memakai checklist (`shouldReadRequiredKelengkapan`, `local-submit-write-bridge.ts:585-587`).
- **Pengguna.** Admin mengelola; Pegawai dan Ketua Tim mengikuti.
- **Catatan.** Halaman **revisi** pegawai dan halaman **kirim ulang** PPK masih memakai pencocokan bertingkat yang berbeda dari server. Lihat D-1.

## PB-4.3 — Klasifikasi dokumen (Cara Pembayaran) tidak terstruktur

- **Solusi.** **Master Klasifikasi Dokumen** berbentuk hierarki induk–anak (`arsip.master_klasifikasi_arsip.parent_id`, kode unik bila diisi). Hanya node daun yang bisa dipakai. Daftar pilihan disaring per **Tahun Anggaran** (`src/routes/api/kasubag/klasifikasi/index.ts:130-137,165-172`) oleh `berkas-klasifikasi-eligibility.ts`:
  - Cara Pembayaran yang berkasnya untuk TA tersebut sudah ditutup **tidak bisa dipilih lagi** pada TA yang sama.
  - Induk yang semua anaknya tersaring ikut hilang (`filterKlasifikasiTreeForBerkasSelection`, `:129-174`).
- **Pengguna.** KSBU mengelola dan memakai di `/kasubag/klasifikasi`.
- **Keterkaitan.** Kunci pengelompokan di PB-6.

---

# PB-5 — Realisasi Anggaran & Pembedaan Dokumen Tidak Tertib

## PB-5.1 — Dokumen bernilai uang dan tidak bernilai uang diperlakukan sama

- **Solusi.** Dua jalur pengisian:

  | | **Material** | **Non-Material** |
  |---|---|---|
  | Rantai yang diisi | Fungsi → Kegiatan → **Komponen** → Jenis → Kategori → Detail | Fungsi → Kegiatan → **Nama Dokumen** (teks bebas) |
  | Nominal realisasi | wajib > 0 (`validateNominalForMaterial`) | tidak diisi |
  | Status setelah diajukan | "Divalidasi PPK" → … → "Selesai" | langsung **"Tersimpan"** |
  | Masuk antrean PPK/PPSPM | ya | tidak |
  | Judul otomatis | `{nama daun} {tahun} {nama pengaju}` | `{nama dokumen} {tahun} {nama pengaju}` (`local-submit-write-bridge.ts:413`) |
  | Boleh diubah pemilik | hanya saat "Perlu Revisi" untuk Pegawai | saat "Tersimpan" dan lampiran belum dibersihkan (`src/routes/api/dokumen.$id.ts:490-505`) |
  | Boleh dihapus permanen | tidak | ya, bila "Tersimpan" dan tidak berada di berkas (`dokumen.$id.ts:675-800`) |

  Nama daun diturunkan berurutan: Detail → Kategori → Jenis → Komponen → Kegiatan (`resolveLocalSubmitLeafName`, `:458-502`).
- **Keputusan perancangan.** Jalur non-material adalah **satu-satunya perpindahan status di luar modul transisi terpusat**. `buildLocalSubmitTransitionPlan` (`local-submit-write-bridge.ts:436-456`) menetapkan "Tersimpan" langsung dengan aksi riwayat `STORE`. Jalur material memanggil transisi `DRAFT --SUBMIT--> IN_PPK_VALIDATION`. Pada kedua jalur, baris dibuat sebagai `DRAFT` lalu status diperbarui **dalam transaksi yang sama** (`executeLocalSubmitWritePlan`, `:363-396`), sehingga status Draf tidak pernah terlihat di luar transaksi.
- **Perubahan dari rancangan lama.** Jalur Draf lama dihapus (commit `8faf1c8`):
  - `POST /api/dokumen` (buat draf) sudah tidak ada; `src/routes/api/dokumen/index.ts` hanya punya GET.
  - Cabang Draf di `POST /api/dokumen/$id/submit` dihapus; rute itu kini hanya menerima revisi pegawai (`src/routes/api/dokumen.$id.submit.ts:98-110`).
  - Endpoint khusus ubah nominal `PATCH /api/dokumen/$id/nominal` dihapus (commit `d1f147e`).
- **Pengguna.** Pegawai menentukan karakteristik; PPK dan PPSPM hanya menangani material.
- **Keterkaitan.** Hanya dokumen material yang bisa diberkaskan (PB-6), nominal dijumlahkan di PB-7, dan hanya non-material yang masuk PB-9.

## PB-5.2 — Rantai permintaan tidak terikat ke dokumen material

- **Solusi.** Kolom `komponen_id` (dengan FK), serta `jenis_permintaan_id`, `kategori_permintaan_id`, dan `detail_permintaan_id` pada dokumen. Sejak migrasi 0020 (D-20) ketiga kolom terakhir juga punya FK ke tabel master masing-masing (`ON DELETE restrict`), sama seperti `komponen_id` (lihat B.4).
- **Keterkaitan.** Menyambungkan PB-4.1 ke PB-7.

---

# PB-6 — Dokumen Selesai Tercecer & Penyimpanan Terbatas

**Peran aplikasi.** Aplikasi ini bukan sistem kearsipan resmi dan tidak berintegrasi dengan aplikasi mana pun. Tugasnya menyimpan dokumen selesai per berkas dan menyediakan unduhan. Istilah kearsipan diganti di tampilan: "berkas", "masa simpan", "pembersihan file". Identifier internal lama (`berkas_arsip`, `USUL_MUSNAH`, `DIMUSNAHKAN`) dipertahankan di basis data.

## Model berkas

```
Pengklasifikasian Dokumen ─┐
                           ├─▶ BERKAS TERBUKA ──(Tutup Berkas: Nomor SPM + Masa Simpan Minimal)──▶ BERKAS TERTUTUP
Penambahan Dokumen ────────┘   (satu per Cara Pembayaran × Tahun Anggaran)                            │ (Usulkan Pembersihan)
                                                                                                      ▼
                                             BERKAS TERTUTUP ◀──(Batalkan Usulan)── USUL PEMBERSIHAN
                                                                                                      │ (Bersihkan File — ketik "BERSIHKAN FILE BERKAS")
                                                                                                      ▼
                                                                                        FILE DIBERSIHKAN (akhir; metadata tetap)
```

## PB-6.1 — Dokumen selesai tidak terkumpul rapi per berkas

- **Solusi.** Dua pintu masuk yang bermuara ke satu struktur:
  - **Pengklasifikasian Dokumen** (`/kasubag/inbox` → detail `/kasubag/dokumen/$id` → `POST /api/kasubag/dokumen/$id/archive`). Hanya untuk dokumen berstatus "Selesai" (`src/routes/api/kasubag/dokumen.$id.archive.ts:103`). Dokumen masuk sebagai item `WORKFLOW`, dan statusnya tidak berubah.
  - **Penambahan Dokumen** (`/kasubag/penambahan-arsip`). Entri manual KSBU tanpa alur persetujuan, berisi Fungsi → Kegiatan → Komponen → Nama Dokumen, tanggal, Cara Pembayaran, TA, nominal, keterangan wajib, dan lampiran opsional. Dokumen masuk sebagai item `MANUAL`.

  KSBU memilih **Cara Pembayaran dan Tahun Anggaran**. TA dipilih di formulir, **tidak diturunkan dari tanggal dokumen**:
  - skema `tahun_anggaran` 2000–2100 di `dokumen.$id.archive.ts:65`;
  - pilihan UI dari tahun berjalan +2 sampai −5 (`src/lib/utils/tahun.ts:5-11`);
  - bawaan formulir manual adalah tahun berjalan (`src/routes/kasubag/penambahan-arsip.tsx:230`).

  Aturan get-or-create (`getOrCreateOpenBerkasForKlasifikasi`, `src/lib/archive/berkas-arsip-service.ts:249-272`):
  - belum ada berkas untuk (Cara Pembayaran, TA) → berkas terbuka baru dibuat;
  - sudah ada yang terbuka → item masuk ke berkas itu;
  - sudah ada tetapi tertutup → ditolak dengan pesan "Berkas untuk Cara Pembayaran ini TA … sudah ditutup" (`:851-876`).

  Keunikan (Cara Pembayaran, TA) ditegakkan **oleh basis data** dengan unique index `berkas_arsip_klasifikasi_tahun_unique` (`src/db/schema/arsip/berkas-arsip.ts:60-61`). Satu dokumen hanya boleh berada di satu berkas (partial unique index pada `dokumen_id` dan `manual_arsip_id`, `:101-106`).
- **Use case.** Lima honor SAKERNAS yang selesai ditambah satu bukti bayar service AC (manual) masuk ke berkas "Honor — TA 2026" yang masih terbuka.
- **Pengguna.** KSBU.
- **Perubahan dari rancangan lama.** Aturan lama "satu berkas **terbuka** per Cara Pembayaran" (partial unique index `berkas_arsip_open_klasifikasi_unique`) diganti "satu berkas per Cara Pembayaran **per Tahun Anggaran**", terbuka maupun tertutup (migrasi `0018_berkas_tahun_anggaran.sql`, commit `56c6a6d`). Konsekuensinya, setelah ditutup, Cara Pembayaran itu **baru bisa dipakai lagi pada TA berikutnya**. Berkas lama diisi TA dari tahun `created_at` zona Asia/Jakarta.

## PB-6.2 — Berkas tidak pernah "ditutup" dengan Nomor SPM

- **Solusi.** Aksi **Tutup Berkas** (`POST /api/kasubag/berkas/$id/close`). Nomor SPM wajib, dan **Masa Simpan Minimal** dipilih dari 1/3/5/10 Tahun/Permanen (`MANUAL_ARCHIVE_RETENTION_LABELS`, `src/lib/archive/retention.ts:1-7`; skema `.strict()` di `src/lib/schemas/berkas-arsip.ts:46-58`). Basis data mewajibkan waktu dan pelaku penutupan untuk berkas tertutup (`berkas_arsip_closed_metadata_check`). Dialog penutupan adalah konfirmasi biasa **tanpa ketik-persis** (`src/routes/kasubag/berkas/-components/CloseBerkasDialog.tsx:232-242`), dan menjelaskan bahwa Cara Pembayaran itu tidak bisa dipakai lagi untuk TA yang sama.
- **Pengguna.** KSBU.
- **Keterkaitan.** Tanggal penutupan menjadi dasar umur berkas dan jatuh tempo (PB-6.3).

## PB-6.3 — Tidak ada cara membersihkan file lama dari disk terbatas

- **Solusi.** Pembersihan dua tahap setelah berkas ditutup:
  1. **Berkas Tertutup** (`/kasubag/berkas/tertutup`) menampilkan Umur Berkas dan penanda Jatuh Tempo. Keduanya dihitung saat halaman dibuka (`computeBerkasAging`), tanpa penjadwal.
  2. **Usulkan Pembersihan** mengubah status ke "Usul Pembersihan", yang bisa **dibatalkan** kembali ke "Tersimpan".
  3. **Pembersihan Berkas** (`/kasubag/pembersihan`): tombol **Bersihkan File** meminta frasa ketik-persis `BERSIHKAN FILE BERKAS`. Frasa ini **ditegakkan di server** (`z.literal`, `src/routes/api/kasubag/berkas/$id/lifecycle.ts:28`; konstanta di `src/lib/archive/berkas-arsip-page-format.ts:16`).

  File fisik dihapus. Kandidat file diturunkan dari keanggotaan berkas, tidak pernah dari input klien. Metadata, Nomor SPM, daftar item, dan seluruh riwayat tetap ada. Akses file berikutnya dijawab **410** "Data file sudah dimusnahkan".
- **Pengguna.** KSBU.
- **Keterkaitan.** Menjaga PB-3. Dokumen dari berkas yang dibersihkan dikeluarkan dari perhitungan PB-7.2.

## PB-6.4 — Mengunduh file masih satu per satu

- **Solusi.** Ekspor ZIP melalui modul bersama `src/lib/export/document-zip.ts`:

  | Ekspor | Endpoint |
  |---|---|
  | Per berkas | `GET /api/kasubag/berkas/$id/export-zip` |
  | Laporan Saya | `POST /api/laporan/saya/export-zip` |
  | Laporan Kegiatan / Monitoring Dokumen Tim | `POST /api/laporan/kegiatan/export-zip` |

  Aturan ekspor:
  - Maksimal **500 dokumen** per ekspor, ditegakkan di server (413; `DOCUMENT_ZIP_MAX_ENTRIES`, `document-zip.ts:71,136`, dan konstanta di ketiga rute).
  - File di atas 250 MB dilewati (`:72,222-224`).
  - Setiap ZIP diawali `DAFTAR_ISI.txt`.
  - Tiap dokumen diotorisasi ulang di server.

  Ekspor CSV metadata juga tersedia di halaman berkas.
- **Pengguna.** Pegawai, Ketua Tim, KSBU.

## PB-6.5 — Unggahan yang ditinggalkan memenuhi disk (baru)

- **Proses sekarang.** Tidak relevan di proses kertas. Masalah ini muncul karena file diunggah **sebelum** formulir disimpan, sehingga formulir yang ditinggalkan meninggalkan file tanpa pemilik.
- **Solusi** (commit `fc59b44`). File hasil unggahan disimpan di area **tertunda** dan baru dipindah ke lokasi resmi saat pengajuan atau penyimpanan berhasil. Pemindahan terjadi di dalam transaksi; bila gagal, file dikembalikan (`moveSubmitFilesOrRollback`, `src/routes/api/dokumen/submit.ts:321-359`). Area tertunda dibersihkan tiga cara:

  | Cara | Mekanisme |
  |---|---|
  | Browser | Hook `useDiscardPendingUploadsOnLeave` mengirim `POST /api/upload?cleanup=pending` saat halaman ditinggalkan. Server hanya menghapus file tertunda milik pengguna itu (`src/routes/api/upload.ts:178-269`). |
  | Server, otomatis | Setelah setiap unggahan, paling sering sejam sekali per proses (`upload.ts:139`), server menghapus file tertunda berumur > 24 jam yang tidak dirujuk (`src/lib/storage/pending-upload-sweeper.ts`). |
  | Manual atau terjadwal | `pnpm storage:sweep-pending` (`src/scripts/sweep-pending-uploads.ts`). |

- **Pengguna.** Sistem (otomatis); operator server (skrip).

## Tabel penamaan (label tampilan vs identifier internal)

| Konsep | Label tampilan | Identifier internal |
|---|---|---|
| Berkas menerima dokumen | Terbuka | `status_berkas = OPEN` (`status_arsip` wajib NULL, `berkas_arsip_open_status_arsip_null_check`) |
| Berkas dikunci | Tertutup | `status_berkas = CLOSED` |
| Tertutup, belum diusulkan | Tersimpan | `status_arsip = AKTIF` |
| Dicalonkan pembersihan | Usul Pembersihan | `status_arsip = USUL_MUSNAH` |
| File sudah dihapus | File Dibersihkan | `status_arsip = DIMUSNAHKAN` |
| (tidak dipakai) | — | `status_arsip = INAKTIF` (nilai enum mati, `src/lib/constants/archive-status.ts`) |
| Klasifikasi | Cara Pembayaran | `klasifikasi_id` |
| Tahun | Tahun Anggaran (TA) | `tahun_anggaran` |
| Masa retensi | Masa Simpan Minimal | `retensi_aktif` |
| Bersihkan file | Bersihkan File | aksi `approve_destruction` |

---

# PB-7 — Pelaporan Kinerja & Rekap Realisasi Manual

## PB-7.1 — Pegawai & Ketua Tim tidak bisa memantau dokumen

- **Solusi.**
  - **Laporan Saya** (`/pegawai/laporan/saya`): dokumen milik sendiri.
  - **Laporan Kegiatan** (`/pegawai/laporan/kegiatan`, grup PJ Kegiatan): dokumen **final** (Selesai atau Tersimpan) dari semua kegiatan yang dipimpin (`scope=final`, `src/lib/laporan/kegiatan-scope.ts:12-15`).

  Keduanya punya filter bertingkat dan ekspor ZIP. Server hanya mengembalikan dokumen dari kegiatan milik penugasan pemanggil (`src/routes/api/laporan/kegiatan.ts:59-68`).
- **Pengguna.** Pegawai; Ketua Tim.

## PB-7.2 — Tidak ada rekap realisasi untuk pihak berwenang

- **Solusi.** Satu tampilan **`MonitoringRealisasiView.tsx`** dipakai tiga halaman dengan drill-down Fungsi → Kegiatan → Komponen → Dokumen (atau per pegawai), filter periode (Triwulan, Tahunan, Semua, Kustom), dan state filter di URL:

  | Halaman | Pengguna | Isi (`GET /api/laporan/kinerja`) |
  |---|---|---|
  | Nominal Realisasi `/ppk/monitoring-realisasi` | PPK | material "Selesai" dengan komponen (`kinerja.ts:168-172`) |
  | Nominal Realisasi `/ppspm/monitoring-realisasi` | PPSPM | sama |
  | Laporan Kinerja `/penanggung-jawab-kinerja/laporan-kinerja` | PJ Kinerja | di atas **ditambah** non-material "Tersimpan" yang lampirannya belum dibersihkan (`scope=laporan_kinerja`, `:174-184`) |

  Aturan cakupan:
  - `scope=laporan_kinerja` **hanya boleh dipakai PJ Kinerja**; peran lain mendapat 403 (`kinerja.ts:121-127`, commit `c40c469`).
  - Dokumen dari berkas yang file-nya sudah dibersihkan **dikeluarkan** (`:141-152`).
  - **Dokumen manual KSBU tidak ikut dihitung.** API hanya membaca `dokumen_transaksi`; nominal manual hanya dijumlahkan di total berkas (`src/lib/archive/berkas-arsip-read-model.ts:375,798`).
  - Maksimal 2000 baris per permintaan.
- **Pengguna.** PPK, PPSPM, PJ Kinerja. Admin tidak.

## PB-7.3 — Ketua Tim tidak tahu dokumen timnya tertahan di mana (baru)

- **Proses sekarang.** Ketua tim menanyai anggota satu per satu apakah SPJ-nya sudah diteken PPK atau PPSPM.
- **Solusi.** Halaman **Monitoring Dokumen Tim** (`/pegawai/monitoring-dokumen-tim`, commit `16f6e1a`). Halaman ini menampilkan dokumen **yang masih dalam proses maupun final** dari kegiatan yang dipimpin (`scope=monitoring`: Divalidasi PPK, Menunggu PPSPM, Perlu Revisi, Selesai, Tersimpan; `kegiatan-scope.ts:12-15`).
  - Tiap dokumen diberi **posisi**: di PPK, di PPSPM, revisi pengaju, revisi PPK, atau selesai (`getPosisiDokumen`, `:41-55`).
  - Ada kartu ringkasan per posisi.
  - Filter: kegiatan, posisi, pembuat, "tertahan > 3/7/14 hari", dan rentang tanggal.
  - Peringatan muncul bila dokumen tertahan > 7 hari (`monitoring-dokumen-tim.tsx:121`).
  - Tersedia ekspor ZIP (maksimal 500).
- **Pengguna.** Ketua Tim.
- **Keterkaitan.** Memakai kapabilitas PB-1.2 dan status PB-1.1; ekspor memakai PB-6.4.

---

# PB-8 — Kontrol Akses & Akuntabilitas Identitas Lemah

## PB-8.1 — Siapa pun bisa memegang dan memproses berkas

- **Solusi.** Hak akses berbasis peran ditegakkan di server. Handler API mengikuti urutan baku: cek same-origin untuk POST/PUT/PATCH/DELETE (`src/lib/security/same-origin.ts:3`), ambil sesi, cek peran yang dimiliki, validasi Zod, lalu jalankan layanan. Menyembunyikan menu hanya kosmetik. Admin tidak bisa menyetujui atau memberkaskan, karena ADMIN tidak pernah digabung dengan peran lain.
- **Pengguna.** Semua aktor.
- **Catatan.** Satu pengecualian ditemukan: `GET /api/kasubag/klasifikasi` tidak memeriksa sesi (D-3).

## PB-8.2 — Tidak ada pengelolaan identitas & wewenang

- **Solusi.** **Master User** (`/admin/master-data/user`):
  - membuat pengguna dengan **username** (3–30 karakter `a-z0-9._-`, minimal satu huruf) dan NIP/NRP (unik);
  - menetapkan peran dan penugasan Ketua Tim;
  - menonaktifkan atau mengaktifkan pengguna;
  - mereset kata sandi;
  - melihat foto profil pengguna (`GET /api/users/$id/avatar`).

  Setiap pengguna bisa mengganti kata sandi dan foto profilnya di `/profile`.

  **Login memakai username atau NIP** (`or(eq(users.username, …), eq(users.nipNrp, …))`, `src/lib/auth/local-auth-service.ts:155`). Email kini opsional dan bukan identitas login (migrasi `0017_username_login_identity.sql`).

  Rincian keamanan (Lampiran E):
  - kata sandi di-hash **Argon2id**;
  - sesi berlaku **8 jam**;
  - batas percobaan login **5 gagal per 10 menit**, lalu jeda 15 menit;
  - logout mencabut sesi saat itu;
  - ganti atau reset kata sandi dan penonaktifan akun mencabut **semua** sesi pengguna (`revokeAllUserSessions`, `src/lib/users/local-user-passwords.ts:44,92`, `local-user-mutations.ts:328`).
- **Pengguna.** Admin; semua pengguna untuk profil sendiri.
- **Perubahan dari rancangan lama.** Login dengan email diganti username/NIP (commit `80fb6d3`).

## PB-8.3 — Lampiran bisa dibuka siapa saja

- **Solusi.** File disimpan di luar direktori publik. Lampiran dokumen dibuka dengan meminta **URL bertanda tangan HMAC-SHA256** yang terikat pada pengguna, sesi, dokumen, dan indeks lampiran. Masa berlakunya **15 menit untuk pratinjau** dan **1 jam untuk unduh** (`src/lib/storage/document-file-access.ts:56-57,61-135`).

  Hak baca diperiksa ulang pada setiap permintaan (`canSessionReadDocument`, `:344-375`). Yang berhak:
  - pemilik dokumen;
  - PPK dan PPSPM untuk status tertentu;
  - KSBU untuk dokumen "Selesai";
  - Ketua Tim untuk dokumen non-Draf di kegiatannya.

  Admin murni ditolak. Respons 410 dikirim bila lampiran sudah dibersihkan ("Data file sudah dibersihkan") atau berkasnya sudah dibersihkan ("Data file sudah dimusnahkan"). File di dalam berkas dan dokumen manual dibuka lewat rute KSBU yang memeriksa sesi dan peran, **tanpa** token HMAC.
- **Unggahan.** Maksimal **5 MB**. Server memeriksa ekstensi (pdf, doc, docx, xls, xlsx, jpg, jpeg, png), MIME yang dilaporkan, kecocokan ekstensi dengan MIME, dan **tanda tangan isi file** (magic bytes) (`src/lib/storage/local-upload.ts:107-140,229-259`; `src/lib/upload/document-upload-policy.ts`).
- **Pengguna.** Semua aktor yang berhak.

---

# PB-9 — Dokumen Kerja Non-Material Menumpuk Tanpa Batas

## PB-9.1 — Tidak ada yang berwenang & tidak ada mekanisme memangkas

- **Solusi.** Menu **Pembersihan Dokumen** (`/pegawai/pembersihan-dokumen`, grup PJ Kegiatan) untuk Ketua Tim. Halaman ini menampilkan dokumen non-material "Tersimpan" dari kegiatan yang dipimpin, beserta umur dokumen dan penanda "lama" untuk dokumen berumur > **90 hari** (`NON_MATERIAL_STALE_DAYS`, `src/lib/dokumen/pembersihan.ts:10`).

  Eksekusi meminta frasa ketik-persis **`BERSIHKAN`**, maksimal **200 dokumen** per permintaan. Keduanya ditegakkan server (`z.literal`, `.max(PEMBERSIHAN_BATCH_LIMIT)`, `src/routes/api/pembersihan-dokumen.bersihkan.ts:14-19`).

  Server menilai ulang setiap dokumen (`buildPembersihanPlan`, `src/lib/dokumen/pembersihan-service.ts:55-99`). Dokumen ditolak bila:
  - bukan non-material murni;
  - bukan "Tersimpan";
  - bukan kegiatan yang dipimpin;
  - sudah dibersihkan;
  - ada di berkas.
- **Keputusan perancangan.** Yang dibersihkan hanya file fisik:
  - status tidak diubah;
  - daftar lampiran tidak dikosongkan;
  - kondisi dicatat di `lampiran_dibersihkan_at/by/alasan`;
  - jejak ditulis ke tabel audit (PB-3.2).
- **Pengguna.** Ketua Tim.
- **Keterkaitan.** PB-5.1, PB-1.2, PB-6.3, PB-8.3, PB-3.2.

---

## 4. Bagaimana Semua Masalah Saling Terhubung

```
                ┌──────── PB-8 Kontrol Akses & Identitas ────────────────────────┐
                │ gabungan peran di server · username/NIP · URL lampiran HMAC     │
                └───────────────────────────┬────────────────────────────────────┘
                                            │ menegakkan
 PB-4 Data Master ──────► PB-1 Transparansi ──────► PB-2 Persetujuan ──────► PB-3 Jejak Audit
 (A) Fungsi→Kegiatan→     status + posisi,          modul transisi terpusat   riwayat dokumen +
     Komponen             Ketua Tim, checklist      PPK → PPSPM, revisi,      riwayat berkas +
 (B) Jenis→Kategori→      exact-match               kembalikan, konflik       tabel audit
     Detail                     │                         │ Selesai                  │
 Kelengkapan exact-match        │                         │                          │
 Cara Pembayaran (hierarki)     ▼                         ▼                          │
        └────► PB-5 Material vs Non-Material ────► PB-6 Pemberkasan & Unduh ─────────┤
               (nominal │ Nama Dokumen)            (Cara Pembayaran × TA) Terbuka →   │
                    │   │                          Tertutup → Usul → Dibersihkan;     │
                    │   │                          unggahan tertunda dibersihkan      │
                    │   └──────► PB-9 Pembersihan Non-Material (Ketua Tim, >90 hari) ─┘
                    ▼
               PB-7 Pelaporan: Laporan Saya · Laporan Kegiatan · Monitoring Dokumen Tim ·
               Nominal Realisasi (PPK/PPSPM) · Laporan Kinerja (PJ Kinerja)
               — tanpa dokumen manual KSBU, tanpa berkas yang file-nya dibersihkan

 Aplikasi mandiri: TIDAK bertukar data dengan KipApp, SAKIP, SPIDER, Drive, atau aplikasi arsip.
```

---

## 5. Peta Cepat: Masalah → Lokasi di Kode

| Area | Berkas/rute kunci |
|---|---|
| Transisi status | `src/lib/fsm.ts`, `src/lib/constants/document-status.ts`, `src/lib/types/fsm.ts`, `src/lib/dokumen/transition-conflict.ts` |
| Pengajuan dokumen | `src/routes/api/dokumen/submit.ts`, `src/lib/dokumen/local-submit-write-bridge.ts`, `local-submit-repository.ts`, `local-submit-drizzle-adapter.ts`, `submit-file-preflight.ts` |
| Validasi kirim ulang | `src/lib/dokumen/resubmit-validation.ts`, `src/routes/api/dokumen.$id.submit.ts`, `src/routes/api/ppk/resubmit/$id.ts` |
| Formulir bertahap | `src/routes/pegawai/dokumen/aju.tsx`, `src/components/dokumen/form/Step*.tsx`, `src/components/dokumen/KelengkapanChecklist.tsx` |
| Ubah/hapus dokumen | `src/routes/api/dokumen.$id.ts` (GET/PATCH/DELETE) |
| Validasi PPK | `src/routes/ppk/*`, `src/routes/api/ppk/dokumen/$id/{approve,reject}.ts`, `src/routes/api/ppk/resubmit/$id.ts`, `src/routes/api/ppk/kembalikan/$id.ts` |
| Persetujuan PPSPM | `src/routes/ppspm/*`, `src/routes/api/ppspm/dokumen/$id/{approve,reject}.ts` |
| Pemberkasan | `src/lib/archive/berkas-arsip-service.ts`, `berkas-arsip-read-model.ts`, `berkas-klasifikasi-eligibility.ts`, `src/routes/api/kasubag/dokumen.$id.archive.ts`, `src/routes/api/kasubag/berkas/**`, `src/routes/kasubag/berkas/**` |
| Siklus & pembersihan berkas | `src/lib/constants/archive-status.ts`, `src/lib/archive/retention.ts`, `berkas-arsip-physical-destruction.ts`, `src/routes/api/kasubag/berkas/$id/lifecycle.ts`, `src/routes/kasubag/pembersihan/index.tsx` |
| Penambahan dokumen manual | `src/lib/manual-arsip.ts`, `src/lib/schemas/manual-arsip.ts`, `src/lib/storage/manual-arsip-pending-attachments.ts`, `src/routes/api/kasubag/manual-arsip/*`, `src/routes/kasubag/penambahan-arsip.tsx` |
| Tahun anggaran | `drizzle/0018_berkas_tahun_anggaran.sql`, `src/lib/utils/tahun.ts`, `src/routes/api/kasubag/klasifikasi/index.ts` |
| Ekspor ZIP | `src/lib/export/document-zip.ts`, `laporan-zip-entries.ts`, `src/routes/api/kasubag/berkas/$id/export-zip.ts`, `src/routes/api/laporan/{saya,kegiatan}.export-zip.ts`, `src/components/laporan/ExportZipDialog.tsx` |
| Data master | `src/lib/master-data/*`, `src/routes/admin.master-data.*.tsx`, `src/routes/api/master-*.ts` |
| Ketua Tim | `src/db/schema/master/ketua-tim-assignments.ts`, `src/lib/schemas/ketua-tim.ts`, `src/routes/api/ketua-tim/*`, `src/routes/api/users/me/ketua-tim.ts` |
| Pembersihan non-material | `src/lib/dokumen/pembersihan.ts`, `pembersihan-service.ts`, `src/routes/api/pembersihan-dokumen*.ts`, `src/routes/pegawai/pembersihan-dokumen.tsx` |
| Monitoring Realisasi & Laporan Kinerja | `src/components/kinerja/MonitoringRealisasiView.tsx`, `src/lib/laporan/{monitoring-rows,periode}.ts`, `src/routes/api/laporan/kinerja.ts` |
| Laporan pegawai & Monitoring Dokumen Tim | `src/routes/pegawai/laporan/*`, `src/routes/pegawai/monitoring-dokumen-tim.tsx`, `src/lib/laporan/kegiatan-scope.ts`, `src/routes/api/laporan/{saya,kegiatan}.ts`, `src/components/laporan/{HierarchicalFilter,FilterToolbar}.tsx` |
| Auth & sesi | `src/lib/auth/*` (`local-auth-service.ts`, `session-*.ts`, `login-rate-limit.ts`, `password.ts`), `src/routes/api/auth/*`, `src/db/schema/auth/*` |
| Peran & navigasi | `src/lib/auth/local-server-auth.ts`, `src/lib/users/role-assignment.ts`, `src/lib/guards.ts`, `src/config/navigation.ts`, `src/components/layout/AppSidebar.tsx` |
| Same-origin | `src/lib/security/same-origin.ts` |
| Penyimpanan & akses file | `src/lib/storage/*`, `src/lib/upload/document-upload-policy.ts`, `src/routes/api/upload.ts`, `src/routes/api/files/access.ts`, `src/routes/api/dokumen/{preview,download}-url.ts` |
| Unggahan tertunda | `src/hooks/useDiscardPendingUploadsOnLeave.ts`, `src/lib/storage/pending-upload-*.ts`, `src/scripts/sweep-pending-uploads.ts` |
| Jejak audit | `src/db/schema/dokumen/log-aktivitas.ts`, `src/db/schema/audit/audit-log.ts`, `src/db/schema/arsip/berkas-arsip.ts`, `src/routes/api/activity-log.ts`, `src/components/activity-log/ActivityLogView.tsx` |
| Tema, identitas, bantuan | `src/db/schema/app/app-settings.ts`, `src/lib/schemas/settings.ts`, `src/routes/api/settings/*`, `src/routes/admin.settings.tsx`, `src/routes/bantuan.tsx` |

---

# A. Changelog sejak `7927ca5`

41 commit; 182 berkas di `src/` + `drizzle/` berubah (+4.632 / −5.348 baris). Commit `1a09093`–`cf4c5ce` hanya berisi diagram dan dokumen.

| # | Dulu (`7927ca5`) | Sekarang (`b6c3292`) | Bukti |
|---|---|---|---|
| 1 | Login dengan **email** | Login dengan **username atau NIP**; email opsional, tidak unik | `drizzle/0017_username_login_identity.sql`; `local-auth-service.ts:155`; commit `80fb6d3` |
| 2 | Satu berkas **terbuka** per Cara Pembayaran (partial unique index) | Satu berkas per **(Cara Pembayaran, Tahun Anggaran)**, terbuka maupun tertutup; TA dipilih KSBU | `drizzle/0018_berkas_tahun_anggaran.sql`; `berkas-arsip.ts:60-63`; commit `56c6a6d` |
| 3 | Master Jenis Dokumen ada (tidak dipakai) | Tabel, kolom `dokumen_transaksi.jenis_dokumen_id`, rute, dan halaman admin dihapus | `drizzle/0019_drop_master_jenis_dokumen.sql`; commit `3834a95` |
| 4 | `PATCH /api/dokumen/$id/nominal` | Dihapus | commit `d1f147e` |
| 5 | `POST /api/dokumen` (buat draf) dan cabang Draf di `/api/dokumen/$id/submit` | Dihapus; pengajuan hanya lewat `POST /api/dokumen/submit` | commit `8faf1c8`; `dokumen.$id.submit.ts:98-110` |
| 6 | Checklist server memakai pencocokan bertingkat | Exact-match 6 kolom, sama dengan formulir pengajuan | `local-submit-drizzle-adapter.ts:159-173`; commit `d45bcd9` |
| 7 | `scope=laporan_kinerja` terbuka untuk PPK/PPSPM | Hanya PJ Kinerja (403 untuk yang lain) | `kinerja.ts:121-127`; commit `c40c469` |
| 8 | Kirim ulang tanpa validasi prasyarat terpadu | Nominal > 0, minimal satu lampiran, dan kelengkapan wajib dicek untuk SUBMIT, RESUBMIT, dan RESUBMIT_PPK | `resubmit-validation.ts`; commit `b6c3292` |
| 9 | Tidak ada penjaga konflik bersamaan | `UPDATE … WHERE status = asal` + `DokumenTransitionConflictError` | `transition-conflict.ts`; commit `fc59b44` |
| 10 | File unggahan yang ditinggalkan menetap | Area tertunda + pembersihan saat halaman ditinggalkan + sweeper 24 jam + skrip `storage:sweep-pending` | `useDiscardPendingUploadsOnLeave.ts`, `pending-upload-sweeper.ts`; commit `fc59b44` |
| 11 | File lampiran lama langsung dihapus saat diganti | Dihapus **setelah** basis data diperbarui, dan hanya bila tidak dirujuk | `local-attachment-reference-cleanup.ts`; commit `fc59b44` |
| 12 | Ketua Tim hanya punya Laporan Kegiatan & Pembersihan | Tambahan **Monitoring Dokumen Tim** (dokumen dalam proses + posisi + peringatan tertahan) dalam grup menu "PJ Kegiatan" | `routes/pegawai/monitoring-dokumen-tim.tsx`, `kegiatan-scope.ts`; commit `16f6e1a` |
| 13 | Tidak ada halaman bantuan | Halaman **Bantuan** `/bantuan` (tautan di sidebar, layout, dan login) | `src/routes/bantuan.tsx`; commit `7ee6307` |
| 14 | Admin tidak melihat foto profil pengguna | `GET /api/users/$id/avatar` + tampilan di Master User | commit `ad79004` |
| 15 | Rute ketua-tim tanpa Zod | Skema `src/lib/schemas/ketua-tim.ts` | commit `8ace388` |
| 16 | Orkestrator submit 777 baris + kompensasi DB/file | Dihapus; diganti pemindahan file di dalam transaksi + rollback | `submit-runtime-orchestrator.ts` & `submit-db-file-compensation.ts` dihapus; commit `fc59b44` |
| 17 | `POST /api/kasubag/manual-arsip/$id/attachments` | Dihapus; lampiran manual ikut alur unggahan tertunda | `manual-arsip-pending-attachments.ts`; commit `fc59b44` |
| 18 | Kode mati: `RoleSwitcher.tsx`, `UserMenu.tsx`, `StatsBento.tsx`, `DokumenFormContext.tsx`, `dokumen-form-reducer.ts`, `KinerjaPagePrimitives.tsx`, `ui/avatar.tsx`, `ui/card.tsx`, `src/db/index.ts`, `src/lib/db/*`, `lib/constants/{env,index,tables}.ts` | Dihapus | commit `6a1b158`, `d067410`, `630f9ae`, `34c8c0a`, `d61214d` |
| 19 | *(koreksi dokumen, bukan perubahan kode)* Versi lama menyebut hanya Admin yang melihat Activity Log lintas pengguna | Admin **dan PJ Kinerja** (`scope=all`); sudah begitu sejak `7927ca5` | `activity-log.ts:15` di `7927ca5` dan HEAD |

---

# B. Fakta untuk Bab IV

## B.1 Use case per modul (dicocokkan dengan `src/config/navigation.ts` dan rute nyata)

Penomoran UC mengikuti kerangka Bab IV (24 UC). Kolom "Catatan kode" menandai hal yang **harus disesuaikan** di kerangka.

| UC | Nama | Aktor | Menu (label → path) | Endpoint utama | Catatan kode |
|---|---|---|---|---|---|
| **M1 Akses dan Akun** |||||
| UC-01 | Login | Semua | `/login` | `POST /api/auth/login` | **Username/NIP**, bukan email |
| UC-02 | Logout | Semua | header | `POST /api/auth/logout` | |
| UC-03 | Mengelola Profil dan Kata Sandi | Semua | Profil → `/profile` | `/api/users/me`, `/api/users/me/change-password` | Foto profil: `/api/users/me` (DELETE untuk menghapus) |
| UC-04 | Berpindah Peran Aktif | Peran operasional | header (`AppLayout.tsx:332`) | `POST /api/auth/role-switch` | Hanya mengubah ruang kerja UI |
| **M2 Pengajuan Dokumen** |||||
| UC-05 | Mengajukan Dokumen | Pegawai | Ajukan Dokumen → `/pegawai/dokumen/aju` | `POST /api/upload`, `POST /api/dokumen/submit` | |
| UC-06 | Mengelola Dokumen yang Diajukan | Pegawai | Dokumen Diajukan → `/pegawai/dokumen` (+ `/$id`, `/$id/edit`) | `GET/PATCH/DELETE /api/dokumen/$id` | Ubah/hapus hanya non-material Tersimpan |
| UC-07 | Merevisi Dokumen yang Ditolak | Pegawai | Revisi Dokumen → `/pegawai/revisi`, `/pegawai/dokumen/$id/revisi` | `POST /api/dokumen/$id/submit` | |
| UC-08 | Membersihkan Lampiran Dokumen Non-Material | Ketua Tim | PJ Kegiatan › Pembersihan Dokumen → `/pegawai/pembersihan-dokumen` | `POST /api/pembersihan-dokumen/bersihkan` | |
| **M3 Persetujuan Berjenjang** |||||
| UC-09 | Memvalidasi Dokumen | PPK | Validasi Dokumen → `/ppk/inbox`, `/ppk/dokumen/$id` | `POST /api/ppk/dokumen/$id/{approve,reject}` | |
| UC-10 | Menindaklanjuti Dokumen yang Ditolak PPSPM | PPK | Revisi Dokumen → `/ppk/revisi`, `/ppk/dokumen/$id/resubmit` | `PATCH/POST /api/ppk/resubmit/$id`, `POST /api/ppk/kembalikan/$id` | |
| UC-11 | Menyetujui Dokumen | PPSPM | Persetujuan Dokumen → `/ppspm/inbox`, `/ppspm/dokumen/$id` | `POST /api/ppspm/dokumen/$id/{approve,reject}` | |
| UC-12 | Melihat Dokumen yang Telah Diproses | PPK, PPSPM | Dokumen Tervalidasi `/ppk/tervalidasi`, Dokumen Tidak Valid `/ppk/ditolak`, Dokumen Ditolak `/ppspm/ditolak`, Dokumen Selesai `/ppspm/selesai` | GET per halaman | |
| **M4 Pemberkasan** |||||
| UC-13 | Mengklasifikasikan Dokumen ke Berkas | KSBU | Pengklasifikasian Dokumen → `/kasubag/inbox`, `/kasubag/dokumen/$id` | `POST /api/kasubag/dokumen/$id/archive` | + pilih **Tahun Anggaran** |
| UC-14 | Menambahkan Dokumen tanpa Alur Persetujuan | KSBU | Penambahan Dokumen → `/kasubag/penambahan-arsip` | `POST /api/kasubag/manual-arsip` | + Tahun Anggaran |
| UC-15 | Mengelola Berkas | KSBU | Berkas Terbuka `/kasubag/berkas`, Berkas Tertutup `/kasubag/berkas/tertutup`, detail `/kasubag/berkas/$id` | `/api/kasubag/berkas/**` (close, export-zip) | |
| UC-16 | Membersihkan File Berkas | KSBU | Pembersihan Berkas → `/kasubag/pembersihan` | `POST /api/kasubag/berkas/$id/lifecycle` | |
| UC-17 | Mengelola Klasifikasi Dokumen | KSBU | Master Klasifikasi Dokumen → `/kasubag/klasifikasi` | `/api/kasubag/klasifikasi(/$id)` | |
| **M5 Pelaporan dan Pemantauan** |||||
| UC-18 | Melihat Laporan Saya | Pegawai | Laporan Saya → `/pegawai/laporan/saya` | `GET /api/laporan/saya`, `POST …/export-zip` | |
| UC-19 | Melihat Laporan Kegiatan | Ketua Tim | PJ Kegiatan › Laporan Kegiatan → `/pegawai/laporan/kegiatan` | `GET /api/laporan/kegiatan?scope=final` | |
| *UC-baru* | **Memantau Dokumen Tim** | Ketua Tim | PJ Kegiatan › Monitoring Dokumen Tim → `/pegawai/monitoring-dokumen-tim` | `GET /api/laporan/kegiatan?scope=monitoring` | **Belum ada di kerangka 24 UC**; bisa dijadikan UC sendiri atau alur alternatif UC-19 (D-Q1) |
| UC-20 | Memantau Nominal Realisasi | PPK, PPSPM | Nominal Realisasi → `/ppk/monitoring-realisasi`, `/ppspm/monitoring-realisasi` | `GET /api/laporan/kinerja` | |
| UC-21 | Melihat Laporan Kinerja | PJ Kinerja | Laporan Kinerja → `/penanggung-jawab-kinerja/laporan-kinerja` | `GET /api/laporan/kinerja?scope=laporan_kinerja` | |
| **M6 Administrasi Sistem** |||||
| UC-22 | Mengelola Pengguna dan Penugasan Ketua Tim | Admin | Master User → `/admin/master-data/user` | `/api/users/**`, `/api/ketua-tim/**` | |
| UC-23 | Mengelola Data Master | Admin | Departemen Fungsi, Master Kegiatan, Master Komponen, Jenis Permintaan, Kategori Permintaan, Detail Permintaan, Kelengkapan Dokumen → `/admin/master-data/*` | `/api/master-*` | **Hapus "jenis dokumen"** dari deskripsi UC-23 |
| UC-24 | Mengatur Tampilan Aplikasi | Admin | Settings → `/admin/settings` | `/api/settings/{theme,general,epoch}` | Tema `se`/`sp`/`st` (`src/lib/schemas/settings.ts:3`) |

Fungsi yang ada di menu tetapi di luar 24 UC:
- **Activity Log** untuk enam peran. Lingkup: sendiri, atau semua untuk Admin dan PJ Kinerja.
- **Dashboard** per peran.
- **Bantuan** (`/bantuan`).

Menu "Settings" untuk lima peran non-admin **tidak punya tujuan** (D-5).

## B.2 Tabel transisi status dokumen (`src/lib/fsm.ts`)

Status (`src/lib/constants/document-status.ts:1-8`): `DRAFT`, `IN_PPK_VALIDATION`, `IN_PPSPM_APPROVAL`, `NEED_REVISION`, `COMPLETED`, `TERSIMPAN`. Aksi (`fsm.ts:35-42`): `SUBMIT`, `APPROVE`, `REJECT`, `RESUBMIT`, `RESUBMIT_PPK`, `KEMBALIKAN`.

| # | Status asal | Aksi | Aktor | Status tujuan | `current_step` | `revision_target` | Tahap | Prasyarat (modul transisi + rute) | Aksi riwayat | Baris |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | DRAFT | SUBMIT | PEGAWAI | IN_PPK_VALIDATION | PPK | null | 1 | material; ≥1 lampiran; kelengkapan wajib; nominal > 0; komponen terisi; bila mengaku Ketua Tim harus ditugaskan | `SUBMIT` | 20-25 |
| 2 | IN_PPK_VALIDATION | APPROVE | PPK | IN_PPSPM_APPROVAL | PPSPM | null | 2 | peran PPK; penjaga konflik | `PPK_APPROVE` | 26-31 |
| 3 | IN_PPK_VALIDATION | REJECT | PPK | NEED_REVISION | PPK | **USER** | 1 | catatan 10–2000 karakter | `PPK_REJECT` | 32-37 |
| 4 | IN_PPSPM_APPROVAL | APPROVE | PPSPM | COMPLETED | null | null | 2 | peran PPSPM; penjaga konflik | `PPSPM_APPROVE` | 38-43 |
| 5 | IN_PPSPM_APPROVAL | REJECT | PPSPM | NEED_REVISION | PPSPM | **PPK** | 1 | catatan 10–2000 karakter | `PPSPM_REJECT` | 44-49 |
| 6 | NEED_REVISION | RESUBMIT | PEGAWAI | IN_PPK_VALIDATION | PPK | null | 1 | `revision_target = USER` (`fsm.ts:104-109`); pemilik; material; `validateResubmitRequirements` | `RESUBMIT` | 50-55 |
| 7 | NEED_REVISION | RESUBMIT_PPK | PPK | IN_PPSPM_APPROVAL | PPSPM | null | 2 | `revision_target = PPK` (`:110-115`); `validateResubmitRequirements` | `RESUBMIT_PPK` | 56-61 |
| 8 | NEED_REVISION | KEMBALIKAN | PPK | NEED_REVISION | PPK | **USER** | 1 | `revision_target = PPK` (dicek rute, `kembalikan/$id.ts:76`) | `PPK_KEMBALIKAN` | 62-67 |
| — | DRAFT | *(tanpa aksi modul transisi)* | PEGAWAI | **TERSIMPAN** | null | null | 1 | non-material; ≥1 lampiran; Nama Dokumen terisi | `STORE` | `local-submit-write-bridge.ts:436-450` |

Pemeriksaan di `transition()` (`fsm.ts:87-124`), berurutan:
1. Aktor berhak.
2. Pada REJECT, `revisionTarget` harus USER atau PPK.
3. Pada RESUBMIT, target saat ini harus USER.
4. Pada RESUBMIT_PPK, target saat ini harus PPK.
5. Kunci `status:aksi` terdaftar.

Status akhir adalah `COMPLETED` dan `TERSIMPAN`. `COMPLETED` tidak berubah saat diberkaskan.

## B.3 State berkas (`status_berkas` × `status_arsip`)

| `status_berkas` | `status_arsip` | Label | Aksi keluar (aktor KSBU) | Tujuan | Peristiwa riwayat |
|---|---|---|---|---|---|
| — | — | (belum ada) | klasifikasi / penambahan dokumen / `POST /api/kasubag/berkas/open` | OPEN, NULL | `BERKAS_DIBUKA` |
| OPEN | NULL (wajib) | Terbuka | Tutup Berkas (Nomor SPM + Masa Simpan) | CLOSED, AKTIF | `BERKAS_DITUTUP` |
| CLOSED | AKTIF | Tersimpan | `propose_destruction` | CLOSED, USUL_MUSNAH | `BERKAS_DIPINDAHKAN_KE_USUL_MUSNAH` |
| CLOSED | USUL_MUSNAH | Usul Pembersihan | `cancel_proposal` | CLOSED, AKTIF | `METADATA_ARSIP_AKTIF_DIPERBARUI` |
| CLOSED | USUL_MUSNAH | Usul Pembersihan | `approve_destruction` + frasa `BERSIHKAN FILE BERKAS` | CLOSED, DIMUSNAHKAN | `BERKAS_DIMUSNAHKAN` |
| CLOSED | DIMUSNAHKAN | File Dibersihkan | — (akhir) | — | — |
| CLOSED | INAKTIF | — | nilai enum mati; tidak pernah dituju | — | — |

Constraint yang menjaga state ini (`berkas-arsip.ts:62-76`):
- `status_berkas ∈ {OPEN, CLOSED}`;
- `status_arsip ∈ {AKTIF, INAKTIF, USUL_MUSNAH, DIMUSNAHKAN}` atau NULL;
- OPEN ⇒ `status_arsip` NULL;
- OPEN ⇔ `closed_at`/`closed_by` NULL, CLOSED ⇔ keduanya terisi.

Pemetaan aksi ke peristiwa ada di `berkas-arsip-service.ts:988-999`.

## B.4 Tabel untuk ERD

Konvensi: PK `id uuid` kecuali disebut lain. **FK** = foreign key fisik. **Logis** = relasi tanpa FK. `R` = `ON DELETE restrict`, `C` = cascade, `SN` = set null, `NA` = no action.

**Skema `auth`**

| Tabel | Kolom kunci | Constraint / index | FK |
|---|---|---|---|
| `users` | `username`, `email` (nullable), `password_hash`, `password_hash_algorithm` (default `argon2id`), `nama_lengkap`, `display_name`, `nip_nrp`, `departemen` (teks), `avatar_*`, `is_active`, `deactivated_at/by`, `last_login_at` | UNIQUE `username`, UNIQUE `nip_nrp`; CHECK format username (migrasi 0017) | `deactivated_by` → users: **logis** |
| `roles` | `nama` | UNIQUE `nama` | — |
| `user_roles` | PK komposit (`user_id`, `role_id`) | — | `user_id` → users (C); `role_id` → roles (C) |
| `sessions` | `token_hash`, `expires_at`, `revoked_at`, `remember_me`, `user_agent`, `ip_address` | UNIQUE `token_hash` | `user_id` → users (C) |

**Skema `master`**

| Tabel | Kolom kunci | Constraint / index | FK |
|---|---|---|---|
| `master_fungsi` | `nama`, `is_active` | UNIQUE `nama` | — |
| `master_kegiatan` | `fungsi_id`, `nama` | UNIQUE (`fungsi_id`, `nama`) WHERE aktif | `fungsi_id` → fungsi (R) |
| `master_komponen` | `kegiatan_id`, `nama` | UNIQUE (`kegiatan_id`, `nama`) WHERE aktif | `kegiatan_id` → kegiatan (R) |
| `master_jenis_permintaan` | `nama` | UNIQUE `nama` WHERE aktif | — (global) |
| `master_kategori_permintaan` | `jenis_permintaan_id`, `nama` | UNIQUE (`jenis`, `nama`) WHERE aktif | → jenis (R) |
| `master_detail_permintaan` | `kategori_permintaan_id`, `nama` | UNIQUE (`kategori`, `nama`) WHERE aktif | → kategori (R) |
| `master_kelengkapan_dokumen` | `kegiatan_id`, `is_ketua_tim`, `nama_dokumen`, `required`, `komponen_id`, `jenis/kategori/detail_permintaan_id` | CHECK kategori⇒jenis, detail⇒kategori; INDEX rantai 6 kolom | kegiatan (C); komponen, jenis, kategori, detail (R) |
| `ketua_tim_assignments` | `user_id`, `kegiatan_id`, `created_by` | UNIQUE `kegiatan_id` | user (C); kegiatan (C); `created_by` → users (SN) |

**Skema `dokumen`**

| Tabel | Kolom kunci | Constraint / index | FK |
|---|---|---|---|
| `dokumen_transaksi` | `judul`, `status` (default DRAFT), `current_step`, `revision_target`, `revision_notes`, `lampiran_urls` (jsonb: `kelengkapan_id`, `nama`, `url`, `uploaded_at`), `tahun`, `tanggal` (teks), `is_ketua_tim`, `is_non_material`, `nama_dokumen`, `keterangan_detail`, `nominal_realisasi` NUMERIC(15,2), `lampiran_dibersihkan_at/by/alasan` | CHECK nominal NULL atau ≥ 0; CHECK alasan ∈ {`BERKAS_DIMUSNAHKAN`, `PEMBERSIHAN_NON_MATERIAL`}; 17 index | `fungsi_id` (R), `kegiatan_jenis_id` → kegiatan (R), `komponen_id` (R), `created_by` → users (NA), `lampiran_dibersihkan_by` → users (SN), `jenis_permintaan_id` → master_jenis_permintaan (R), `kategori_permintaan_id` → master_kategori_permintaan (R), `detail_permintaan_id` → master_detail_permintaan (R) (FK sejak migrasi 0020, D-20) |
| `log_aktivitas` | `aksi`, `catatan`, `step_urutan`, `timestamp` | INDEX (`dokumen_id`, `timestamp`) | `dokumen_id` (C), `user_id` (NA) |

Lampiran dokumen **bukan tabel**: disimpan sebagai array JSON di `lampiran_urls`, dan indeks array menjadi identitas lampiran.

**Skema `arsip`**

| Tabel | Kolom kunci | Constraint / index | FK |
|---|---|---|---|
| `master_klasifikasi_arsip` | `nama`, `kode`, `parent_id`, `is_active` | UNIQUE `nama`; UNIQUE `kode` WHERE not null | `parent_id` → dirinya (SN) |
| `berkas_arsip` | `klasifikasi_id`, **`tahun_anggaran`**, snapshot kode/nama klasifikasi, `status_berkas`, `status_arsip`, `nomor_spm`, `retensi_aktif`, `masa_aktif_berakhir`, `closed_at/by` | **UNIQUE (`klasifikasi_id`, `tahun_anggaran`)**; CHECK TA 2000–2100 + 4 CHECK state (B.3) | klasifikasi (R), `closed_by`/`created_by` → users (NA) |
| `berkas_arsip_item` | `berkas_id`, `source_type` (WORKFLOW/MANUAL), `dokumen_id`, `manual_arsip_id`, `added_by` | UNIQUE `dokumen_id` WHERE not null; UNIQUE `manual_arsip_id` WHERE not null; CHECK tepat satu referensi sesuai tipe | berkas, dokumen, manual_arsip, users (NA) |
| `berkas_arsip_activity` | `event_type`, `actor_user_id`, `source_type`, `workflow_document_id`, `manual_document_id`, `metadata_snapshot` | CHECK 8 jenis peristiwa; CHECK referensi sesuai tipe | berkas, users, dokumen, manual_arsip (NA) |
| `manual_arsip` | `nama`, `tanggal`, `keterangan` (wajib), `nominal_realisasi`, `fungsi/kegiatan/komponen_id` (wajib), `klasifikasi_id`, `status_arsip`, kolom siklus lama (`inactivated_*`, `proposed_destroy_*`, `destroyed_*`) | CHECK status_arsip; CHECK nominal ≥ 0 | fungsi, kegiatan, komponen (R); klasifikasi (SN); users (NA) |
| `manual_arsip_attachment` | `logical_path`, `original_filename`, `judul_lampiran`, `content_type`, `size_bytes` | CHECK ukuran ≥ 0; CHECK judul tidak kosong | manual_arsip, users (NA) |

**Skema `audit` dan `app`**

| Tabel | Kolom kunci | Constraint | FK |
|---|---|---|---|
| `audit.audit_log` | `entity_type` (hanya `DOKUMEN`), **`entity_id`**, `aksi` (3 nilai), `metadata_snapshot` | CHECK entity_type & aksi | `actor_user_id` → users (SN); **`entity_id`: logis, sengaja tanpa FK** |
| `app.app_settings` | PK `key` (teks), `value` jsonb | — | **`updated_by`: logis** |

Catatan ERD:
- **Kolom tahun berbeda.** Dokumen punya `tahun`, dari tahun pengajuan. Berkas punya `tahun_anggaran`, yang dipilih KSBU. Keduanya tidak dihubungkan oleh constraint.
- **Tabel migrasi Drizzle.** Tabel `drizzle.__drizzle_migrations` dibuat oleh drizzle-kit, bukan bagian domain.

## B.5 Urutan pemanggilan untuk 6 diagram sequence

**(1) Submit dokumen**: `POST /api/dokumen/submit` (`src/routes/api/dokumen/submit.ts`)
1. `requireSameOrigin` (`:403`) → JSON → `createAndSubmitDokumenSchema.safeParse` (`:412`).
2. `validateNominalForMaterial` (`:421`) → `validateWorkflowChainForCharacteristic` (`:431`).
3. `handleLocalDbSubmit`: `getLocalServerSession` → `createLocalSubmitActorFromSession` (harus PEGAWAI) (`:72-81`).
4. `buildSubmitMovePlan` → `preflightSubmitFiles` (file tertunda ada dan milik pengguna) (`:83-100`).
5. `prepareLocalSubmitWriteBridge` (`local-submit-write-bridge.ts:262-352`):
   - `checkLampiranNotEmpty`
   - `getKegiatanById`
   - `checkRequiredKelengkapan` (query exact-match)
   - `hasKetuaTimAssignment` (bila mengaku Ketua Tim)
   - `resolveLocalSubmitLeafName`
   - `buildLocalSubmitTransitionPlan` (`transition(DRAFT, SUBMIT, PEGAWAI)` atau TERSIMPAN langsung)
6. `executeLocalSubmitWritePlan` dalam satu transaksi (`:363-396`):
   - `createDokumen` (DRAFT)
   - `updateDokumenStatus`
   - `appendLog` (SUBMIT/STORE)
   - `afterWrites`: `moveSubmitFilesOrRollback` (tertunda → resmi)
7. Bila transaksi gagal setelah file dipindah: `rollbackLocalAttachmentMovements`. Respons 201 atau 500.

**(2) Unggah lampiran**: `POST /api/upload` (`src/routes/api/upload.ts`)
1. `requireSameOrigin` → `getLocalServerSession` (tanpa cek peran khusus) (`:61-67`).
2. `?cleanup=pending` → cabang pembersihan (`:70`, `:178-269`).
3. `request.formData()` → `file`, `kelengkapan_id`, `nama_dokumen` (`:74-95`).
4. `validateLocalUploadFileMetadata`: ekstensi, MIME, pasangan, ukuran > 0 dan ≤ 5 MB, sanitasi nama (`local-upload.ts:107-140`).
5. `writeLocalUploadContent`: cek ulang ukuran dan **magic bytes**, lalu tulis ke path tertunda `<user>/<kelengkapan>_<ts>_<nama>` (`local-upload.ts:180-197,229-259`).
6. `maybeSweepStalePendingUploads` (fire-and-forget, maksimal sejam sekali) → 201 `{ url, … }`.

**(3) Approve PPK**: `POST /api/ppk/dokumen/$id/approve`
1. `requireSameOrigin` → `getLocalServerSession` → `hasLocalRole(session, 'PPK')` (403) (`:27-36`).
2. `approveDokumenSchema.safeParse` (`:46`) → SELECT dokumen (`:60-66`).
3. `transition(status, 'APPROVE', 'PPK')` (`:86`).
4. Transaksi (`:94-116`):
   - `UPDATE dokumen_transaksi SET status=IN_PPSPM_APPROVAL, current_step=PPSPM … WHERE id=$id AND status='IN_PPK_VALIDATION'`; bila 0 baris → konflik;
   - `INSERT log_aktivitas (aksi 'PPK_APPROVE')`.
5. 200.

**(4) Klasifikasi dokumen ke berkas**: `POST /api/kasubag/dokumen/$id/archive`
1. `requireSameOrigin` → sesi → `hasLocalRole(KEPALA_SUB_BAGIAN_UMUM)` (`:52-57`).
2. Zod (`klasifikasi_id`, `tahun_anggaran` 2000–2100) (`:65-76`).
3. SELECT dokumen; tolak bila bukan COMPLETED (`:102-103`).
4. `db.transaction` (`:106-117`):
   - `getOrCreateOpenBerkasForKlasifikasi`: validasi daun, lalu `resolveExistingBerkasForKlasifikasi` (tolak bila tertutup), lalu `insertOpenBerkas … ON CONFLICT DO NOTHING` dan baca ulang bila kalah balapan (`berkas-arsip-service.ts:249-272,630,851-876`);
   - `addWorkflowDocumentToOpenBerkas`: INSERT `berkas_arsip_item` (WORKFLOW), dengan unique `dokumen_id` mencegah duplikasi;
   - INSERT `berkas_arsip_activity` (BERKAS_DIBUKA bila baru, lalu DOKUMEN_PERSETUJUAN_DIKLASIFIKASIKAN).
5. 200.

**(5) Akses lampiran**: dua langkah
1. Klien memanggil `GET /api/dokumen/$id/preview/$lampiranIndex` (atau `/download/`, atau `/api/ppk|ppspm/dokumen/$id/...`). Rute ini menjalankan `createDocumentLampiranAccessUrlResponse` (`document-file-access.ts:61-135`):
   - sesi (401);
   - validasi UUID dan indeks (404);
   - `loadDocumentAccessContext`;
   - `canRouteAccessDocument` / `canSessionReadDocument` (403);
   - `resolveDocumentLampiranReference`: 410 bila lampiran dibersihkan atau berkas dibersihkan (`:266-300`);
   - `createInternalFileAccessUrl`: token HMAC `v1.<payload>.<sig>` dengan `expiresAt` +15 menit atau +1 jam;
   - respons `{ signedUrl }`.
2. Browser membuka `GET /api/files/access?token=…`. Rute ini menjalankan `handleInternalFileAccessRequest` (`internal-file-access.ts:82-`):
   - verifikasi tanda tangan (`timingSafeEqual`) dan kedaluwarsa;
   - sesi masih sah serta cocok dengan `subjectUserId`/`sessionId` (`:100`);
   - status diperiksa ulang: 410 bila sudah dibersihkan (`:175-189`);
   - `readLocalLogicalPathFile` (normalisasi path, anti traversal);
   - stream file dengan `secureHeaders` dan `Content-Disposition`.

**(6) Pembersihan non-material**: `POST /api/pembersihan-dokumen/bersihkan`
1. `requireSameOrigin` → sesi → `hasLocalRole('PEGAWAI')` (`:32-39`).
2. `bodySchema.strict()`: `dokumen_ids` 1–200 UUID, `confirmation` = `'BERSIHKAN'` (`:14-19,42-45`).
3. SELECT `ketua_tim_assignments` milik pengguna; 403 bila kosong (`:47-62`).
4. `executePembersihanLampiran` (`pembersihan-service.ts:130-239`):
   - `listCandidatesByIds` + `listDokumenIdsInBerkasArsip`;
   - `buildPembersihanPlan` (5 alasan tolak);
   - `loadProtectedLogicalPaths` (path yang dirujuk baris lain);
   - `deleteLogicalFilesSafely` (hapus file, idempoten untuk file hilang);
   - `applyCleanup` dalam transaksi: UPDATE `lampiran_dibersihkan_at/by/alasan='PEMBERSIHAN_NON_MATERIAL'` + INSERT `audit_log (DOKUMEN_LAMPIRAN_DIBERSIHKAN)` (`:345-360`).
5. 200 `{ report: requested / cleaned / skipped / failed }`.

---

# C. Fakta untuk Bab V

## C.1 Versi dependensi utama

Versi terpasang dibaca dari `node_modules/<pkg>/package.json` pada 27-09-2026. Beberapa paket dideklarasikan `latest` di `package.json`, sehingga versi pasti bergantung pada `pnpm-lock.yaml`.

| Paket | Deklarasi | Terpasang |
|---|---|---|
| react / react-dom | ^19.2.0 | 19.2.4 |
| @tanstack/react-start | latest | 1.167.16 |
| @tanstack/react-router | latest | 1.168.10 |
| nitro | npm:nitro-nightly@latest | 3.0.1-20260329-223454-55f30f48 |
| vite | ^7.3.1 | 7.3.1 |
| typescript | ^5.7.2 | 5.9.3 |
| drizzle-orm | ^0.45.2 | 0.45.2 |
| drizzle-kit | ^0.31.10 | 0.31.10 |
| pg (driver PostgreSQL) | ^8.20.0 | 8.20.0 |
| zod | ^4.3.6 | 4.3.6 |
| argon2 | ^0.44.0 | 0.44.0 |
| archiver (ZIP) | ^8.0.0 | 8.0.0 |
| tailwindcss | ^4.1.18 | 4.2.2 |
| @base-ui/react | ^1.3.0 | 1.3.0 |
| vitest | ^3.0.5 | 3.2.4 |
| @playwright/test | ^1.59.1 | 1.59.1 |
| PostgreSQL (Docker) | `postgres:16` | `infra/docker/postgres/docker-compose.yml` |

## C.2 Struktur direktori

```
src/
├── routes/            file-based routing (TanStack Start)
│   ├── api/           95 handler HTTP (same-origin → sesi → peran → Zod → layanan)
│   ├── pegawai/ ppk/ ppspm/ kasubag/ penanggung-jawab-kinerja/   halaman per peran
│   ├── admin.*.tsx    halaman admin (flat route)
│   └── login.tsx, bantuan.tsx, profile.tsx, forbidden.tsx, dokumen/* (legacy)
├── components/        61 komponen (ui/, layout/, dokumen/, laporan/, kinerja/, activity-log/, …)
├── lib/               108 modul domain: fsm.ts, auth/, users/, dokumen/, archive/, storage/,
│                      export/, laporan/, master-data/, schemas/ (Zod), security/, upload/, constants/
├── db/                client, schema/{auth,master,dokumen,arsip,audit,app}, seed/
├── hooks/  config/navigation.ts  scripts/ (generate-password-hash, sweep-pending-uploads)
drizzle/               0000–0019 migrasi SQL + meta/_journal.json
tests/                 fsm.test.ts, unit/** (Vitest), e2e/** (Playwright)
infra/docker/postgres/ docker-compose + init/001-create-schemas.sql
```

## C.3 Jumlah berkas dan baris per lapisan

Dihitung dari `.ts`/`.tsx` non-tes pada 27-09-2026.

| Lapisan | Lokasi | Berkas | Baris |
|---|---|---|---|
| Presentasi: halaman | `src/routes/**` selain `api/` (`.tsx`, termasuk `-components/`) | 75 | 27.435 |
| Presentasi: komponen | `src/components/` | 61 | 13.192 |
| Endpoint | `src/routes/api/` | 95 | 13.013 |
| Domain / layanan | `src/lib/` | 108 | 19.457 |
| Data | `src/db/` | 32 | 1.588 |
| Lainnya | `src/hooks/` 4 (207), `src/config/` 1 (228), `src/scripts/` 2 (74) | 7 | 509 |
| Migrasi | `drizzle/*.sql` | 20 | — |

Halaman per peran: pegawai 13, ppk 10, ppspm 7, kasubag 10, PJ Kinerja 3, admin 13 (termasuk layout).

## C.4 Pengujian

**Hasil run Vitest** (`pnpm test` = `vitest run`, 27-09-2026 14:05, branch `fix/temuan-bab-v` setelah perbaikan Bagian D, termasuk D-24):
- **112 berkas tes lulus dari 112**;
- **1.152 kasus lulus, 1 dilewati, 0 gagal** (1.153 total);
- durasi 35,32 detik;
- `tsc --noEmit` bersih;
- sebelum perbaikan (HEAD `043449d`): 108 berkas, 1.060 lulus + 1 dilewati (1.061). Selisih: +4 berkas, +92 kasus;
- satu-satunya kasus yang dilewati ada di `tests/unit/arsiparis/berkas-arsip-folder-pages.test.ts`.

**Tes integrasi** (`pnpm test:integration` = `vitest run --config vitest.integration.config.ts`, 27-09-2026, setelah D-20): **1 berkas, 3 kasus lulus**. Berkas `tests/integration/dokumen-transaksi-permintaan-fk.test.ts` menyambung ke PostgreSQL asli (bukan `db` yang di-mock) dan membuktikan FK D-20 menolak UUID yang tidak ada di master (kode `23503`); setiap kasus di dalam transaksi yang di-ROLLBACK. Folder `tests/integration/**` dikecualikan dari `pnpm test`, jadi angka Vitest di atas **tidak** memuat tes ini dan tetap sama sebelum/sesudah D-20. Butuh Postgres lokal jalan (lihat C.5).

Sebaran per modul (folder `tests/unit/*`). Jumlah kasus per modul adalah **perkiraan** dari hitungan `it(`/`test(`; totalnya mengikuti hasil run di atas.

| Modul | Berkas | ± Kasus |
|---|---|---|
| Modul transisi status (`tests/fsm.test.ts`) | 1 | **49** |
| arsiparis (pemberkasan, manual, ekspor berkas, klasifikasi) | 22 | ~318 |
| storage (unggah, tertunda, akses file, pembersihan) | 26 | ~245 |
| dokumen (submit, revisi, guard transisi, PATCH, checklist kelengkapan, pembersihan, master data) | 27 | ~257 |
| laporan (kinerja, periode, filter) | 7 | 91 |
| auth (sesi, login, rate-limit, peran, navigasi) | 10 | 66 |
| export | 1 | 19 |
| utils | 3 | 19 |
| components | 3 | 16 |
| security (same-origin) | 1 | 9 |
| users | 3 | ~16 |
| dashboard / db / profile / kelengkapan / hooks / pegawai / styles | 8 | ~33 |

Rincian 49 kasus `tests/fsm.test.ts`:

| Kelompok | Kasus |
|---|---|
| Transisi valid (8 transisi, termasuk `NEED_REVISION --KEMBALIKAN--> NEED_REVISION`) | 8 |
| Validasi aktor (termasuk KEMBALIKAN: PPK valid; PEGAWAI, PPSPM, KSBU, ADMIN ditolak) | 21 |
| `revisionTarget` pada REJECT | 3 |
| Validasi RESUBMIT | 4 |
| Validasi KEMBALIKAN (status sumber salah ditolak; hasil `revision_target` = USER) | 4 |
| Kombinasi tidak valid | 7 |
| Bentuk error | 2 |

Seluruh 8 transisi kini punya tes unit (D-14 selesai). Tes rute `POST /api/ppk/kembalikan/$id` ada di `tests/unit/dokumen/dokumen-transition-guards-route.test.ts`: 400 bila `revision_target` ≠ PPK atau status bukan NEED_REVISION, 403 untuk peran non-PPK, catatan otomatis memuat "Alasan penolakan PPSPM", 409 saat balapan.

**Playwright (e2e)**, 68 kasus di `tests/e2e/`:

| Spec | Kasus |
|---|---|
| `approval-flow.spec.ts` | 27 |
| `spec-06-user-management.spec.ts` | 21 |
| `submit-flow.spec.ts` | 20 |

**Tidak dijalankan** di pembaruan ini karena butuh aplikasi dan basis data hidup, dan **tidak ada skrip e2e di `package.json`**. Jalankan manual dengan `pnpm exec playwright test` [BELUM TERVERIFIKASI konfigurasi baseURL/seed].

## C.5 Menjalankan dan deploy di LAN

1. **Basis data:** `docker compose -f infra/docker/postgres/docker-compose.yml up -d`. Image `postgres:16`, port **hanya `127.0.0.1:5432`**, TZ Asia/Jakarta; `init/001-create-schemas.sql` membuat skema.
2. **Konfigurasi `.env`** (dari `.env.example`):
   - `DATABASE_URL`
   - `DMS_LOCAL_STORAGE_ROOT` (folder file)
   - `DMS_FILE_TOKEN_SECRET` (wajib; tanpa ini akses file melempar error, `internal-file-access.ts:72-80`)
   - `APP_URL` (dipakai cek same-origin)
   - `DMS_SESSION_COOKIE_SECURE`
3. **Migrasi dan seed:**
   - `pnpm db:migrate` (atau `db:local:migrate` dengan `.env.migration`);
   - `pnpm db:seed`. Seed pengguna dev butuh `DMS_DEV_SEED_PASSWORD_HASH`; hash dibuat dengan `pnpm auth:hash-password`.
   - **Tes:** `pnpm test` (unit, tanpa basis data). Tambahan `pnpm test:integration` menguji constraint langsung ke Postgres; butuh langkah 1–3 di atas (container jalan, `DATABASE_URL` di `.env` benar, migrasi sudah diterapkan).
4. **Pengembangan:** `pnpm dev` (Vite, port 3000). `host: true` membuat server bisa diakses dari LAN. `allowedHosts: true` dan plugin `basicSsl()` menjalankan **HTTPS sertifikat self-signed** (`vite.config.ts:11-24`).
5. **Produksi LAN:** `pnpm build`, lalu `pnpm start` (`node .output/server/index.mjs` dengan `.env`). Port default server Nitro [BELUM TERVERIFIKASI]. Cookie sesi `Secure` aktif di production atau bila `DMS_SESSION_COOKIE_SECURE` diset.
6. **Perawatan:** jadwalkan `pnpm storage:sweep-pending` (misalnya harian) bila perlu; sweeper otomatis juga berjalan setelah unggahan.

## C.6 Halaman per peran untuk tangkapan layar

| Peran | Halaman (path) |
|---|---|
| Umum | Login `/login` (username/NIP), Bantuan `/bantuan`, Profil `/profile`, Forbidden `/forbidden` |
| Pegawai | Dashboard `/pegawai`; Ajukan Dokumen `/pegawai/dokumen/aju` (tiap langkah: karakteristik, komponen, jenis/kategori/detail, nominal, unggah, review); Dokumen Diajukan `/pegawai/dokumen`; Detail `/pegawai/dokumen/$id` (timeline); Ubah `/pegawai/dokumen/$id/edit`; Revisi `/pegawai/revisi`, `/pegawai/dokumen/$id/revisi`; Laporan Saya `/pegawai/laporan/saya` (+ dialog ekspor ZIP); Activity Log `/pegawai/activity-log` |
| Ketua Tim | Monitoring Dokumen Tim `/pegawai/monitoring-dokumen-tim`; Laporan Kegiatan `/pegawai/laporan/kegiatan`; Pembersihan Dokumen `/pegawai/pembersihan-dokumen` (+ dialog ketik `BERSIHKAN`) |
| PPK | Dashboard `/ppk`; Validasi Dokumen `/ppk/inbox`; Detail `/ppk/dokumen/$id` (dialog tolak); Tervalidasi `/ppk/tervalidasi`; Tidak Valid `/ppk/ditolak`; Revisi `/ppk/revisi`; Kirim Ulang `/ppk/dokumen/$id/resubmit`; Nominal Realisasi `/ppk/monitoring-realisasi`; Activity Log |
| PPSPM | Dashboard `/ppspm`; Persetujuan `/ppspm/inbox`; Detail `/ppspm/dokumen/$id`; Ditolak `/ppspm/ditolak`; Selesai `/ppspm/selesai`; Nominal Realisasi `/ppspm/monitoring-realisasi`; Activity Log |
| KSBU | Dashboard `/kasubag`; Pengklasifikasian `/kasubag/inbox` + `/kasubag/dokumen/$id` (pilih Cara Pembayaran + TA); Penambahan Dokumen `/kasubag/penambahan-arsip`; Berkas Terbuka `/kasubag/berkas`; Detail Berkas `/kasubag/berkas/$id` (timeline, dialog Tutup Berkas); Berkas Tertutup `/kasubag/berkas/tertutup` (umur, Jatuh Tempo); Pembersihan Berkas `/kasubag/pembersihan` (toggle Usulan/Sudah Dibersihkan, dialog ketik frasa); Master Klasifikasi `/kasubag/klasifikasi`; Activity Log |
| PJ Kinerja | Dashboard `/penanggung-jawab-kinerja`; Laporan Kinerja `/penanggung-jawab-kinerja/laporan-kinerja` (drill-down, filter periode); Activity Log (semua pengguna) |
| Admin | Dashboard `/admin`; Master User `/admin/master-data/user` (peran, Ketua Tim, reset sandi, nonaktif, foto); Fungsi, Kegiatan, Komponen, Jenis, Kategori, Detail, Kelengkapan `/admin/master-data/*`; Settings `/admin/settings` (tema se/sp/st, sub-judul); Activity Log `/admin/activity-log` (semua pengguna) |

## C.7 Pemetaan fitur → karakteristik ISO/IEC 25010 (bahan butir kuesioner)

| Karakteristik | Sub-karakteristik | Fitur yang bisa ditanyakan | Bukti kode |
|---|---|---|---|
| **Functional suitability** | Kelengkapan, kebenaran, kesesuaian | Pengajuan dua jalur; persetujuan PPK→PPSPM; revisi/kembalikan; pemberkasan per Cara Pembayaran × TA; laporan & realisasi; pembersihan | B.1, B.2, B.3 |
| **Performance efficiency** | Perilaku waktu, kapasitas | Batas ZIP 500 dokumen, batas pembersihan 200, laporan kinerja maksimal 2000 baris, Activity Log maksimal 500 baris, unggah 5 MB | `document-zip.ts:71`, `pembersihan.ts:21`, `kinerja.ts`, `activity-log.ts` |
| **Compatibility** | Ko-eksistensi / interoperabilitas | Berdiri sendiri; hasil ekspor ZIP/CSV bisa dipakai di aplikasi lain secara manual | §1 |
| **Interaction capability (usability)** | Kemudahan dipelajari, operabilitas, perlindungan dari kesalahan, estetika | Formulir bertahap + checklist; halaman Bantuan; menu per peran; konfirmasi ketik-persis; guard perubahan belum disimpan; toast; tiga tema; peringatan dokumen tertahan | `aju.tsx`, `bantuan.tsx`, `navigation.ts`, `ConfirmDialog` |
| **Reliability** | Kematangan, toleransi kesalahan, pemulihan | Transaksi submit + rollback file; penjaga konflik bersamaan; penghapusan file idempoten; hapus file lama setelah commit; sweeper file tertunda; 1.152 tes unit lulus | §PB-2.2, PB-6.5, C.4 |
| **Security** | Kerahasiaan, integritas, non-repudiasi, akuntabilitas, autentisitas | Argon2id; sesi 8 jam dengan token hash; rate-limit login; peran dicek server; same-origin; URL lampiran HMAC; cek magic bytes; riwayat dokumen/berkas/audit | Lampiran E |
| **Maintainability** | Modularitas, dapat diuji, dapat dimodifikasi | Lapisan route/lib/db; modul transisi terpusat; injeksi repository; Zod di setiap batas; 112 berkas tes | Lampiran B, C.4 |
| **Flexibility (portability)** | Kemampuan instal, adaptabilitas | Docker PostgreSQL; berjalan di LAN tanpa internet; konfigurasi via `.env` | C.5 |

---

# D. Ketidakkonsistenan & Pertanyaan Terbuka

Dampak: **T** = Tinggi (bisa salah data/akses atau salah ditulis di skripsi), **S** = Sedang, **R** = Rendah.

| # | Temuan | Bukti | Dampak |
|---|---|---|---|
| D-1 | **✅ SELESAI `69f7fbe`.** Fungsi murni bersama `matchesKelengkapanSelection` (`src/lib/kelengkapan-match.ts`, exact-match 6 kolom) dipakai `KelengkapanChecklist`, halaman revisi Pegawai, dan kirim ulang PPK; `GET /api/ppk/resubmit/$id` kini mengirim `komponen_id`. *Temuan asli:* **Checklist di halaman revisi ≠ server.** Halaman revisi Pegawai dan kirim ulang PPK memakai `matchesCurrentChain` yang **bertingkat**: bila ada detail, hanya detail yang dicocokkan; bila rantai kosong, semua item kegiatan tampil. Halaman PPK **tidak punya cabang komponen**. Server memakai exact-match 6 kolom. Checklist yang dilihat pengguna bisa berbeda dari yang ditegakkan server. | `src/routes/pegawai/dokumen/$id/revisi.tsx:97-121`, `src/routes/ppk/dokumen/$id/resubmit.tsx:96-113` vs `local-submit-drizzle-adapter.ts:159-173` | **T** |
| D-2 | **✅ SELESAI `f26ee7f`.** `validateNominalUpdate` di kedua PATCH: Material > 0, Non-Material menolak nominal; dicek sebelum file dipindah. *Temuan asli:* **Nominal masih bisa diubah tanpa cek > 0.** `PATCH /api/dokumen/$id` menerima `nominalRealisasi` `min(0)`/nullable, termasuk untuk dokumen **non-material**. `PATCH /api/ppk/resubmit/$id` (simpan tanpa kirim) juga tanpa cek > 0. Pengecekan > 0 baru terjadi saat kirim ulang. | `src/routes/api/dokumen.$id.ts:560-565`, `src/lib/schemas/dokumen.ts:54`, `src/routes/api/ppk/resubmit/$id.ts:328-331` | **S** |
| D-3 | **✅ SELESAI `3a34192`.** GET memakai `requireKepalaSubBagianUmum` (401/403). Pemanggilnya hanya halaman `/kasubag/*`. Audit endpoint lain: lihat D-23. *Temuan asli:* **`GET /api/kasubag/klasifikasi` tanpa cek sesi/peran.** Siapa pun di jaringan bisa membaca pohon klasifikasi dan kelayakan berkas per TA. Rute tulis di berkas yang sama tetap memakai `requireKepalaSubBagianUmum`. | `src/routes/api/kasubag/klasifikasi/index.ts:130-` | **S** |
| D-4 | **Setelah ditutup, (Cara Pembayaran, TA) terkunci permanen.** Bila KSBU salah menutup berkas atau butuh SPM kedua pada TA yang sama, tidak ada jalan keluar di aplikasi. Perlu keputusan: memang diinginkan (1 Cara Pembayaran = 1 SPM per TA), atau perlu fitur "buka kembali". | `0018`, `berkas-arsip-service.ts:851-876`, `CloseBerkasDialog.tsx` | **T** (keputusan bisnis) |
| D-5 | **✅ SELESAI `e4abe5a`.** Item dihapus dari 5 peran; ADMIN: "Pengaturan Aplikasi" (path `/admin/settings` tetap); "Activity Log" → "Log Aktivitas" di semua peran. *Temuan asli:* Menu **"Settings" tanpa tujuan** di 5 peran non-admin. | `src/config/navigation.ts` (item tanpa `to`) | R |
| D-6 | **Pencampuran ADMIN diringkas diam-diam**, bukan ditolak. Pesan "ADMIN tidak boleh digabung…" tidak pernah muncul karena `normalizeAdminRolePayload` sudah membuang peran lain. Admin bisa mengira PPK tersimpan padahal tidak. | `role-assignment.ts:6-16`, `users/index.ts:107-108` | S |
| D-7 | **Pemeriksaan "admin aktif terakhir" tanpa kunci baris.** Dua permintaan bersamaan secara teori bisa menonaktifkan dua admin terakhir. | `local-user-mutations.ts:437-449` | R |
| D-8 | **Rate-limit login di memori proses**, dengan kunci IP dari `x-forwarded-for`/`x-real-ip` yang bisa dipalsukan klien. Hilang saat restart dan tidak berbagi antarproses. | `login-rate-limit.ts:28`, `login.ts:96-103` | S |
| D-9 | **"Ingat saya" 30 hari tidak terjangkau.** Konstanta dan kolom `sessions.remember_me` ada, tetapi `loginSchema` tidak punya field-nya. | `session-constants.ts:10`, `src/lib/schemas/auth.ts` | R |
| D-10 | **Validasi MIME memakai tipe yang dilaporkan browser.** Untuk DOCX/XLSX, magic bytes hanya membuktikan "berkas ZIP". | `document-upload-policy.ts:166-193` | R |
| D-11 | **Token HMAC hanya untuk lampiran dokumen.** File berkas dan dokumen manual cukup dengan cek sesi + peran KSBU. Sebutkan dengan tepat di skripsi. | `berkas-arsip-file-access.ts`, `manual-arsip.ts` | R (dokumentasi) |
| D-12 | **✅ SELESAI `df8ea51`, `d328ad7`, dikuatkan `38f5900`.** Submit: `is_ketua_tim` = hasil penugasan di server (klaim Ketua Tim palsu tetap ditolak; Ketua Tim yang mengirim `false` dicatat & dicek sebagai Ketua Tim). **Keputusan produk yang disengaja:** nilai `is_ketua_tim` dikunci sejak SUBMIT untuk seluruh umur dokumen dan **tidak pernah** diverifikasi ulang ke penugasan terkini — bila admin mencabut penugasan Ketua Tim setelah dokumen diajukan, dokumen itu tetap memakai checklist Ketua Tim. Sejak D-24 (`38f5900`), `kegiatanId` tidak lagi diterima `PATCH /api/dokumen/$id` sama sekali, sehingga tidak ada jalan apa pun untuk mengubah `is_ketua_tim` setelah submit — lebih ketat dari rencana semula (menghitung ulang saat kegiatan berubah). Ditegaskan komentar kebijakan di `resubmit-validation.ts` dan tes yang menguncinya. Baris lama sebelum `df8ea51` tidak dikoreksi. *Temuan asli:* **Pengaju bisa mengaku "Anggota" meski ia Ketua Tim.** Server hanya memverifikasi klaim Ketua Tim, bukan klaim Anggota, sehingga checklist Anggota yang lebih ringan bisa dipakai lewat manipulasi permintaan. | `local-submit-write-bridge.ts:294-306` | S |
| D-13 | **Pembersihan non-material menghapus file sebelum menandai basis data.** Bila `applyCleanup` gagal setelah file terhapus, dokumen tidak berlabel "dibersihkan" padahal file hilang. Aksi idempoten bisa diulang, tetapi tidak atomik. | `pembersihan-service.ts:187-236` | R |
| D-14 | **✅ SELESAI `89a93bd`.** +10 kasus di `tests/fsm.test.ts` (39 → 49) dan +6 kasus rute. *Temuan asli:* **Tes modul transisi tidak mencakup `KEMBALIKAN`.** Kerangka Bab IV menargetkan "seluruh transisi lolos unit testing", jadi target ini **belum terpenuhi** untuk transisi #8. | `tests/fsm.test.ts` (39 kasus, 0 untuk KEMBALIKAN) | **T** (klaim skripsi) |
| D-15 | **Tidak ada skrip e2e** di `package.json`; Playwright harus dipanggil manual. | `package.json:8-24` | R |
| D-16 | **Kode peninggalan:** | | R |
| | • `/api/dokumen/rename-pending` tidak dipanggil klien mana pun | `src/routes/api/dokumen/rename-pending.ts` | |
| | • parameter dry-run `useLocalAuthDryRun`/`useLocalPreflightDryRun` | `submit.ts:40-45` | |
| | • label aksi `UPDATE_NOMINAL` dan komentar "update nominal" | `aksi-labels.ts:58-68`, `activity-log.ts:48` | |
| | • cabang `MULTIPLE_OPEN_BERKAS` (mustahil sejak unique index) | `berkas-klasifikasi-eligibility.ts:89-127` | |
| | • enum `INAKTIF` dan peristiwa `BERKAS_DIPINDAHKAN_KE_INAKTIF` | `archive-status.ts`, `berkas-arsip.ts:149` | |
| | • kolom siklus lama di `manual_arsip` dan `retensi_inaktif`/`masa_inaktif_berakhir` | skema arsip | |
| | • tipe `AppUser.email: string` | `src/lib/types/auth.ts:9` | |
| | • rute legacy `/dokumen`, `/dokumen/$id`, `/dokumen/$id/edit`, `/dokumen/aju`, `/dokumen/aji` (salah ketik, redirect), `/dokumen/saya` | `src/routes/dokumen/*` | |
| D-17 | **✅ SELESAI `ae966d2`.** Label → "Karakteristik". *Temuan asli:* **Label UI "Jenis Dokumen"** di ringkasan sukses halaman revisi dan kirim ulang (dan teks konfirmasi pengajuan) sebenarnya menampilkan **karakteristik** (Material/Non-Material), bukan master yang sudah dihapus. Istilahnya membingungkan; sebaiknya "Karakteristik". | `revisi.tsx:402-405`, `resubmit.tsx:418`, `aju.tsx:1184` | R |
| D-18 | **✅ SELESAI `ae966d2`.** Pesan: "Berkas untuk Cara Pembayaran ini pada TA {tahun} sudah ditutup." *Temuan asli:* **`CLOSED_UNAVAILABLE_REASON` tidak menyebut TA.** Pesan kelayakan tidak menjelaskan bahwa kuncinya per tahun. | `berkas-klasifikasi-eligibility.ts:42` | R |
| D-19 | **Tabel `audit.audit_log` tidak punya tampilan UI.** Klaim "pemeriksa bisa menelusuri dokumen yang dihapus" hanya berlaku lewat akses basis data langsung. | tidak ditemukan pembaca di halaman | S (klaim skripsi) |
| D-20 | **✅ SELESAI `8816524`.** Ketiga kolom kini `.references()` ke `master_jenis_permintaan` / `master_kategori_permintaan` / `master_detail_permintaan` (`ON DELETE restrict`, `ON UPDATE no action`, sama dengan `komponen_id`); migrasi `drizzle/0020_dokumen_transaksi_permintaan_fk.sql` (ditulis manual, idempoten). Sebelum migrasi dicek: 0 baris yatim dari 24 dokumen. Dibuktikan oleh tes integrasi pertama, `tests/integration/dokumen-transaksi-permintaan-fk.test.ts` (Postgres asli, transaksi di-ROLLBACK, harus gagal `23503`), dijalankan lewat `pnpm test:integration`. *Temuan asli:* **`dokumen_transaksi.jenis/kategori/detail_permintaan_id` tanpa FK.** Integritasnya hanya dijaga aplikasi. Gambarkan sebagai relasi logis di ERD, atau tambahkan FK. | skema `dokumen-transaksi.ts:58-60`; migrasi 0000 | S |
| D-21 | **Kerangka Bab IV tertinggal dari kode:** UC-01 masih "email"; UC-23 masih "jenis dokumen"; UC-13/UC-14 belum menyebut Tahun Anggaran; Monitoring Dokumen Tim belum ada; Activity Log PJ Kinerja lintas pengguna belum disebut. | `docs/planning/ubah-alur-v1/kerangka-bab-iv.md` (commit `171fc5d`) vs B.1 | **T** (dokumen) |
| D-22 | **PPK/PPSPM bisa membaca semua dokumen** pada status tertentu tanpa pembatasan unit/kegiatan. Sesuai desain satu satker; sebutkan sebagai asumsi. | `document-file-access.ts:381-395` | R |
| D-23 | **✅ SELESAI `b4ee1a8`** (baru, dari audit D-3). 11 handler GET master data (`master-fungsi`, `master-kegiatan`, `master-komponen` +`$id`, `master-jenis` +`$id`, `master-kategori` +`$id`, `master-detail` +`$id`, `master-kelengkapan`) sebelumnya bertanda "Public read endpoint" dan bisa dibaca siapa pun di LAN tanpa login. Kini memakai `requireAnyLocalSession` (401 tanpa sesi; peran apa pun boleh, karena semua pemanggilnya sudah di halaman yang dijaga peran). Mutasi tetap ADMIN-only. Endpoint lain tanpa `getLocalServerSession` langsung memakai pembungkus yang memeriksa sesi + peran (`requireBerkasArsipApiSession`, `requireManualArsipApiSession`, `createDocumentLampiranAccessUrlResponse`, `authorizeCleanupRequest`) atau memang publik (`auth/login`, `auth/logout`, `auth/session`; `auth/role-switch` memeriksa token sesi sendiri). | `src/routes/api/master-*.ts` | S (klaim keamanan) |
| D-24 | **✅ SELESAI `38f5900`.** `updateDokumenSchema` kini `.strict()` dan hanya berisi `lampiranUrls`, `nominalRealisasi`, `keteranganDetail`, `namaDokumen` — persis yang dikirim `edit.tsx` (Non-Material) dan `revisi.tsx` (Material, sebelum resubmit). `kegiatanId`, `fungsiId`, `komponenId`, `jenisPermintaanId`, `kategoriPermintaanId`, `detailPermintaanId`, `tahun`, `tanggal`, `judul` ditolak 400 sebelum dokumen dibaca. *Temuan asli:* **`PATCH /api/dokumen/$id` menerima perubahan metadata yang tidak dikirim UI mana pun** (`kegiatanId`, `fungsiId`, `komponenId`, `tahun`, `tanggal`, `judul`) untuk dokumen Material saat revisi, tanpa menyelaraskan jenis/kategori/detail. Permintaan manual bisa membuat rantai tidak konsisten (mis. komponen baru dengan jenis lama). | `src/routes/api/dokumen.$id.ts`, `updateDokumenSchema` | S |

**Pertanyaan yang butuh keputusan Anda**

| # | Pertanyaan |
|---|---|
| Q1 | Monitoring Dokumen Tim dijadikan **UC baru** (menjadi 25 UC) atau **alur alternatif UC-19**? |
| Q2 | D-4: kunci permanen (Cara Pembayaran, TA) setelah ditutup, apakah memang aturan bisnis? Bila ya, tulis sebagai keputusan perancangan di Bab IV. |
| Q3 | D-1 dan D-2 diperbaiki sebelum pengujian sistem, atau dicatat sebagai keterbatasan? |
| Q4 | D-14: tambah kasus uji `KEMBALIKAN` di `tests/fsm.test.ts` sebelum Bab V, supaya klaim "seluruh transisi lolos unit testing" benar? |
| Q5 | Kolom "tahun" dokumen dan "tahun anggaran" berkas dibiarkan independen (dokumen TA 2025 bisa masuk berkas TA 2026)? |

---

## Lampiran — Catatan Teknis

> Bukan bagian dari permasalahan bisnis.

### A. Tumpukan teknologi

- **Framework:** TanStack Start (React 19 + TypeScript, file-based routing, server Nitro, Vite 7).
- **Basis data:** PostgreSQL 16 lokal + Drizzle ORM (driver `pg`).
- **Autentikasi:** lokal dengan cookie sesi `dms_session`.
- **Penyimpanan file:** filesystem lokal (`DMS_LOCAL_STORAGE_ROOT`).
- **Validasi:** Zod 4 di setiap batas.
- **ZIP:** archiver.
- **UI:** Tailwind CSS 4 + Base UI.
- **Pengujian:** Vitest + Playwright.
- **Manajer paket:** pnpm.

### B. Arsitektur kode

Berlapis dan diiris per fitur. Urutan standar handler: `requireSameOrigin` → `getLocalServerSession` → `hasLocalRole` → Zod → layanan `src/lib/*`. Menurut laporan graf graphify, node paling terhubung setelah `cn()` adalah `getLocalServerSession()` (230 sisi), `db` (185), `hasLocalRole()` (182), dan `requireSameOrigin()` (151). Layanan menerima repository lewat injeksi (`{ repository }`, `{ deps }`), sehingga logika submit, pemberkasan, pembersihan, dan ZIP diuji tanpa basis data.

### C. Basis data: enam skema

`auth`, `master`, `dokumen`, `arsip`, `audit`, `app`. Rinciannya ada di B.4. Aturan bisnis yang ditegakkan constraint:
- satu berkas per (Cara Pembayaran, TA);
- satu ketua tim per kegiatan;
- satu dokumen di satu berkas;
- item berkas punya tepat satu referensi sesuai tipe;
- berkas tertutup wajib punya waktu dan pelaku penutupan;
- berkas terbuka tidak punya status arsip;
- kategori kelengkapan butuh jenis, detail butuh kategori;
- nominal ≥ 0;
- username unik dan berformat;
- NIP unik.

### D. Berjalan mandiri di LAN

Seluruh komponen (DB, auth, file) lokal; tidak perlu internet. Postur: handoff internal/LAN terkontrol, bukan kesiapan produksi publik. PostgreSQL hanya mendengar di `127.0.0.1`.

### E. Keamanan tingkat implementasi

| Aspek | Nilai | Bukti |
|---|---|---|
| Hash kata sandi | Argon2id, memori 64 MiB, time cost 3, parallelism 1; hanya menerima hash `$argon2id$` | `src/lib/auth/password.ts:4,9-14,37-39` |
| Token sesi | 32 byte acak (base64url); disimpan sebagai SHA-256; lookup via hash | `session-constants.ts:6-7`, `session-token.ts:13-25` |
| Masa sesi | 8 jam (`SESSION_DURATION_SECONDS`) | `session-constants.ts:9` |
| Cookie | `dms_session` HttpOnly, SameSite=Lax, Secure di production atau bila `DMS_SESSION_COOKIE_SECURE`; `dms_active_role` tidak HttpOnly (hanya UI) | `session-cookies.ts` |
| Rate-limit login | 5 gagal / 10 menit → jeda 15 menit, 429 + `Retry-After`; kunci `identifier + IP`; hanya 401 yang dihitung | `login-rate-limit.ts:3-5`, `login.ts:60` |
| Pencabutan sesi | logout (satu sesi); ganti/reset sandi & nonaktif (semua sesi) | `logout.ts:24`, `local-user-passwords.ts:44,92`, `local-user-mutations.ts:328` |
| CSRF | same-origin untuk POST/PUT/PATCH/DELETE (Origin → Referer; host, `APP_URL`, `X-Forwarded-*`) | `same-origin.ts:3` |
| URL lampiran | HMAC-SHA256 `v1.<payload>.<sig>`, `timingSafeEqual`; pratinjau 15 menit, unduh 1 jam; terikat pengguna + sesi | `file-access-token.ts:27,210-216`, `document-file-access.ts:56-57` |
| Unggah | 5 MB; 8 ekstensi; 7 MIME; pasangan ekstensi↔MIME; magic bytes; sanitasi nama | `document-upload-policy.ts`, `local-upload.ts` |
| Aksi destruktif | frasa ketik-persis ditegakkan server (`BERSIHKAN`, `BERSIHKAN FILE BERKAS`); kandidat file dari data, bukan input klien; idempoten | §PB-6.3, PB-9 |
| Batas | ZIP 500 dokumen (413), file ZIP > 250 MB dilewati, pembersihan 200/permintaan | `document-zip.ts:71-72`, `pembersihan.ts:21` |

### F. Tampilan & pengalaman pengguna

- **Tema:** tiga tema (`se`, `sp`, `st`), dipilih Admin, berlaku global (`app.app_settings`), dengan sinkronisasi antartab lewat `epoch`.
- **Identitas aplikasi:** sub-judul bisa diatur.
- **Profil:** foto profil pengguna.
- **Umpan balik:** toast global dan dialog konfirmasi terpusat, dengan mode ketik-persis.
- **Formulir:** guard "perubahan belum disimpan"; unggahan tertunda dibuang saat halaman ditinggalkan.
- **Bantuan** (`/bantuan`, 354 baris): tautan di footer sidebar (`AppSidebar.tsx:271`), layout (`AppLayout.tsx:464`), dan halaman login.

### G. Catatan istilah "DMS"

Aplikasi ini bukan Document Management System klasik: tidak ada versioning, pencarian isi, atau editing kolaboratif. Yang dikelola adalah **record transaksi** (`dokumen_transaksi`) beserta status dan lampirannya. Prefiks `dms_`/`DMS_` di cookie dan env hanyalah peninggalan nama.

### H. Istilah yang sudah tidak dipakai

| Istilah usang | Pengganti / status |
|---|---|
| `BENDAHARA`, `IN_BENDAHARA_APPROVAL`, `/bendahara` | `PPSPM`, `IN_PPSPM_APPROVAL`, `/ppspm` |
| Status `ARCHIVED`; aksi `ARCHIVE`, `SKIP` | dihapus |
| `/arsiparis` | `/kasubag` |
| Tahap "Arsip Inaktif" | dibuang |
| Kategori manual Pemeliharaan/Pengadaan/Lain-lain | diganti Komponen |
| "Laporan Klasifikasi", "Pencarian Arsip global" | dihapus |
| Master Jenis Dokumen, `jenis_dokumen_id` | dihapus (0019) |
| Login email | username/NIP (0017) |
| "1 berkas terbuka per Cara Pembayaran" | 1 berkas per Cara Pembayaran × TA (0018) |
| `PATCH /api/dokumen/$id/nominal`, `POST /api/dokumen`, jalur Draf | dihapus |
| `RoleSwitcher.tsx` | perpindahan peran di `AppLayout.tsx` |
| Orkestrator submit runtime | dihapus (`fc59b44`) |

---

*Dokumen ini bahan penulisan Bab IV dan Bab V. Bila perilaku aplikasi berubah, kode di `src/` dan `drizzle/` adalah otoritas: perbarui dokumen ini mengikuti kode.*

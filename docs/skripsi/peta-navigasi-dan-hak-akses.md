# Peta Struktur Navigasi dan Hak Akses

Sumber: `src/config/navigation.ts` (isi menu), `src/lib/constants/routes.ts` (rute), guard rute (`guardRole` di layout tiap peran), `src/components/layout/AppSidebar.tsx` (penyaringan menu Ketua Tim), dan pemeriksaan peran di `src/routes/api/*`. Isi menu sama dengan Gambar 4.30a dan 4.30b.

## 1. Prinsip akses

| Aturan | Penjelasan |
|---|---|
| Satu peran = satu prefiks rute | `/pegawai`, `/ppk`, `/ppspm`, `/kasubag`, `/penanggung-jawab-kinerja`, `/admin`. Setiap layout memanggil `guardRole(<peran>)`; pengguna tanpa peran itu diarahkan ke `/forbidden`, yang belum login ke `/login`. |
| Peran aktif menentukan menu | Setelah login, menu mengikuti **peran aktif** (`NAV_CONFIG[activeRole]`). Pengguna dengan beberapa peran non-admin berpindah lewat pengalih peran di header. |
| Admin tidak mewarisi peran lain | Akun Admin terpisah; `/admin` hanya untuk peran ADMIN, dan Admin tidak bisa membuka `/ppk`, `/pegawai`, dan seterusnya. |
| Ketua Tim bukan peran | Ketua Tim adalah penugasan per kegiatan (`ketua_tim_assignments`) pada pengguna berperan Pegawai. Tiga menu grup "PJ Kegiatan" hanya tampil bila pengguna punya minimal satu penugasan. |
| Pemeriksaan ganda | Menu disembunyikan di sidebar, rute dijaga di klien, dan **API memeriksa ulang di server** (mis. `/api/ppk/inbox` menolak non-PPK dengan 403). Menyembunyikan menu hanya kenyamanan, bukan pengaman. |
| Halaman umum | `/login` (tanpa login), `/profile` dan `/bantuan` (semua pengguna yang login), `/forbidden` (halaman penolakan). |

## 2. Matriks menu × peran

Tanda: ✔ = tersedia di menu peran itu; ✔* = hanya bila punya penugasan Ketua Tim; — = tidak tersedia.

### 2.1 Menu umum (semua peran)

| Menu | Rute | Pegawai | PPK | PPSPM | KSBU | PJ Kinerja | Admin |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Dashboard | `/<peran>` | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Profil | `/profile` | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Log Aktivitas | `/<peran>/activity-log` (admin: `/admin/activity-log`) | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Bantuan (tautan footer/sidebar) | `/bantuan` | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |

Cakupan data Log Aktivitas: Admin dan PJ Kinerja melihat aktivitas **semua pengguna** (`GLOBAL_SCOPE_ROLES`); peran lain hanya melihat aktivitasnya sendiri.

### 2.2 Menu per peran

| Grup | Menu | Rute | Siapa yang bisa |
|---|---|---|---|
| MANAGEMENT | Ajukan Dokumen | `/pegawai/dokumen/aju` | Pegawai |
| MANAGEMENT | Dokumen Diajukan | `/pegawai/dokumen` (detail: `/pegawai/dokumen/$id`) | Pegawai (dokumen miliknya) |
| MANAGEMENT | Revisi Dokumen | `/pegawai/revisi` (`/pegawai/dokumen/$id/revisi`) | Pegawai |
| MANAGEMENT | Laporan Saya | `/pegawai/laporan/saya` | Pegawai |
| PJ Kegiatan | Monitoring Dokumen Tim | `/pegawai/monitoring-dokumen-tim` | Pegawai berstatus Ketua Tim ✔* |
| PJ Kegiatan | Laporan Kegiatan | `/pegawai/laporan/kegiatan` | Pegawai berstatus Ketua Tim ✔* |
| PJ Kegiatan | Pembersihan Dokumen | `/pegawai/pembersihan-dokumen` | Pegawai berstatus Ketua Tim ✔* |
| VALIDASI | Validasi Dokumen | `/ppk/inbox` | PPK |
| VALIDASI | Dokumen Tervalidasi | `/ppk/tervalidasi` | PPK |
| VALIDASI | Dokumen Tidak Valid | `/ppk/ditolak` | PPK |
| VALIDASI | Revisi Dokumen | `/ppk/revisi` (`/ppk/dokumen/$id/resubmit`) | PPK |
| PERSETUJUAN | Persetujuan Dokumen | `/ppspm/inbox` | PPSPM |
| PERSETUJUAN | Dokumen Ditolak | `/ppspm/ditolak` | PPSPM |
| PERSETUJUAN | Dokumen Selesai | `/ppspm/selesai` | PPSPM |
| MONITORING | Nominal Realisasi | `/ppk/monitoring-realisasi` dan `/ppspm/monitoring-realisasi` | PPK, PPSPM |
| PEMBERKASAN | Pengklasifikasian Dokumen | `/kasubag/inbox` | KSBU |
| PEMBERKASAN | Penambahan Dokumen | `/kasubag/penambahan-arsip` | KSBU |
| PEMBERKASAN | Berkas Terbuka | `/kasubag/berkas` (`/kasubag/berkas/$id`) | KSBU |
| PEMBERKASAN | Berkas Tertutup | `/kasubag/berkas/tertutup` | KSBU |
| PEMBERKASAN | Pembersihan Berkas | `/kasubag/pembersihan` | KSBU |
| PEMBERKASAN | Master Klasifikasi Dokumen | `/kasubag/klasifikasi` | KSBU |
| KINERJA | Laporan Kinerja | `/penanggung-jawab-kinerja/laporan-kinerja` | PJ Kinerja |
| MANAJEMEN SISTEM | Master User | `/admin/master-data/user` | Admin |
| MANAJEMEN SISTEM | Departemen Fungsi | `/admin/master-data/fungsi` | Admin |
| MANAJEMEN SISTEM | Master Kegiatan | `/admin/master-data/kegiatan` | Admin |
| MANAJEMEN SISTEM | Master Komponen | `/admin/master-data/komponen` | Admin |
| REFERENSI DOKUMEN | Jenis Permintaan | `/admin/master-data/jenis` | Admin |
| REFERENSI DOKUMEN | Kategori Permintaan | `/admin/master-data/kategori` | Admin |
| REFERENSI DOKUMEN | Detail Permintaan | `/admin/master-data/detail` | Admin |
| REFERENSI DOKUMEN | Kelengkapan Dokumen | `/admin/master-data/kelengkapan` | Admin |
| SYSTEM | Pengaturan Aplikasi | `/admin/settings` | Admin |

### 2.3 Ringkasan jumlah menu

| Peran | Menu di sidebar | Rincian |
|---|:-:|---|
| Pegawai | 10 (7 tanpa Ketua Tim) | Dashboard, 4 Management, 3 PJ Kegiatan*, Profil, Log Aktivitas |
| PPK | 8 | Dashboard, 4 Validasi, Nominal Realisasi, Profil, Log Aktivitas |
| PPSPM | 7 | Dashboard, 3 Persetujuan, Nominal Realisasi, Profil, Log Aktivitas |
| KSBU | 9 | Dashboard, 6 Pemberkasan, Profil, Log Aktivitas |
| PJ Kinerja | 4 | Dashboard, Laporan Kinerja, Profil, Log Aktivitas |
| Admin | 12 | Dashboard, 4 Manajemen Sistem, 4 Referensi Dokumen, Profil, Log Aktivitas, Pengaturan Aplikasi |

## 3. Akses di sisi API (penegakan di server)

| Fungsi | Endpoint | Peran yang diizinkan |
|---|---|---|
| Inbox PPK, daftar PPK | `/api/ppk/*` | PPK |
| Inbox/daftar PPSPM | `/api/ppspm/*` | PPSPM |
| Pemberkasan | `/api/kasubag/*` | KSBU |
| Laporan Kinerja (material + non-material) | `/api/laporan/kinerja?scope=laporan_kinerja` | PJ Kinerja saja |
| Monitoring Nominal Realisasi (material saja) | `/api/laporan/kinerja` | PJ Kinerja, PPK, PPSPM |
| Laporan Kegiatan, Monitoring Dokumen Tim | `/api/laporan/kegiatan` | Pegawai yang terdaftar Ketua Tim (data dibatasi pada kegiatan yang dipimpinnya) |
| Pembersihan Dokumen non-material | `/api/pembersihan-dokumen` | Pegawai berstatus Ketua Tim, pada kegiatannya |
| Laporan Saya | `/api/laporan/saya` | Pegawai (dokumennya sendiri) |
| Log Aktivitas (semua pengguna) | `/api/activity-log?scope=all` | Admin, PJ Kinerja |
| Ubah data master | `/api/master-*` (POST/PUT/DELETE) | Admin |
| Baca data master (untuk dropdown formulir) | `/api/master-*` (GET) | Semua pengguna yang login |
| Pengaturan aplikasi dan tema | `/api/settings/*` | Ubah: Admin. Baca (GET): tanpa pembatasan peran di handler |

## 4. Catatan untuk penulisan dan sidang

1. Menu Ketua Tim ada **dua lapis pengaman**: disaring di sidebar (`KETUA_TIM_ONLY_NAV_IDS`), dan halamannya menampilkan keadaan kosong bila pengguna bukan Ketua Tim; API membatasi data pada kegiatan yang dipimpin. Ini mendukung penjelasan Ketua Tim sebagai *capability*, bukan role.
2. Monitoring Nominal Realisasi dipakai bersama PPK dan PPSPM, sedangkan Laporan Kinerja (yang memuat dokumen non-material) khusus PJ Kinerja. Ini dibedakan di server lewat parameter `scope`.
3. Pada deck, KF-08 menyebut Ketua Tim sebagai salah satu pengguna total nominal. Di menu, Ketua Tim memperoleh nominal lewat Laporan Kegiatan, bukan lewat menu "Nominal Realisasi" milik PPK dan PPSPM. Samakan rumusan di buku bila perlu.
4. Pemeriksaan peran di `guardRole` dan pengalihan ke `/forbidden` terjadi di klien; kekuatan aksesnya ada pada pemeriksaan ulang di API (KNF-06).

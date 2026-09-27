# Hasil Validasi Activity Diagram Sistem Usulan terhadap Kode

**Sumber:** `docs/Validasi Activity Diagram Sistem Usulan - untuk Claude Code.md.md`
**Metode:** membaca kode sumber langsung (route handler, `fsm.ts`, skema Zod, komponen form). Tidak ada perubahan kode maupun diagram — ini murni laporan audit.

---

## Diagram A — Gambar 4.13 Pengajuan Dokumen

### SESUAI

| Langkah | Bukti |
|---|---|
| 1. Tanggal tidak boleh melewati hari ini | Dicek di **dua sisi**. Klien: `aju.tsx:431-438` (`handleNextFromInformation`). Server: Zod `.refine` di `schemas/dokumen.ts:68-77` |
| 2. Deteksi Ketua Tim dan badge | Dipanggil saat kegiatan dipilih (`aju.tsx:348`), lewat `checkChairmanStatus` `aju.tsx:487-510` → `GET /api/users/me/is-ketua-tim/$kegiatanId` (lookup `ketua_tim_assignments`) |
| 3. Decision material / non-material | `StepKarakteristik`. Material: Komponen → Jenis → Kategori → Detail (`aju.tsx:402-407`). Detail dilewati bila `!kategoriHasDetail` (`aju.tsx:407`). Non-material: `StepNamaDokumen` (teks bebas) |
| 4. Checklist exact-match 6 kolom, identik klien–server | Klien: API memfilter `kegiatan_id` + `is_ketua_tim` (`KelengkapanChecklist.tsx:122-127`), lalu 4 kolom lain lewat `matchesCurrentSelection` (`:59-72`). Server: `buildRequiredKelengkapanCondition` (`local-submit-drizzle-adapter.ts:159-173`) memakai 6 kolom yang sama dengan aturan NULL-match yang sama. Perbaikan di commit `d45bcd9` memang sudah ada di kode saat ini. Lampiran tambahan di luar checklist didukung (`user-custom-*`) |
| 5. Decision kelengkapan wajib (loop balik) | Tombol "Lanjut" dari langkah Unggah nonaktif selama `missingRequired.length > 0` (`aju.tsx:410-411`, `:1029`) |
| 7. Tombol Ajukan nonaktif | `submitDisabled={submitting \|\| missingRequired.length > 0 \|\| lampiranUrls.length === 0}` (`aju.tsx:1055`) |
| 8. Validasi ulang di server | Semua tahap ada (lihat urutan persisnya di bagian TIDAK SESUAI #3). Pesan error dikembalikan sebagai 400/403 |
| 9. Material: satu transaksi, lalu file dipindah | `executeLocalSubmitWritePlan` (`local-submit-write-bridge.ts:366-397`): insert `DRAFT` (`:371`, `:421`) → update status hasil `transition(DRAFT, SUBMIT, PEGAWAI)` (`:452-455`, `:377`) → `log_aktivitas` `SUBMIT` (`:383`), semuanya dalam satu `db.transaction`. File baru dipindah **setelah** transaksi commit (`submit.ts:113` lalu `:118-136`) |
| 9. Non-material tanpa `fsm.ts` | `buildLocalSubmitTransitionPlan` (`local-submit-write-bridge.ts:439-450`) langsung membuat `TERSIMPAN` dengan aksi `STORE` tanpa memanggil `transition()`. Klaim "satu-satunya jalur di luar FSM" juga terbukti: grep semua penulisan `status:` di `src/routes/api` dan `src/lib` tidak menemukan jalur lain |
| `DRAFT` transien | Hanya hidup di dalam transaksi submit (insert `DRAFT` lalu langsung update di transaksi yang sama). Tidak ada endpoint simpan draf |
| Nominal `> 0` di server | `validateNominalForMaterial`: `nominalRealisasi <= 0` → error (`schemas/dokumen.ts:139-148`). Catatan: Zod-nya sendiri `.min(0)` (`:79-83`), tetapi pengecekan `> 0` dilakukan tepat sesudahnya (`submit.ts:430-436`) |

### TIDAK SESUAI

1. **Urutan isian di langkah 1–3.** Diagram menulis "isi fungsi dan tanggal, lalu pilih kegiatan". Di kode, urutan tampilan pada langkah "Informasi Dasar" adalah Fungsi → Kegiatan → Karakteristik → rantai (Komponen/Jenis/Kategori/Detail) atau Nama Dokumen → **Tanggal di paling akhir** (`aju.tsx:826-960`; `StepFungsiTanggal` dirender dua kali, `showTanggal={false}` di atas dan `showFungsi={false}` di bawah). Validasi tanggal baru berjalan saat menekan "Lanjut".

2. **Nominal realisasi tidak berada di cabang material langkah 3.** Nominal diisi di langkah mayor ke-2, "Unggah Dokumen", bersama lampiran (`aju.tsx:73-77`, "Unggah lampiran serta lengkapi nominal atau keterangan detail"). Gerbangnya `canAdvanceFromStep6` (`aju.tsx:410-421`). Untuk non-material, Keterangan Detail juga diisi di langkah ini. Sebaiknya dipindah ke aksi setelah checklist atau digabung dengan aksi unggah.

3. **Urutan validasi server (langkah 8) sedikit berbeda dari aturan global #4.** Urutan sebenarnya:
   1. Zod (`submit.ts:421`)
   2. Nominal > 0 (`:430`)
   3. Komponen (material) atau Nama Dokumen (non-material) (`:440`)
   4. **Autentikasi dan peran Pegawai** (`:66-75`) — tidak ada di diagram
   5. **Keberadaan file** di area pending (`preflightSubmitFiles`, `:77-94`)
   6. Lampiran ≥ 1 (`local-submit-write-bridge.ts:269-271`) — muncul setelah cek file
   7. **Kegiatan ada** (`:273-276`) — tidak ada di diagram
   8. Kelengkapan wajib (`:279-304`)
   9. Penugasan Ketua Tim (`:306-318`, hasilnya 403)
   10. Transaksi
   11. Pindah file

   Urutan intinya (skema → nominal/rantai → file → kelengkapan → Ketua Tim → transaksi → pindah file) **benar**. Hanya "validasi rantai" di server yang jauh lebih tipis dari klaimnya (lihat poin 4).

4. **"Validasi rantai (jenis/kategori/detail)" di server hanya mengecek keberadaan `komponenId`.** `validateWorkflowChainForCharacteristic` (`schemas/dokumen.ts:158-180`) hanya mewajibkan `komponenId` untuk material dan `namaDokumen` untuk non-material. Server **tidak** mewajibkan Jenis/Kategori, tidak memeriksa konsistensi induk–anak (misalnya kategori benar milik jenis tersebut), dan tidak memeriksa aturan "Detail boleh dilewati hanya bila Kategori simpul daun". Semua itu hanya ditegakkan di UI (`aju.tsx:405-407`). Di diagram, sebaiknya labelnya diganti menjadi "validasi nominal & Komponen/Nama Dokumen", atau rantai lengkap ditandai sebagai validasi sisi klien.

5. **Posisi langkah 6 (unggah ke area pending).** File diunggah ke `POST /api/upload` **seketika** saat dipilih di checklist (`FileUploadButton.tsx:159`, `upload.ts:99-126`, path pending di `local-upload.ts:207`). Artinya unggah terjadi **sebelum** decision "kelengkapan terpenuhi?", sebagai bagian dari loop unggah. Di diagram, langkah 6 seharusnya berada di dalam loop 4–5, bukan sesudahnya.

### Ada di kode tetapi tidak tergambar

- **Cek non-material di server bersyarat.** Kelengkapan non-material di server hanya dibaca bila `jenisPermintaanId` dikirim (`shouldReadRequiredKelengkapan`, `local-submit-write-bridge.ts:515-517`), dan klien tidak pernah mengirimnya untuk non-material. Jadi non-material praktis tidak punya checklist wajib, tetapi tetap wajib ≥ 1 lampiran. Konsisten dengan klien, yang juga melewati checklist untuk non-material (`KelengkapanChecklist.tsx:113-118`).
- **Judul dibuat otomatis** dengan format `{nama simpul terdalam} {tahun} {nama pegawai}` (`:413`, `resolveLocalSubmitLeafName` `:458-502`). `tahun` diturunkan dari tanggal (`aju.tsx:335`).
- **Gagal pindah file setelah commit.** Bila pemindahan file gagal, respons berstatus 500 dengan `compensationRequired: true` (`submit.ts:124-135`), tetapi dokumen **sudah tersimpan** berstatus `IN_PPK_VALIDATION`. Modul `submit-db-file-compensation.ts` dan `submit-runtime-orchestrator.ts` tidak di-import oleh route mana pun. Ini sebaiknya dijadikan catatan batasan, tidak perlu digambar.
- **Tinjauan dengan dialog konfirmasi.** Langkah 3 "Tinjauan" memakai dialog konfirmasi (`aju.tsx:1163-1167`). Ini cocok dengan "Pegawai meninjau lalu menekan Ajukan".

### Kesimpulan A: **perlu revisi kecil**

Yang perlu diubah:
- (a) urutan tanggal dan posisi nominal/keterangan;
- (b) pindahkan "unggah ke pending" ke dalam loop sebelum decision kelengkapan;
- (c) ubah label validasi rantai di server.

Poin kritis diagram ini (exact-match 6 kolom, jalur `TERSIMPAN` yang melewati FSM, urutan commit lalu pindah file, nominal > 0) semuanya **SESUAI**.

---

## Diagram B — Gambar 4.14 Persetujuan Berjenjang dan Revisi

### SESUAI

| Langkah | Bukti |
|---|---|
| 1–2. Kotak masuk PPK | `ppk/inbox.ts:32` (filter `IN_PPK_VALIDATION`) |
| 3. PPK setuju (#2) | `ppk/dokumen/$id/approve.ts`: cek status `:75-79` → `transition(...,'APPROVE','PPK')` `:82` → transaksi update + log `PPK_APPROVE` `:88-112` |
| 3. PPK tolak (#3), catatan 10–2000 | `ppk/dokumen/$id/reject.ts`: Zod `rejectDokumenSchema` `:43-49` (`schemas/dokumen.ts:103-105`, min 10 max 2000) → `transition(...,'REJECT','PPK','USER')` `:83` → `revision_notes` + log `PPK_REJECT` `:88-113` |
| 3. Pegawai `RESUBMIT` (#6) dengan guard `revision_target = USER` | `dokumen.$id.submit.ts`: guard di handler `:84-88`, pemilik dokumen `:75`, `transition(...,'RESUBMIT',...)` `:96` (FSM juga mengecek, `fsm.ts:104-109`) → kembali ke `IN_PPK_VALIDATION` |
| 4. Kotak masuk PPSPM | `ppspm/inbox.ts:25` |
| 5. PPSPM setuju (#4) dengan penjaga persetujuan ganda | `ppspm/dokumen/$id/approve.ts`: cek status masih `IN_PPSPM_APPROVAL` `:58` **dan** cek belum ada log `PPSPM_APPROVE` `:60-75` → transisi `:77` → `COMPLETED` |
| 5. PPSPM tolak (#5), catatan 10–2000 | `ppspm/dokumen/$id/reject.ts`: Zod yang **sama** (`:34`) → `transition(...,'REJECT','PPSPM','PPK')` `:81` → `revision_target = PPK`, log `PPSPM_REJECT` `:84-109`. Jadi validasi catatan ada di **kedua** titik penolakan |
| 6. `RESUBMIT_PPK` (#7) langsung ke PPSPM | **Terkonfirmasi.** `ppk/resubmit/$id.ts:400` → FSM `NEED_REVISION:RESUBMIT_PPK` → `IN_PPSPM_APPROVAL` (`fsm.ts:56-61`). Tidak melewati `IN_PPK_VALIDATION`. Guard `revision_target = PPK` ada di handler `:396-398` dan di FSM `:110-115` |
| 6. `KEMBALIKAN` (#8) dengan catatan otomatis | **Terkonfirmasi.** `ppk/kembalikan/$id.ts`: konstanta `KEMBALIKAN_CATATAN = 'Dikembalikan ke pegawai oleh PPK'` (`:10`), tanpa body dan tanpa form catatan. Guard `status = NEED_REVISION && revision_target = PPK` (`:57`) → `revision_target → USER` → log `PPK_KEMBALIKAN` (`:84-90`) |
| Aturan "`REJECT` ditentukan status asal" | `fsm.ts:19-68`: kunci transisi adalah `status:aksi`. Parameter `revisionTarget` pada `REJECT` hanya divalidasi bernilai `USER`/`PPK` (`:95-102`) dan tidak memengaruhi hasil |
| "Kembalikan hanya setelah PPSPM menolak" | Benar. `KEMBALIKAN` mensyaratkan `revision_target = PPK`, dan nilai itu hanya diset oleh PPSPM `REJECT` |
| Guard `revision_target` dicek di handler, bukan hanya UI | Ya, di ketiga handler: `dokumen.$id.submit.ts:84`, `ppk/resubmit/$id.ts:396`, `ppk/kembalikan/$id.ts:57` |

### TIDAK SESUAI / PERLU KONFIRMASI

1. **TIDAK SESUAI — setelah `KEMBALIKAN`, Pegawai tidak lagi membaca catatan penolakan PPSPM.** Handler **menimpa** `revision_notes` dengan teks generik (`ppk/kembalikan/$id.ts:74`). Halaman revisi Pegawai menampilkan `dok.revision_notes` (`pegawai/dokumen/$id/revisi.tsx:650`, `pegawai/revisi.tsx:199`). Akibatnya, pada jalur "dikembalikan ke Pegawai" yang dibaca Pegawai adalah "Dikembalikan ke pegawai oleh PPK". Alasan asli PPSPM hanya tersisa di tab **Riwayat** (catatan log `PPSPM_REJECT`). Label merge "Pegawai membaca catatan penolakan" di diagram jadi kurang tepat untuk jalur ini. Pilihannya: ubah labelnya menjadi "membaca catatan pengembalian (alasan PPSPM ada di riwayat)", atau jadikan ini temuan untuk diperbaiki di kode.

2. **PERLU KONFIRMASI — penjaga persetujuan ganda PPSPM tidak atomik.** Cek status (`:58`) dan cek log (`:60-75`) berjalan **di luar** transaksi. `UPDATE` hanya memakai `WHERE id = ...` (`:91`), tanpa `AND status = 'IN_PPSPM_APPROVAL'` dan tanpa row lock. Secara logika guard-nya ada, jadi diagram SESUAI. Namun dua klik yang benar-benar bersamaan secara teoretis bisa lolos keduanya; akibatnya hanya log ganda, karena status akhirnya sama-sama `COMPLETED`. Pola cek-lalu-update yang sama dipakai di semua handler transisi. Kalau skripsi mengklaim "tahan kondisi balapan" untuk persetujuan, klaim itu belum didukung kode.

3. **PERLU KONFIRMASI — "memperbaiki lalu mengajukan ulang" adalah dua request terpisah dan tidak divalidasi ulang.**
   - **Pegawai:** `PATCH /api/dokumen/:id` (lampiran + nominal), lalu `POST /api/dokumen/:id/submit` (`revisi.tsx:310-322`). Keduanya bukan satu transaksi. Handler resubmit **tidak** memvalidasi ulang kelengkapan wajib, jumlah lampiran, maupun nominal > 0. `PATCH` hanya mensyaratkan nominal `.min(0)` (`schemas/dokumen.ts:54`), jadi nol diterima server.
   - **PPK (`RESUBMIT_PPK`):** Zod hanya dijalankan **bila** `lampiranUrls` dikirim (`ppk/resubmit/$id.ts:357-365`, juga `PATCH :236-244`). Tanpa lampiran, `nominalRealisasi` dari body langsung ditulis tanpa validasi Zod. Selain itu, `revision_notes` tidak dikosongkan saat `RESUBMIT_PPK` (`:425-430`), jadi catatan PPSPM lama masih melekat.

   Ini bukan salah model diagram, tetapi celah pada validasi server. Bila Bab IV mengklaim "setiap endpoint JSON divalidasi Zod", klaim itu **tidak berlaku penuh** untuk endpoint `ppk/resubmit`.

### Ada di kode tetapi tidak tergambar

- **Nama aksi di `log_aktivitas` berbeda dari nama aksi FSM:** `PPK_APPROVE`, `PPK_REJECT`, `PPSPM_APPROVE`, `PPSPM_REJECT`, `RESUBMIT`, `RESUBMIT_PPK`, `PPK_KEMBALIKAN`. Perlu diperhatikan kalau diagram atau tabel menyebut nama log.
- **Guard `KEMBALIKAN` hanya di handler, tidak di FSM.** `fsm.ts:151-152` hanya mengecek peran PPK dan status `NEED_REVISION`, bukan `revision_target`. Argumen `'USER'` di `kembalikan/$id.ts:63` diabaikan. Tidak ada salah perilaku karena handler sudah mengecek, tetapi klaim "semua guard transisi ada di `fsm.ts`" tidak berlaku untuk `KEMBALIKAN`.
- **PPK bisa "simpan tanpa transisi".** `PATCH /api/ppk/resubmit/:id` menyimpan lampiran/nominal tanpa mengirim dokumen. Ini sub-langkah dari "PPK memperbaiki".
- **Non-material ditolak di endpoint resubmit Pegawai** (`dokumen.$id.submit.ts:90-94`). Wajar, karena non-material tidak masuk alur ini.
- **Cek pemilik dokumen** (`created_by === session.user.id`) pada resubmit dan edit Pegawai.

### Kesimpulan B: **perlu revisi kecil**

Semua poin kritis (#7 langsung ke PPSPM, #8 catatan otomatis, guard `revision_target` di handler, catatan 10–2000 di kedua titik, penjaga persetujuan ganda) **SESUAI**. Satu-satunya perubahan pada diagram adalah label "Pegawai membaca catatan penolakan" pada jalur setelah `KEMBALIKAN`. Poin 2 dan 3 di atas lebih tepat dijadikan catatan batasan atau temuan kode daripada perubahan diagram.

---

## Diagram C — Gambar 4.16 Pemberkasan Dokumen

### SESUAI

| Langkah | Bukti |
|---|---|
| 1a. Pintu masuk dokumen `COMPLETED` | Kotak masuk KSBU: `kasubag/inbox.ts:28-29` (`COMPLETED` dan belum ada di `berkas_arsip_item`). Endpoint `POST /api/kasubag/dokumen/:id/archive` menolak dokumen yang bukan `COMPLETED` (`dokumen.$id.archive.ts:103`). Peran: `KEPALA_SUB_BAGIAN_UMUM` (`:57`) |
| 1b. Pintu masuk dokumen manual | `POST /api/kasubag/manual-arsip` → `createManualArsipRecord` (`lib/manual-arsip.ts:199-290`). Field sesuai UC-14: fungsi, kegiatan, komponen (hierarki divalidasi `:934-962`), nama, tanggal, keterangan wajib, nominal wajib (`schemas/manual-arsip.ts:91-107`) |
| 2. Keduanya wajib memilih cara pembayaran + tahun anggaran | `klasifikasi_id` + `tahun_anggaran` wajib di Zod kedua jalur (`dokumen.$id.archive.ts:64-65`, `schemas/manual-arsip.ts:101-102`) |
| 3–4. Get-or-create per pasangan | `getOrCreateOpenBerkasForKlasifikasi` (`berkas-arsip-service.ts:243-266`), dipakai **kedua** jalur (`archive.ts:108`, `manual-arsip.ts:266`) |
| 4. Tiga cabang decision | `resolveExistingBerkasForKlasifikasi` (`:808-833`): ada yang `OPEN` → dipakai; ada tetapi tidak ada yang `OPEN` → error `BERKAS_KLASIFIKASI_CLOSED` "Berkas untuk Cara Pembayaran ini TA X sudah ditutup" (`:826-831`); belum ada → `insertOpenBerkasWithActivity` (`:835-858`, status `OPEN` + riwayat `BERKAS_DIBUKA`) |
| Unique index komposit di basis data | `drizzle/0018_berkas_tahun_anggaran.sql:28`: `UNIQUE (klasifikasi_id, tahun_anggaran)`. Satu dokumen hanya bisa masuk satu berkas: `0007:190` dan `:192` |
| Prasyarat klasifikasi aktif dan simpul daun, dicek di server | `validateOperationalKlasifikasiSelection` (`berkas-klasifikasi-eligibility.ts:55-87`): `KLASIFIKASI_INACTIVE` (`:68`), `KLASIFIKASI_PARENT` (`:75`). Dijalankan di awal get-or-create (`berkas-arsip-service.ts:248`) |
| 5. Tambah dokumen dan catat riwayat | `addWorkflowDocumentToOpenBerkas` (`:287-316`, riwayat `DOKUMEN_PERSETUJUAN_DIKLASIFIKASIKAN`) dan `addManualDocumentToOpenBerkas` (`:318-347`, riwayat `DOKUMEN_MANUAL_DITAMBAHKAN`) ke `berkas_arsip_activity`. Pada kedua jalur UI, get-or-create + tambah item + riwayat berada dalam **satu** `db.transaction` |
| 7–9. Tutup berkas: ≥ 1 isi, Nomor SPM, Masa Simpan, tanpa konfirmasi ketik-persis | `POST /api/kasubag/berkas/:id/close`. Zod `closeBerkasMetadataSchema` (`schemas/berkas-arsip.ts:49-60`): `nomor_spm` wajib, `retensi_aktif` ("Masa Simpan Minimal") wajib. Service `closeBerkasArsip` (`berkas-arsip-service.ts:349-395`): harus `OPEN` → `BERKAS_EMPTY` bila isi < 1 (`:362-365`) → `UPDATE ... WHERE status_berkas = 'OPEN'` (atomik) → riwayat `BERKAS_DITUTUP`. Klien: toast "Data belum lengkap" (`kasubag/berkas/$id.tsx:327-333`). Tidak ada konfirmasi ketik-persis |
| Pembersihan file berkas di luar cakupan | Benar. Ada di `.../lifecycle` dan `pembersihan-*`, terpisah dari alur ini |

### TIDAK SESUAI / PERLU KONFIRMASI

1. **PERLU KONFIRMASI — penanganan race condition get-or-create tidak efektif di jalur yang dipakai UI.** Kodenya punya logika tangkap `23505` lalu baca ulang (`berkas-arsip-service.ts:258-264`, `isUniqueConflict` `:1007-1014`). Namun di kedua jalur UI (`archive.ts:106-119`, `manual-arsip.ts:210`) get-or-create berjalan **di dalam** `db.transaction`. Di PostgreSQL (driver `pg`, `db/client.ts:4-5`), transaksi yang terkena unique violation berstatus *aborted*, sehingga query baca-ulang ikut gagal dan request yang kalah balapan mendapat 500 "Gagal mengklasifikasikan dokumen". Integritas data tetap terjaga (tidak mungkin ada berkas ganda, karena dijamin unique index), tetapi request yang kalah **tidak** otomatis bergabung ke berkas pemenang. Jalur retry hanya efektif di `POST /api/kasubag/berkas/open`, yang non-transaksional. Catatan: ini kesimpulan dari membaca kode ditambah perilaku standar PostgreSQL, **belum diuji dengan menjalankannya**. Klaim KNF-11 "tahan kondisi balapan" sebaiknya dirumuskan sebagai "tidak mungkin terbentuk berkas ganda", bukan "semua request konkuren berhasil".

2. **TIDAK SESUAI (ringan) — urutan pintu masuk manual.** Diagram: isi data dan **unggah dokumen** → pilih cara bayar/tahun → get-or-create. Kode: cara bayar dan tahun ada di **form yang sama** dengan data dokumen (satu submit). Server membuat rekaman manual, lalu menjalankan get-or-create, lalu menambah item. **Setelah itu baru** lampiran diunggah lewat request terpisah `POST /manual-arsip/:id/attachments` (`penambahan-arsip.tsx:1384-1440`). Lampiran juga **opsional**: nol lampiran diterima (`:1400-1413`). Kalau lampiran gagal diunggah, dokumen manual tetap sudah masuk berkas.

3. **TIDAK SESUAI (struktur) — decision langkah 6 "pengelompokan selesai?" tidak ada di kode.** Menambah dokumen (langkah 5) dan menutup berkas (langkah 7–9) adalah dua operasi yang benar-benar terpisah:
   - endpoint berbeda (`/dokumen/:id/archive` atau `/manual-arsip` vs `/berkas/:id/close`);
   - halaman berbeda (detail dokumen di kotak masuk KSBU atau Penambahan Arsip vs halaman detail berkas `kasubag/berkas/$id.tsx`);
   - penutupan bisa dilakukan kapan saja.

   Decision langkah 6 adalah keputusan manusia (KSBU), bukan logika sistem. Penyederhanaan ini masih bisa dipertanggungjawabkan kalau diberi catatan, misalnya "penutupan berkas adalah aktivitas terpisah yang dapat dipicu kapan pun dari halaman berkas". Alternatif yang lebih jujur: pecah menjadi dua aktivitas, atau beri penanda *interruptible region* / *start* kedua untuk "KSBU membuka berkas terbuka".

4. **Ringan — cabang "isi < 1" di UI.** Untuk berkas kosong, UI tidak memunculkan pesan "data belum lengkap". Form atau aksi tutup tidak ditampilkan sama sekali (`isBerkasEmptyForClose`, `$id.tsx:2123`, dan `:324-325`). Pesan hanya muncul dari server ("Berkas kosong tidak dapat ditutup"). Loop "kembali ke langkah 7" hanya relevan untuk Nomor SPM dan Masa Simpan.

### Ada di kode tetapi tidak tergambar

- **Riwayat `BERKAS_DIBUKA`** dicatat saat berkas baru dibuat (`:835-858`). Bisa digabung ke "sistem membuat berkas baru".
- **Konflik data:** error `BERKAS_KLASIFIKASI_CONFLICT` bila ada > 1 berkas `OPEN` untuk pasangan yang sama (`:815-820`). Cabang pengaman, boleh diabaikan.
- **Error `CONFLICT` "Dokumen sudah terhubung ke berkas"** bila dokumen yang sama diklasifikasi dua kali (`insertBerkasItemSafely` `:860-874`).
- **Field tambahan saat tutup:** "Tanggal Tutup Berkas" opsional (`closed_at`), dan `masa_aktif_berakhir` dihitung otomatis dari Masa Simpan (`buildCloseBerkasPlan` `:521-541`).
- **Tutup berkas tidak dalam satu transaksi dengan riwayat.** Update status dan insert riwayat `BERKAS_DITUTUP` adalah dua panggilan terpisah (repository default, sekitar `:684` dan `:749`). Status tetap aman karena memakai `UPDATE` bersyarat `OPEN`, tetapi riwayat bisa hilang bila insert kedua gagal.
- **Endpoint tanpa pemanggil di UI:** `POST /api/kasubag/berkas/open` dan `POST /api/kasubag/berkas/:id/items` ada tetapi tidak dipanggil dari UI. Jangan dijadikan dasar diagram.
- **Status dokumen tidak berubah setelah dimasukkan ke berkas** (tetap `COMPLETED`). Penandanya adalah baris di `berkas_arsip_item`. Ini sesuai dengan tabel status di spesifikasi.

### Kesimpulan C: **perlu revisi kecil hingga sedang**

- **Wajib:** beri catatan atau pisahkan aktivitas "tambah dokumen ke berkas" dari "tutup berkas" (langkah 6).
- **Wajib:** perbaiki urutan unggah pada pintu manual: lampiran diunggah setelah pemberkasan dan bersifat opsional.
- **Untuk teks Bab IV:** sesuaikan klaim "tahan kondisi balapan".

Aturan get-or-create, unique index komposit, guard klasifikasi daun + aktif, dan syarat tutup berkas semuanya **SESUAI**.

---

## Ringkasan

| Diagram | Kesimpulan | Yang harus direvisi |
|---|---|---|
| A — 4.13 | Revisi kecil | Urutan tanggal/nominal; posisi "unggah ke pending"; label validasi rantai di server |
| B — 4.14 | Revisi kecil | Label catatan yang dibaca Pegawai setelah `KEMBALIKAN` |
| C — 4.16 | Revisi kecil–sedang | Pisahkan/beri catatan penutupan berkas; urutan dan sifat opsional lampiran manual |

Temuan yang menyangkut **kode**, bukan diagram (belum diubah):
- `KEMBALIKAN` menimpa catatan PPSPM.
- Resubmit Pegawai dan PPK tidak memvalidasi ulang kelengkapan dan nominal > 0.
- Endpoint `ppk/resubmit` hanya menjalankan Zod bila ada `lampiranUrls`.
- Penjaga persetujuan ganda tidak atomik.
- Retry race get-or-create di dalam transaksi tidak efektif.

### Cara memverifikasi sendiri
- [ ] Buka `src/routes/api/ppk/kembalikan/$id.ts:74`. Pastikan `revisionNotes: KEMBALIKAN_CATATAN` menimpa catatan lama (temuan B-1).
- [ ] Di aplikasi: tolak dokumen sebagai PPSPM dengan catatan tertentu → Kembalikan sebagai PPK → buka halaman revisi Pegawai. Catatan yang tampil seharusnya "Dikembalikan ke pegawai oleh PPK".
- [ ] Buka `src/lib/schemas/dokumen.ts:158-180`. Pastikan server hanya mewajibkan `komponenId` untuk material (temuan A-4).
- [ ] Buka `src/routes/pegawai/dokumen/aju.tsx:826-960`. Periksa urutan tampilan: Tanggal ada di bawah rantai Komponen/Jenis/Kategori (temuan A-1).
- [ ] Buka `src/routes/api/kasubag/dokumen.$id.archive.ts:106-119` bersama `src/lib/archive/berkas-arsip-service.ts:258-264`. Pastikan retry `23505` berjalan di dalam `db.transaction` (temuan C-1).
- [ ] Di aplikasi: buat dokumen manual tanpa lampiran di Penambahan Arsip. Dokumen seharusnya tetap tersimpan dan masuk berkas (temuan C-2).

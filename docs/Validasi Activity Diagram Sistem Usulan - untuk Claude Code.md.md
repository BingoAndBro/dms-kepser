# Spesifikasi Activity Diagram Sistem Usulan — untuk Validasi Terhadap Kode

**Tujuan dokumen ini:** memberi cukup detail logis (aktor, aksi, guard/decision, transisi, status objek) dari tiga *activity diagram* "sistem usulan" yang sudah digambar di Miro, agar dapat divalidasi terhadap kode aplikasi sungguhan. Untuk setiap langkah, tolong periksa: (a) apakah aksi/urutan ini memang ada di kode, (b) file/handler mana yang mengimplementasikannya, (c) apakah guard/kondisinya sama persis dengan yang tertulis di sini, dan (d) apakah ada langkah di kode yang **tidak tergambar** di sini atau sebaliknya.

Tandai setiap temuan dengan salah satu dari: `SESUAI`, `TIDAK SESUAI` (jelaskan bedanya + file:baris), `TIDAK DITEMUKAN` (kode tidak melakukan ini), atau `PERLU KONFIRMASI` (ambigu/butuh keputusan penulis).

Konteks tumpukan teknologi: Node.js + Nitro, TanStack Start (`createFileRoute(...).server.handlers`, **bukan** `createServerFn`), Drizzle ORM, PostgreSQL, validasi Zod pada setiap endpoint berpayload JSON (GET dan upload FormData dikecualikan secara wajar). Modul transisi status terpusat (`fsm.ts`) adalah fungsi murni — **tidak menulis log sendiri**; route handler yang menulis `log_aktivitas`/`audit_log` dalam transaksi yang sama setelah `fsm.ts` mengonfirmasi transisi valid.

---

## Aturan bisnis global yang relevan untuk ketiga diagram

Gunakan ini sebagai acuan silang saat memvalidasi setiap decision node di bawah — jangan hanya cocokkan nama aksinya, tapi juga guard-nya.

1. **Tabel transisi status dokumen (8 transisi, 6 aksi):**

   | # | Status asal | Aksi | Aktor | Status tujuan | `revision_target` baru | Prasyarat |
   |---|---|---|---|---|---|---|
   | 1 | `DRAFT` | `SUBMIT` | Pegawai | `IN_PPK_VALIDATION` | — | Lampiran ≥1; kelengkapan wajib terpenuhi; Komponen terisi; nominal > 0; bila mengaku Ketua Tim, penugasannya ada |
   | 2 | `IN_PPK_VALIDATION` | `APPROVE` | PPK | `IN_PPSPM_APPROVAL` | — | — |
   | 3 | `IN_PPK_VALIDATION` | `REJECT` | PPK | `NEED_REVISION` | `USER` | catatan wajib |
   | 4 | `IN_PPSPM_APPROVAL` | `APPROVE` | PPSPM | `COMPLETED` | — | belum pernah disetujui PPSPM (penjaga persetujuan ganda) |
   | 5 | `IN_PPSPM_APPROVAL` | `REJECT` | PPSPM | `NEED_REVISION` | `PPK` | catatan wajib |
   | 6 | `NEED_REVISION` | `RESUBMIT` | Pegawai | `IN_PPK_VALIDATION` | — | `revision_target = USER` |
   | 7 | `NEED_REVISION` | `RESUBMIT_PPK` | PPK | `IN_PPSPM_APPROVAL` | — | `revision_target = PPK` |
   | 8 | `NEED_REVISION` | `KEMBALIKAN` | PPK | `NEED_REVISION` | `USER` | `revision_target = PPK`; catatan dibuat **otomatis** |

   Poin penting: `REJECT` tujuannya ditentukan oleh **status asal**, bukan oleh `revision_target`. Yang bergantung pada `revision_target` hanyalah pilihan `RESUBMIT` vs `RESUBMIT_PPK`/`KEMBALIKAN`.

2. **Jalur non-material** `DRAFT → TERSIMPAN` (aksi log `STORE`) adalah **satu-satunya** jalur dokumen di luar modul transisi status terpusat — disengaja, bukan bug.

3. **Checklist kelengkapan** dicocokkan *exact-match* pada **enam kolom**: kegiatan, status Ketua Tim/anggota (`is_ketua_tim`), komponen, jenis, kategori, detail permintaan. Aturan ini **harus identik** antara validasi klien dan validasi server (bukan sekadar "mirip").

4. **Urutan validasi server saat submit** (Gambar 4.13, langkah 7): validasi skema (Zod) → validasi nominal & rantai (jenis/kategori/detail) → keberadaan file → kelengkapan wajib (aturan sama dengan klien) → penugasan Ketua Tim (bila diklaim) → **baru kemudian** satu transaksi (buat dokumen → ubah status → tulis log) → pindahkan file dari area *pending* ke lokasi tetap.

5. **Pemberkasan — get-or-create berkas** per pasangan **(cara pembayaran, tahun anggaran)**, ditegakkan oleh *unique index* komposit `(klasifikasi_id, tahun_anggaran)` di basis data (bukan hanya di kode):
   - klasifikasi harus aktif dan simpul daun (leaf) di hierarki;
   - jika berkas untuk pasangan itu sudah `OPEN` → dokumen ditambahkan ke sana;
   - jika sudah `CLOSED` → permintaan ditolak;
   - jika belum ada → berkas baru dibuat.

6. **Tutup berkas** mensyaratkan: ≥1 isi (item), Nomor SPM terisi, dan Masa Simpan Minimal terisi. **Tanpa** konfirmasi ketik-persis (beda dengan pembersihan file berkas yang pakai `BERSIHKAN FILE BERKAS`).

7. **Dua pintu masuk pemberkasan** bermuara ke *get-or-create* yang sama: (a) dokumen `COMPLETED` dari kotak masuk KSBU; (b) dokumen manual KSBU (tanpa alur persetujuan) — keduanya wajib memilih cara pembayaran + tahun anggaran.

8. Tidak ada fitur "simpan draf" — `DRAFT` adalah status **transien**, hanya hidup di dalam satu transaksi submit.

---

## Diagram A — Gambar 4.13: Pengajuan Dokumen

**Aktor/swimlane:** Pegawai, Sistem.

**Urutan aksi (sesuai kode, per kerangka Bab IV 4.2.2.1):**

1. Pegawai mengisi fungsi dan tanggal (**tidak boleh melewati hari ini**), lalu memilih kegiatan.
   - *Cek kode:* validasi tanggal ≤ hari ini ada di mana (klien dan/atau server)?
2. Sistem mendeteksi apakah Pegawai adalah Ketua Tim kegiatan tersebut (lookup `ketua_tim_assignments`), menampilkan badge.
3. **Decision — karakteristik dokumen:**
   - **[material]** → pilih Komponen → Jenis → Kategori → Detail Permintaan (Detail **boleh dilewati** bila Kategori sudah simpul daun) → isi nominal realisasi (**> 0**).
   - **[non-material]** → isi Nama Dokumen (teks bebas).
4. Sistem menampilkan checklist kelengkapan (*exact-match* 6 kolom — lihat aturan global #3). Pegawai boleh menambah lampiran tambahan di luar checklist.
5. **Decision — kelengkapan wajib terpenuhi?**
   - **[belum]** → kembali ke langkah unggah (merge ke titik sebelum decision ini).
   - **[terpenuhi]** → lanjut.
6. Setiap file diunggah ke **area pending** milik pengguna (belum masuk basis data).
7. Pegawai meninjau lalu menekan **Ajukan**. Tombol ini **nonaktif** sampai kelengkapan wajib terpenuhi (guard UI, dimodelkan sebagai decision di lane Sistem).
8. Server memvalidasi ulang **dalam urutan tetap** (lihat aturan global #4): skema → nominal & rantai → keberadaan file → kelengkapan wajib → penugasan Ketua Tim.
   - **Decision — valid?** [tidak valid] → tampilkan pesan kesalahan → kembali ke tinjau/ajukan. [valid] → lanjut.
9. **Decision — karakteristik (hasil, menentukan efek transaksi):**
   - **[material]** → satu transaksi: simpan dokumen, set status `IN_PPK_VALIDATION`, tulis `log_aktivitas` → lalu pindahkan file dari *pending* ke lokasi tetap. Objek akhir: `Dokumen [Menunggu Validasi PPK]`.
   - **[non-material]** → satu transaksi: simpan dokumen, set status `TERSIMPAN` (aksi log `STORE`), tulis `log_aktivitas` — **tanpa melewati modul transisi status terpusat** (lihat aturan global #2) → pindahkan file. Objek akhir: `Dokumen [Tersimpan]`.

**Catatan wajib pada diagram:** alur fisik (dokumen kertas) tetap berjalan berdampingan — sistem mendampingi, bukan menggantikan. Ini bukan langkah kode, murni anotasi naratif; boleh diabaikan saat validasi kode.

**Yang perlu divalidasi Claude Code terhadap sumber:**
- Apakah urutan validasi server (skema → nominal/rantai → file → kelengkapan → penugasan Ketua Tim) benar-benar berurutan seperti ini di handler (kemungkinan besar di `src/routes/api/dokumen*` atau sejenis)?
- Apakah checklist server memang *exact-match* 6 kolom yang identik dengan klien (per catatan kerangka: ini baru diperbaiki di commit `d45bcd9`, jadi pastikan sudah konsisten di kode saat ini)?
- Apakah jalur non-material benar-benar menulis status `TERSIMPAN` **tanpa** memanggil `fsm.ts`/modul transisi (per klaim "satu-satunya jalur di luar modul transisi status")?
- Apakah pemindahan file dari *pending* ke lokasi tetap terjadi **setelah** commit transaksi basis data (urutan: DB commit dulu, baru file dipindah)?
- Apakah nominal realisasi memang divalidasi `> 0` (bukan `>= 0`) di server?

---

## Diagram B — Gambar 4.14: Persetujuan Berjenjang dan Revisi Dokumen

**Aktor/swimlane:** PPK, Pegawai, PPSPM, Sistem. (Urutan lane pada Miro: PPK | Pegawai | Sistem | PPSPM — dipilih agar garis antar-lane paling sedikit berpotongan; ini murni tata letak visual, tidak relevan untuk validasi kode.)

**Aturan penting yang menentukan struktur diagram ini:** "Kembalikan" (aksi `KEMBALIKAN`) hanya tersedia **setelah PPSPM menolak**, tidak tersedia dari penolakan PPK langsung ke Pegawai.

**Urutan aksi:**

1. **Mulai:** dokumen berstatus `Dokumen [Menunggu Validasi PPK]` masuk ke kotak masuk PPK.
2. PPK memeriksa dokumen dan lampiran dari kotak masuk.
3. **Decision — PPK setuju/tolak?**
   - **[setuju]** → aksi `APPROVE` (transisi #2) → sistem meneruskan dokumen ke PPSPM + mencatat riwayat aktivitas → masuk kotak masuk PPSPM.
   - **[tolak]** → aksi `REJECT` (transisi #3) → PPK **wajib menulis catatan penolakan** (10–2000 karakter) → sistem mengembalikan dokumen ke Pegawai + mencatat riwayat aktivitas → **merge**: Pegawai membaca catatan penolakan → memperbaiki data/lampiran/nominal → mengajukan ulang (`RESUBMIT`, transisi #6, hanya berlaku bila `revision_target = USER`) → sistem meneruskan dokumen ke PPK + mencatat riwayat → kembali ke langkah 2 (merge ke kotak masuk PPK).
4. PPSPM memeriksa dokumen dan lampiran dari kotak masuk.
5. **Decision — PPSPM setuju/tolak?**
   - **[setuju]** → aksi `APPROVE` (transisi #4, **dengan guard: belum pernah disetujui PPSPM sebelumnya** — penjaga persetujuan ganda) → sistem menyelesaikan dokumen + mencatat riwayat → objek akhir `Dokumen [Selesai]` → **final**.
   - **[tolak]** → aksi `REJECT` (transisi #5) → PPSPM **wajib menulis catatan penolakan** → sistem mengembalikan dokumen ke PPK + mencatat riwayat (`revision_target` diset ke `PPK`) → PPK membaca catatan penolakan PPSPM.
6. **Decision — tindak lanjut PPK atas penolakan PPSPM** (ini adalah UC-10, "Menindaklanjuti Dokumen yang Ditolak PPSPM"):
   - **[diperbaiki PPK]** → PPK memperbaiki lampiran atau nominal → aksi `RESUBMIT_PPK` (transisi #7, guard `revision_target = PPK`) → sistem mengirim ulang dokumen langsung ke PPSPM + mencatat riwayat → kembali ke langkah 4 (merge ke kotak masuk PPSPM, **tidak** lewat PPK lagi).
   - **[dikembalikan ke Pegawai]** → aksi `KEMBALIKAN` (transisi #8, guard `revision_target = PPK`, hasil `revision_target → USER`) → sistem **membuat catatan pengembalian otomatis** (bukan PPK yang mengetik) dan mengembalikan dokumen ke Pegawai → merge ke langkah 3 (Pegawai membaca catatan → perbaiki → `RESUBMIT` ke PPK, **bukan** langsung ke PPSPM).

**Poin kritis untuk divalidasi (ini yang paling berisiko salah dimodelkan):**
- Pastikan transisi #7 (`RESUBMIT_PPK`) benar-benar meneruskan **langsung ke PPSPM**, tidak balik dulu ke status `IN_PPK_VALIDATION` yang berarti PPK harus approve lagi.
- Pastikan transisi #8 (`KEMBALIKAN`) benar-benar membuat catatan **otomatis** oleh sistem (bukan field kosong / bukan PPK yang mengisi form catatan) — cek pesan/template teks catatan otomatis di kode.
- Pastikan guard `revision_target` dicek di route handler (bukan hanya di UI) untuk membedakan apakah `RESUBMIT` (Pegawai, dari `revision_target=USER`) vs `RESUBMIT_PPK`/`KEMBALIKAN` (PPK, dari `revision_target=PPK`) yang valid dipanggil pada suatu dokumen `NEED_REVISION`.
- Pastikan penjaga persetujuan ganda PPSPM (tidak bisa `APPROVE` dua kali) benar-benar dicek — biasanya berupa pengecekan status masih `IN_PPSPM_APPROVAL` sebelum transisi, atau row lock/transaksi.
- Cek apakah panjang catatan penolakan (10–2000 karakter) benar-benar divalidasi Zod di kedua titik penolakan (PPK dan PPSPM) — bukan hanya salah satunya.
- Catatan anotasi "alur dokumen fisik tetap berjalan berdampingan" murni naratif, tidak perlu divalidasi ke kode.

---

## Diagram C — Gambar 4.16: Pemberkasan Dokumen

**Aktor/swimlane:** KSBU, Sistem.

**Urutan aksi:**

1. **Decision — dua pintu masuk:**
   - **[dokumen dari alur persetujuan]** → KSBU memilih dokumen `COMPLETED` dari kotak masuk dokumen selesai (UC-13, Mengklasifikasikan Dokumen ke Berkas).
   - **[dokumen tanpa alur persetujuan]** → KSBU mengisi data dokumen manual dan mengunggah dokumen elektronik (UC-14, Menambahkan Dokumen tanpa Alur Persetujuan: fungsi → kegiatan → komponen → nama dokumen, tanggal, keterangan, nominal, lampiran).
   - Kedua jalur **merge** ke langkah berikutnya — keduanya mewajibkan KSBU memilih cara pembayaran dan tahun anggaran.
2. KSBU memilih cara pembayaran dan tahun anggaran.
3. Sistem mencari berkas untuk pasangan (cara pembayaran, tahun anggaran) yang dipilih — **get-or-create**.
4. **Decision — status pencarian berkas** (aturan global #5):
   - **[berkas tertutup/`CLOSED`]** → tampilkan pesan "berkas sudah ditutup" → kembali ke langkah 2 (KSBU harus pilih pasangan lain).
   - **[berkas belum ada]** → sistem membuat berkas baru (status `OPEN`) → merge ke langkah berikut.
   - **[berkas terbuka/`OPEN`]** → merge langsung ke langkah berikut (tanpa membuat baru).
5. Sistem menambahkan dokumen ke berkas dan mencatat riwayat berkas (`berkas_arsip_activity`). Objek: `Berkas [Terbuka]`.
6. **Decision — pengelompokan (proses menambah dokumen ke berkas ini) selesai?**
   - **[belum selesai]** → kembali ke langkah 1 (KSBU lanjut memproses dokumen lain — tanpa loop balik ke awal untuk *dokumen yang sama*, ini loop di tingkat "dokumen berikutnya").
   - **[selesai]** → lanjut ke keputusan penutupan berkas.
7. KSBU mengisi Nomor SPM dan Masa Simpan Minimal (langkah menuju penutupan berkas — aturan global #6).
8. Sistem memeriksa isi dan data berkas.
9. **Decision — siap ditutup?**
   - **[tidak lengkap]** (kurang dari salah satu: ≥1 isi, Nomor SPM, masa simpan) → tampilkan pesan "data belum lengkap" → kembali ke langkah 7.
   - **[lengkap]** → sistem menutup berkas (status `CLOSED`) dan mencatat riwayat berkas → objek akhir `Berkas [Tertutup]` → **final**.

**Catatan pemodelan yang perlu diperiksa konsistensinya dengan kode:**
- Kerangka Bab IV menyebut "pemberkasan satu dokumen berakhir setelah masuk berkas bila berkas belum siap ditutup (tanpa loop ke awal)" — pastikan pemahaman ini konsisten: yaitu, menambahkan **satu dokumen** ke berkas adalah operasi yang selesai di langkah 5, dan keputusan menutup berkas (langkah 7–9) adalah operasi **terpisah** yang bisa dipicu KSBU kapan saja (tidak harus langsung setelah menambah dokumen). Jika di kode kedua operasi ini benar-benar terpisah (endpoint berbeda, dipanggil dari halaman berkas terbuka yang berbeda dari halaman kotak masuk), diagram ini menyederhanakan dua use case (UC-13/14 dan bagian dari UC-15) menjadi satu alur — sebutkan di catatan validasi apakah penyederhanaan ini menyesatkan atau masih dapat dipertanggungjawabkan.
- Pastikan *get-or-create* benar-benar menangani race condition (disebut di KNF-11 "tahan kondisi balapan") — cek apakah ada penanganan `unique constraint violation` / retry di kode, karena ini tidak tergambar sebagai decision terpisah di diagram (disederhanakan sebagai bagian dari langkah 3–4).
- Pastikan klasifikasi yang dipakai memang harus simpul daun (leaf) dan aktif — dicek di mana (server-side) sebelum get-or-create dijalankan; ini tidak muncul sebagai decision eksplisit di diagram tapi disebut sebagai prasyarat di kerangka.
- Perhatikan bahwa "Membersihkan File Berkas" (UC-16, Gambar 4.19 di kerangka lama) **tidak termasuk** dalam diagram C ini — itu proses terpisah (status_arsip: Aktif → Usul Pembersihan → File Dibersihkan) yang divalidasi lewat *State Machine Diagram* Status Berkas (Gambar 4.20), bukan bagian dari activity diagram pemberkasan. Jangan menganggap diagram C tidak lengkap karena tidak menggambarkan ini — itu memang di luar cakupannya secara sengaja.

---

## Ringkasan referensi silang status untuk validasi cepat

| Status dokumen | Final? | Diagram tempat muncul |
|---|---|---|
| `DRAFT` | Transien (tidak pernah persisten di luar transaksi) | A (implisit, sebelum submit) |
| `IN_PPK_VALIDATION` | Tidak | A (hasil), B (awal & setelah resubmit) |
| `IN_PPSPM_APPROVAL` | Tidak | B |
| `NEED_REVISION` | Tidak | B |
| `COMPLETED` | Ya (untuk dokumen material) | B (akhir), C (pintu masuk) |
| `TERSIMPAN` | Ya (untuk dokumen non-material) | A (hasil non-material) |

| Status berkas | Final? | Diagram tempat muncul |
|---|---|---|
| `OPEN`/`Terbuka` | Tidak | C |
| `CLOSED`/`Tertutup` | Ya (untuk *activity diagram* ini; masih ada siklus status_arsip lanjutan di luar cakupan diagram C) | C |

---

## Format keluaran yang diharapkan dari validasi

Untuk masing-masing dari ketiga diagram, mohon berikan:

1. Daftar langkah yang **`SESUAI`** — tidak perlu detail panjang, cukup nomor langkah + file/handler yang membuktikannya.
2. Daftar langkah yang **`TIDAK SESUAI`** — jelaskan apa yang sebenarnya terjadi di kode, sertakan path file dan (bila memungkinkan) nomor baris atau nama fungsi.
3. Daftar langkah/guard di kode yang **tidak tergambar** di diagram ini (kalau ada) — supaya bisa dipertimbangkan apakah perlu ditambahkan ke diagram atau memang sengaja disederhanakan.
4. Kesimpulan singkat per diagram: apakah diagram ini **layak dipakai apa adanya**, **perlu revisi kecil** (sebutkan bagian mana), atau **perlu revisi signifikan**.

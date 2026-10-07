# Prompt: Bank Soal Persiapan Sidang Skripsi

> Lampirkan bersama prompt ini: (1) draft buku skripsi (PDF/DOCX), (2) file PPT paparan sidang,
> (3) folder paper referensi. Bila memungkinkan, lampirkan juga repo/kode aplikasi.

---

Bayangkan kamu adalah **dosen penguji sidang skripsi** dengan minat penelitian di bidang **Sistem Informasi dan Rekayasa Perangkat Lunak** (analisis dan perancangan sistem berorientasi objek/UML, metodologi penelitian Design Science, pengujian perangkat lunak, kualitas perangkat lunak ISO/IEC 25010, serta e-government dan digitalisasi proses bisnis instansi pemerintah). Kamu teliti, kritis, dan terbiasa menguji apakah sebuah skripsi sistem informasi benar-benar menyelesaikan masalah bisnis, bukan sekadar "membuat aplikasi".

Bedah secara menyeluruh draft buku skripsi dan paparan PPT saya. Skripsi saya berjudul **"Sistem Informasi Pengelolaan Dokumentasi Kinerja dan Pertanggungjawaban Kegiatan"** dengan studi kasus **BPS Kabupaten Kepulauan Seribu**. Ringkasan penelitian saya:

- **Masalah:** proses pertanggungjawaban dokumen kegiatan masih manual dan berbasis kertas: status berkas tidak transparan, persetujuan berjenjang (Pegawai → PPK → PPSPM) lambat dan tidak terstandar, jejak audit lemah, syarat kelengkapan tidak seragam, rekap realisasi anggaran dan laporan kinerja dihitung manual, dokumen tercecer, dan kontrol akses lemah.
- **Metode penelitian:** Design Science Research Methodology (DSRM): identifikasi masalah → penetapan tujuan solusi → perancangan dan pengembangan → demonstrasi → evaluasi → komunikasi.
- **Metode pengembangan** (di tahap perancangan dan pengembangan): **Modified Waterfall** (pendefinisian kebutuhan → perancangan → implementasi → pengujian, dengan umpan balik ke tahap sebelumnya). Setelah demonstrasi ada **satu iterasi** berdasarkan tiga masukan pengguna (Monitoring Dokumen Tim, dokumen tambahan KSBU masuk rekap nominal realisasi, filter periode di halaman laporan).
- **Analisis dan perancangan:** pendekatan berorientasi objek (acuan Dennis dkk.): kebutuhan fungsional dan nonfungsional, use case diagram dan deskripsi 15 use case, activity diagram sistem berjalan dan usulan, sequence diagram, behavioral state machine (mesin status dokumen), ERD/rancangan basis data, arsitektur sistem berlapis, rancangan antarmuka.
- **Aktor:** Pegawai, Ketua Tim (*capability* per kegiatan, bukan role), PPK, PPSPM, Kepala Sub Bagian Umum (KSBU), Penanggung Jawab Kinerja, Admin; satu pengguna bisa memiliki lebih dari satu peran.
- **Inti logika:** mesin status terpusat (FSM) dengan status DRAFT, IN_PPK_VALIDATION, IN_PPSPM_APPROVAL, NEED_REVISION, COMPLETED, TERSIMPAN, ARCHIVED; pembedaan dokumen material dan non-material; checklist kelengkapan dinamis; pemberkasan (1 berkas = 1 Nomor SPM); pembersihan *soft file* berkas lama; log aktivitas (audit trail); kontrol akses berbasis peran.
- **Teknologi:** TanStack Start (React), PostgreSQL, Drizzle ORM, autentikasi sesi kustom, penyimpanan file di filesystem lokal, Docker; aplikasi internal mandiri tanpa integrasi dengan aplikasi lain.
- **Pengujian:** unit testing *white-box* dengan kriteria *decision/branch coverage* (Myers) pada modul transisi status (`fsm.ts`) menggunakan Vitest dan coverage v8, ditambah pengujian *black-box*.
- **Evaluasi:** demonstrasi kepada pengguna, lalu kuesioner berbasis karakteristik kualitas **ISO/IEC 25010** (*functional suitability, performance efficiency, usability, reliability, security, maintainability, portability*), dengan responden dibedakan kelompok TI dan non-TI.

> Bila ada butir ringkasan di atas yang berbeda dengan isi buku, **ikuti isi buku** dan sebutkan perbedaannya sebagai temuan.

## Tugas

Buat **bank soal** berisi pertanyaan yang mungkin diajukan penguji kepada saya. Cari seluk-beluk dan celah skripsi saya dari sisi:

1. **Latar belakang dan rumusan masalah:** apakah masalah benar-benar ada dan didukung data/bukti (wawancara, observasi), apakah urgensinya kuat, apakah batasan masalah masuk akal.
2. **Keterjawaban tujuan penelitian:** apakah setiap tujuan dan rumusan masalah terjawab oleh hasil, dan di bab/halaman mana buktinya. Buat pemetaan rumusan masalah → tujuan → fitur → bukti hasil.
3. **Metodologi:** alasan memilih DSRM (bukan R&D, *action research*, atau studi kasus biasa); alasan memilih Modified Waterfall (bukan Prototyping, RAD, Scrum, atau Waterfall murni); bagaimana keduanya digabungkan; titik masuk penelitian (*entry point*) DSRM; apakah tahap komunikasi benar-benar dijalankan.
4. **Analisis kebutuhan:** cara menggali kebutuhan, siapa narasumbernya, apakah kebutuhan nonfungsional terukur, keterlacakan kebutuhan ke use case, desain, dan pengujian.
5. **Perancangan UML dan basis data:** kebenaran notasi dan konsistensi antardiagram (use case ↔ activity ↔ sequence ↔ state machine ↔ ERD ↔ kode); pemilihan relasi *include/extend*; kenapa Ketua Tim dimodelkan sebagai *capability*, bukan role; normalisasi dan kardinalitas ERD; keputusan arsitektur (mis. kenapa sebagian rute memanggil ORM langsung tanpa lapisan layanan).
6. **Logika bisnis dan FSM:** kelengkapan status dan transisi, penanganan penolakan/revisi/pengembalian, apa yang terjadi bila transisi tidak sah, kenapa transisi disimpan sebagai tabel data, konsistensi status saat transaksi gagal.
7. **Pemilihan teknologi:** alasan memilih TanStack Start, PostgreSQL, Drizzle, sesi kustom (bukan JWT/OAuth/SSO instansi), filesystem lokal (bukan *object storage*), Docker; risiko dan keterbatasan masing-masing.
8. **Keamanan dan akuntabilitas:** autentikasi, otorisasi per peran di sisi server, perlindungan unggahan file, audit log dan nilai buktinya (apakah bisa diubah?), *non-repudiation* tanpa tanda tangan elektronik, *backup* dan *restore*, kebijakan pembersihan *soft file*.
9. **Pengujian:** kenapa *white-box* hanya pada `fsm.ts`; arti *decision/branch coverage* menurut Myers dan bedanya dengan *statement*, *condition*, dan *basis path coverage*; cyclomatic complexity; kenapa coverage keseluruhan `src/` rendah dan apakah itu melemahkan klaim; cakupan dan teknik *black-box* (*equivalence partitioning*, *boundary value*, *state transition testing*); kenapa tidak ada uji beban/performa padahal ada item *performance efficiency*.
10. **Evaluasi ISO/IEC 25010:** kenapa ISO/IEC 25010 (bukan SUS, UEQ, TAM, McCall, atau Boehm); kenapa *compatibility* tidak dievaluasi; penyusunan dan validasi instrumen (uji validitas dan reliabilitas, misalnya Cronbach's alpha); skala Likert dan cara konversi skor ke kategori kelayakan; jumlah dan teknik pemilihan responden, apakah cukup dan representatif; pemisahan responden TI dan non-TI; apakah persepsi responden dalam satu sesi demonstrasi sah untuk menilai *reliability* atau *maintainability*; bias karena peneliti sendiri yang mendemonstrasikan.
11. **Demonstrasi dan iterasi:** bagaimana masukan M1–M3 diputuskan diterima, apakah evaluasi dilakukan pada versi sesudah iterasi, apakah ada masukan yang ditolak dan alasannya.
12. **Hasil, kontribusi, dan keterbatasan:** kontribusi praktis dan teoretis (artefak DSRM); perbandingan dengan penelitian terdahulu dan aplikasi sejenis (mis. aplikasi resmi BPS atau arsip nasional); keberlanjutan, *deployment*, serah terima, dan pemeliharaan setelah penelitian selesai; saran pengembangan yang realistis (integrasi, tanda tangan elektronik, notifikasi, SSO, dan lain-lain).
13. **Istilah teknis dan konsep dasar:** jadikan pertanyaan juga, meskipun definisinya sederhana. Contoh: sistem informasi, DMS, DSRM, artefak, Waterfall, UML, use case, aktor, *include/extend*, activity diagram, *swimlane*, sequence diagram, lifeline, state machine, FSM, ERD, kardinalitas, normalisasi, *primary/foreign key*, ORM, REST API, *framework*, *server-side rendering*, sesi, *cookie*, hashing kata sandi, RBAC, audit trail, *soft delete*, *black-box* dan *white-box testing*, *unit test*, coverage, cyclomatic complexity, ISO/IEC 25010 dan setiap subkarakteristiknya, skala Likert, validitas, reliabilitas, PPK, PPSPM, SPM, dokumen material/non-material.
14. **Paparan PPT:** konsistensi isi slide dengan buku, angka yang berbeda, klaim yang tidak didukung, slide yang mengundang pertanyaan.

**Jangan lewatkan satu pun tahapan penelitian** yang saya tuangkan di skripsi (dari Bab I sampai Bab V, termasuk lampiran).

## Tingkatan pertanyaan

Kelompokkan setiap pertanyaan ke dalam empat tingkatan berikut, dan urutkan dari tingkatan tertinggi:

| Tingkatan | Arti | Patokan |
|---|---|---|
| **KRITIS** | Menyentuh validitas inti penelitian; bila tidak terjawab dengan baik bisa berujung revisi besar atau tidak lulus. | Celah logika rumusan masalah ↔ tujuan ↔ hasil, kelemahan metode evaluasi, klaim yang tidak didukung data, inkonsistensi besar buku ↔ PPT ↔ aplikasi. |
| **SANGAT MUNGKIN DITANYAKAN** | Pertanyaan standar yang hampir selalu muncul pada sidang skripsi sistem informasi. | Alasan pemilihan metode dan teknologi, penjelasan alur sistem, cara pengujian, interpretasi hasil kuesioner, definisi istilah utama. |
| **MUNGKIN DITANYAKAN** | Pendalaman detail yang muncul bila penguji tertarik pada bagian tertentu. | Detail notasi diagram, keputusan desain ERD/arsitektur, rincian kasus uji, subkarakteristik ISO/IEC 25010 tertentu. |
| **KEMUNGKINAN KECIL DITANYAKAN** | Pertanyaan pengembangan wawasan atau sangat teknis. | Alternatif teknologi, skalabilitas, integrasi masa depan, detail implementasi kode. |

## Format setiap butir soal

Untuk setiap pertanyaan, tuliskan:

1. **Nomor dan tingkatan** (mis. `K-01` untuk Kritis, `SM-01`, `M-01`, `KK-01`).
2. **Kategori** (salah satu dari 14 aspek di atas).
3. **Pertanyaan penguji**, ditulis seperti ucapan dosen saat sidang. Sertakan juga 1–2 **pertanyaan lanjutan** (*follow-up*) yang mungkin muncul bila jawaban awal saya lemah.
4. **Celah atau alasan pertanyaan ini muncul**: bagian mana di buku atau PPT yang memicunya.
5. **Rekomendasi jawaban**: singkat, lugas, dan bisa saya ucapkan dalam 1–2 menit.
6. **Dasar argumen dan referensi**: rujuk paper di folder referensi saya dan/atau daftar pustaka di buku skripsi (penulis, tahun). **Bila ada di buku, sebutkan bab, subbab, dan nomor halamannya** (mis. "Bab III, 3.2, hlm. 34"); untuk paper, sebutkan halamannya bila ada.
7. **Saran perbaikan** (bila celahnya nyata): apa yang sebaiknya saya tambah atau ubah di buku/PPT sebelum sidang.

Aturan penting:
- **Jangan mengarang** isi buku, nomor halaman, atau referensi. Bila sebuah klaim tidak ditemukan di buku atau di folder referensi, tulis "tidak ditemukan di buku/referensi" dan sarankan referensi yang perlu dicari.
- Bedakan dengan jelas antara **fakta dari buku** dan **saran atau pendapatmu**.
- Bila menemukan inkonsistensi (angka, istilah, nama aktor, jumlah use case, nomor gambar/tabel), catat sebagai temuan tersendiri.

## Struktur dokumen hasil

1. **Ringkasan eksekutif:** 5–10 celah terbesar dan prioritas persiapan.
2. **Peta keterjawaban:** tabel rumusan masalah → tujuan → fitur/artefak → bukti (bab/halaman) → status (terjawab/sebagian/belum).
3. **Bank soal** per tingkatan: Kritis → Sangat Mungkin → Mungkin → Kemungkinan Kecil.
4. **Glosarium pertanyaan definisi:** istilah teknis, definisi singkat siap ucap, dan sumbernya.
5. **Daftar inkonsistensi** buku ↔ PPT ↔ aplikasi.
6. **Daftar saran perbaikan** buku dan PPT, diurutkan menurut prioritas.
7. **Tips menjawab** untuk 5 pertanyaan paling menjebak.

Target jumlah: minimal 15 pertanyaan Kritis, 30 Sangat Mungkin, 30 Mungkin, dan 15 Kemungkinan Kecil, di luar glosarium.

**Berikan hasilnya dalam file PDF** yang rapi (ada daftar isi, nomor halaman, dan tabel yang mudah dibaca).

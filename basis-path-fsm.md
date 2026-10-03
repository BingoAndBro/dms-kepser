# Basis Path Testing: Modul Transisi Status (`src/lib/fsm.ts`)

Lingkup: unit testing white-box untuk dua fungsi di `src/lib/fsm.ts`, yaitu `transition()` (baris 81–127) dan `isActorValidForAction()` (baris 129–156), diuji oleh `tests/fsm.test.ts`. Teknik yang dipakai adalah basis path testing menurut Pressman (2010, edisi 7, hlm. 485–491): flow graph, cyclomatic complexity V(G), jalur independen, dan kasus uji.

Acuan awal adalah `laporan-kelayakan-unit-whitebox.md` bagian 3.2–3.6. Laporan itu dibuat ketika `tests/fsm.test.ts` baru berisi 49 kasus. Berkas ini memakai 52 kasus (UT-FSM-001 s.d. UT-FSM-052, daftar lengkap di `hasil-unit-whitebox-fsm.md` bagian e). Perubahan utama dibanding acuan awal:

- Jalur J7 di `transition()` dan jalur A7 (`default`) di `isActorValidForAction()` sekarang sudah punya kasus uji (UT-FSM-050 dan UT-FSM-052).
- `isActorValidForAction()` digambar dengan aturan yang diminta: `switch` = satu simpul predikat bercabang tujuh, dan setiap `return` = satu simpul. Ekspresi boolean di dalam `return` tidak dipecah, karena tidak membelokkan alur kontrol.

Tidak ada berkas di `src/` yang diubah, dan tidak ada kasus uji yang ditambahkan, karena setiap jalur independen sudah punya kasus uji.

Gambar flow graph ada di `docs/skripsi/gambar/`:

| Fungsi | Sumber Graphviz | PNG | SVG |
|---|---|---|---|
| `transition()` | `flowgraph-fsm-transition.dot` | `flowgraph-fsm-transition.png` | `flowgraph-fsm-transition.svg` |
| `isActorValidForAction()` | `flowgraph-fsm-isactorvalid.dot` | `flowgraph-fsm-isactorvalid.png` | `flowgraph-fsm-isactorvalid.svg` |

Simpul berupa lingkaran bernomor. Simpul predikat diarsir abu-abu. Sisi diberi label T (benar), F (salah), atau nama `case`. Dirender dengan Graphviz 16.1.0 (`dot -Tsvg` dan `dot -Tpng -Gdpi=200`).

---

## 1. Fungsi `transition()` (baris 81–127)

### 1.1 Tabel simpul

Kondisi majemuk dipecah menjadi satu simpul per kondisi: L96 berisi tiga kondisi (simpul 4, 5, 6), L104 dua kondisi (simpul 8, 9), dan L110 dua kondisi (simpul 11, 12). Pemanggilan `isActorValidForAction()` di L87 digabung dengan pengujian hasilnya di L88 menjadi satu simpul, karena keduanya berurutan tanpa percabangan. Hal yang sama berlaku untuk L117–119.

| Simpul | Baris | Keterangan | Predikat? |
|---:|---|---|:---:|
| 1 | 87–88 | Periksa kewenangan aktor. Apakah aktor **tidak** berwenang? | ya |
| 2 | 89–92 | Kembalikan galat "aktor tidak bisa melakukan aksi". | |
| 3 | 95 | Apakah aksinya REJECT? | ya |
| 4 | 96 | Apakah `revisionTarget` kosong? | ya |
| 5 | 96 | Apakah `revisionTarget` bukan USER? | ya |
| 6 | 96 | Apakah `revisionTarget` bukan PPK? | ya |
| 7 | 97–100 | Kembalikan galat "REJECT butuh target USER atau PPK". | |
| 8 | 104 | Apakah aksinya RESUBMIT? | ya |
| 9 | 104 | Apakah `revisionTarget` bukan USER? | ya |
| 10 | 105–108 | Kembalikan galat "RESUBMIT hanya untuk target USER". | |
| 11 | 110 | Apakah aksinya RESUBMIT_PPK? | ya |
| 12 | 110 | Apakah `revisionTarget` bukan PPK? | ya |
| 13 | 111–114 | Kembalikan galat "RESUBMIT_PPK hanya untuk target PPK". | |
| 14 | 117–119 | Susun kunci `status:aksi`, cari di tabel `TRANSITIONS`. Apakah transisinya **tidak** ada? | ya |
| 15 | 120–123 | Kembalikan galat "transisi tidak valid". | |
| 16 | 126 | Kembalikan hasil sukses dari tabel. | |
| 17 | 127 | Keluar dari fungsi. | |

Jumlah simpul N = 17, terdiri dari 10 simpul predikat dan 7 simpul biasa. Setiap simpul predikat bercabang dua.

### 1.2 Daftar sisi

| No | Dari | Ke | Label | | No | Dari | Ke | Label |
|---:|---:|---:|:---:|---|---:|---:|---:|:---:|
| 1 | 1 | 2 | T | | 14 | 8 | 11 | F |
| 2 | 1 | 3 | F | | 15 | 9 | 10 | T |
| 3 | 2 | 17 | | | 16 | 9 | 11 | F |
| 4 | 3 | 4 | T | | 17 | 10 | 17 | |
| 5 | 3 | 8 | F | | 18 | 11 | 12 | T |
| 6 | 4 | 7 | T | | 19 | 11 | 14 | F |
| 7 | 4 | 5 | F | | 20 | 12 | 13 | T |
| 8 | 5 | 6 | T | | 21 | 12 | 14 | F |
| 9 | 5 | 8 | F | | 22 | 13 | 17 | |
| 10 | 6 | 7 | T | | 23 | 14 | 15 | T |
| 11 | 6 | 8 | F | | 24 | 14 | 16 | F |
| 12 | 7 | 17 | | | 25 | 15 | 17 | |
| 13 | 8 | 9 | T | | 26 | 16 | 17 | |

Jumlah sisi E = 26.

Alasan beberapa sisi:

- 4 → 7 (T): bila `revisionTarget` kosong, operand kanan `||` tidak dievaluasi dan galat langsung dikembalikan.
- 5 → 8 (F): bila `revisionTarget` = USER, operand `&&` berikutnya tidak dievaluasi, kondisi L96 bernilai salah, dan alur keluar dari blok `if` REJECT ke L104.
- 8 → 11 (F): bila aksi bukan RESUBMIT, operand kanan `&&` tidak dievaluasi.

### 1.3 Cyclomatic complexity V(G)

| Cara | Perhitungan | Hasil |
|---|---|---:|
| Jumlah region | 10 region tertutup + 1 region luar | **11** |
| E − N + 2 | 26 − 17 + 2 | **11** |
| P + 1 | 10 simpul predikat (semuanya biner) + 1 | **11** |

Ketiganya sama, V(G) = **11**.

Region dihitung dari gambar `flowgraph-fsm-transition.png`, yang tidak memiliki sisi bersilangan. Setiap region disebut dengan simpul-simpul pada batasnya:

| Region | Simpul pembatas | Dibentuk oleh predikat |
|---|---|---|
| R1 | 1, 2, 17, 7, 4, 3 | simpul 1 |
| R2 | 3, 4, 5, 8 | simpul 3 |
| R3 | 4, 7, 6, 5 | simpul 4 |
| R4 | 5, 6, 8 | simpul 5 |
| R5 | 6, 7, 17, 13, 12, 11, 8 | simpul 6 |
| R6 | 8, 9, 11 | simpul 8 |
| R7 | 9, 10, 17, 15, 14, 11 | simpul 9 |
| R8 | 11, 12, 14 | simpul 11 |
| R9 | 12, 13, 17, 16, 14 | simpul 12 |
| R10 | 14, 15, 17, 16 | simpul 14 |
| R11 | region luar (dibatasi 1, 3, 8, 9, 10, 17, 2) | |

### 1.4 Jalur independen dan kasus uji

Jalur dasar adalah J2 (jalur sukses tanpa REJECT/RESUBMIT/RESUBMIT_PPK). Jalur lain disusun dengan membalik satu predikat setiap kali. Setiap jalur menambahkan minimal satu sisi yang belum dilalui jalur sebelumnya:

| Jalur | Sisi baru |
|---|---|
| J2 | 1→3, 3→8, 8→11, 11→14, 14→16, 16→17 |
| J1 | 1→2, 2→17 |
| J3 | 14→15, 15→17 |
| J4 | 3→4, 4→7, 7→17 |
| J5 | 4→5, 5→8 |
| J6 | 5→6, 6→8 |
| J7 | 6→7 |
| J8 | 8→9, 9→10, 10→17 |
| J9 | 9→11 |
| J10 | 11→12, 12→13, 13→17 |
| J11 | 12→14 |

Jumlah sisi baru 6 + 2 + 2 + 3 + 2 + 2 + 1 + 3 + 1 + 3 + 1 = 26, jadi kesebelas jalur bersama-sama melalui semua sisi.

Pesan galat dicocokkan persis oleh tes. Singkatannya mengikuti `hasil-unit-whitebox-fsm.md` bagian e:

- **G-AKTOR**: `Actor '<peran>' tidak bisa melakukan aksi '<aksi>' pada status '<status>'`
- **G-REJECT**: `REJECT requires revisionTarget: 'USER' or 'PPK'`
- **G-RESUBMIT**: `RESUBMIT only valid when revisionTarget is 'USER'`
- **G-RESUBMIT_PPK**: `RESUBMIT_PPK only valid when revisionTarget is 'PPK'`
- **G-TABEL**: `Transisi '<aksi>' dari '<status>' tidak valid`

Kolom "Hasil aktual" berasal dari dua sumber: (1) status lulus pada run bagian 3, dan (2) nilai kembalian `transition()` yang dicetak langsung untuk masukan kasus wakil (lihat bagian 3.3).

| Jalur | Urutan simpul | Kondisi masukan | Kasus wakil (kasus lain di jalur yang sama) | Hasil yang diharapkan | Hasil aktual | Status |
|---|---|---|---|---|---|---|
| J1 | 1-2-17 | Aktor tidak berwenang untuk aksi dan status itu | UT-FSM-010: DRAFT, SUBMIT, PPK (011–013, 015, 016, 020, 022, 024, 026–032, 041, 042, 044, 045, 048, 049, 051, 052) | gagal, G-AKTOR | `success: false`, `Actor 'PPK' tidak bisa melakukan aksi 'SUBMIT' pada status 'DRAFT'` | lulus |
| J2 | 1-3-8-11-14-16-17 | Aktor berwenang; aksi SUBMIT, APPROVE, atau KEMBALIKAN; transisi ada di tabel | UT-FSM-001: DRAFT, SUBMIT, PEGAWAI (002, 004, 008, 009, 014, 017, 025, 033) | sukses: IN_PPK_VALIDATION, step PPK, target null, urutan 1 | `success: true`, IN_PPK_VALIDATION, PPK, null, 1 | lulus |
| J3 | 1-3-8-11-14-15-17 | Aktor berwenang; aksi bukan REJECT/RESUBMIT/RESUBMIT_PPK; transisi tidak ada di tabel | UT-FSM-043: COMPLETED, SUBMIT, PEGAWAI (047) | gagal, G-TABEL | `success: false`, `Transisi 'SUBMIT' dari 'COMPLETED' tidak valid` | lulus |
| J4 | 1-3-4-7-17 | REJECT oleh aktor berwenang; `revisionTarget` kosong | UT-FSM-034: IN_PPK_VALIDATION, REJECT, PPK, tanpa target (tidak ada) | gagal, G-REJECT | `success: false`, G-REJECT | lulus |
| J5 | 1-3-4-5-8-11-14-16-17 | REJECT oleh aktor berwenang; `revisionTarget` = USER | UT-FSM-003: IN_PPK_VALIDATION, REJECT, PPK, USER (018, 035) | sukses: NEED_REVISION, PPK, USER, 1 | `success: true`, NEED_REVISION, PPK, USER, 1 | lulus |
| J6 | 1-3-4-5-6-8-11-14-16-17 | REJECT oleh aktor berwenang; `revisionTarget` = PPK | UT-FSM-005: IN_PPSPM_APPROVAL, REJECT, PPSPM, PPK (019, 036) | sukses: NEED_REVISION, PPSPM, PPK, 1 | `success: true`, NEED_REVISION, PPSPM, PPK, 1 | lulus |
| J7 | 1-3-4-5-6-7-17 | REJECT oleh aktor berwenang; `revisionTarget` terisi tetapi bukan USER/PPK | UT-FSM-050: IN_PPK_VALIDATION, REJECT, PPK, `'X'` (tidak ada) | gagal, G-REJECT | `success: false`, G-REJECT | lulus |
| J8 | 1-3-8-9-10-17 | RESUBMIT oleh PEGAWAI; `revisionTarget` bukan USER | UT-FSM-038: NEED_REVISION, RESUBMIT, PEGAWAI, PPK (tidak ada) | gagal, G-RESUBMIT | `success: false`, G-RESUBMIT | lulus |
| J9 | 1-3-8-9-11-14-16-17 | RESUBMIT oleh PEGAWAI; `revisionTarget` = USER; status NEED_REVISION | UT-FSM-006: NEED_REVISION, RESUBMIT, PEGAWAI, USER (021, 037) | sukses: IN_PPK_VALIDATION, PPK, null, 1 | `success: true`, IN_PPK_VALIDATION, PPK, null, 1 | lulus |
| J10 | 1-3-8-11-12-13-17 | RESUBMIT_PPK oleh PPK; `revisionTarget` bukan PPK | UT-FSM-040: NEED_REVISION, RESUBMIT_PPK, PPK, USER (tidak ada) | gagal, G-RESUBMIT_PPK | `success: false`, G-RESUBMIT_PPK | lulus |
| J11 | 1-3-8-11-12-14-16-17 | RESUBMIT_PPK oleh PPK; `revisionTarget` = PPK; status NEED_REVISION | UT-FSM-007: NEED_REVISION, RESUBMIT_PPK, PPK, PPK (023, 039) | sukses: IN_PPSPM_APPROVAL, PPSPM, null, 2 | `success: true`, IN_PPSPM_APPROVAL, PPSPM, null, 2 | lulus |

Satu kasus menempuh jalur di luar basis set: **UT-FSM-046** (IN_PPK_VALIDATION, RESUBMIT, PEGAWAI, USER) menempuh 1-3-8-9-11-14-15-17 dan menghasilkan G-TABEL. Jalur ini adalah gabungan linier dari jalur-jalur basis, sehingga tidak menambah V(G).

Jumlah: 24 (J1) + 9 (J2) + 2 + 1 + 3 + 3 + 1 + 1 + 3 + 1 + 3 = 51, ditambah UT-FSM-046 = **52 kasus**. Setiap kasus menempuh tepat satu jalur.

**Pencocokan dengan hitungan cabang coverage** (bagian 3.2). UT-FSM-033 memanggil `transition()` dua kali, jadi J2 = 10 panggilan.

| Keputusan | Hitungan dari coverage (T / F) | Hitungan dari tabel jalur |
|---|---|---|
| Simpul 1 (L88) | 24 / 29 | T: J1 = 24. F: J2 10 + J3 2 + J4 1 + J5 3 + J6 3 + J7 1 + J8 1 + J9 3 + J10 1 + J11 3 + UT-046 1 = 29 |
| Simpul 3 (L95) | 8 / 21 | T: J4–J7 = 1 + 3 + 3 + 1 = 8. F: 29 − 8 = 21 |
| L96 (simpul 4–6 sebagai satu `if`) | 2 / 6 | T: J4 + J7 = 2. F: J5 + J6 = 6 |
| L104 (simpul 8–9 sebagai satu `if`) | 1 / 26 | T: J8 = 1. F: 29 − 2 − 1 = 26 |
| L110 (simpul 11–12 sebagai satu `if`) | 1 / 25 | T: J10 = 1. F: 26 − 1 = 25 |
| Simpul 14 (L119) | 3 / 22 | T: J3 2 + UT-046 1 = 3. F: J2 10 + J5 3 + J6 3 + J9 3 + J11 3 = 22 |

Semua angka cocok.

---

## 2. Fungsi `isActorValidForAction()` (baris 129–156)

### 2.1 Tabel simpul

Sesuai aturan yang dipakai, `switch` digambar sebagai satu simpul predikat bercabang tujuh, dan setiap pernyataan `return` menjadi satu simpul. Ekspresi boolean di dalam `return` (misalnya `role === ROLES.PPK && status === DOC_STATUS.NEED_REVISION`) hanya menghitung nilai kembalian dan tidak membelokkan alur kontrol, jadi tidak dipecah. Kombinasi kondisi di dalam ekspresi itu sudah dibahas terpisah di `hasil-unit-whitebox-fsm.md` bagian d.

| Simpul | Baris | Keterangan | Predikat? |
|---:|---|---|:---:|
| 1 | 134 | Aksi apa yang diminta? (`switch (action)`) | ya, 7 cabang |
| 2 | 135–136 | SUBMIT: kembalikan "apakah peran PEGAWAI?" | |
| 3 | 137–141 | APPROVE: kembalikan "apakah (status IN_PPK_VALIDATION dan peran PPK) atau (status IN_PPSPM_APPROVAL dan peran PPSPM)?" | |
| 4 | 142–146 | REJECT: kembalikan pemeriksaan yang sama dengan APPROVE. | |
| 5 | 147–148 | RESUBMIT: kembalikan "apakah peran PEGAWAI?" | |
| 6 | 149–150 | RESUBMIT_PPK: kembalikan "apakah peran PPK?" | |
| 7 | 151–152 | KEMBALIKAN: kembalikan "apakah peran PPK dan status NEED_REVISION?" | |
| 8 | 153–154 | Aksi tidak dikenal (`default`): kembalikan `false`. | |
| 9 | 155–156 | Keluar dari fungsi. | |

N = 9, terdiri dari 1 simpul predikat dan 8 simpul biasa.

### 2.2 Daftar sisi

| No | Dari | Ke | Label |
|---:|---:|---:|---|
| 1 | 1 | 2 | SUBMIT |
| 2 | 1 | 3 | APPROVE |
| 3 | 1 | 4 | REJECT |
| 4 | 1 | 5 | RESUBMIT |
| 5 | 1 | 6 | RESUBMIT_PPK |
| 6 | 1 | 7 | KEMBALIKAN |
| 7 | 1 | 8 | default |
| 8 | 2 | 9 | |
| 9 | 3 | 9 | |
| 10 | 4 | 9 | |
| 11 | 5 | 9 | |
| 12 | 6 | 9 | |
| 13 | 7 | 9 | |
| 14 | 8 | 9 | |

E = 14.

### 2.3 Cyclomatic complexity V(G)

| Cara | Perhitungan | Hasil |
|---|---|---:|
| Jumlah region | 6 region tertutup + 1 region luar | **7** |
| E − N + 2 | 14 − 9 + 2 | **7** |
| P + 1 | simpul 1 bercabang 7, dihitung 7 − 1 = 6; maka 6 + 1 | **7** |

Ketiganya sama, V(G) = **7**.

Region pada `flowgraph-fsm-isactorvalid.png`: setiap dua cabang yang bersebelahan membentuk satu region tertutup, yaitu R1 (1, 2, 9, 3), R2 (1, 3, 9, 4), R3 (1, 4, 9, 5), R4 (1, 5, 9, 6), R5 (1, 6, 9, 7), R6 (1, 7, 9, 8). Ditambah R7 (region luar, dibatasi 1, 2, 9, 8). Total 7.

### 2.4 Jalur independen dan kasus uji

`isActorValidForAction()` tidak diekspor, jadi diuji lewat `transition()`. Nilai kembalian `true` terlihat sebagai lolosnya simpul 1 di `transition()`, dan nilai `false` terlihat sebagai galat G-AKTOR.

Pada setiap jalur, nilai kembalian bisa `true` atau `false`, tergantung peran dan status. Bagi flow graph keduanya adalah jalur yang sama. Kolom "Kasus lain" mencakup kedua kemungkinan nilai itu. Rincian kasus per nilai kembalian ada di `hasil-unit-whitebox-fsm.md` bagian c (keputusan 8–13).

| Jalur | Urutan simpul | Kondisi masukan | Kasus wakil (kasus lain di jalur yang sama) | Hasil yang diharapkan | Hasil aktual | Status |
|---|---|---|---|---|---|---|
| A1 | 1-2-9 | aksi = SUBMIT | UT-FSM-001: DRAFT, SUBMIT, PEGAWAI (009–013, 043, 047) | `true`, sehingga `transition()` lolos pemeriksaan aktor dan sukses | `success: true`, IN_PPK_VALIDATION | lulus |
| A2 | 1-3-9 | aksi = APPROVE | UT-FSM-002: IN_PPK_VALIDATION, APPROVE, PPK (004, 014–017, 041, 044, 048, 049) | `true`, `transition()` sukses | `success: true`, IN_PPSPM_APPROVAL | lulus |
| A3 | 1-4-9 | aksi = REJECT | UT-FSM-003: IN_PPK_VALIDATION, REJECT, PPK, USER (005, 018–020, 034–036, 042, 045, 050, 051) | `true`, `transition()` sukses | `success: true`, NEED_REVISION | lulus |
| A4 | 1-5-9 | aksi = RESUBMIT | UT-FSM-006: NEED_REVISION, RESUBMIT, PEGAWAI, USER (021, 022, 037, 038, 046) | `true`, `transition()` sukses | `success: true`, IN_PPK_VALIDATION | lulus |
| A5 | 1-6-9 | aksi = RESUBMIT_PPK | UT-FSM-007: NEED_REVISION, RESUBMIT_PPK, PPK, PPK (023, 024, 039, 040) | `true`, `transition()` sukses | `success: true`, IN_PPSPM_APPROVAL | lulus |
| A6 | 1-7-9 | aksi = KEMBALIKAN | UT-FSM-008: NEED_REVISION, KEMBALIKAN, PPK, USER (025–033) | `true`, `transition()` sukses | `success: true`, NEED_REVISION, target USER | lulus |
| A7 | 1-8-9 | aksi di luar keenam nilai `FSMAction` | UT-FSM-052: DRAFT, `'X' as FSMAction`, PEGAWAI (tidak ada) | `false`, `transition()` gagal dengan G-AKTOR | `success: false`, `Actor 'PEGAWAI' tidak bisa melakukan aksi 'X' pada status 'DRAFT'` | lulus |

Jumlah kasus per jalur: A1 8, A2 10, A3 12, A4 6, A5 5, A6 10, A7 1, total **52 kasus**. Karena UT-FSM-033 (di A6) memanggil `transition()` dua kali, A6 menghasilkan 11 panggilan, sehingga total panggilan 8 + 10 + 12 + 6 + 5 + 11 + 1 = **53**. Angka per jalur sama dengan hitungan cabang `switch` dari coverage (bagian 3.2): SUBMIT 8, APPROVE 10, REJECT 12, RESUBMIT 6, RESUBMIT_PPK 5, KEMBALIKAN 11, `default` 1.

---

## 3. Hasil run dan coverage

### 3.1 Data run

| Item | Nilai |
|---|---|
| Tanggal | 3 Oktober 2026, mulai 09:04:14 |
| Commit | `c2c0842faa11bcb25e73c132024cd34c3e332eaa` (`src/` dan `tests/` tidak diubah) |
| Vitest | 3.2.4 (`vitest/3.2.4 win32-x64 node-v24.14.0`) |
| Penyedia coverage | `@vitest/coverage-v8` 3.2.4, `experimentalAstAwareRemapping: true` dari `vitest.config.ts` |
| Node | v24.14.0 |
| OS | Windows 11 Home |

Perintah:

```bash
npx vitest run tests/fsm.test.ts --coverage --coverage.include=src/lib/fsm.ts --coverage.reportsDirectory=<scratch>/cov-fsm --coverage.reporter=text --coverage.reporter=json --coverage.reporter=json-summary --reporter=verbose --reporter=json --outputFile.json=<scratch>/fsm-run.json
```

`<scratch>` adalah folder sementara di luar repo, supaya laporan di `coverage/` tidak tertimpa.

Keluaran teks Vitest (kode warna ANSI dibuang):

```text
 Test Files  1 passed (1)
      Tests  52 passed (52)
   Start at  09:04:14
   Duration  2.05s (transform 130ms, setup 0ms, collect 161ms, tests 12ms, environment 0ms, prepare 858ms)

 % Coverage report from v8
----------|---------|----------|---------|---------|-------------------
File      | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
----------|---------|----------|---------|---------|-------------------
All files |     100 |      100 |     100 |     100 |                   
 fsm.ts   |     100 |      100 |     100 |     100 |                   
----------|---------|----------|---------|---------|-------------------
```

Laporan JSON: 52 kasus, 52 lulus, 0 gagal.

### 3.2 Coverage `src/lib/fsm.ts`

Dari `coverage-summary.json`:

| Metrik | Tertutup dari total | Persentase |
|---|---:|---:|
| Statement | **25 dari 25** | 100% |
| Branch | **36 dari 36** | 100% |
| Function | 3 dari 3 | 100% |
| Line | 25 dari 25 | 100% |

Hitungan eksekusi per cabang dari `coverage-final.json`. Angka ini yang dicocokkan dengan tabel jalur di 1.4 dan 2.4.

| Cabang | Jenis | Baris | Hitungan |
|---|---|---|---|
| 0 | `if` | 88 | T 24, F 29 |
| 1 | `if` | 95 | T 8, F 21 |
| 2 | `if` | 96 | T 2, F 6 |
| 3 | operand `\|\|`/`&&` | 96 | a 8, b 7, c 4 |
| 4 | `if` | 104 | T 1, F 26 |
| 5 | operand `&&` | 104 | a 27, b 5 |
| 6 | `if` | 110 | T 1, F 25 |
| 7 | operand `&&` | 110 | a 26, b 4 |
| 8 | `if` | 119 | T 3, F 22 |
| 9 | `switch` | 134 | SUBMIT 8, APPROVE 10, REJECT 12, RESUBMIT 6, RESUBMIT_PPK 5, KEMBALIKAN 11, `default` 1 |
| 10 | operand `\|\|`/`&&` | 139–140 | 10, 3, 8, 3 |
| 11 | operand `\|\|`/`&&` | 144–145 | 12, 6, 7, 4 |
| 12 | operand `&&` | 152 | 11, 7 |

Hitungan operand juga cocok dengan simpul 4–6 dan 8–9 di `transition()`:

- Simpul 4 (operand a, L96) dievaluasi 8 kali (J4–J7).
- Simpul 5 (operand b) dievaluasi 7 kali (semua kecuali J4).
- Simpul 6 (operand c) dievaluasi 4 kali (J6 3 + J7 1).
- Simpul 9 (operand b, L104) dievaluasi 5 kali (J8 1 + J9 3 + UT-FSM-046 1).
- Simpul 12 (operand b, L110) dievaluasi 4 kali (J10 1 + J11 3).

Fungsi: `transition` dan `isActorValidForAction` masing-masing dipanggil 53 kali, `makeError` 31 kali. Nilai 31 sama dengan jumlah panggilan yang berakhir di simpul galat (J1 24 + J3 2 + J4 1 + J7 1 + J8 1 + J10 1 + UT-FSM-046 1).

### 3.3 Cara memperoleh hasil aktual

Kolom "Hasil aktual" di 1.4 dan 2.4 diambil dari keluaran `transition()` untuk masukan kasus wakil. Nilainya dicetak dengan skrip sementara di luar repo (`npx tsx`) yang mengimpor `src/lib/fsm.ts` tanpa mengubahnya. Contoh keluaran:

```text
010 {"success":false,"newStatus":"DRAFT","newCurrentStep":null,"newRevisionTarget":null,"stepUrutan":null,"error":"Actor 'PPK' tidak bisa melakukan aksi 'SUBMIT' pada status 'DRAFT'"}
043 {"success":false,"newStatus":"COMPLETED","newCurrentStep":null,"newRevisionTarget":null,"stepUrutan":null,"error":"Transisi 'SUBMIT' dari 'COMPLETED' tidak valid"}
050 {"success":false,"newStatus":"IN_PPK_VALIDATION","newCurrentStep":null,"newRevisionTarget":null,"stepUrutan":null,"error":"REJECT requires revisionTarget: 'USER' or 'PPK'"}
052 {"success":false,"newStatus":"DRAFT","newCurrentStep":null,"newRevisionTarget":null,"stepUrutan":null,"error":"Actor 'PEGAWAI' tidak bisa melakukan aksi 'X' pada status 'DRAFT'"}
```

Status "lulus" berasal dari laporan JSON run 3.1. Tes yang sama mencocokkan isi `error` secara persis, jadi kelulusannya juga membuktikan simpul galat mana yang dicapai.

---

## 4. Catatan

### 4.1 Jalur yang tidak bisa dicapai secara normal

1. **A7 (`default`, L153–154) di `isActorValidForAction()`.** Parameter `action` bertipe `FSMAction`, yang hanya berisi enam nilai, dan keenamnya punya `case`. Pemanggil yang lolos pemeriksaan tipe TypeScript tidak bisa mencapai `default`. UT-FSM-052 mencapainya dengan cast `'X' as FSMAction`. Cast ini meniru nilai yang lolos dari pemeriksaan tipe saat runtime, misalnya dari masukan yang tidak tervalidasi. Jalur ini dibiarkan ada di kode sebagai pengaman.

2. **Kombinasi predikat yang mustahil di `transition()`.** Graf `transition()` memuat jalur yang secara struktur ada tetapi tidak mungkin ditempuh. Basis set di 1.4 tidak memakai satu pun dari jalur ini:
   - Setelah simpul 3 bernilai T (aksi = REJECT), simpul 8 (aksi = RESUBMIT) dan simpul 11 (aksi = RESUBMIT_PPK) pasti bernilai F. Jadi jalur seperti 1-3-4-5-8-9-… atau 1-3-4-5-8-11-12-… tidak mungkin.
   - Setelah simpul 8 bernilai T (aksi = RESUBMIT), simpul 11 pasti F. Jadi 1-3-8-9-11-12-… tidak mungkin.
   - Pada jalur REJECT yang sukses (J5, J6), simpul 14 pasti F. REJECT hanya lolos simpul 1 pada status IN_PPK_VALIDATION atau IN_PPSPM_APPROVAL, dan kedua kunci itu ada di tabel `TRANSITIONS`. Jadi 1-3-4-5-8-11-14-15-17 tidak mungkin.
   - Hal yang sama berlaku untuk APPROVE dan KEMBALIKAN pada J3: aktor berwenang hanya pada status yang punya entri di tabel. Karena itu J3 (simpul 14 = T) hanya bisa dicapai lewat SUBMIT dari status selain DRAFT (UT-FSM-043, 047).

### 4.2 Jalur di luar basis set

- 1-3-8-9-11-14-15-17 (RESUBMIT bertarget USER dari status selain NEED_REVISION) feasible dan teruji oleh UT-FSM-046.
- 1-3-8-11-12-14-15-17 (RESUBMIT_PPK bertarget PPK dari status selain NEED_REVISION, misalnya `transition('IN_PPK_VALIDATION', 'RESUBMIT_PPK', 'PPK', 'PPK')`) feasible tetapi belum punya kasus uji. Jalur ini bukan jalur independen baru, karena semua sisinya sudah dilalui basis set (sisi 12→14 oleh J11, 14→15 dan 15→17 oleh J3). Sesuai aturan, kasus uji tidak ditambahkan.

### 4.3 Batas basis path pada modul ini

- Delapan transisi sah disimpan sebagai data di tabel `TRANSITIONS` (L19–68), bukan sebagai percabangan. Di flow graph, seluruh isi tabel diwakili satu simpul predikat (simpul 14). Basis path membuktikan logika pencarian tabel teruji, tetapi tidak membuktikan setiap baris tabel benar. Kebenaran tiap baris dibuktikan oleh UT-FSM-001 s.d. 008, yang memeriksa keempat field hasil untuk setiap transisi sah.
- Fungsi `makeError()` (L70–79) tidak bercabang (V(G) = 1). Fungsi ini tercakup oleh setiap jalur galat, sehingga tidak digambar terpisah.
- Pada `isActorValidForAction()`, aturan "ekspresi boolean di dalam `return` bukan percabangan alur" membuat V(G) = 7. Bila operand `&&`/`||` di dalam `return` dihitung sebagai predikat, V(G) menjadi 14 (laporan kelayakan 3.3). Kecukupan pengujian kondisi di dalam ekspresi itu dibuktikan terpisah lewat tabel kombinasi kondisi di `hasil-unit-whitebox-fsm.md` bagian d.

### 4.4 Perbedaan dengan acuan awal (`laporan-kelayakan-unit-whitebox.md` 3.2–3.6)

| Bagian acuan | Isi acuan (49 kasus) | Keadaan sekarang (52 kasus) |
|---|---|---|
| 3.3, V(G) `transition()` tingkat kondisi | 11 | 11, tidak berubah (`src/` tidak berubah) |
| 3.3, V(G) `isActorValidForAction()` | 7 (keputusan) / 14 (kondisi) | 7, memakai aturan `return` = satu simpul |
| 3.4, simpul dan sisi `transition()` | N = 17, E = 26 | sama; penomoran simpul juga sama |
| 3.5, J1 | 22 kasus | 24 kasus (+051, +052) |
| 3.5, J7 | BELUM ADA | UT-FSM-050 |
| 3.6, A3 (REJECT) | tanpa 050, 051 | ditambah 050, 051 |
| 3.6, A7 (`default`) | BELUM ADA | UT-FSM-052 |
| Coverage `fsm.ts` | statement 24 dari 25, branch 35 dari 36 | statement 25 dari 25, branch 36 dari 36 |

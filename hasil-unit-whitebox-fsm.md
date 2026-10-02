# Hasil Unit Testing White-box: Modul Transisi Status (`src/lib/fsm.ts`)

Lingkup: unit testing white-box **hanya** untuk `src/lib/fsm.ts`, diuji oleh `tests/fsm.test.ts`. Kriteria kecukupan: **decision/branch coverage menurut Myers**, yaitu setiap keputusan harus menghasilkan nilai benar dan salah minimal satu kali. Flow graph dan basis path tidak dipakai. Acuan: `laporan-kelayakan-unit-whitebox.md` bagian 3.7 dan 3.8.

Tidak ada berkas di `src/` yang diubah. Ketiga kasus baru berperilaku persis seperti perkiraan, dan semua asersi pesan galat yang diperketat lulus tanpa mengubah masukan.

---

## a. Data run

| Item | Nilai |
|---|---|
| Tanggal | 2 Oktober 2026 |
| Commit dasar | `4da7ed1a186fdb3788e6e0b6918dbd41e1d3ed17`, ditambah perubahan yang belum di-commit pada daftar (f). `src/` sama persis dengan commit dasar. |
| Vitest | 3.2.4 (`vitest/3.2.4 win32-x64 node-v24.14.0`) |
| Penyedia coverage | `@vitest/coverage-v8` 3.2.4, `experimentalAstAwareRemapping: true` (pemetaan AST) |
| Node | v24.14.0 |
| pnpm | 10.33.0 |
| OS | Windows 11 Home |

Run coverage FSM dijalankan sebelum kontainer Postgres dinyalakan. Ini tidak memengaruhi hasilnya, karena `fsm.ts` hanya mengimpor konstanta dan tipe, tanpa modul basis data.

Bukti bahwa pemetaan AST aktif: penyebut cabang `fsm.ts` adalah 36 dan penyebut statement 25. Angka ini sama dengan run mode AST di laporan kelayakan (35/36, 24/25). Mode bawaan v8 memberi penyebut cabang 32.

Perintah yang dipakai:

```bash
npx vitest run tests/fsm.test.ts --coverage --coverage.include=src/lib/fsm.ts --coverage.reportsDirectory=<scratch>/cov-fsm --reporter=verbose --reporter=json --outputFile.json=<scratch>/fsm-run.json
```
```bash
docker compose -f infra/docker/postgres/docker-compose.yml up -d
```
```bash
pnpm test
```

`<scratch>` adalah folder sementara di luar repo. Laporan coverage FSM sengaja tidak ditulis ke `coverage/` supaya laporan lama di sana tidak tertimpa. `pnpm test` dijalankan tanpa opsi tambahan; `maxWorkers: 4` sekarang berasal dari `vitest.config.ts`.

---

## b. Coverage `src/lib/fsm.ts` dari `tests/fsm.test.ts`

Angka diambil dari `coverage-summary.json` run di atas.

| Metrik | Tertutup / Total | Persentase |
|---|---:|---:|
| Statement | 25 / 25 | 100% |
| Branch | **36 / 36** | 100% |
| Function | 3 / 3 | 100% |
| Line | 25 / 25 | 100% |

Sebelum perubahan ini (laporan kelayakan, mode AST): statement 24/25, branch 35/36. Cabang yang tadinya tidak tertutup adalah `default` pada L153–154. Sekarang cabang itu tertutup oleh UT-FSM-052.

Keluaran teks Vitest, disalin apa adanya (kode warna ANSI dibuang):

```text
 Test Files  1 passed (1)
      Tests  52 passed (52)
   Start at  20:46:04
   Duration  1.49s (transform 97ms, setup 0ms, collect 125ms, tests 11ms, environment 0ms, prepare 566ms)

 % Coverage report from v8
----------|---------|----------|---------|---------|-------------------
File      | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
----------|---------|----------|---------|---------|-------------------
All files |     100 |      100 |     100 |     100 |                   
 fsm.ts   |     100 |      100 |     100 |     100 |                   
----------|---------|----------|---------|---------|-------------------
```

Hitungan eksekusi per cabang dari `coverage-final.json` (run yang sama). Angka ini dipakai untuk mencocokkan tabel (c) dan (d).

| Cabang | Jenis | Baris | Hitungan per lokasi |
|---|---|---|---|
| 0 | `if` | 88 | benar 24, salah 29 |
| 1 | `if` | 95 | benar 8, salah 21 |
| 2 | `if` | 96 | benar 2, salah 6 |
| 3 | operand `\|\|`/`&&` | 96 | a 8, b 7, c 4 |
| 4 | `if` | 104 | benar 1, salah 26 |
| 5 | operand `&&` | 104 | a 27, b 5 |
| 6 | `if` | 110 | benar 1, salah 25 |
| 7 | operand `&&` | 110 | a 26, b 4 |
| 8 | `if` | 119 | benar 3, salah 22 |
| 9 | `switch` | 134 | SUBMIT 8, APPROVE 10, REJECT 12, RESUBMIT 6, RESUBMIT_PPK 5, KEMBALIKAN 11, `default` 1 |
| 10 | operand `\|\|`/`&&` | 139–140 | c1 10, c2 3, c3 8, c4 3 |
| 11 | operand `\|\|`/`&&` | 144–145 | c1 12, c2 6, c3 7, c4 4 |
| 12 | operand `&&` | 152 | c1 11, c2 7 |

Fungsi: `makeError` dipanggil 31 kali, `transition` 53 kali, `isActorValidForAction` 53 kali. Ada 53 panggilan dari 52 kasus karena UT-FSM-033 memanggil `transition()` dua kali.

Catatan definisi: cabang `if` dan `switch` di atas setara dengan decision coverage. Cabang operand `&&`/`||` dihitung tertutup bila operand itu **pernah dievaluasi**, bukan bila pernah bernilai benar dan salah (lihat laporan kelayakan 1.2). Karena itu kombinasi kondisi di bagian (d) diperiksa manual.

---

## c. Tabel keputusan gaya Myers (13 keputusan)

ID kasus memakai format UT-FSM-nnn dari daftar (e). Angka di kolom terakhir adalah hitungan eksekusi dari coverage dan cocok dengan jumlah panggilan dari kasus yang tercantum.

| No | Keputusan (baris) | Situasi untuk hasil BENAR | Situasi untuk hasil SALAH | Kasus uji (BENAR) | Kasus uji (SALAH) | Hitungan B / S |
|---|---|---|---|---|---|---|
| 1 | `!actorValid` (L88) | Aktor tidak berhak untuk aksi dan status itu, atau aksi tidak dikenal | Aktor berhak | 010–013, 015, 016, 020, 022, 024, 026–032, 041, 042, 044, 045, 048, 049, 051, 052 | 001–009, 014, 017–019, 021, 023, 025, 033–040, 043, 046, 047, 050 | 24 / 29 |
| 2 | `action === REJECT` (L95) | REJECT oleh aktor yang sah | Aksi lain oleh aktor yang sah | 003, 005, 018, 019, 034–036, 050 | 001, 002, 004, 006–009, 014, 017, 021, 023, 025, 033, 037–040, 043, 046, 047 | 8 / 21 |
| 3 | `!rt \|\| (rt !== 'USER' && rt !== 'PPK')` (L96) | `revisionTarget` kosong, atau terisi dengan nilai selain USER/PPK | `revisionTarget` = USER atau PPK | 034 (kosong), 050 (`'X'`) | 003, 005, 018, 019, 035, 036 | 2 / 6 |
| 4 | `action === RESUBMIT && rt !== 'USER'` (L104) | RESUBMIT dengan target selain USER | Aksi bukan RESUBMIT, atau RESUBMIT dengan target USER | 038 | RESUBMIT bertarget USER: 006, 021, 037, 046. Aksi lain: 001–005, 007–009, 014, 017–019, 023, 025, 033, 035, 036, 039, 040, 043, 047 | 1 / 26 |
| 5 | `action === RESUBMIT_PPK && rt !== 'PPK'` (L110) | RESUBMIT_PPK dengan target selain PPK | Aksi bukan RESUBMIT_PPK, atau RESUBMIT_PPK dengan target PPK | 040 | RESUBMIT_PPK bertarget PPK: 007, 023, 039. Aksi lain: 001–006, 008, 009, 014, 017–019, 021, 025, 033, 035–037, 043, 046, 047 | 1 / 25 |
| 6 | `!result` (L119) | Tidak ada entri `status:aksi` di tabel `TRANSITIONS` | Entri ada | 043, 046, 047 | 001–009, 014, 017–019, 021, 023, 025, 033, 035–037, 039 | 3 / 22 |
| 7 | `switch (action)` (L134) | Aksi cocok dengan salah satu dari 6 `case` (rincian di bawah) | Aksi tidak cocok dengan `case` mana pun, sehingga masuk `default` (L153) | Lihat rincian di bawah tabel | 052 | 51 / 1 |
| 8 | `role === PEGAWAI` (L136, SUBMIT) | Peran PEGAWAI | Peran lain | 001, 009, 043, 047 | 010–013 | 4 / 4 |
| 9 | `(IN_PPK && PPK) \|\| (IN_PPSPM && PPSPM)` (L139–140, APPROVE) | (IN_PPK_VALIDATION, PPK) atau (IN_PPSPM_APPROVAL, PPSPM) | Kombinasi lain | 002, 004, 014, 017 | 015, 016, 041, 044, 048, 049 | 4 / 6 |
| 10 | `(IN_PPK && PPK) \|\| (IN_PPSPM && PPSPM)` (L144–145, REJECT) | (IN_PPK_VALIDATION, PPK) atau (IN_PPSPM_APPROVAL, PPSPM) | Kombinasi lain | 003, 005, 018, 019, 034–036, 050 | 020, 042, 045, 051 | 8 / 4 |
| 11 | `role === PEGAWAI` (L148, RESUBMIT) | Peran PEGAWAI | Peran lain | 006, 021, 037, 038, 046 | 022 | 5 / 1 |
| 12 | `role === PPK` (L150, RESUBMIT_PPK) | Peran PPK | Peran lain | 007, 023, 039, 040 | 024 | 4 / 1 |
| 13 | `role === PPK && status === NEED_REVISION` (L152, KEMBALIKAN) | PPK pada status NEED_REVISION | Peran lain, atau status lain | 008, 025, 033 | 026–032 | 4 / 7 |

Hitungan B / S untuk keputusan 8–13 dihitung dari daftar kasus, karena coverage AST tidak mencatat hasil benar/salah ekspresi `return`. Jumlahnya cocok dengan hitungan cabang `switch` di bagian (b): SUBMIT 8, APPROVE 10, REJECT 12, RESUBMIT 6, RESUBMIT_PPK 5, KEMBALIKAN 11. Hitungan 4 pada baris 13 berasal dari UT-FSM-033 yang memanggil `transition()` dua kali.

Rincian keputusan 7 (7 hasil `switch`):

| Hasil | Baris | Kasus uji | Hitungan |
|---|---|---|---|
| `case SUBMIT` | 135 | 001, 009–013, 043, 047 | 8 |
| `case APPROVE` | 137 | 002, 004, 014–017, 041, 044, 048, 049 | 10 |
| `case REJECT` | 142 | 003, 005, 018–020, 034–036, 042, 045, 050, 051 | 12 |
| `case RESUBMIT` | 147 | 006, 021, 022, 037, 038, 046 | 6 |
| `case RESUBMIT_PPK` | 149 | 007, 023, 024, 039, 040 | 5 |
| `case KEMBALIKAN` | 151 | 008, 025–033 | 11 |
| `default` | 153 | 052 | 1 |

**Semua 13 keputusan sudah terpenuhi pada setiap hasilnya.** Decision coverage `fsm.ts` = 36/36 cabang menurut pengukuran (b).

Catatan untuk UT-FSM-052: `default` tidak bisa dicapai lewat tipe `FSMAction`. Kasus ini memakai cast `'X' as FSMAction` untuk mensimulasikan nilai di luar tipe, misalnya dari masukan yang tidak tervalidasi saat runtime.

---

## d. Kombinasi kondisi pada keputusan majemuk

Notasi: B = benar, S = salah, – = tidak dievaluasi karena short-circuit.

**Keputusan 3 (L96)**: a = `!rt`, b = `rt !== 'USER'`, c = `rt !== 'PPK'`

| a | b | c | Hasil | Contoh `revisionTarget` | Kasus |
|---|---|---|---|---|---|
| B | – | – | B | `undefined` | 034 |
| S | S | – | S | `'USER'` | 003, 018, 035 |
| S | B | S | S | `'PPK'` | 005, 019, 036 |
| S | B | B | B | `'X'` | **050 (baru)** |

- Setiap kondisi sudah bernilai B dan S: a (B 034, S 003), b (B 005, S 003), c (B 050, S 005).
- Kombinasi yang mustahil:
  - a = B dengan b/c terevaluasi: bila a benar, operand kanan `||` tidak dievaluasi.
  - (S, S, B) dan (S, S, S): bila b salah, c tidak dievaluasi karena `&&`. Tanpa short-circuit pun (S, S, S) tidak mungkin, karena berarti `rt` bernilai `'USER'` sekaligus `'PPK'`.
- Varian `null` dan `''` memberi kombinasi yang sama dengan `undefined` (a = B) dan tidak diuji terpisah.

**Keputusan 4 (L104)**: a = `action === RESUBMIT`, b = `rt !== 'USER'`

| a | b | Hasil | Kasus |
|---|---|---|---|
| S | – | S | semua aksi selain RESUBMIT yang lolos L88 dan L96, mis. 001 |
| B | B | B | 038 (target PPK) |
| B | S | S | 006, 021, 037, 046 |

Kombinasi (S, B) dan (S, S) mustahil karena b tidak dievaluasi bila a salah. RESUBMIT dengan target kosong menghasilkan kombinasi yang sama dengan 038 dan tidak diuji terpisah.

**Keputusan 5 (L110)**: a = `action === RESUBMIT_PPK`, b = `rt !== 'PPK'`

| a | b | Hasil | Kasus |
|---|---|---|---|
| S | – | S | mis. 001 |
| B | B | B | 040 (target USER) |
| B | S | S | 007, 023, 039 |

Kombinasi (S, B) dan (S, S) mustahil karena short-circuit.

**Keputusan 9 (L139–140, APPROVE)**: c1 = `status === IN_PPK_VALIDATION`, c2 = `role === PPK`, c3 = `status === IN_PPSPM_APPROVAL`, c4 = `role === PPSPM`

| c1 | c2 | c3 | c4 | Hasil | Kasus |
|---|---|---|---|---|---|
| B | B | – | – | B | 002, 014 |
| B | S | S | – | S | 015 |
| S | – | B | B | B | 004, 017 |
| S | – | B | S | S | 016 |
| S | – | S | – | S | 041, 044, 048, 049 |

Setiap kondisi sudah bernilai B dan S. Kombinasi yang mustahil:
- c1 = B dan c3 = B: satu dokumen tidak bisa berstatus IN_PPK_VALIDATION dan IN_PPSPM_APPROVAL sekaligus. Karena itu baris (B, S, B, ·) tidak ada.
- c1 = B dan c2 = B dengan c3/c4 terevaluasi: operand kanan `||` dilewati.
- c1 = S atau c2 = S dengan operand berikutnya dalam `&&` yang sama terevaluasi: dilewati oleh short-circuit (tanda –).

**Keputusan 10 (L144–145, REJECT)**: struktur sama dengan keputusan 9

| c1 | c2 | c3 | c4 | Hasil | Kasus |
|---|---|---|---|---|---|
| B | B | – | – | B | 003, 018, 034, 035, 050 |
| B | S | S | – | S | 020 |
| S | – | B | B | B | 005, 019, 036 |
| S | – | B | S | S | **051 (baru)**: REJECT oleh PPK di IN_PPSPM_APPROVAL |
| S | – | S | – | S | 042, 045 |

c4 sekarang pernah bernilai S (051), jadi setiap kondisi sudah bernilai B dan S. Kombinasi yang mustahil sama dengan keputusan 9.

**Keputusan 13 (L152, KEMBALIKAN)**: c1 = `role === PPK`, c2 = `status === NEED_REVISION`

| c1 | c2 | Hasil | Kasus |
|---|---|---|---|
| B | B | B | 008, 025, 033 |
| B | S | S | 030, 031, 032 |
| S | – | S | 026–029 |

Kombinasi (S, B) dan (S, S) mustahil karena c2 tidak dievaluasi bila c1 salah.

**Ringkasan:** setiap kombinasi yang feasible pada keenam keputusan majemuk sudah punya kasus uji, dan setiap kondisi sederhana di dalamnya sudah bernilai benar dan salah.

---

## e. Daftar kasus uji FSM

Urutan mengikuti urutan dalam `tests/fsm.test.ts`. UT-FSM-001 s.d. UT-FSM-049 sama dengan nomor #1–#49 di `laporan-kelayakan-unit-whitebox.md` dan `lampiran-kasus-uji-unit.csv`. Tiga kasus baru (050–052) ditaruh di blok `describe` terakhir supaya nomor lama tidak bergeser.

Status lulus/gagal berasal dari laporan JSON run FSM pada bagian (a): 52 lulus, 0 gagal. Pada kolom revisionTarget, – berarti argumen tidak diberikan.

Pesan galat yang diharapkan (dicocokkan persis oleh tes):

- **G-AKTOR**: `Actor '<peran>' tidak bisa melakukan aksi '<aksi>' pada status '<status>'`
- **G-REJECT**: `REJECT requires revisionTarget: 'USER' or 'PPK'`
- **G-RESUBMIT**: `RESUBMIT only valid when revisionTarget is 'USER'`
- **G-RESUBMIT_PPK**: `RESUBMIT_PPK only valid when revisionTarget is 'PPK'`
- **G-TABEL**: `Transisi '<aksi>' dari '<status>' tidak valid`

| ID | Nama kasus asli | Status | Aksi | Peran | revisionTarget | Hasil yang diharapkan | Hasil |
|---|---|---|---|---|---|---|---|
| UT-FSM-001 | DRAFT + SUBMIT → IN_PPK_VALIDATION, step=1 | DRAFT | SUBMIT | PEGAWAI | – | sukses: IN_PPK_VALIDATION, current step PPK, target null, step 1 | lulus |
| UT-FSM-002 | IN_PPK_VALIDATION + APPROVE → IN_PPSPM_APPROVAL, step=2 | IN_PPK_VALIDATION | APPROVE | PPK | – | sukses: IN_PPSPM_APPROVAL, PPSPM, null, 2 | lulus |
| UT-FSM-003 | IN_PPK_VALIDATION + REJECT → NEED_REVISION, target=USER, step=1 | IN_PPK_VALIDATION | REJECT | PPK | USER | sukses: NEED_REVISION, PPK, USER, 1 | lulus |
| UT-FSM-004 | IN_PPSPM_APPROVAL + APPROVE → COMPLETED, step=2 | IN_PPSPM_APPROVAL | APPROVE | PPSPM | – | sukses: COMPLETED, null, null, 2 | lulus |
| UT-FSM-005 | IN_PPSPM_APPROVAL + REJECT → NEED_REVISION, target=PPK, step=1 | IN_PPSPM_APPROVAL | REJECT | PPSPM | PPK | sukses: NEED_REVISION, PPSPM, PPK, 1 | lulus |
| UT-FSM-006 | NEED_REVISION + RESUBMIT → IN_PPK_VALIDATION, step=1 | NEED_REVISION | RESUBMIT | PEGAWAI | USER | sukses: IN_PPK_VALIDATION, PPK, null, 1 | lulus |
| UT-FSM-007 | NEED_REVISION + RESUBMIT_PPK → IN_PPSPM_APPROVAL, step=2 | NEED_REVISION | RESUBMIT_PPK | PPK | PPK | sukses: IN_PPSPM_APPROVAL, PPSPM, null, 2 | lulus |
| UT-FSM-008 | NEED_REVISION + KEMBALIKAN → NEED_REVISION, target=USER, step=1 | NEED_REVISION | KEMBALIKAN | PPK | USER | sukses: NEED_REVISION, PPK, USER, 1 | lulus |
| UT-FSM-009 | SUBMIT by PEGAWAI → success | DRAFT | SUBMIT | PEGAWAI | – | sukses | lulus |
| UT-FSM-010 | SUBMIT by PPK → error | DRAFT | SUBMIT | PPK | – | gagal, G-AKTOR | lulus |
| UT-FSM-011 | SUBMIT by PPSPM → error | DRAFT | SUBMIT | PPSPM | – | gagal, G-AKTOR | lulus |
| UT-FSM-012 | SUBMIT by KEPALA_SUB_BAGIAN_UMUM → error | DRAFT | SUBMIT | KEPALA_SUB_BAGIAN_UMUM | – | gagal, G-AKTOR | lulus |
| UT-FSM-013 | SUBMIT by ADMIN → error | DRAFT | SUBMIT | ADMIN | – | gagal, G-AKTOR | lulus |
| UT-FSM-014 | APPROVE by PPK on IN_PPK_VALIDATION → success | IN_PPK_VALIDATION | APPROVE | PPK | – | sukses | lulus |
| UT-FSM-015 | APPROVE by PPSPM on IN_PPK_VALIDATION → error | IN_PPK_VALIDATION | APPROVE | PPSPM | – | gagal, G-AKTOR | lulus |
| UT-FSM-016 | APPROVE by PPK on IN_PPSPM_APPROVAL → error | IN_PPSPM_APPROVAL | APPROVE | PPK | – | gagal, G-AKTOR | lulus |
| UT-FSM-017 | APPROVE by PPSPM on IN_PPSPM_APPROVAL → success | IN_PPSPM_APPROVAL | APPROVE | PPSPM | – | sukses | lulus |
| UT-FSM-018 | REJECT by PPK on IN_PPK_VALIDATION → success | IN_PPK_VALIDATION | REJECT | PPK | USER | sukses | lulus |
| UT-FSM-019 | REJECT by PPSPM on IN_PPSPM_APPROVAL → success | IN_PPSPM_APPROVAL | REJECT | PPSPM | PPK | sukses | lulus |
| UT-FSM-020 | REJECT by PPSPM on IN_PPK_VALIDATION → error | IN_PPK_VALIDATION | REJECT | PPSPM | USER | gagal, G-AKTOR | lulus |
| UT-FSM-021 | RESUBMIT by PEGAWAI → success | NEED_REVISION | RESUBMIT | PEGAWAI | USER | sukses | lulus |
| UT-FSM-022 | RESUBMIT by PPK → error | NEED_REVISION | RESUBMIT | PPK | USER | gagal, G-AKTOR | lulus |
| UT-FSM-023 | RESUBMIT_PPK by PPK → success | NEED_REVISION | RESUBMIT_PPK | PPK | PPK | sukses | lulus |
| UT-FSM-024 | RESUBMIT_PPK by PEGAWAI → error | NEED_REVISION | RESUBMIT_PPK | PEGAWAI | PPK | gagal, G-AKTOR | lulus |
| UT-FSM-025 | KEMBALIKAN by PPK → success | NEED_REVISION | KEMBALIKAN | PPK | USER | sukses | lulus |
| UT-FSM-026 | KEMBALIKAN by PEGAWAI → error | NEED_REVISION | KEMBALIKAN | PEGAWAI | USER | gagal, G-AKTOR | lulus |
| UT-FSM-027 | KEMBALIKAN by PPSPM → error | NEED_REVISION | KEMBALIKAN | PPSPM | USER | gagal, G-AKTOR | lulus |
| UT-FSM-028 | KEMBALIKAN by KEPALA_SUB_BAGIAN_UMUM → error | NEED_REVISION | KEMBALIKAN | KEPALA_SUB_BAGIAN_UMUM | USER | gagal, G-AKTOR | lulus |
| UT-FSM-029 | KEMBALIKAN by ADMIN → error | NEED_REVISION | KEMBALIKAN | ADMIN | USER | gagal, G-AKTOR | lulus |
| UT-FSM-030 | KEMBALIKAN from IN_PPK_VALIDATION → error | IN_PPK_VALIDATION | KEMBALIKAN | PPK | USER | gagal, G-AKTOR | lulus |
| UT-FSM-031 | KEMBALIKAN from IN_PPSPM_APPROVAL → error | IN_PPSPM_APPROVAL | KEMBALIKAN | PPK | USER | gagal, G-AKTOR | lulus |
| UT-FSM-032 | KEMBALIKAN from COMPLETED → error | COMPLETED | KEMBALIKAN | PPK | USER | gagal, G-AKTOR | lulus |
| UT-FSM-033 | KEMBALIKAN result always targets USER | NEED_REVISION | KEMBALIKAN | PPK | (1) PPK; (2) – | kedua panggilan: newRevisionTarget = USER | lulus |
| UT-FSM-034 | REJECT without revisionTarget → error | IN_PPK_VALIDATION | REJECT | PPK | – | gagal, G-REJECT | lulus |
| UT-FSM-035 | REJECT with USER target from PPK step → success | IN_PPK_VALIDATION | REJECT | PPK | USER | sukses | lulus |
| UT-FSM-036 | REJECT with PPK target from Ppspm step → success | IN_PPSPM_APPROVAL | REJECT | PPSPM | PPK | sukses | lulus |
| UT-FSM-037 | RESUBMIT with USER target → success | NEED_REVISION | RESUBMIT | PEGAWAI | USER | sukses | lulus |
| UT-FSM-038 | RESUBMIT with PPK target → error | NEED_REVISION | RESUBMIT | PEGAWAI | PPK | gagal, G-RESUBMIT | lulus |
| UT-FSM-039 | RESUBMIT_PPK with PPK target → success | NEED_REVISION | RESUBMIT_PPK | PPK | PPK | sukses | lulus |
| UT-FSM-040 | RESUBMIT_PPK with USER target → error | NEED_REVISION | RESUBMIT_PPK | PPK | USER | gagal, G-RESUBMIT_PPK | lulus |
| UT-FSM-041 | DRAFT + APPROVE → error | DRAFT | APPROVE | PPK | – | gagal, G-AKTOR | lulus |
| UT-FSM-042 | DRAFT + REJECT → error | DRAFT | REJECT | PPK | USER | gagal, G-AKTOR | lulus |
| UT-FSM-043 | COMPLETED + SUBMIT → error | COMPLETED | SUBMIT | PEGAWAI | – | gagal, G-TABEL | lulus |
| UT-FSM-044 | NEED_REVISION + APPROVE → error | NEED_REVISION | APPROVE | PPK | – | gagal, G-AKTOR | lulus |
| UT-FSM-045 | NEED_REVISION + REJECT → error (use RESUBMIT) | NEED_REVISION | REJECT | PPK | USER | gagal, G-AKTOR | lulus |
| UT-FSM-046 | IN_PPK_VALIDATION + RESUBMIT → error | IN_PPK_VALIDATION | RESUBMIT | PEGAWAI | USER | gagal, G-TABEL | lulus |
| UT-FSM-047 | IN_PPSPM_APPROVAL + SUBMIT → error | IN_PPSPM_APPROVAL | SUBMIT | PEGAWAI | – | gagal, G-TABEL | lulus |
| UT-FSM-048 | All failures preserve original status | DRAFT | APPROVE | PPK | – | gagal; newStatus tetap DRAFT; newCurrentStep, newRevisionTarget, stepUrutan null; G-AKTOR | lulus |
| UT-FSM-049 | Error message is descriptive | DRAFT | APPROVE | PPK | – | gagal; pesan memuat "tidak bisa" dan sama persis dengan G-AKTOR | lulus |
| UT-FSM-050 | IN_PPK_VALIDATION + REJECT (target X) → error | IN_PPK_VALIDATION | REJECT | PPK | `'X'` | gagal, G-REJECT | lulus |
| UT-FSM-051 | IN_PPSPM_APPROVAL + REJECT by PPK → error | IN_PPSPM_APPROVAL | REJECT | PPK | PPK | gagal, G-AKTOR | lulus |
| UT-FSM-052 | DRAFT + X (unknown action) → error | DRAFT | `'X' as FSMAction` | PEGAWAI | – | gagal, G-AKTOR (aksi `'X'`) | lulus |

Pada kasus sukses 001–008, keempat field hasil (newStatus, newCurrentStep, newRevisionTarget, stepUrutan) diperiksa. Pada kasus sukses lain hanya `success === true` dan `error` kosong yang diperiksa. Pada setiap kasus gagal, tes memeriksa `success === false` dan isi `error` secara persis.

### Hasil `pnpm test` penuh

`pnpm test` dijalankan tiga kali tanpa perubahan kode di antaranya. Run 1 dan 2 berjalan tanpa kontainer Postgres. Sebelum run 3, Postgres dinyalakan dengan `docker compose -f infra/docker/postgres/docker-compose.yml up -d` (kontainer `kepser-postgres`, status `healthy`), sesuai lingkungan pengembangan yang semestinya. **Run 3 adalah run resmi.**

| Run | Postgres | Mulai | Berkas tes | Kasus lulus | Gagal | Dilewati | Total kasus | Durasi |
|---|---|---|---|---:|---:|---:|---:|---|
| 1 | mati | 20:46:40 | 114 lulus, 1 gagal (dari 115) | 1.223 | 1 | 1 | 1.225 | 68,89 dtk |
| 2 | mati | 20:48:04 | 115 lulus (dari 115) | 1.224 | 0 | 1 | 1.225 | 35,08 dtk |
| **3** | **jalan** | 20:54:57 | **115 lulus (dari 115)** | **1.224** | **0** | **1** | **1.225** | 42,91 dtk |

- Kasus yang gagal di run 1 bukan kasus FSM, melainkan `tests/unit/dokumen/master-data-read-requires-session.test.ts` › `master-fungsi returns 401 and reads nothing without a session`. Galatnya `Test timed out in 5000ms` (kasus berjalan 5.553 ms).
- Kasus yang sama lulus dalam 1.623 ms di run 2 dan 2.296 ms di run 3.
- Ini kasus yang sama yang tercatat pernah gagal sesaat di `penjelasan-proyek-aplikasi.md` dan dibahas di laporan kelayakan 4.3 (impor dingin modul rute di dalam badan tes). Penyebab pastinya, termasuk apakah ada kaitannya dengan Postgres yang mati, [BELUM TERVERIFIKASI]. Run 2 lulus walaupun Postgres juga mati.
- Dengan `maxWorkers: 4`, ketiga run selesai tanpa crash kehabisan memori.
- Kasus yang dilewati tetap satu: `tests/unit/arsiparis/berkas-arsip-folder-pages.test.ts` (`it.skip`).
- Total 1.225 = 1.222 (run sebelumnya) + 3 kasus FSM baru.

`npx tsc --noEmit -p .` juga dijalankan setelah perubahan dan selesai tanpa galat (exit 0).

---

## f. Berkas yang diubah

| Berkas | Perubahan |
|---|---|
| `tests/fsm.test.ts` | (1) Tiga kasus baru di blok `describe('Additional Decision Outcomes')`: UT-FSM-050, 051, dan 052. (2) `assertError(result, expectedError)` sekarang mencocokkan `error` secara persis. Sebelumnya helper ini hanya memeriksa bahwa `error` berupa string. Semua 23 pemanggilan `assertError` yang sudah ada (termasuk `it.each` KEMBALIKAN) diberi pesan yang diharapkan. (3) UT-FSM-048 dan 049 diberi asersi pesan persis; asersi lama tetap ada. (4) Konstanta dan fungsi pembentuk pesan galat ditambahkan, juga `import type { FSMAction }`. Masukan dan nama kasus yang sudah ada tidak berubah. |
| `vitest.config.ts` | Blok `coverage` dipertahankan. Ditambah `coverage.experimentalAstAwareRemapping: true` dan `test.maxWorkers: 4`. |
| `.gitignore` | Ditambah `coverage/` dan `coverage-ast/`. |
| `package.json`, `pnpm-lock.yaml` | Tidak diubah di pekerjaan ini. Isinya `@vitest/coverage-v8@3.2.4` dari pemeriksaan kelayakan sebelumnya, dan dipertahankan sesuai permintaan. |
| `docs/skripsi/penjelasan-proyek-aplikasi.md` | Bagian C.4: angka run, riwayat, jumlah kasus `fsm.test.ts` (49 → 52), dan rincian per kelompok. |
| `hasil-unit-whitebox-fsm.md` (baru) | Berkas ini. |

Tidak diubah: semua berkas di `src/`. Berkas `laporan-kelayakan-unit-whitebox.md` dan `lampiran-kasus-uji-unit.csv` juga tidak diubah, sehingga keduanya masih mencatat 49 kasus FSM dan 1.222 kasus total.

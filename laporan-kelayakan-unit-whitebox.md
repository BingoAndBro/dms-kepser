# Laporan Kelayakan Unit Testing White-box

Tanggal pemeriksaan: 2 Oktober 2026. Repo: `kepser-postgres-migration`, branch `migration/postgres-local`, commit `4da7ed1a186fdb3788e6e0b6918dbd41e1d3ed17`.

## Ringkasan

**Jawaban singkat: belum cukup untuk klaim "unit testing white-box dengan statement dan branch coverage" atas seluruh `src/`.** Klaim yang masih bisa dipertanggungjawabkan saat ini lebih sempit, yaitu lapisan logika sisi server (`src/lib` tanpa modul klien, konstanta, dan tipe) ditambah basis path testing untuk `src/lib/fsm.ts`.

| Hal | Temuan |
|---|---|
| Penyedia coverage | Belum terpasang sebelum pemeriksaan. Sekarang dipasang `@vitest/coverage-v8@3.2.4`, versinya disamakan dengan `vitest@3.2.4`. |
| Hasil tes | 115 berkas tes, 1.222 kasus: **1.221 lulus, 0 gagal, 1 dilewati** (konsisten di 3 run yang selesai). |
| Coverage seluruh `src/` (mode AST, definisi gaya Istanbul) | Statement **30,42%**, branch **24,93%**, function 24,08%, line 31,69%. |
| Coverage seluruh `src/` (mode bawaan v8) | Statement 28,75%, branch **73,60%**, function 63,47%. **Angka branch mode bawaan tidak bisa dipakai sebagai bukti branch coverage** (bagian 1.2). |
| `src/lib` sisi server, 98 berkas (mode AST) | Statement **74,45%**, branch **69,28%**. |
| `src/routes/api`, 97 berkas (mode AST) | Statement 43,90%, branch 38,58%. Ada 33 berkas rute API yang 0%. |
| `src/lib/fsm.ts` | Statement 96%, branch 97,22% (35/36). V(G) `transition()` = 7 (per keputusan) atau 11 (per kondisi). **10 dari 11 jalur independen sudah punya kasus uji.** Ada 1 kombinasi kondisi majemuk di `isActorValidForAction` yang belum diuji, dan cabang `default` pada L153 tidak tertutup. |
| Sifat tes | 54 berkas unit terisolasi, 30 menjalankan handler rute bersama banyak modul, 20 *static source guard* (membaca teks sumber, tidak mengeksekusi kode), 11 memakai sistem berkas nyata. |
| Risiko demonstrasi | (1) `pnpm test` dengan jumlah worker bawaan **crash kehabisan memori** di mesin ini saat RAM sedang terpakai banyak. (2) Kasus pertama `master-data-read-requires-session` butuh ±1,2 detik untuk impor dingin. Ini masih jauh di bawah batas 5 detik, tetapi bisa mendekatinya bila mesin sedang sangat sibuk. |

---

## 0. Yang saya ubah (belum di-commit)

| Berkas | Perubahan |
|---|---|
| `package.json` | Menambah devDependency `"@vitest/coverage-v8": "3.2.4"`. |
| `pnpm-lock.yaml` | Diperbarui oleh `pnpm add -D @vitest/coverage-v8@3.2.4`. Ada 46 paket baru, termasuk `ast-v8-to-istanbul@0.3.12`. |
| `vitest.config.ts` | Menambah blok `test.coverage`: `provider: 'v8'`, `all: true`, `include: ['src/**/*.{ts,tsx}']`, `exclude: ['src/**/*.test.{ts,tsx}', 'src/**/*.spec.{ts,tsx}', 'src/**/*.d.ts']`, `reporter: ['text', 'json-summary', 'json', 'html']`, `reportsDirectory: './coverage'`. Blok ini hanya aktif bila `--coverage` dipakai, jadi `pnpm test` biasa tidak berubah perilakunya. Reporter `json` saya tambahkan di luar permintaan (text, json-summary, html) karena nomor baris cabang yang belum tertutup hanya tersedia di `coverage-final.json`. |
| `coverage/` (baru) | Laporan mode bawaan v8 (text, json-summary, json, html). |
| `coverage-ast/` (baru) | Laporan mode AST (`--coverage.experimentalAstAwareRemapping=true`). |
| `lampiran-kasus-uji-unit.csv` (baru) | Lampiran per kasus uji (bagian 5). |
| `laporan-kelayakan-unit-whitebox.md` (baru) | Berkas ini. |

Tidak ada berkas di `src/` maupun `tests/` yang diubah (`git status --short src tests` kosong). `coverage/` dan `coverage-ast/` belum ada di `.gitignore`. Tambahkan ke `.gitignore` atau hapus sebelum commit.

Perintah yang dijalankan:

```bash
pnpm test --coverage --maxWorkers=2 --minWorkers=1 --reporter=default --reporter=json --outputFile.json=<scratch>/vitest-run1.json
```
```bash
pnpm test --coverage --coverage.experimentalAstAwareRemapping=true --coverage.reportsDirectory=./coverage-ast --maxWorkers=2 --minWorkers=1 --reporter=default --reporter=json --outputFile.json=<scratch>/vitest-run2.json
```
```bash
pnpm test --maxWorkers=4 --minWorkers=1 --reporter=default --reporter=json --outputFile.json=<scratch>/vitest-run4.json
```
```bash
npx vitest run tests/fsm.test.ts --coverage --coverage.include=src/lib/fsm.ts --coverage.experimentalAstAwareRemapping=true
```

`--maxWorkers` hanyalah opsi saat run dan tidak mengubah tes. Opsi ini perlu karena run dengan worker bawaan gagal (bagian 1.3).

---

## 1. Coverage

### 1.1 Konfigurasi

- **Sebelum pemeriksaan:** coverage belum dikonfigurasi. `vitest.config.ts` hanya berisi `environment`, `include: ['tests/**/*.test.ts']`, dan `exclude` untuk `tests/integration/**`. Tidak ada paket `@vitest/coverage-*` di `node_modules`.
- **Sesudah:** penyedia **V8** (`@vitest/coverage-v8@3.2.4`) dengan konfigurasi seperti di bagian 0. Semua 384 berkas `src/**/*.{ts,tsx}` ikut dihitung, termasuk yang tidak pernah diimpor tes. Tidak ada berkas tes di dalam `src/`.
- Penyedia V8 di Vitest 3.2.4 punya dua cara memetakan data V8 ke format Istanbul. Hasil keduanya sangat berbeda, jadi saya menjalankan keduanya:
  - **mode bawaan** (`v8-to-istanbul`), dan
  - **mode AST** (`experimentalAstAwareRemapping: true`, memakai `ast-v8-to-istanbul`). Vitest sendiri menandai opsi ini *experimental*, dengan keterangan "more accurate results compared to default mode".

### 1.2 Apa yang dihitung sebagai "branch" (setara decision atau condition coverage?)

**Mode bawaan (v8-to-istanbul):**

- **"Statement" = baris.** Total statement (58.716) persis sama dengan total line (58.716), jadi statement coverage di mode ini sebenarnya line coverage.
- **"Branch" = rentang blok yang dilaporkan V8**, termasuk rentang badan fungsi itu sendiri. Contoh untuk `fsm.ts`: cabang #0 adalah seluruh badan `makeError` (L70–79), #1 seluruh `transition` (L81–127), dan #17 seluruh `isActorValidForAction`. Fungsi yang tidak pernah dipanggil tidak menyumbang cabang ke penyebut. Mekanisme internal V8-nya [BELUM TERVERIFIKASI dari kode sumber V8], tetapi efeknya terlihat jelas pada data:
  - `src/lib/dokumen/pembersihan-service.ts`: statement 19,42%, **branch 18/18 = 100%**. Di mode AST nilainya 16/64 = 25%.
  - `src/lib/api-client.ts`: statement 3,73%, branch **0/0, dilaporkan 100%**. Mode AST: 0/52.
  - `src/routeTree.gen.ts`: statement 0%, **branch 1/1 = 100%**.
  - 18 berkas di `src/lib` + `src/routes/api` punya 0 cabang di mode bawaan, dan semuanya dilaporkan 100%.
- Kesimpulan: **angka branch mode bawaan bukan decision coverage dan bukan condition coverage.** Angkanya menggelembung: 73,60% di mode bawaan, sedangkan mode AST memberi 24,93% untuk kode yang sama.

**Mode AST (ast-v8-to-istanbul, semantik Istanbul):** jenis cabang yang muncul di data (`branchMap.type`):

| Jenis | Konstruksi | Lokasi yang dihitung |
|---|---|---|
| `if` | `if` / `if-else`, termasuk `if` tanpa `else` | 2 lokasi: hasil benar dan hasil salah |
| `cond-expr` | operator ternary `a ? b : c` | 2 lokasi |
| `switch` | `switch` | 1 lokasi per `case`, termasuk `default` |
| `binary-expr` | operand `&&`, `\|\|`, `??` | 1 lokasi **per operand** |
| `default-arg` | nilai bawaan parameter | 1 lokasi |

- `if`, `cond-expr`, dan `switch` **setara decision coverage**: setiap hasil keputusan harus terjadi minimal sekali.
- `binary-expr` dihitung tertutup bila operand itu **pernah dievaluasi** (tidak dilompati short-circuit), **bukan** bila operand pernah bernilai benar dan salah. Jadi ini **bukan condition coverage**. Buktinya ada di `fsm.ts`:
  - L96 `!revisionTarget || (revisionTarget !== 'USER' && revisionTarget !== 'PPK')`: ketiga operand berstatus tertutup (hitungan 7, 6, 3 dari `fsm.test.ts` saja). Padahal kondisi ketiga belum pernah bernilai **benar**, karena tidak ada kasus dengan `revisionTarget` selain `USER`/`PPK`/kosong.
  - L144–145 (`REJECT`): keempat operand berstatus tertutup, padahal `role === PPSPM` tidak pernah bernilai **salah** saat `status === IN_PPSPM_APPROVAL`.
- **Ringkasnya:** branch coverage mode AST = decision coverage + "setiap operand logika pernah dievaluasi". Ukuran ini lebih kuat daripada decision coverage murni, tetapi **lebih lemah daripada condition coverage** maupun decision/condition coverage dalam istilah Myers. Untuk `fsm.ts`, condition coverage dibuktikan manual di bagian 3.8.
- Di mode AST, "statement" adalah statement AST yang sebenarnya (18.099), dan "line" adalah baris yang memuat statement (16.250).

**Rekomendasi:** angka yang dikutip di skripsi sebaiknya dari mode AST, dengan menyebut "Vitest 3.2.4, penyedia V8, pemetaan AST (experimentalAstAwareRemapping)". Alternatif yang tidak berlabel *experimental* adalah `@vitest/coverage-istanbul`, yang memakai instrumentasi Istanbul dengan definisi cabang yang sama [angkanya BELUM TERVERIFIKASI; tidak dijalankan].

### 1.3 Data run

| Item | Nilai |
|---|---|
| Commit | `4da7ed1a186fdb3788e6e0b6918dbd41e1d3ed17` (`src/` dan `tests/` bersih terhadap commit ini) |
| Alat | Vitest 3.2.4, Node v24.14.0, pnpm 10.33.0, Windows 11 Home (12 prosesor logis) |
| Berkas tes dalam `pnpm test` | **115**: 114 di `tests/unit/**` dan `tests/fsm.test.ts`. `tests/integration/` (2 berkas) dan `tests/e2e/` (3 spesifikasi Playwright) tidak termasuk. |
| Kasus | **1.222**: 1.221 lulus, 0 gagal, 1 dilewati, 0 todo. Jumlah suite yang dilaporkan Vitest (berkas + describe) 289. |
| `it.each` | 35 pemakaian di 20 berkas. Setiap baris dihitung sebagai satu kasus dalam angka 1.222. |

| Run | Mode | Worker | Hasil | Durasi |
|---|---|---|---|---|
| 0 | coverage v8 bawaan | bawaan (±11) | **crash**: `VirtualAlloc failed` / `JavaScript heap out of memory` | — |
| 1 | coverage v8 bawaan | 2 | 115 berkas lulus; 1.221 lulus, 1 dilewati | 84,36 dtk |
| 2 | coverage AST | 2 | 115 berkas lulus; 1.221 lulus, 1 dilewati | 60,90 dtk |
| 3 | tanpa coverage (`pnpm test` murni) | bawaan | **crash**, kehabisan memori | — |
| 4 | tanpa coverage | 4 | 115 berkas lulus; 1.221 lulus, 1 dilewati | 34,12 dtk |

Saat pemeriksaan, memori virtual yang tersisa hanya 2.155 MB dari 27.801 MB (memori fisik bebas 1.432 MB dari 10.085 MB). Crash itu adalah kegagalan alokasi memori proses Node, bukan kegagalan tes. Dampaknya untuk demonstrasi dibahas di bagian 4.3.

### 1.4 Tabel coverage per lapisan

Persentase dihitung dari covered/total dan dibulatkan ke 2 desimal. Keluaran teks Vitest memotong (bukan membulatkan) angka, jadi bisa berbeda 0,01.

**Tabel 1.A — mode AST (disarankan untuk dikutip)**

| Lapisan | Berkas | Statement % | Branch % | Function % | Line % |
|---|---:|---:|---:|---:|---:|
| src/lib/ (berkas langsung) | 19 | 53.28 (284/533) | 41.46 (204/492) | 64.15 (68/106) | 55.19 (271/491) |
| src/lib/archive/ | 15 | 76.15 (907/1191) | 71.84 (704/980) | 80.77 (210/260) | 78.93 (824/1044) |
| src/lib/auth/ | 9 | 65.22 (165/253) | 59.28 (99/167) | 67.16 (45/67) | 66.67 (152/228) |
| src/lib/constants/ | 4 | 80.77 (21/26) | 0.00 (0/6) | 0.00 (0/2) | 84.00 (21/25) |
| src/lib/dokumen/ | 14 | 70.69 (316/447) | 72.14 (259/359) | 75.81 (94/124) | 73.01 (303/415) |
| src/lib/export/ | 3 | 90.30 (149/165) | 80.77 (63/78) | 88.57 (31/35) | 91.61 (142/155) |
| src/lib/laporan/ | 5 | 86.19 (181/210) | 80.84 (173/214) | 92.45 (49/53) | 89.56 (163/182) |
| src/lib/master-data/ | 3 | 0.00 (0/34) | 0.00 (0/32) | 0.00 (0/1) | 0.00 (0/25) |
| src/lib/schemas/ | 9 | 91.81 (157/171) | 87.29 (103/118) | 96.88 (31/32) | 94.30 (149/158) |
| src/lib/security/ | 1 | 90.16 (55/61) | 90.00 (27/30) | 100.00 (11/11) | 92.16 (47/51) |
| src/lib/storage/ | 19 | 73.64 (1137/1544) | 69.47 (728/1048) | 82.03 (210/256) | 75.03 (1094/1458) |
| src/lib/types/ | 3 | 71.43 (5/7) | 100.00 (2/2) | 75.00 (3/4) | 71.43 (5/7) |
| src/lib/upload/ | 1 | 91.80 (56/61) | 76.74 (33/43) | 90.91 (10/11) | 94.83 (55/58) |
| src/lib/users/ | 4 | 42.22 (95/225) | 38.60 (88/228) | 51.06 (24/47) | 41.59 (89/214) |
| src/lib/utils/ | 4 | 83.61 (51/61) | 79.41 (27/34) | 92.86 (13/14) | 86.79 (46/53) |
| **subtotal src/lib** | **113** | **71.74 (3579/4989)** | **65.52 (2510/3831)** | **78.10 (799/1023)** | **73.64 (3361/4564)** |
| src/routes/api | 97 | 43.90 (1682/3831) | 38.58 (1042/2701) | 50.94 (189/371) | 45.77 (1562/3413) |
| src/db | 32 | 40.58 (84/207) | 46.43 (13/28) | 9.76 (8/82) | 39.71 (81/204) |
| src/components | 65 | 2.10 (48/2287) | 0.47 (11/2326) | 0.61 (5/826) | 2.26 (46/2038) |
| src/routes (selain api) | 68 | 1.54 (97/6293) | 1.31 (75/5728) | 1.39 (25/1795) | 1.57 (87/5547) |
| lainnya: src/hooks/ | 4 | 20.00 (14/70) | 10.71 (3/28) | 35.71 (5/14) | 19.05 (12/63) |
| lainnya: src/scripts/ | 2 | 0.00 (0/32) | 0.00 (0/16) | 0.00 (0/6) | 0.00 (0/31) |
| lainnya: src/config/ | 1 | 100.00 (1/1) | – (0/0) | – (0/0) | 100.00 (1/1) |
| lainnya: src/router.tsx | 1 | 0.00 (0/3) | – (0/0) | 0.00 (0/2) | 0.00 (0/3) |
| lainnya: src/routeTree.gen.ts | 1 | 0.00 (0/386) | – (0/0) | 0.00 (0/163) | 0.00 (0/386) |
| **subtotal lainnya** | **9** | **3.05 (15/492)** | **6.82 (3/44)** | **2.70 (5/185)** | **2.69 (13/484)** |
| **TOTAL src/** | **384** | **30.42 (5505/18099)** | **24.93 (3654/14658)** | **24.08 (1031/4282)** | **31.69 (5150/16250)** |

Subset untuk pertimbangan cakupan klaim (mode AST):

| Subset | Berkas | Statement % | Branch % | Function % | Line % |
|---|---:|---:|---:|---:|---:|
| src/lib sisi server: src/lib tanpa 8 modul klien¹, `constants/`, dan `types/` | 98 | **74.45** (3514/4720) | **69.28** (2485/3587) | 80.51 (789/980) | 76.54 (3299/4310) |
| src/lib + src/routes/api | 210 | 59.65 (5261/8820) | 54.38 (3552/6532) | 70.88 (988/1394) | 61.71 (4923/7977) |
| src/lib sisi server + src/routes/api | 195 | 60.76 (5196/8551) | 56.09 (3527/6288) | 72.39 (978/1351) | 62.94 (4861/7723) |
| 8 modul klien¹ saja | 8 | 16.53 (39/236) | 9.75 (23/236) | 18.92 (7/37) | 16.22 (36/222) |

¹ Modul klien: `api-client.ts`, `api-mutation.ts`, `auth-state.ts`, `guards.ts`, `file-helpers.ts`, `storage-client.ts`, `workspace-label.ts`, `dev-logger.ts`. Penentuannya dari isi kode (fetch dari peramban, `redirect` router, unduhan, state klien) dan dari pengimpornya, yang sebagian besar komponen atau halaman UI.

**Tabel 1.B — mode bawaan v8 (pembanding saja; branch tidak valid, lihat 1.2)**

| Lapisan | Berkas | Statement % | Branch % | Function % | Line % |
|---|---:|---:|---:|---:|---:|
| src/lib/ (berkas langsung) | 19 | 70.46 (1159/1645) | 74.35 (200/269) | 72.62 (61/84) | 70.46 (1159/1645) |
| src/lib/archive/ | 15 | 78.97 (2636/3338) | 78.65 (641/815) | 80.89 (182/225) | 78.97 (2636/3338) |
| src/lib/auth/ | 9 | 64.23 (404/629) | 88.49 (123/139) | 76.79 (43/56) | 64.23 (404/629) |
| src/lib/constants/ | 4 | 95.00 (171/180) | – (0/0) | 0.00 (0/1) | 95.00 (171/180) |
| src/lib/dokumen/ | 14 | 80.09 (1199/1497) | 87.50 (280/320) | 86.73 (85/98) | 80.09 (1199/1497) |
| src/lib/export/ | 3 | 93.94 (341/363) | 90.53 (86/95) | 95.65 (22/23) | 93.94 (341/363) |
| src/lib/laporan/ | 5 | 90.95 (412/453) | 86.36 (152/176) | 92.50 (37/40) | 90.95 (412/453) |
| src/lib/master-data/ | 3 | 5.17 (3/58) | 50.00 (1/2) | 33.33 (1/3) | 5.17 (3/58) |
| src/lib/schemas/ | 9 | 95.92 (588/613) | 84.21 (96/114) | 89.47 (17/19) | 95.92 (588/613) |
| src/lib/security/ | 1 | 89.16 (74/83) | 84.09 (37/44) | 100.00 (11/11) | 89.16 (74/83) |
| src/lib/storage/ | 19 | 74.74 (2704/3618) | 80.18 (720/898) | 85.17 (178/209) | 74.74 (2704/3618) |
| src/lib/types/ | 3 | 75.00 (12/16) | 83.33 (5/6) | 66.67 (4/6) | 75.00 (12/16) |
| src/lib/upload/ | 1 | 93.92 (139/148) | 80.00 (32/40) | 88.89 (8/9) | 93.92 (139/148) |
| src/lib/users/ | 4 | 38.63 (231/598) | 56.18 (50/89) | 56.25 (18/32) | 38.63 (231/598) |
| src/lib/utils/ | 4 | 87.50 (98/112) | 73.17 (30/41) | 100.00 (13/13) | 87.50 (98/112) |
| **subtotal src/lib** | **113** | **76.18 (10171/13351)** | **80.48 (2453/3048)** | **82.03 (680/829)** | **76.18 (10171/13351)** |
| src/routes/api | 97 | 48.66 (5158/10599) | 62.86 (853/1357) | 61.42 (156/254) | 48.66 (5158/10599) |
| src/db | 32 | 60.92 (717/1177) | 69.23 (18/26) | 10.91 (6/55) | 60.92 (717/1177) |
| src/components | 65 | 3.15 (323/10240) | 50.00 (29/58) | 15.38 (20/130) | 3.15 (323/10240) |
| src/routes (selain api) | 68 | 1.44 (311/21649) | 44.38 (71/160) | 17.36 (21/121) | 1.44 (311/21649) |
| **subtotal lainnya** (hooks, scripts, config, router, routeTree) | **9** | **11.82 (201/1700)** | **57.14 (8/14)** | **50.00 (5/10)** | **11.82 (201/1700)** |
| **TOTAL src/** | **384** | **28.75 (16881/58716)** | **73.60 (3432/4663)** | **63.47 (888/1399)** | **28.75 (16881/58716)** |

Contoh dampak pemilihan mode: subtotal `src/lib` di mode bawaan sudah "lulus 80% branch" (80,48%), padahal di mode AST nilainya 65,52%. `src/components` tampak 50% branch di mode bawaan, padahal hanya 0,47% di mode AST.

---

## 2. Celah coverage di `src/lib` dan `src/routes/api`

Kriteria: branch coverage **mode AST** di bawah 80%, diurutkan dari yang terendah (bila sama, cabang terbanyak lebih dulu). Berkas dengan 0 cabang (32 berkas) tidak dimasukkan karena persentase branch-nya tidak bermakna. Hasilnya **124 dari 210 berkas**. Sebagai pembanding, mode bawaan memberi 120 berkas di bawah 80%.

"Baris cabang belum tertutup" adalah baris awal setiap lokasi cabang dengan hitungan 0. Untuk `if`, hasil salah tanpa `else` dicatat pada baris `if`-nya.

Ringkasan penilaian: **112 berkas "perlu tes tambahan"** dan **12 berkas "wajar dikecualikan"**. Yang wajar dikecualikan adalah 7 modul klien, `user-response.ts`, 2 pembungkus query Drizzle tanpa logika (`local-user-queries.ts`, `session-repository.ts`; lebih tepat diuji di `tests/integration`), `workspace-label.ts`, dan utilitas reset data pengembangan.

Pola yang paling sering:

- **47 berkas 0%**, karena tidak pernah diimpor tes. Dari jumlah itu, 33 rute API (daftar inbox per peran, `users/*`, `settings/*`, `ketua-tim/*`, `master-*.$id`) dan 14 modul lib. Yang paling penting adalah `src/lib/auth/local-server-auth.ts`: modul ini menjadi penjaga sesi untuk hampir semua rute, tetapi di setiap tes selalu di-mock. Berikutnya `src/lib/master-data/kelengkapan-chain.ts`, validasi rantai master yang punya 13 titik return.
- **11 rute master data CRUD** (8–12%). Tes yang ada hanya menguji penjaga sesi pada GET (D-23). Validasi POST/PATCH/DELETE, konflik 409, dan 404 belum tersentuh.
- **Rute transisi dokumen** (`ppk/dokumen/$id/approve|reject`, `ppspm/dokumen/$id/approve|reject`, `ppk/kembalikan/$id`, `dokumen.$id.submit`, `ppk/resubmit/$id`) di kisaran 38–65%. Tes yang ada fokus pada konflik 409 dan rollback. Rantai penjaga 401/403/400/404/500 belum diuji.
- **Cabang yang tidak tercapai:** di rute approve/reject/kembalikan, `if (!result.success)` setelah pengecekan status tidak bisa bernilai benar karena status sudah dicek sesuai aturan FSM. Cabang ini wajar dikecualikan, misalnya dengan anotasi `/* v8 ignore next */` disertai alasan, bila rute ikut dalam lingkup klaim.

| No | Berkas | Branch (AST) | Branch (v8 bawaan) | Stmt (AST) | Baris cabang belum tertutup | Penilaian | Kasus uji yang kurang / alasan |
|---:|---|---:|---:|---:|---|---|---|
| 1 | `src/routes/api/users/me.ts` | 0% (0/88) | 0% (0/1) | 0% | 30, 45, 51-52, 54-55, 62-65, 67, 82-84, 90, 93, 97, 101, 107, 119, 163, 177, 180, 184, 188, 194, 215, 241, 262-265, 267, 286, 293, 303, 317, 322-323, 325-326, 328-330, 332-334, 340 | **perlu tes tambahan** | GET profil: tanpa sesi 401, user tidak ada 404; POST/DELETE avatar: bukan multipart 400, MIME/ukuran salah 400, storage tidak tersedia 503, galat 500 |
| 2 | `src/routes/api/master-kelengkapan.$id.ts` | 0% (0/84) | 0% (0/1) | 0% | 17-18, 35, 37-40, 51-52, 54-55, 57-58, 60-61, 65-67, 75, 86, 94, 111, 118-119, 121-122, 124-125, 127-128, 130, 138-139, 141-142, 144-145, 147-148, 150-151, 154, 161-167, 187, 204, 206, 208, 220, 224, 235 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; id tidak ada → 404; PATCH body tidak valid → 400; PATCH nama duplikat → 409; reaktivasi saat induk nonaktif → 400; DELETE masih dipakai → 409; galat DB → 500; validasi rantai kelengkapan gagal → 400 |
| 3 | `src/routes/api/users/$id.ts` | 0% (0/58) | 0% (0/1) | 0% | 28, 34, 38, 45, 58, 61, 67, 71, 83, 90, 95, 98-99, 103-104, 108, 111-112, 116, 120, 127, 131, 140 | **perlu tes tambahan** | non-ADMIN 403; id bukan UUID 400; user tidak ada 404; PATCH: username/NIP tidak valid 400, campur ADMIN + peran lain 400, demosi diri 400, duplikat 409 |
| 4 | `src/lib/api-client.ts` | 0% (0/52) | 100% (0/0) | 1.88% | 28-29, 31, 37, 41, 43, 45, 52, 59, 64-65, 72, 76, 80, 84, 89, 101, 103, 108, 113, 117, 125, 137 | **wajar dikecualikan** | kode sisi peramban (fetch/redirect/unduh/sessionStorage) yang dipakai komponen UI; di luar lingkup unit sisi server, perilakunya diuji lewat e2e |
| 5 | `src/lib/guards.ts` | 0% (0/48) | 0% (0/1) | 0% | 13, 16, 19, 21, 23-25, 29, 44, 46-48, 53, 58, 60, 62 | **wajar dikecualikan** | kode sisi peramban (fetch/redirect/unduh/sessionStorage) yang dipakai komponen UI; di luar lingkup unit sisi server, perilakunya diuji lewat e2e (redirect TanStack Router) |
| 6 | `src/lib/api-mutation.ts` | 0% (0/41) | 0% (0/1) | 0% | 16, 20, 24, 28, 32, 36, 46, 48, 58, 61, 66, 72, 75 | **wajar dikecualikan** | kode sisi peramban (fetch/redirect/unduh/sessionStorage) yang dipakai komponen UI; di luar lingkup unit sisi server, perilakunya diuji lewat e2e |
| 7 | `src/lib/users/local-user-queries.ts` | 0% (0/40) | 100% (0/0) | 0% | 47, 83, 85, 89, 96-97, 99-101, 110, 113-116, 126-130, 135, 143 | **wajar dikecualikan** | hanya membungkus query Drizzle tanpa logika selain pemetaan; lebih tepat diuji di tests/integration dengan Postgres nyata |
| 8 | `src/routes/api/master-kegiatan.$id.ts` | 0% (0/40) | 0% (0/1) | 0% | 11-12, 23, 34, 42, 54, 58, 68, 73-74, 86, 93-95, 111, 123, 134, 138, 146 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; id tidak ada → 404; PATCH body tidak valid → 400; PATCH nama duplikat → 409; reaktivasi saat induk nonaktif → 400; DELETE masih dipakai → 409; galat DB → 500 |
| 9 | `src/routes/api/ketua-tim/kegiatan/$kegiatanId.ts` | 0% (0/40) | 0% (0/1) | 0% | 13, 17, 29, 48, 57-58, 67, 70, 90, 100-101, 112, 114, 117, 129, 142, 163-164 | **perlu tes tambahan** | tanpa sesi 401; bukan ADMIN/KSBU 403; PATCH body tidak valid 400; pegawai bukan anggota 400; galat 500 |
| 10 | `src/routes/api/laporan/saya.ts` | 0% (0/38) | 0% (0/1) | 0% | 17-18, 21, 35, 93-102, 104-110 | **perlu tes tambahan** | tanpa sesi 401; galat 500; filter periode/status dari query string diteruskan benar |
| 11 | `src/lib/master-data/kelengkapan-chain.ts` | 0% (0/32) | 100% (0/0) | 0% | 20-23, 25-27, 34-35, 42, 49-50, 54, 60-61, 75 | **perlu tes tambahan** | PRIORITAS: 13 titik return. Kasus: komponen kosong; jenis kosong; kategori kosong; komponen tidak ada; komponen beda kegiatan; jenis tidak ada; kategori tidak ada; kategori beda jenis; detail tidak ada; detail beda kategori; detail valid → null; tanpa detail tetapi kategori punya detail aktif → pesan wajib; tanpa detail dan kategori tanpa detail → null (db di-mock) |
| 12 | `src/routes/api/ketua-tim/index.ts` | 0% (0/32) | 0% (0/1) | 0% | 12, 16, 39, 48-49, 56, 64, 89-90, 101, 103, 113, 135-136, 149, 151, 154, 164 | **perlu tes tambahan** | tanpa sesi 401; peran salah 403; POST body tidak valid 400; kegiatan/pegawai tidak ada 404; sukses 201; galat 500 |
| 13 | `src/lib/auth-state.ts` | 0% (0/28) | 0% (0/1) | 0% | 26, 30, 35, 39, 42, 47-48, 50, 59, 62, 74, 76, 88, 102-103 | **wajar dikecualikan** | kode sisi peramban (fetch/redirect/unduh/sessionStorage) yang dipakai komponen UI; di luar lingkup unit sisi server, perilakunya diuji lewat e2e |
| 14 | `src/routes/api/master-fungsi.$id.ts` | 0% (0/28) | 0% (0/1) | 0% | 11-12, 23, 34, 42, 50, 54, 65, 72-73, 88, 101, 105, 113 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; id tidak ada → 404; PATCH body tidak valid → 400; PATCH nama duplikat → 409; reaktivasi saat induk nonaktif → 400; DELETE masih dipakai → 409; galat DB → 500 |
| 15 | `src/routes/api/activity-log.ts` | 0% (0/26) | 0% (0/1) | 0% | 34-37, 63, 66, 68, 73, 115, 148-151, 154, 159, 162 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON; filter aksi/rentang tanggal; label aksi lewat formatAksiLabel |
| 16 | `src/routes/api/ppspm/inbox.ts` | 0% (0/26) | 0% (0/1) | 0% | 17-18, 21-23, 26-28, 50, 74, 76, 81-82 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON (peran PPSPM). Dapat digabung dalam satu berkas tabel-driven seperti master-data-read-requires-session |
| 17 | `src/lib/auth/session-repository.ts` | 0% (0/25) | 100% (0/0) | 0% | 59-60, 64, 74, 101, 106, 120, 134, 148, 154, 158 | **wajar dikecualikan** | akses tabel sesi lewat Drizzle; logikanya tipis (insert/select/update). Uji di tests/integration |
| 18 | `src/routes/api/kasubag/inbox.ts` | 0% (0/24) | 0% (0/1) | 0% | 19-20, 23-25, 31-33, 63, 86, 88, 96 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON (peran KSBU) |
| 19 | `src/routes/api/ppk/inbox.ts` | 0% (0/20) | 0% (0/1) | 0% | 19, 23, 28-30, 33-35, 62, 64 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON (peran PPK) |
| 20 | `src/routes/api/ketua-tim/$id.ts` | 0% (0/17) | 0% (0/1) | 0% | 11, 15, 27, 35, 37, 40, 55 | **perlu tes tambahan** | cross-origin 403; tanpa sesi 401; peran salah 403; id tidak valid 400; tidak ada 404; galat 500 |
| 21 | `src/lib/auth/local-server-auth.ts` | 0% (0/16) | 100% (0/0) | 0% | 33, 43, 46, 53, 69, 74, 90, 99-101 | **perlu tes tambahan** | PRIORITAS: dipakai hampir semua rute tetapi selalu di-mock. Kasus getLocalServerSession: tanpa cookie → null; repositori melempar → null; sesi tidak ada → null; peran tidak valid → null; peran aktif tak terselesaikan → null; sesi valid → objek sesi. hasAnyLocalRole([]) → false; requireAnyLocalSession tanpa sesi → Response 401 |
| 22 | `src/routes/api/settings/general.ts` | 0% (0/16) | 0% (0/1) | 0% | 19, 32-33, 44, 47-48, 60, 68 | **perlu tes tambahan** | tanpa sesi 401; PUT bukan ADMIN 403; body tidak valid 400; galat 500 |
| 23 | `src/routes/api/users/$id/reset-password.ts` | 0% (0/16) | 0% (0/1) | 0% | 19, 22, 27, 31, 43, 45, 52 | **perlu tes tambahan** | non-ADMIN 403; id bukan UUID 400; kata sandi tidak memenuhi aturan 400; galat 500 |
| 24 | `src/routes/api/pembersihan-dokumen.ts` | 0% (0/14) | 0% (0/1) | 0% | 15-18, 34, 44, 90, 92, 107 | **perlu tes tambahan** | tanpa sesi 401; galat 500; daftar kandidat memakai buildPembersihanPlan |
| 25 | `src/routes/api/auth/role-switch.ts` | 0% (0/14) | 0% (0/1) | 0% | 22, 31, 42, 53, 58, 62, 68 | **perlu tes tambahan** | tanpa sesi 401; body tidak valid 400; peran tidak dimiliki 403; sukses set cookie peran aktif |
| 26 | `src/routes/api/users/$id/deactivate.ts` | 0% (0/14) | 0% (0/1) | 0% | 15, 18, 24, 28, 33, 40-41 | **perlu tes tambahan** | non-ADMIN 403; id bukan UUID 400; menonaktifkan diri sendiri 400; galat 500 |
| 27 | `src/lib/auth/password.ts` | 0% (0/12) | 100% (1/1) | 0% | 26, 38, 42, 48, 54 | **perlu tes tambahan** | hashPassword lalu verifyPassword benar → true; sandi salah → false; hash rusak → false/galat. Murah, tanpa mock |
| 28 | `src/lib/users/local-user-passwords.ts` | 0% (0/12) | 100% (1/1) | 0% | 22, 40, 61, 66, 70, 88 | **perlu tes tambahan** | reset: user tidak ada; change: sandi lama salah → gagal; sukses mencabut sesi lain (db & password di-mock) |
| 29 | `src/routes/api/settings/theme.ts` | 0% (0/12) | 0% (0/1) | 0% | 17, 26, 36, 39-40, 52 | **perlu tes tambahan** | tanpa sesi 401; PUT non-ADMIN 403; tema tidak dikenal 400; galat 500 |
| 30 | `src/routes/api/users/$id/activate.ts` | 0% (0/12) | 0% (0/1) | 0% | 15, 18, 24, 28, 35-36 | **perlu tes tambahan** | non-ADMIN 403; id bukan UUID 400; galat 500 |
| 31 | `src/routes/api/users/$id/avatar.ts` | 0% (0/12) | 0% (0/1) | 0% | 19, 25, 29, 42, 52 | **perlu tes tambahan** | tanpa sesi 401; id tidak valid 400; tanpa avatar/berkas hilang 404 |
| 32 | `src/routes/api/ketua-tim/user/$userId.ts` | 0% (0/10) | 0% (0/1) | 0% | 10, 14, 28, 31, 57-58 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON; userId tidak valid 400 |
| 33 | `src/lib/dokumen/aksi-labels.ts` | 0% (0/8) | 0% (0/1) | 0% | 33, 68-69, 72 | **perlu tes tambahan** | formatAksiLabel aksi dikenal & tidak dikenal (fallback ke kode); resolveAksiRole UPDATE_NOMINAL oleh pembuat → PEGAWAI, oleh orang lain → KSBU; aksi tak dikenal → null. Fungsi murni |
| 34 | `src/routes/api/pegawai/revisi.ts` | 0% (0/8) | 0% (0/1) | 0% | 17, 19, 56-57 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON (peran PEGAWAI) |
| 35 | `src/routes/api/ppk/ditolak.ts` | 0% (0/8) | 0% (0/1) | 0% | 17-18, 49, 51 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON (peran PPK) |
| 36 | `src/routes/api/ppk/revisi.ts` | 0% (0/8) | 0% (0/1) | 0% | 17-18, 46-47 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON (peran PPK) |
| 37 | `src/routes/api/ppk/tervalidasi.ts` | 0% (0/8) | 0% (0/1) | 0% | 17-18, 45, 47 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON (peran PPK) |
| 38 | `src/routes/api/ppspm/ditolak.ts` | 0% (0/8) | 0% (0/1) | 0% | 13-14, 42-43 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON (peran PPSPM) |
| 39 | `src/routes/api/ppspm/selesai.ts` | 0% (0/8) | 0% (0/1) | 0% | 13-14, 38-39 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON (peran PPSPM) |
| 40 | `src/lib/constants/routes.ts` | 0% (0/6) | 100% (0/0) | 44.44% | 83, 88 | **perlu tes tambahan** | getDefaultRouteForRoles: daftar peran kosong dan peran tanpa rute bawaan (2 kasus) |
| 41 | `src/routes/api/auth/session.ts` | 0% (0/6) | 0% (0/1) | 0% | 25, 40, 49 | **perlu tes tambahan** | tanpa sesi → respons anonim; dengan sesi → data user & peran |
| 42 | `src/routes/api/users/me/is-ketua-tim/$kegiatanId.ts` | 0% (0/6) | 0% (0/1) | 0% | 20, 26, 32 | **perlu tes tambahan** | kegiatanId tidak valid 400; bukan ketua → false; ketua → true; galat 500 |
| 43 | `src/lib/storage/local-attachment-reference-cleanup.ts` | 0% (0/4) | 100% (0/0) | 0% | 53, 95 | **perlu tes tambahan** | berkas pengganti masih dirujuk → tidak dihapus; tidak dirujuk → dihapus (filesystem sementara) |
| 44 | `src/routes/api/dokumen/index.ts` | 0% (0/4) | 0% (0/1) | 0% | 19, 23 | **perlu tes tambahan** | tanpa sesi → 401; peran salah → 403; query DB melempar galat → 500; sukses → 200 + bentuk JSON |
| 45 | `src/routes/api/settings/epoch.ts` | 0% (0/4) | 0% (0/1) | 0% | 17, 25 | **perlu tes tambahan** | tanpa sesi 401; galat 500 |
| 46 | `src/routes/api/users/me/ketua-tim.ts` | 0% (0/4) | 0% (0/1) | 0% | 22, 41-42 | **perlu tes tambahan** | galat 500; daftar kegiatan sebagai ketua |
| 47 | `src/lib/workspace-label.ts` | 0% (0/2) | 100% (1/1) | 0% | 10 | **wajar dikecualikan** | satu fungsi label UI 11 baris yang hanya dipakai komponen |
| 48 | `src/routes/api/master-kelengkapan.ts` | 7.81% (5/64) | 60% (6/10) | 22.22% | 17-18, 34, 36-39, 50-51, 53-54, 56-57, 59-60, 64, 81, 85, 108, 127, 130-131, 133-135, 147, 156, 164, 167, 185, 191, 204-207, 223, 231 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; body gagal Zod → 400; induk tidak ada/nonaktif → 400; nama duplikat (unique violation) → 409; sukses → 201; galat DB → 500; rantai kelengkapan tidak valid → 400; filter GET (komponen/jenis/kategori) |
| 49 | `src/routes/api/master-komponen.$id.ts` | 8% (4/50) | 33.33% (2/6) | 10.81% | 11-12, 44, 57-59, 69, 80, 88, 100, 104, 114, 119-120, 132, 139-142, 158, 170, 181, 185, 193 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; id tidak ada → 404; PATCH body tidak valid → 400; PATCH nama duplikat → 409; reaktivasi saat induk nonaktif → 400; DELETE masih dipakai → 409; galat DB → 500 |
| 50 | `src/routes/api/master-detail.ts` | 9.37% (3/32) | 60% (3/5) | 25.53% | 15-16, 35, 73, 77-78, 80-82, 94, 103, 111, 130, 144, 156, 168, 176 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; body gagal Zod → 400; induk tidak ada/nonaktif → 400; nama duplikat (unique violation) → 409; sukses → 201; galat DB → 500; filter GET ?kategoriId |
| 51 | `src/routes/api/master-jenis.$id.ts` | 9.37% (3/32) | 66.66% (2/3) | 13.11% | 11-12, 40, 53, 62, 70, 81, 85, 96, 102-104, 120, 133, 135 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; id tidak ada → 404; PATCH body tidak valid → 400; PATCH nama duplikat → 409; reaktivasi saat induk nonaktif → 400; DELETE masih dipakai → 409; galat DB → 500 |
| 52 | `src/routes/api/master-fungsi.ts` | 11.11% (2/18) | 66.66% (2/3) | 18.91% | 12-13, 52, 61, 69, 81, 90, 101 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; body gagal Zod → 400; induk tidak ada/nonaktif → 400; nama duplikat (unique violation) → 409; sukses → 201; galat DB → 500 |
| 53 | `src/routes/api/master-jenis.ts` | 11.11% (2/18) | 66.66% (2/3) | 18.91% | 11-12, 49, 58, 66, 77, 88, 99 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; body gagal Zod → 400; induk tidak ada/nonaktif → 400; nama duplikat (unique violation) → 409; sukses → 201; galat DB → 500 |
| 54 | `src/routes/api/master-kategori.ts` | 11.53% (3/26) | 60% (3/5) | 25.53% | 11-12, 31, 64-66, 78, 87, 95, 106, 120, 132, 144 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; body gagal Zod → 400; induk tidak ada/nonaktif → 400; nama duplikat (unique violation) → 409; sukses → 201; galat DB → 500; filter GET ?jenisId |
| 55 | `src/routes/api/master-kegiatan.ts` | 11.53% (3/26) | 60% (3/5) | 25.53% | 11-12, 31, 61-63, 75, 84, 92, 103, 117, 129, 141 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; body gagal Zod → 400; induk tidak ada/nonaktif → 400; nama duplikat (unique violation) → 409; sukses → 201; galat DB → 500; filter GET ?fungsiId |
| 56 | `src/routes/api/master-komponen.ts` | 11.53% (3/26) | 60% (3/5) | 25.53% | 11-12, 31, 61-63, 75, 84, 92, 103, 117, 129, 141 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; body gagal Zod → 400; induk tidak ada/nonaktif → 400; nama duplikat (unique violation) → 409; sukses → 201; galat DB → 500; filter GET ?kegiatanId |
| 57 | `src/routes/api/master-kategori.$id.ts` | 11.76% (4/34) | 33.33% (2/6) | 14.03% | 11-12, 47, 60-62, 72, 81, 89, 92-95, 112, 124, 135, 137 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; id tidak ada → 404; PATCH body tidak valid → 400; PATCH nama duplikat → 409; reaktivasi saat induk nonaktif → 400; DELETE masih dipakai → 409; galat DB → 500 |
| 58 | `src/routes/api/master-detail.$id.ts` | 12.5% (5/40) | 28.57% (2/7) | 14.03% | 15-16, 56, 69, 73-74, 76-78, 88, 97, 105, 108-111, 128, 148, 150, 152, 164, 166 | **perlu tes tambahan** | non-ADMIN → 403; cross-origin → 403; id tidak ada → 404; PATCH body tidak valid → 400; PATCH nama duplikat → 409; reaktivasi saat induk nonaktif → 400; DELETE masih dipakai → 409; galat DB → 500 |
| 59 | `src/lib/file-helpers.ts` | 16% (4/25) | 100% (5/5) | 15.38% | 5-7, 10-17, 42-43 | **wajar dikecualikan** | kode sisi peramban (fetch/redirect/unduh/sessionStorage) yang dipakai komponen UI; di luar lingkup unit sisi server, perilakunya diuji lewat e2e. Catatan: buildFormalFilename bisa diuji murni (dokumen material/non-material, nilai kosong) |
| 60 | `src/routes/api/dokumen.$id.log.ts` | 17.14% (6/35) | 23.07% (3/13) | 47.05% | 17-18, 22, 30-33, 37, 43, 66, 68, 86, 88, 116-117 | **perlu tes tambahan** | matriks akses canAccessDokumenLog: pembuat → boleh; PPK pada status tertentu; PPSPM pada status tertentu; KSBU; Ketua Tim; peran lain → 403; id bukan UUID 404; galat 500 |
| 61 | `src/lib/storage/local-attachment-replacement.ts` | 24.07% (26/108) | 64.1% (25/39) | 34.75% | 126-128, 142, 151, 156, 182, 201, 208, 219, 230-231, 254-257, 266, 287, 332, 337, 349, 379, 388, 403-404, 410, 412, 414, 416, 418, 420, 422-423, 444-445, 461-466, 475, 478, 482-483, 487, 490, 497, 512, 521, 523, 545, 566 | **perlu tes tambahan** | pemilik berbeda ditolak; path bukan pending ditolak; rollback saat pemindahan gagal; cleanup gagal sebagian; localAttachmentIssueStatus per kode isu (L403–423) |
| 62 | `src/lib/dokumen/pembersihan-service.ts` | 25% (16/64) | 100% (18/18) | 16% | 139, 141, 157, 170, 173, 180, 184, 188, 192, 194, 199, 206, 210, 212, 214, 219, 245-246, 249-251, 270, 293, 317-318, 338, 366 | **perlu tes tambahan** | executePembersihanLampiran: tidak ada id layak (L157); baris hilang (L170); dokumen tidak lagi layak; penghapusan berkas gagal sebagian; audit log ditulis. Repositori di-inject |
| 63 | `src/lib/user-response.ts` | 30% (6/20) | 50% (4/8) | 42.85% | 14, 25, 29, 37, 41, 49, 53, 57, 61 | **wajar dikecualikan** | parser respons di sisi klien; cabang yang belum tertutup hanya jalur nilai tak berbentuk objek / gagal parse |
| 64 | `src/lib/dokumen/storage.ts` | 35.29% (6/17) | 18.18% (2/11) | 81.81% | 15-17, 21, 23, 27-30, 44 | **perlu tes tambahan** | buildStorageFilename: dokumen non-material dengan/ tanpa nama_dokumen; material dengan detail_permintaan_nama vs kategori; nilai kosong memakai fallback |
| 65 | `src/routes/api/ppspm/dokumen/$id/reject.ts` | 38.46% (10/26) | 17.64% (3/17) | 55.31% | 28, 30, 32, 39, 41, 61, 65, 81, 86, 105, 114, 118 | **perlu tes tambahan** | cross-origin → 403; tanpa sesi → 401; bukan PPSPM → 403; body gagal Zod → 400; id bukan UUID → 404; dokumen tidak ada → 404; status ≠ IN_PPSPM_APPROVAL → 400; galat lookup/transaksi → 500. Cabang `if (!result.success)` setelah cek status tidak tercapai (cek status sudah sama dengan aturan FSM) → wajar dikecualikan; catatan kosong ditolak |
| 66 | `src/routes/api/upload.ts` | 39.68% (25/63) | 82.85% (29/35) | 38.23% | 62, 70, 129, 153, 166, 187, 191, 200, 206, 211, 220, 241, 252, 256, 276, 284-285, 287, 295-297, 301-304 | **perlu tes tambahan** | mode cleanup (?cleanup=pending) dengan daftar path; body cleanup tidak valid 400; setiap kode LocalUploadError (switch L166) → status yang sesuai; galat non-LocalUploadError → 500 |
| 67 | `src/routes/api/ppk/dokumen/$id/approve.ts` | 41.66% (10/24) | 17.64% (3/17) | 65.85% | 28, 31, 35, 47, 54, 74, 79, 87-88, 108, 117, 121 | **perlu tes tambahan** | cross-origin → 403; tanpa sesi → 401; bukan PPK → 403; body gagal Zod → 400; id bukan UUID → 404; dokumen tidak ada → 404; status ≠ IN_PPK_VALIDATION → 400; galat lookup/transaksi → 500. Cabang `if (!result.success)` setelah cek status tidak tercapai (cek status sudah sama dengan aturan FSM) → wajar dikecualikan |
| 68 | `src/routes/api/ppk/dokumen/$id/reject.ts` | 41.66% (10/24) | 17.64% (3/17) | 63.41% | 28, 31, 35, 48, 55, 75, 80, 88-89, 109, 118, 122 | **perlu tes tambahan** | cross-origin → 403; tanpa sesi → 401; bukan PPK → 403; body gagal Zod → 400; id bukan UUID → 404; dokumen tidak ada → 404; status ≠ IN_PPK_VALIDATION → 400; galat lookup/transaksi → 500. Cabang `if (!result.success)` setelah cek status tidak tercapai (cek status sudah sama dengan aturan FSM) → wajar dikecualikan; catatan revisi kosong ditolak |
| 69 | `src/lib/dev-logger.ts` | 41.66% (5/12) | 66.66% (4/6) | 52.63% | 13, 15-16, 23 | **wajar dikecualikan** | pembungkus console.log yang hanya aktif saat import.meta.env.DEV; tidak ada logika bisnis |
| 70 | `src/lib/users/local-user-mutations.ts` | 42.65% (61/143) | 44.06% (26/59) | 46.57% | 66, 87, 97, 107-108, 110, 115, 129, 132, 137, 140, 160, 164, 166-168, 178, 190, 194-195, 214, 222, 228-229, 231, 236, 246, 262, 273, 276, 289, 294-296, 303, 320-321, 324, 338, 340, 344, 346, 356-358, 376, 384, 396-399, 415, 434, 458, 469, 474, 484 | **perlu tes tambahan** | campuran ADMIN + peran lain ditolak (L66); insert tidak mengembalikan baris (L97); galat peran tidak valid (L110); activate/deactivate user tidak ada; deactivate mencabut sesi; update NIP dinormalisasi |
| 71 | `src/lib/laporan/kegiatan-scope.ts` | 43.75% (7/16) | 87.5% (7/8) | 60% | 43, 45, 47-50, 52, 69 | **perlu tes tambahan** | getPosisiDokumen untuk setiap status (switch L43–52): DRAFT, IN_PPK_VALIDATION, IN_PPSPM_APPROVAL, NEED_REVISION target PPK vs USER, COMPLETED, TERSIMPAN. Fungsi murni, 7 kasus |
| 72 | `src/routes/api/ppk/resubmit/$id.ts` | 44.52% (61/137) | 60.75% (48/79) | 55.42% | 46-47, 50, 55, 57, 64, 106, 116, 147, 172, 174, 176, 223, 225, 229-230, 237, 239, 263, 265, 267, 273, 308, 310, 316, 323, 326, 335, 347, 349, 358, 364, 375, 389, 391, 393, 399, 442, 444, 449, 460, 492, 516, 545 | **perlu tes tambahan** | GET/PATCH/POST: cross-origin, 401, bukan PPK 403, id bukan UUID 404, dokumen bukan milik alur PPK/target bukan PPK 400, validasi ulang kelengkapan gagal 400, galat penyimpanan lampiran → rollback 500. Berkas terbesar ke-2 di api |
| 73 | `src/lib/storage-client.ts` | 50% (14/28) | 58.82% (10/17) | 42.85% | 35, 39, 44, 46, 62, 74, 90, 95, 97, 114 | **wajar dikecualikan** | kode sisi peramban (fetch/redirect/unduh/sessionStorage) yang dipakai komponen UI; di luar lingkup unit sisi server, perilakunya diuji lewat e2e (sebagian sudah diuji; sisa cabang = respons non-OK/JSON tanpa signedUrl) |
| 74 | `src/routes/api/ppspm/dokumen/$id/approve.ts` | 50% (13/26) | 30% (6/20) | 68.08% | 28, 30, 32, 39, 41, 61-62, 79, 82, 111, 115 | **perlu tes tambahan** | cross-origin → 403; tanpa sesi → 401; bukan PPSPM → 403; body gagal Zod → 400; id bukan UUID → 404; dokumen tidak ada → 404; status ≠ IN_PPSPM_APPROVAL → 400; galat lookup/transaksi → 500. Cabang `if (!result.success)` setelah cek status tidak tercapai (cek status sudah sama dengan aturan FSM) → wajar dikecualikan |
| 75 | `src/routes/api/kasubag/berkas/$id.ts` | 50% (8/16) | 42.85% (6/14) | 67.64% | 23, 26, 42, 45, 48, 51, 53 | **perlu tes tambahan** | sesi/akses ditolak diteruskan (L23, L45); id berkas tidak valid (L26, L48); PATCH metadata tidak valid 400; cross-origin |
| 76 | `src/routes/api/dokumen.$id.ts` | 51.16% (88/172) | 57.52% (65/113) | 50.81% | 52, 55, 62, 77-80, 84, 116, 118, 125, 129, 142, 192, 194, 211, 216, 237, 242, 251, 275, 283-284, 286, 294-296, 304, 330-333, 347, 351, 406, 431, 448, 452, 456, 500, 504, 510, 521, 524, 543, 551, 561, 586, 596, 607, 623, 634, 683, 697, 700, 704, 708, 751, 756, 786, 799, 835, 858 | **perlu tes tambahan** | GET matriks akses per peran (PPSPM/PPK/KSBU/Ketua Tim) & status; PATCH: non-pemilik 403, status tidak boleh diedit 400, nominal tidak valid 400; DELETE: bukan DRAFT/NEED_REVISION 400; galat 500. Berkas terbesar di api (873 baris) |
| 77 | `src/lib/storage/document-file-access.ts` | 52.67% (59/112) | 73.43% (47/64) | 52.94% | 76, 81, 86, 97, 101, 106, 113-114, 124, 144, 148, 152, 156, 167, 293, 307, 315, 319, 321, 333-334, 337-338, 353-354, 371, 399-401, 405, 408, 416, 421-422 | **perlu tes tambahan** | tanpa sesi (L76); id bukan UUID (L81); lampiranIndex null (L86); konteks tidak ada (L97); indeks di luar rentang; berkas dimusnahkan; berkas fisik hilang |
| 78 | `src/routes/api/dokumen.$id.submit.ts` | 54.54% (18/33) | 31.81% (7/22) | 67.39% | 30, 33, 37, 41, 87, 91, 96, 100, 106, 114-115, 174 | **perlu tes tambahan** | cross-origin; 401; bukan PEGAWAI 403; id bukan UUID 404; dokumen tidak ada 404; bukan pembuat 403; non-material vs material (L96); status bukan DRAFT 400 |
| 79 | `src/routes/api/kasubag/dokumen.$id.archive.ts` | 55.76% (29/52) | 77.27% (34/44) | 70.83% | 29, 37-38, 53, 55, 60, 81, 102, 121, 182, 220, 233, 252, 262, 285 | **perlu tes tambahan** | helper serialisasi galat (L29–38) dapat diuji murni; jalur: klasifikasi tidak layak 400, berkas tidak ada 404, galat layanan 500 |
| 80 | `src/routes/api/ppspm/dokumen/$id.ts` | 56.25% (18/32) | 22.22% (4/18) | 68.96% | 20-21, 24, 29-30, 42-43, 45, 87, 89, 127, 129, 149 | **perlu tes tambahan** | tanpa sesi 401; bukan PPSPM 403; id bukan UUID 404; dokumen tidak ada 404; status di luar cakupan PPSPM (L29–30); galat 500 |
| 81 | `src/lib/storage/local-storage-diagnostics.ts` | 56.97% (49/86) | 68.08% (64/94) | 70.67% | 172, 257, 299, 308, 378, 380, 399-401, 405, 422, 454, 462, 487, 494, 497, 524, 526, 531, 555, 557, 568, 578, 587, 589, 595, 600, 647, 655 | **perlu tes tambahan** | berkas non-formal dilewati (L172); umur null (L257); inspeksi bukan ok (L299); ENOENT vs galat lain (L308); entri lampiran bukan string (L378–380) |
| 82 | `src/lib/dokumen/local-submit-repository.ts` | 57.14% (8/14) | 80.64% (25/31) | 83.33% | 265, 269, 286, 312-313, 316 | **perlu tes tambahan** | baris tidak ditemukan → null (L265, L286); parse nominal null/number/string tidak valid (L312–316). Fungsi pemetaan, murah |
| 83 | `src/routes/api/users/index.ts` | 57.89% (33/57) | 42.1% (16/38) | 52.72% | 27, 31, 50, 53, 57, 69, 81, 84, 87, 90, 93, 96, 102, 108, 115, 123 | **perlu tes tambahan** | GET: 401/403; POST: cross-origin, 401, 403, setiap validasi field (username, NIP, nama, peran, sandi) → 400 |
| 84 | `src/lib/archive/berkas-arsip-file-access.ts` | 59.25% (48/81) | 64.28% (36/56) | 67.88% | 132, 136, 138, 159, 182, 185, 214, 217, 224, 239, 260, 278, 288, 296, 299, 313, 337, 342, 373, 393, 421, 436, 469, 487-490 | **perlu tes tambahan** | lampiranIndex negatif/bukan bilangan bulat (L132); berkas tidak ada (L138, L185); item tanpa dokumen_id (L214); sumber manual vs workflow (L159); lampiran dimusnahkan |
| 85 | `src/routes/api/kasubag/berkas/$id/close.ts` | 60% (6/10) | 55.55% (5/9) | 77.77% | 17, 20, 23, 28 | **perlu tes tambahan** | cross-origin (L17); sesi ditolak (L20); id tidak valid (L23) |
| 86 | `src/routes/api/kasubag/berkas/$id/items/$itemId/download/$lampiranIndex.ts` | 60% (6/10) | 50% (5/10) | 73.91% | 21, 49, 52, 57 | **perlu tes tambahan** | param tidak valid (L21); berkasId/itemId/lampiranIndex tidak valid (L49–57) → 404 |
| 87 | `src/lib/archive/berkas-arsip-read-model.ts` | 60.3% (117/194) | 70.76% (92/130) | 62.11% | 270, 284, 320, 325-326, 342, 347, 396, 418, 441, 464, 496, 498, 501-504, 511, 562, 593-595, 623, 670, 680, 717, 720, 728, 739, 742, 757, 761, 768, 779-780, 787, 794, 797, 812, 826, 838, 840, 848, 858, 869, 878-880, 884, 896, 905-907, 915, 923 | **perlu tes tambahan** | filter due_only (L270); detail tidak ditemukan (L284); kombinasi filter kosong vs ada (L320); urutan CLOSED vs lainnya (L325); sisa ±70 cabang pemetaan nilai null |
| 88 | `src/routes/api/kasubag/dokumen.$id.ts` | 60.86% (28/46) | 63.33% (19/30) | 70.58% | 19-20, 23, 31, 39-40, 56, 96, 98, 139-140 | **perlu tes tambahan** | parse nominal null/number (L19–23); helper galat (L31); 401/403/404/500 |
| 89 | `src/routes/api/ppk/dokumen/$id.ts` | 61.53% (16/26) | 21.42% (3/14) | 66.66% | 20-21, 24, 37, 41, 45, 89, 99, 122, 124 | **perlu tes tambahan** | parse nominal (L20–24); tanpa sesi 401; bukan PPK 403; id bukan UUID 404; tidak ada 404 |
| 90 | `src/lib/storage/local-pending-move.ts` | 61.66% (37/60) | 72.85% (51/70) | 77.77% | 107, 127, 139, 145, 214, 269, 289, 293, 297, 314, 332, 341, 364-367, 371, 375, 379 | **perlu tes tambahan** | ekstensi tidak valid (L107); sumber sudah formal (L127, L214); target bukan formal (L139); galat bukan LocalPendingMoveError diteruskan (L145, L269) |
| 91 | `src/lib/storage/profile-avatar.ts` | 61.9% (65/105) | 67.21% (41/61) | 69.15% | 78, 81, 95, 98-99, 107, 115, 123, 154, 163, 176, 180, 184, 188, 195, 211, 215-216, 226, 235, 239, 250, 261-262, 279, 307-311, 332 | **perlu tes tambahan** | avatarUpdatedAt null / Date / string (L78–99); MIME tidak diizinkan (L107); ukuran > batas; signature tidak cocok |
| 92 | `src/routes/api/dokumen/download-url.ts` | 62.5% (15/24) | 40% (6/15) | 84.78% | 31, 37, 41, 61, 72, 79, 85, 93, 99 | **perlu tes tambahan** | tanpa sesi (L31); url kosong (L37); path absolut ditolak (L41); docId/lampName tidak ada (L61); nama berkas pola underscore vs dash (L79–85) |
| 93 | `src/routes/api/kasubag/manual-arsip/$id.ts` | 62.5% (10/16) | 56.25% (9/16) | 68.75% | 17, 19, 25, 38, 48, 54 | **perlu tes tambahan** | sesi ditolak (L17); id bukan UUID (L19); tidak ada (L25); cross-origin (L38); body tidak valid (L48); galat non-ManualArsipApiError (L54) |
| 94 | `src/lib/archive/berkas-arsip-page-format.ts` | 63.01% (46/73) | 70.58% (36/51) | 68.11% | 20, 34-35, 95, 101-102, 108-110, 116-117, 129, 133, 137, 139-140, 150, 156-158 | **perlu tes tambahan** | label status CLOSED/DIMUSNAHKAN/null (L20–35); sumber MANUAL (L95); setiap kode peringatan (L101–158). Fungsi murni, ±12 kasus |
| 95 | `src/lib/laporan/monitoring-rows.ts` | 63.15% (24/38) | 78.04% (32/41) | 80% | 98, 165, 177-179, 192-193, 195, 200, 204, 206 | **perlu tes tambahan** | pengaju tanpa id → fallback nama → "unknown" (L165); setiap mode sortBy (L177–206) |
| 96 | `src/routes/api/laporan/saya.export-zip.ts` | 64% (32/50) | 60% (15/25) | 60.41% | 48, 60, 75, 79-80, 82, 84, 148, 151, 162, 165, 175, 185-186, 189 | **perlu tes tambahan** | mode=ticket (L60); tiket kosong/kedaluwarsa (L79–80) → 410; body melebihi batas 413; tanpa sesi 401 |
| 97 | `src/lib/dokumen/local-submit-drizzle-adapter.ts` | 64.28% (9/14) | 86.66% (26/30) | 95.34% | 111, 207, 220, 245, 269 | **perlu tes tambahan** | insert tidak mengembalikan baris (L220); update tanpa baris → konflik (L245); komponenId null (L269) |
| 98 | `src/lib/export/laporan-zip-entries.ts` | 64.28% (9/14) | 94.11% (16/17) | 64.7% | 38, 41, 134 | **perlu tes tambahan** | konteks dokumen tidak ada (L38); referensi lampiran gagal (L41) |
| 99 | `src/routes/api/users/me/change-password.ts` | 65% (13/20) | 52.63% (10/19) | 73.07% | 26, 38-39, 41, 44, 47, 54 | **perlu tes tambahan** | tanpa sesi (L26); sandi lama kosong (L41); sandi baru kosong (L44); sandi baru lemah (L47); sandi lama salah (L54) |
| 100 | `src/routes/api/ppk/kembalikan/$id.ts` | 65.38% (17/26) | 71.42% (20/28) | 80.48% | 39, 41, 47, 74, 83, 114, 118 | **perlu tes tambahan** | cross-origin (L39); tanpa sesi (L41); id bukan UUID (L47); dokumen tidak ada (L74); galat transaksi 500. Cabang L83 (FSM gagal) wajar dikecualikan bila status sudah dicek |
| 101 | `src/routes/api/kasubag/klasifikasi/$id.ts` | 66.37% (77/116) | 76.13% (67/88) | 69.44% | 25, 39, 50-51, 53, 55, 62, 67, 98, 136, 155, 160, 163, 179, 186, 188, 215, 234, 236, 241, 246, 248, 263, 270, 287, 304, 320, 323 | **perlu tes tambahan** | tanpa sesi (L25); deteksi field duplikat dari constraint kode/nama/detail (L39–62) → pesan 409 yang tepat; masih dipakai saat DELETE 409 |
| 102 | `src/lib/archive/phase15-berkas-activity-dev-reset-analysis.ts` | 66.66% (12/18) | 81.81% (9/11) | 45.94% | 115, 125, 251, 260 | **wajar dikecualikan** | utilitas reset data pengembangan (nama berkas "dev-reset"); cabang tersisa = penjaga NODE_ENV production dan nilai count null. Cukup 1 kasus penjaga production bila ingin |
| 103 | `src/routes/api/kasubag/berkas/$id/items.ts` | 66.66% (8/12) | 60% (6/10) | 77.77% | 20, 23, 26, 31 | **perlu tes tambahan** | cross-origin (L20); sesi ditolak (L23); id berkas tidak valid (L26) |
| 104 | `src/lib/manual-arsip.ts` | 67.61% (119/176) | 70.65% (118/167) | 82.56% | 281, 311, 341, 358, 394, 410, 444, 469, 497, 507, 524, 539, 549, 572, 605-608, 610, 640, 655-657, 660, 702, 724-726, 729, 798, 807, 858, 927, 932, 949, 965, 973-974, 1008, 1022, 1037, 1077, 1087, 1113-1114, 1117, 1121, 1126-1127, 1173, 1212 | **perlu tes tambahan** | berkas terbesar di lib (1213 baris). Cabang belum tertutup didominasi jalur galat: sumber tidak ada (L341), tanpa lampiran (L358), rollback gagal (L394), insert gagal (L410), baris tidak ada (L444), dst. ±15 kasus jalur galat |
| 105 | `src/lib/archive/berkas-arsip-api.ts` | 67.64% (23/34) | 90.32% (28/31) | 72% | 70, 89-90, 108, 116-117 | **perlu tes tambahan** | galat bukan BerkasArsipServiceError (L70); setiap kode galat di switch (L89–90); serialisasi galat non-objek (L108) |
| 106 | `src/routes/api/dokumen/submit.ts` | 69.86% (51/73) | 80.51% (62/77) | 83.33% | 88, 105, 115, 128, 132, 169, 192, 204, 206, 210-211, 221, 255, 264, 279, 297, 302, 305, 326 | **perlu tes tambahan** | persiapan lampiran gagal (L88); rollback gagal (L105, L115); setiap kode isu: ketua-tim-assignment-missing (L128), transition-failed (L132), dst. |
| 107 | `src/routes/api/dokumen/preview-url.ts` | 70% (7/10) | 69.23% (9/13) | 88% | 31, 35, 53 | **perlu tes tambahan** | url kosong (L31); path absolut ditolak (L35) |
| 108 | `src/routes/api/laporan/kegiatan.ts` | 70.49% (43/61) | 56.41% (22/39) | 82.05% | 31, 34, 38-39, 49-51, 79, 179, 191-192, 203, 237, 241-243 | **perlu tes tambahan** | helper konversi nilai (L31–39) number/string/Date; displayName null (L49–50); filter scope |
| 109 | `src/lib/storage/manual-arsip-pending-attachments.ts` | 71.42% (20/28) | 75.75% (25/33) | 90.38% | 80, 89, 101, 108, 150, 155-156, 172 | **perlu tes tambahan** | sumber formal ditolak (L80); MIME/nama kosong (L89); ukuran 0 atau > batas (L101); signature tidak cocok (L108); missing-source → 400 vs lainnya 500 (L155–156) |
| 110 | `src/routes/api/auth/login.ts` | 72.22% (13/18) | 72.22% (13/18) | 90.62% | 33, 60, 69, 98-99 | **perlu tes tambahan** | body tidak valid (L33) → 400; loginResult selain 401 (L60); header x-forwarded-for ada/tidak (L98–99) |
| 111 | `src/lib/archive/berkas-arsip-attachment-names.ts` | 72.72% (112/154) | 71.31% (92/129) | 80.67% | 87, 95-96, 99, 109-111, 115, 143, 153, 180, 197, 210, 212, 216-218, 223-224, 233-234, 236, 243, 260-261, 263-265, 270, 290-292, 295, 318, 339, 349, 378 | **perlu tes tambahan** | entri lampiran bukan objek (L87); judul kosong → fallback (L95–99); value null/undefined/bukan array (L109–110) |
| 112 | `src/lib/storage/logical-file-deletion.ts` | 72.72% (40/55) | 83.33% (35/42) | 90.27% | 64, 119, 166, 189, 191, 227-230 | **perlu tes tambahan** | setiap outcome di switch (L119); ENOENT/ENOTDIR vs galat lain (L166); path di luar root (L189) |
| 113 | `src/routes/api/kasubag/berkas/index.ts` | 72.72% (16/22) | 68.42% (13/19) | 83.72% | 40, 47, 50, 53, 56, 59 | **perlu tes tambahan** | setiap parameter query opsional (status_arsip=null, klasifikasi, tahun, search, limit, offset — L40–59). 1 kasus tabel-driven |
| 114 | `src/lib/storage/manual-arsip-upload.ts` | 73.68% (14/19) | 68.18% (15/22) | 81.57% | 64, 119, 127, 131, 156 | **perlu tes tambahan** | jumlah berkas 0 / > maks (L64); MIME tidak diizinkan (L119); ukuran ≤ 0 (L127); ukuran > maks (L131); nama berkas disanitasi (L156) |
| 115 | `src/lib/archive/berkas-arsip-service.ts` | 74.28% (104/140) | 84.05% (116/138) | 82.71% | 279, 300, 331, 368, 388, 416, 441, 466, 483, 514, 577, 603, 644, 660, 674, 690, 701, 726, 744, 766, 781, 819, 859, 1012, 1020, 1034, 1042, 1046, 1048 | **perlu tes tambahan** | jalur "baris tidak ada" pada setiap operasi (L279, L300, L331, L368, L388, L416, L441, …) → BerkasArsipServiceError dengan kode yang tepat |
| 116 | `src/lib/utils/format.ts` | 75% (12/16) | 70% (14/20) | 85.18% | 4, 40, 53-54 | **perlu tes tambahan** | tanggal tidak valid (L4); created_at tidak valid (L40); formatRelativeAge 0 hari dan 1 hari (L53–54). 4 kasus murni |
| 117 | `src/lib/laporan/status-laporan.ts` | 75% (3/4) | 75% (3/4) | 100% | 25 | **perlu tes tambahan** | laporanStatusLabel status tak dikenal → fallback (L25). 1 kasus |
| 118 | `src/routes/api/admin/analyze-storage.ts` | 75% (3/4) | 60% (3/5) | 80% | 20 | **perlu tes tambahan** | sesi bukan ADMIN → 403 (L20). 1 kasus |
| 119 | `src/routes/api/auth/logout.ts` | 75% (3/4) | 66.66% (2/3) | 100% | 22 | **perlu tes tambahan** | logout tanpa cookie sesi (L22). 1 kasus |
| 120 | `src/routes/api/kasubag/klasifikasi/index.ts` | 76.31% (58/76) | 77.94% (53/68) | 87.12% | 54, 63, 66, 73, 78, 117-118, 145-146, 173, 195, 200, 203, 219, 233, 243, 264 | **perlu tes tambahan** | deteksi constraint duplikat (L54–78); urutan kode kosong → nama (L117–118) |
| 121 | `src/lib/auth/local-auth-service.ts` | 76.47% (13/17) | 80.95% (17/21) | 93.54% | 80, 122, 155 | **perlu tes tambahan** | peran tidak valid saat login (L80); displayName fallback (L122); identifier tidak ditemukan (L155) |
| 122 | `src/lib/upload/document-upload-policy.ts` | 76.74% (33/43) | 80% (32/40) | 91.8% | 109, 111, 116, 129, 141, 171, 191-192, 196 | **perlu tes tambahan** | nama tanpa ekstensi/diakhiri titik (L111); MIME tidak diizinkan (L129); konten ArrayBuffer vs Uint8Array (L171) |
| 123 | `src/lib/archive/berkas-arsip-physical-destruction.ts` | 77.14% (81/105) | 83.16% (84/101) | 78.45% | 281, 298, 379, 382, 451, 489, 494, 555, 578, 580, 608, 611, 631, 651, 665, 720 | **perlu tes tambahan** | tanpa kandidat & tanpa path tidak aman (L281); entri lampiran bukan objek/url bukan string (L379–382); kunci kosong (L451); status skipped (L494) |
| 124 | `src/routes/api/kasubag/manual-arsip/index.ts` | 78.57% (11/14) | 76.47% (13/17) | 80% | 21, 32, 58 | **perlu tes tambahan** | sesi ditolak (L21); query tidak valid (L32); body tidak valid (L58) |

---

## 3. Modul transisi status: `src/lib/fsm.ts` dan `tests/fsm.test.ts`

Nomor kasus uji `#n` di bagian ini adalah kolom `nomor` pada `lampiran-kasus-uji-unit.csv`. Kasus #1–#49 adalah seluruh isi `tests/fsm.test.ts`, sesuai urutan dalam berkas.

### 3.1 Isi modul

| Simbol | Diekspor? | Baris | Keterangan |
|---|---|---|---|
| `transition(currentStatus, action, actorRole, revisionTarget?)` | **ya (satu-satunya)** | 81–127 | Fungsi transisi utama. |
| `isActorValidForAction(status, action, role)` | tidak | 129–156 | Validasi aktor; dipanggil di L87. |
| `makeError(currentStatus, error)` | tidak | 70–79 | Membentuk hasil galat; tanpa percabangan. |
| `TRANSITIONS` | tidak (data) | 19–68 | Tabel 8 transisi sah dengan kunci `status:aksi`. |

Delapan transisi sah disimpan sebagai **data** (tabel), bukan percabangan. Akibatnya, di flow graph satu keputusan `if (!result)` (L119) mewakili seluruh 8 baris tabel dan semua kombinasi status:aksi yang tidak sah. Basis path testing membuktikan bahwa logika penelusuran tabel teruji, tetapi **tidak** membuktikan setiap baris tabel benar. Kebenaran tiap baris tabel dibuktikan oleh kasus #1–#8, yang menguji 8 transisi sah satu per satu dan memeriksa keempat field hasilnya. Ini sebaiknya disebut terpisah sebagai pengujian transisi status (black-box) yang melengkapi basis path.

Coverage `fsm.ts` dari `tests/fsm.test.ts` saja (mode AST): statement 96% (24/25), **branch 97,22% (35/36)**, function 100% (3/3). Satu-satunya cabang yang tidak tertutup adalah `default` di L153–154. Di run penuh angkanya sama, karena tes rute juga memanggil `transition()` asli. Di mode bawaan: statement 99,23%, branch 96,87% (31/32).

### 3.2 Titik keputusan `transition()`

| ID | Baris | Keputusan | Kondisi sederhana |
|---|---|---|---|
| K1 | 88 | `!actorValid` | 1 |
| K2 | 95 | `action === REJECT` | 1 |
| K3 | 96 | `!revisionTarget \|\| (revisionTarget !== 'USER' && revisionTarget !== 'PPK')` | **3** (a, b, c) |
| K4 | 104 | `action === RESUBMIT && revisionTarget !== 'USER'` | **2** |
| K5 | 110 | `action === RESUBMIT_PPK && revisionTarget !== 'PPK'` | **2** |
| K6 | 119 | `!result` | 1 |
| | | **Jumlah** | **6 keputusan, 10 kondisi** |

### 3.3 Cyclomatic complexity V(G)

Ada dua tingkat kerincian, dan keduanya perlu disebut di skripsi:

- **Tingkat keputusan:** setiap `if` dihitung satu simpul predikat.
- **Tingkat kondisi (gaya Pressman):** setiap kondisi sederhana dalam kondisi majemuk menjadi simpul predikat tersendiri. Cara ini lebih sesuai untuk basis path karena `fsm.ts` memiliki kondisi majemuk.

| Fungsi | Cara | Tingkat keputusan | Tingkat kondisi |
|---|---|---|---|
| `transition()` | E − N + 2 | 18 − 13 + 2 = **7** | 26 − 17 + 2 = **11** |
| | predikat + 1 | 6 + 1 = **7** | 10 + 1 = **11** |
| | jumlah region | 6 tertutup + 1 luar = **7** | 10 tertutup + 1 luar = **11** |
| `isActorValidForAction()` | E − N + 2 | 14 − 9 + 2 = **7** | 31 − 19 + 2 = **14** |
| | predikat + 1 | 6 (switch 7 cabang) + 1 = **7** | 6 + 7 operator `&&`/`\|\|` + 1 = **14** |
| | jumlah region | **7** | **14** |
| `makeError()` | — | 1 | 1 |

Catatan:

- Region dihitung dengan anggapan graf digambar planar (tanpa sisi bersilangan). Dalam keadaan itu jumlah region = E − N + 2 (rumus Euler), dan setiap simpul predikat biner menambah tepat satu region tertutup. Daftar region ada di bawah flow graph.
- Untuk `isActorValidForAction()` di tingkat kondisi, operand `&&`/`||` di dalam ekspresi `return` dimodelkan sebagai predikat karena evaluasinya short-circuit. Operand terakhir tidak dihitung sebagai predikat karena nilainya langsung menjadi hasil. Model grafnya:
  - 1 simpul `switch` dengan 7 sisi keluar (6 predikat);
  - APPROVE dan REJECT masing-masing 5 simpul (c1, c2, c3, c4, titik temu) dan 7 sisi internal (3 predikat);
  - KEMBALIKAN 3 simpul (c1, c2, titik temu) dan 3 sisi internal (1 predikat);
  - SUBMIT, RESUBMIT, RESUBMIT_PPK, dan `default` masing-masing 1 simpul;
  - 1 simpul keluar, dan 7 sisi menuju simpul keluar.
  
  Totalnya N = 1 + 1 + 5 + 5 + 1 + 1 + 3 + 1 + 1 = 19 dan E = 7 + 7 + 7 + 3 + 7 = 31.

### 3.4 Draf flow graph `transition()`, tingkat kondisi (V(G) = 11)

**Simpul**

| Simpul | Baris | Isi | Predikat? |
|---|---|---|---|
| 1 | 87–88 | panggil `isActorValidForAction`; uji `!actorValid` | P1 |
| 2 | 89–92 | `return makeError(...)` (aktor tidak sah) | |
| 3 | 95 | `action === REJECT` | P2 |
| 4 | 96 | kondisi a: `!revisionTarget` | P3 |
| 5 | 96 | kondisi b: `revisionTarget !== 'USER'` | P4 |
| 6 | 96 | kondisi c: `revisionTarget !== 'PPK'` | P5 |
| 7 | 97–100 | `return makeError(...)` (target REJECT tidak sah) | |
| 8 | 104 | `action === RESUBMIT` | P6 |
| 9 | 104 | `revisionTarget !== 'USER'` | P7 |
| 10 | 105–108 | `return makeError(...)` (RESUBMIT) | |
| 11 | 110 | `action === RESUBMIT_PPK` | P8 |
| 12 | 110 | `revisionTarget !== 'PPK'` | P9 |
| 13 | 111–114 | `return makeError(...)` (RESUBMIT_PPK) | |
| 14 | 117–119 | bentuk `key`, baca `TRANSITIONS[key]`, uji `!result` | P10 |
| 15 | 120–123 | `return makeError(...)` (transisi tidak ada) | |
| 16 | 126 | `return { success: true, ...result }` | |
| 17 | 127 | keluar | |

**Sisi (26)**: B = benar, S = salah

| Dari | Ke | Label | | Dari | Ke | Label |
|---|---|---|---|---|---|---|
| 1 | 2 | B | | 8 | 9 | B |
| 1 | 3 | S | | 8 | 11 | S |
| 2 | 17 | | | 9 | 10 | B |
| 3 | 4 | B | | 9 | 11 | S |
| 3 | 8 | S | | 10 | 17 | |
| 4 | 7 | B | | 11 | 12 | B |
| 4 | 5 | S | | 11 | 14 | S |
| 5 | 6 | B | | 12 | 13 | B |
| 5 | 8 | S | | 12 | 14 | S |
| 6 | 7 | B | | 13 | 17 | |
| 6 | 8 | S | | 14 | 15 | B |
| 7 | 17 | | | 14 | 16 | S |
| | | | | 15 | 17 | |
| | | | | 16 | 17 | |

N = 17, E = 26, sehingga V(G) = 26 − 17 + 2 = **11**.

**Region**: masing-masing dibatasi oleh dua cabang keluar sebuah predikat sampai titik temu pertamanya.

R1 (P1: 1→2→17 dan 1→3→…→17), R2 (P2: 3→8 dan 3→4→5→8), R3 (P3: 4→7 dan 4→5→6→7), R4 (P4: 5→8 dan 5→6→8), R5 (P5: 6→7→17 dan 6→8→…→17), R6 (P6: 8→11 dan 8→9→11), R7 (P7: 9→10→17 dan 9→11→…→17), R8 (P8: 11→14 dan 11→12→14), R9 (P9: 12→13→17 dan 12→14→…→17), R10 (P10: 14→15→17 dan 14→16→17), ditambah R11 (region luar). Total **11**.

**Versi tingkat keputusan (V(G) = 7)**, bila ingin gambar yang lebih ringkas:

- Simpul: 1 (87–88), 2 (89–92), 3 (95), 4 (96, utuh), 5 (97–100), 6 (104, utuh), 7 (105–108), 8 (110, utuh), 9 (111–114), 10 (117–119), 11 (120–123), 12 (126), 13 (keluar).
- Sisi: 1→2, 1→3, 2→13, 3→4, 3→6, 4→5, 4→6, 5→13, 6→7, 6→8, 7→13, 8→9, 8→10, 9→13, 10→11, 10→12, 11→13, 12→13. Total 18.

### 3.5 Jalur independen ↔ kasus uji (tingkat kondisi, 11 jalur)

Basis set disusun dari jalur dasar J2 (sukses tanpa REJECT/RESUBMIT*), lalu setiap predikat dibalik satu per satu. Pemetaan ke kasus uji berasal dari pembacaan kode. Hitungan eksekusi per cabang dari run `fsm.test.ts` saja mengonfirmasinya: L88 benar 22×/salah 28×, sama dengan jumlah kasus J1 (22) dan sisa panggilan (28, karena #33 memanggil `transition` dua kali). L119 benar 3× (#43, #46, #47). L96 benar 1× (#34).

| Jalur | Urutan simpul | Situasi masukan | Hasil yang diharapkan | Kasus uji di `tests/fsm.test.ts` |
|---|---|---|---|---|
| J1 | 1-2-17 | aktor tidak berhak untuk aksi/status | gagal, pesan "Actor '…' tidak bisa …" | #10–13, #15, #16, #20, #22, #24, #26–32, #41, #42, #44, #45, #48, #49 (22 kasus) |
| J2 | 1-3-8-11-14-16-17 | aktor sah, aksi SUBMIT/APPROVE/KEMBALIKAN, transisi ada | sukses | #1, #2, #4, #8, #9, #14, #17, #25, #33 |
| J3 | 1-3-8-11-14-15-17 | aktor sah, aksi bukan REJECT/RESUBMIT*, transisi tidak ada (hanya mungkin untuk SUBMIT dari status ≠ DRAFT) | gagal, "Transisi … tidak valid" | #43, #47 |
| J4 | 1-3-4-7-17 | REJECT, aktor sah, `revisionTarget` kosong | gagal, "REJECT requires revisionTarget" | #34 |
| J5 | 1-3-4-5-8-11-14-16-17 | REJECT, `revisionTarget = 'USER'` | sukses | #3, #18, #35 |
| J6 | 1-3-4-5-6-8-11-14-16-17 | REJECT, `revisionTarget = 'PPK'` | sukses | #5, #19, #36 |
| **J7** | 1-3-4-5-6-7-17 | REJECT, `revisionTarget` terisi tetapi bukan USER/PPK (mis. `'X'`) | gagal, "REJECT requires revisionTarget" | **BELUM ADA** |
| J8 | 1-3-8-9-10-17 | RESUBMIT, `revisionTarget ≠ 'USER'` | gagal, "RESUBMIT only valid …" | #38 |
| J9 | 1-3-8-9-11-14-16-17 | RESUBMIT, `revisionTarget = 'USER'`, status NEED_REVISION | sukses | #6, #21, #37 |
| J10 | 1-3-8-11-12-13-17 | RESUBMIT_PPK, `revisionTarget ≠ 'PPK'` | gagal, "RESUBMIT_PPK only valid …" | #40 |
| J11 | 1-3-8-11-12-14-16-17 | RESUBMIT_PPK, `revisionTarget = 'PPK'` | sukses | #7, #23, #39 |

Jalur tambahan di luar basis set yang juga teruji: 1-3-8-9-11-14-15-17 (RESUBMIT dengan target USER dari status yang salah) → #46. Jalur feasible yang belum teruji: RESUBMIT_PPK dengan target PPK dari status selain NEED_REVISION (1-3-8-11-12-14-15-17), misalnya `transition('IN_PPK_VALIDATION','RESUBMIT_PPK','PPK','PPK')`.

Jumlah kasus per jalur: 22 + 9 + 2 + 1 + 3 + 3 + 0 + 1 + 3 + 1 + 3 = 48, ditambah #46 = 49. Ada redundansi: 49 kasus hanya memakai 33 pola masukan unik (satu di antaranya `it.each` untuk 4 peran), dan 9 pola dipakai lebih dari sekali.

**Kelemahan untuk penyajian basis path:** `assertError()` (baris 5–9 di tes) hanya memeriksa `success === false` dan bahwa `error` berupa string. Akibatnya tes tidak membuktikan jalur mana yang ditempuh. Misalnya #42 "DRAFT + REJECT → error" sebenarnya berhenti di J1 (aktor), bukan di penelusuran tabel. Satu-satunya kasus galat yang memeriksa isi pesan adalah #49 ("tidak bisa"). Supaya tabel basis path bisa mencantumkan "hasil yang diharapkan" yang terverifikasi, setiap kasus galat sebaiknya memeriksa potongan pesannya. Perubahan ini belum diterapkan sesuai aturan pemeriksaan.

### 3.6 Jalur `isActorValidForAction()` (tingkat keputusan, V(G) = 7)

Simpul: 1 (L134 `switch`), 2 (L135–136), 3 (L137–141), 4 (L142–146), 5 (L147–148), 6 (L149–150), 7 (L151–152), 8 (L153–154 `default`), 9 (keluar). Sisi: 1→2 … 1→8 (7 sisi) dan 2…8→9 (7 sisi). V(G) = 14 − 9 + 2 = 7.

| Jalur | Cabang `switch` | Ekspresi bernilai benar | Ekspresi bernilai salah |
|---|---|---|---|
| A1 | SUBMIT (L135) | #1, #9, #43, #47 | #10–13 |
| A2 | APPROVE (L137) | #2, #4, #14, #17 | #15, #16, #41, #44, #48, #49 |
| A3 | REJECT (L142) | #3, #5, #18, #19, #34–36 | #20, #42, #45 |
| A4 | RESUBMIT (L147) | #6, #21, #37, #38, #46 | #22 |
| A5 | RESUBMIT_PPK (L149) | #7, #23, #39, #40 | #24 |
| A6 | KEMBALIKAN (L151) | #8, #25, #33 | #26–32 |
| **A7** | **default (L153)** | — | **BELUM ADA**. Tidak tercapai lewat tipe `FSMAction`; hanya bisa diuji dengan cast, misalnya `'X' as FSMAction`. |

### 3.7 Tabel keputusan gaya Myers (Tabel 5.1)

| No | Keputusan (baris) | Situasi untuk hasil BENAR | Situasi untuk hasil SALAH | Kasus uji (BENAR) | Kasus uji (SALAH) |
|---|---|---|---|---|---|
| 1 | `!actorValid` (L88) | aktor tidak berhak, mis. SUBMIT oleh PPK, APPROVE oleh PPSPM di IN_PPK_VALIDATION, KEMBALIKAN di luar NEED_REVISION | aktor berhak untuk aksi dan status itu | #10–13, #15, #16, #20, #22, #24, #26–32, #41, #42, #44, #45, #48, #49 | #1–9, #14, #17–19, #21, #23, #25, #33–40, #43, #46, #47 |
| 2 | `action === REJECT` (L95) | REJECT oleh PPK di IN_PPK_VALIDATION atau oleh PPSPM di IN_PPSPM_APPROVAL | aksi lain dengan aktor sah | #3, #5, #18, #19, #34–36 | #1, #2, #4, #6–9 dst. |
| 3 | `!rt \|\| (rt !== USER && rt !== PPK)` (L96) | `revisionTarget` kosong; **atau** terisi tetapi bukan USER/PPK | `revisionTarget` = USER atau PPK | #34 (kosong). **Situasi "nilai tak dikenal" BELUM.** | #3, #5, #18, #19, #35, #36 |
| 4 | `action === RESUBMIT && rt !== USER` (L104) | RESUBMIT oleh PEGAWAI dengan target ≠ USER | aksi bukan RESUBMIT, atau RESUBMIT dengan target USER | #38 | #6, #21, #37, #46, dan semua kasus non-RESUBMIT yang lolos K1 |
| 5 | `action === RESUBMIT_PPK && rt !== PPK` (L110) | RESUBMIT_PPK oleh PPK dengan target ≠ PPK | bukan RESUBMIT_PPK, atau target PPK | #40 | #7, #23, #39 |
| 6 | `!result` (L119) | tidak ada entri `status:aksi` di tabel, mis. COMPLETED:SUBMIT | entri ada | #43, #46, #47 | #1–9, #14, #17–19, #21, #23, #25, #33, #35–37, #39 |
| 7 | `switch (action)` (L134) | 7 hasil: lihat 3.6 | — | 6 cabang teruji | **`default` (L153) BELUM** |
| 8 | `role === PEGAWAI` (L136, SUBMIT) | PEGAWAI | peran lain | #1, #9, #43, #47 | #10–13 |
| 9 | ekspresi APPROVE (L139–140) | (IN_PPK, PPK) atau (IN_PPSPM, PPSPM) | selain itu | #2, #4, #14, #17 | #15, #16, #41, #44, #48, #49 |
| 10 | ekspresi REJECT (L144–145) | (IN_PPK, PPK) atau (IN_PPSPM, PPSPM) | selain itu | #3, #5, #18, #19, #34–36 | #20, #42, #45 |
| 11 | `role === PEGAWAI` (L148, RESUBMIT) | PEGAWAI | peran lain | #6, #21, #37, #38, #46 | #22 |
| 12 | `role === PPK` (L150, RESUBMIT_PPK) | PPK | peran lain | #7, #23, #39, #40 | #24 |
| 13 | `role === PPK && status === NEED_REVISION` (L152) | PPK di NEED_REVISION | peran lain, atau status lain | #8, #25, #33 | #26–32 |

**Hasil keputusan yang belum punya kasus uji: hanya `default` pada keputusan 7 (L153).** Semua keputusan biner sudah teruji pada kedua hasilnya, jadi decision coverage `fsm.ts` = 35/36. Kekurangan lainnya berada di tingkat kondisi (3.8).

### 3.8 Kombinasi kondisi pada keputusan majemuk

Notasi: B = benar, S = salah, – = tidak dievaluasi (short-circuit).

**K3 (L96)**: a = `!rt`, b = `rt !== 'USER'`, c = `rt !== 'PPK'`

| a | b | c | Hasil | Contoh `revisionTarget` | Kasus |
|---|---|---|---|---|---|
| B | – | – | B | `undefined` | #34 |
| B | – | – | B | `null` / `''` | belum (varian dengan hasil sama) |
| S | S | – | S | `'USER'` | #3, #18, #35 |
| S | B | S | S | `'PPK'` | #5, #19, #36 |
| S | B | **B** | **B** | `'X'` | **BELUM** |

Condition coverage: a (B #34, S #3) terpenuhi; b (B #5, S #3) terpenuhi; **c hanya pernah S**, jadi tidak terpenuhi. MC/DC untuk c membutuhkan pasangan (S,B,B) dan (S,B,S); yang pertama belum ada.

**K4 (L104)**: a = `action === RESUBMIT`, b = `rt !== 'USER'`

| a | b | Hasil | Kasus |
|---|---|---|---|
| S | – | S | semua aksi lain yang lolos K1, mis. #1 |
| B | B | B | #38 (target PPK). Varian target kosong belum. |
| B | S | S | #6, #21, #37, #46 |

Semua kombinasi feasible sudah teruji.

**K5 (L110)**: a = `action === RESUBMIT_PPK`, b = `rt !== 'PPK'`

| a | b | Hasil | Kasus |
|---|---|---|---|
| S | – | S | mis. #1 |
| B | B | B | #40 |
| B | S | S | #7, #23, #39 |

Semua kombinasi feasible sudah teruji.

**Ekspresi APPROVE (L139–140)**: c1 = status IN_PPK, c2 = role PPK, c3 = status IN_PPSPM, c4 = role PPSPM

| c1 | c2 | c3 | c4 | Hasil | Kasus |
|---|---|---|---|---|---|
| B | B | – | – | B | #2, #14 |
| B | S | S | – | S | #15 |
| S | – | B | B | B | #4, #17 |
| S | – | B | S | S | #16 |
| S | – | S | – | S | #41, #44 (#48, #49 sama dengan #41) |

Kombinasi (B, S, B, …) tidak mungkin terjadi karena status tidak bisa bernilai dua sekaligus. **Semua kombinasi feasible sudah teruji.**

**Ekspresi REJECT (L144–145)**: struktur sama dengan APPROVE

| c1 | c2 | c3 | c4 | Hasil | Kasus |
|---|---|---|---|---|---|
| B | B | – | – | B | #3, #18, #34, #35 |
| B | S | S | – | S | #20 |
| S | – | B | B | B | #5, #19, #36 |
| S | – | B | **S** | **S** | **BELUM**: REJECT oleh PPK (atau peran lain) di IN_PPSPM_APPROVAL |
| S | – | S | – | S | #42, #45 |

c4 tidak pernah bernilai S, jadi condition coverage untuk ekspresi ini tidak terpenuhi.

**KEMBALIKAN (L152)**: c1 = role PPK, c2 = status NEED_REVISION

| c1 | c2 | Hasil | Kasus |
|---|---|---|---|
| B | B | B | #8, #25, #33 |
| B | S | S | #30, #31, #32 |
| S | – | S | #26–29 |

Semua kombinasi sudah teruji.

**Ringkasan kekurangan FSM (4 kasus baru, tanpa mengubah perilaku):**

1. `transition('IN_PPK_VALIDATION','REJECT','PPK','X')` → gagal (J7; K3 c = B).
2. `transition('IN_PPSPM_APPROVAL','REJECT','PPK','PPK')` → gagal (ekspresi REJECT, c4 = S).
3. `transition('DRAFT','X' as FSMAction,'PEGAWAI')` → gagal (A7, `default` L153).
4. (opsional, di luar basis set) `transition('IN_PPK_VALIDATION','RESUBMIT_PPK','PPK','PPK')` → gagal di tabel.

Ditambah perbaikan asersi galat agar memeriksa pesan (3.5).

Observasi perilaku (bukan cacat, tetapi perlu disadari saat menulis): untuk REJECT, nilai `revisionTarget` dari pemanggil hanya divalidasi (harus USER atau PPK). Target yang dipakai berasal dari tabel. `transition('IN_PPSPM_APPROVAL','REJECT','PPSPM','USER')` tetap sukses dengan `newRevisionTarget = 'PPK'`. Hal yang sama berlaku untuk KEMBALIKAN (#33 menguji ini dengan sengaja).

### 3.9 Penyajian

V(G) setiap fungsi ≤ 15: `transition()` 7/11, `isActorValidForAction()` 7/14. Karena itu tidak perlu teknik khusus. Bila seluruh modul dijumlahkan di tingkat kondisi (11 + 14 + 1 = 26), angkanya melewati 15, jadi **sajikan per fungsi**:

1. `transition()`: flow graph tingkat kondisi (17 simpul) dengan tabel 11 jalur.
2. `isActorValidForAction()`: flow graph tingkat keputusan (switch 7 cabang), ditambah tabel kombinasi kondisi (3.8) untuk ekspresi `&&`/`||` di dalamnya. Cara ini lebih terbaca daripada graf 15 simpul.
3. `TRANSITIONS`: tabel keadaan × aksi, dipetakan ke kasus #1–#8 sebagai pengujian transisi status.

---

## 4. Sifat tes yang ada

### 4.1 Klasifikasi

Kategori ditentukan dari isi tiap berkas: `vi.mock`, impor `#/routes/api/*` dan pemanggilan `handlers`, `readFileSync` atas berkas sumber, serta `node:fs/promises` atau `mkdtemp`. Hasilnya saya cek manual untuk berkas yang ambigu.

- **A. Unit terisolasi**: memanggil fungsi `src/lib` secara langsung; dependensi diganti `vi.mock` atau fake yang di-inject; tidak menjalankan handler rute dan tidak memakai disk nyata.
- **B. Handler rute + beberapa modul**: memanggil `Route.options.server.handlers.<METHOD>`. Hanya DB, auth, atau storage yang di-mock; modul `src/lib` lain (FSM, validasi, schema, storage) berjalan asli.
- **C. Static source guard**: membaca teks berkas sumber (`readFileSync`) lalu `toContain(...)`. Kode tidak dieksekusi.
- **D. Unit dengan sistem berkas nyata**: memanggil `src/lib` dengan direktori sementara di disk.

| Folder | Berkas | A | B | C | D |
|---|---:|---:|---:|---:|---:|
| tests/ (fsm.test.ts) | 1 | 1 | – | – | – |
| tests/unit/ (akar) | 1 | 1 | – | – | – |
| arsiparis | 22 | 11 | 8 | 1 | 2 |
| auth | 10 | 7 | 2 | 1 | – |
| components | 3 | 1 | – | 2 | – |
| dashboard | 1 | – | – | 1 | – |
| db | 2 | 1 | – | 1 | – |
| dokumen | 27 | 9 | 9 | 9 | – |
| export | 1 | – | – | – | 1 |
| hooks | 1 | 1 | – | – | – |
| laporan | 11 | 5 | 4 | 2 | – |
| pegawai | 1 | – | – | 1 | – |
| profile | 1 | – | – | 1 | – |
| security | 1 | 1 | – | – | – |
| storage | 25 | 11 | 6 | – | 8 |
| styles | 1 | – | – | 1 | – |
| users | 3 | 2 | 1 | – | – |
| utils | 3 | 3 | – | – | – |
| **Total berkas** | **115** | **54** | **30** | **20** | **11** |
| **Total kasus** | **1.222** | **516** | **464** | **101** | **141** |

Berkas campuran dimasukkan ke kategori dominannya:

- B+C: `arsiparis/berkas-arsip-folder-pages`, `arsiparis/berkas-export-zip`, `laporan/export-zip`.
- B+D: `arsiparis/manual-arsip-route`, `dokumen/submit-file-move-transaction`, `storage/upload-route-local`, `storage/raw-preview-internal-url-runtime`.
- A+C: `dokumen/kelengkapan-match`, `laporan/status-laporan`.
- A+D: `export/document-zip`.

Contoh tiap kategori:

- **A:** `tests/fsm.test.ts` (fungsi murni, tanpa mock); `tests/unit/users/create-user-unique-violation.test.ts` (`vi.mock('#/db/client')` dan `#/lib/auth/password`, lalu memanggil `createLocalUserWithRoles`); `tests/unit/arsiparis/berkas-arsip-service.test.ts` (repositori fake di-inject).
- **B:** `tests/unit/laporan/kegiatan-route.test.ts` (6 `vi.mock`, 12 modul `src` diimpor, menjalankan 3 rute); `tests/unit/dokumen/dokumen-transition-guards-route.test.ts` (4 rute transisi + FSM asli + schema Zod asli); `tests/unit/dokumen/master-data-read-requires-session.test.ts` (11 handler GET).
- **C:** `tests/unit/dokumen/cross-role-list-parity-source.test.ts` (membaca 10+ berkas `.tsx`); `tests/unit/auth/login-form-identifier.test.ts`; `tests/unit/styles/color-budget.test.ts`.
- **D:** `tests/unit/storage/pending-upload-sweeper.test.ts` (`mkdtemp` + `utimes`); `tests/unit/storage/local-pending-move.test.ts`; `tests/unit/arsiparis/berkas-arsip-physical-destruction.test.ts` (termasuk `symlink`).

**Catatan untuk klaim white-box:** `tests/TEST-CLASSIFICATION.md` menyebut semua berkas white-box. Ada dua hal yang perlu diluruskan di sana:

1. Dokumen itu masih menyebut "seluruh 90 file", padahal sekarang ada 115.
2. 20 berkas kategori C (101 kasus) tidak mengeksekusi kode, jadi **tidak menyumbang coverage** dan secara metodologis lebih tepat disebut pemeriksaan statis atau *source guard*. Sebaiknya kategori ini dipisahkan dari hitungan "unit test white-box" di skripsi.

### 4.2 Tes yang lebih tepat disebut integration test

- **Sudah diakui** di `TEST-CLASSIFICATION.md`: `dokumen/submit-file-move-transaction.test.ts` (handler rute + sistem berkas nyata + DB tiruan dalam transaksi) dan `storage/pending-upload-sweeper.test.ts` (sistem berkas nyata).
- **Sebaiknya ikut digolongkan integration** (komponen + sistem berkas, atau banyak modul nyata sekaligus):
  - `arsiparis/manual-arsip-route.test.ts`: rute + disk di `.tmp/manual-arsip-route-storage`.
  - `storage/upload-route-local.test.ts`: rute upload + disk.
  - `storage/raw-preview-internal-url-runtime.test.ts`: rute + disk.
  - `laporan/kegiatan-route.test.ts`: 3 rute laporan dengan 12 modul nyata; hanya DB/auth yang di-mock.
  - `dokumen/dokumen-transition-guards-route.test.ts`: 4 rute, FSM, schema, dan rollback lampiran.
- Sisa kategori B (23 berkas) masih bisa disebut *unit test pada tingkat handler (component test)* karena semua I/O eksternal di-mock. Namun di skripsi perlu dinyatakan bahwa unitnya adalah handler rute, bukan satu fungsi.
- Kategori D (11 berkas): bisa disebut unit test dengan *test fixture* sistem berkas sementara. Bila mengikuti ISTQB secara ketat, ini integrasi dengan sistem berkas OS. Pilih satu sebutan dan konsisten.

### 4.3 Kasus yang dilewati dan tes yang pernah gagal sesaat

**Kasus yang dilewati** (#138 di CSV, status `dilewati`): `tests/unit/arsiparis/berkas-arsip-folder-pages.test.ts:641`, `it.skip('keeps the active folder page constrained to open and active sections')`.

- Isinya static source guard: membaca 11 berkas, antara lain `src/routes/kasubag/berkas/index.tsx`, `$id.tsx`, `CloseBerkasDialog.tsx`, `navigation.ts`, dan `routeTree.gen.ts`, lalu mencocokkan puluhan string label lama.
- Komentar di atasnya (L638–640) menyatakan asersi ini akan ditulis ulang di section-06 (halaman UI) setelah label, rute, dan siklus 2 tahap RP-01 diterapkan. Dengan kata lain, tes ini sengaja dinonaktifkan karena UI-nya berubah.
- Risiko saat demonstrasi: **tidak ada risiko gagal** karena tes tidak dijalankan. Hanya saja ringkasan run akan menampilkan "1 skipped". Bila ingin hasil bersih, tes ini perlu ditulis ulang atau dihapus. Apakah asersinya lulus bila `skip` dibuka [BELUM TERVERIFIKASI].

**`master-data-read-requires-session.test.ts` yang pernah gagal sesaat**: dicatat di `docs/skripsi/penjelasan-proyek-aplikasi.md:1049` ("sempat gagal sekali, tetapi lulus saat dijalankan ulang"). Log kegagalannya tidak tersimpan, jadi **penyebab pastinya [BELUM TERVERIFIKASI]**. Dugaan terkuat berdasarkan data run:

- Setiap kasus memanggil `await import('#/routes/api/...')` di dalam badan tes. Impor pertama memuat dan mentransformasi modul rute beserta dependensinya (TanStack Start, Drizzle, schema) saat itu juga.
- Kasus pertama (`master-fungsi returns 401…`) butuh **1.190 ms** (run 1), **1.233 ms** (run 2), dan **1.260 ms** (run 4). Kasus berikutnya hanya 17–60 ms, dan kasus yang sama di blok kedua 1–4 ms. Kasus ini adalah yang paling lambat dari 1.222 kasus di ketiga run.
- Batas waktu bawaan Vitest 5.000 ms tidak diubah di konfigurasi. Dengan ±11 worker paralel dan mesin yang sedang kekurangan memori, impor dingin itu bisa membengkak beberapa kali lipat. Kegagalan yang hilang saat diulang tanpa perubahan kode cocok dengan pola timeout seperti ini.
- **Risiko gagal lagi saat demonstrasi: ada, tetapi rendah** dalam kondisi normal (±1,2 dtk vs batas 5 dtk). Risiko naik bila mesin sedang sibuk.

**Risiko yang lebih besar untuk demonstrasi:** di mesin ini, `pnpm test` dengan jumlah worker bawaan **crash kehabisan memori** (run 0 dan run 3), sedangkan `--maxWorkers=4` selalu lulus. Mitigasi:

- tutup aplikasi berat dan jalankan `pnpm test -- --maxWorkers=4`; atau
- tetapkan `maxWorkers` di `vitest.config.ts`; dan
- untuk tes master-data: impor modul rute di `beforeAll`, atau beri `testTimeout` khusus berkas itu.

Kedua perubahan terakhir menyentuh konfigurasi atau tes, jadi tidak saya terapkan.

---

## 5. Kesiapan disajikan per kasus uji

### 5.1 Lampiran CSV

`lampiran-kasus-uji-unit.csv` (UTF-8 dengan BOM, pemisah koma, baris CRLF agar langsung terbaca Excel):

- Kolom: `nomor, modul, berkas, describe, nama kasus, status, durasi_ms`.
- **1.222 baris**: 1.221 `lulus`, 1 `dilewati`. Setiap baris `it.each` dihitung sendiri, misalnya #26–#29 untuk `KEMBALIKAN by %s → error`.
- `modul` = nama subfolder `tests/unit/<modul>/`. `fsm` untuk `tests/fsm.test.ts`, `lainnya` untuk `tests/unit/kelengkapan-duplicate-validation.test.ts`.
- `describe` = rantai `describe` yang digabung dengan ` > `.
- Urutan: berkas diurutkan alfabetis, lalu urutan kasus dalam berkas. `durasi_ms` berasal dari run 4 (tanpa coverage, 4 worker).

Per modul:

| Modul | Berkas | Kasus | Lulus | Gagal | Dilewati |
|---|---:|---:|---:|---:|---:|
| arsiparis | 22 | 335 | 334 | 0 | 1 |
| auth | 10 | 66 | 66 | 0 | 0 |
| components | 3 | 16 | 16 | 0 | 0 |
| dashboard | 1 | 7 | 7 | 0 | 0 |
| db | 2 | 7 | 7 | 0 | 0 |
| dokumen | 27 | 272 | 272 | 0 | 0 |
| export | 1 | 19 | 19 | 0 | 0 |
| fsm | 1 | 49 | 49 | 0 | 0 |
| hooks | 1 | 4 | 4 | 0 | 0 |
| lainnya | 1 | 6 | 6 | 0 | 0 |
| laporan | 11 | 166 | 166 | 0 | 0 |
| pegawai | 1 | 2 | 2 | 0 | 0 |
| profile | 1 | 6 | 6 | 0 | 0 |
| security | 1 | 9 | 9 | 0 | 0 |
| storage | 25 | 221 | 221 | 0 | 0 |
| styles | 1 | 2 | 2 | 0 | 0 |
| users | 3 | 16 | 16 | 0 | 0 |
| utils | 3 | 19 | 19 | 0 | 0 |
| **Total** | **115** | **1.222** | **1.221** | **0** | **1** |

### 5.2 Penilaian penamaan `describe`/`it`

Angka di bawah berasal dari klasifikasi otomatis berbasis kata kunci (regex) atas 1.222 judul `it` dan 174 judul `describe` unik. Angka ini perkiraan heuristik, bukan hasil baca manual satu per satu.

- **Bahasa:** dominan **Inggris**. Dari judul `it`: 955 (78%) murni Inggris, 154 (13%) Inggris bercampur istilah domain Indonesia ("dokumen", "berkas", "lampiran"), 26 (2%) Indonesia, dan 87 (7%) tidak terklasifikasi (umumnya notasi seperti `DRAFT + SUBMIT → …`). Judul `describe` banyak yang berupa nama fungsi atau fitur (`FSM transition()`, `seedDevelopmentUsers`).
- **Konsistensi:** gaya berbeda antarberkas. `fsm.test.ts` memakai notasi `STATUS + AKSI → HASIL` yang konsisten dan baik untuk tabel. Berkas lain memakai kalimat bebas ("rejects …", "returns 403 for …", "keeps …"). Kode `D-23`, `RP-01` sesekali muncul di judul `describe`.
- **Kondisi dan hasil yang diharapkan:** ±459 judul (38%) memuat penanda kondisi sekaligus hasil (mis. "returns 403 for default local submit when the local actor is not PEGAWAI-compatible"). Ada 49 judul yang sangat pendek (≤ 4 kata), misalnya "Error message is descriptive", "rejects self admin demotion", dan "hashes tokens deterministically"; judul seperti ini biasanya hanya menyebut hasil tanpa kondisi.
- **Duplikasi nama:** 1 kasus. `db/seed-users.test.ts:29–30` memakai `it.each` dengan 2 nilai tanpa placeholder `%s`, sehingga dua baris CSV bernama sama.
- **Saran penyajian di skripsi** (yang ditulis dalam bahasa Indonesia): pertahankan nama asli sebagai bukti yang bisa ditelusuri, lalu tambahkan kolom "Deskripsi (Indonesia)", "Kondisi", dan "Hasil yang diharapkan" di tabel lampiran. Untuk `fsm.test.ts`, ketiga kolom itu bisa diturunkan langsung dari notasinya.

### 5.3 Usulan skema ID (belum diterapkan)

Format **`UT-<MODUL>-<nnn>`**, dengan `nnn` berurutan per modul mengikuti urutan CSV.

| Modul CSV | Kode | | Modul CSV | Kode |
|---|---|---|---|---|
| fsm | FSM | | laporan | LAP |
| arsiparis | ARS | | pegawai | PEG |
| auth | AUT | | profile | PRF |
| components | KMP | | security | SEC |
| dashboard | DSB | | storage | STO |
| db | DB | | styles | STY |
| dokumen | DOK | | users | USR |
| export | EKS | | utils | UTL |
| hooks | HOK | | lainnya | LAN |

Contoh: #1 → `UT-FSM-001`, #26–#29 → `UT-FSM-026` … `UT-FSM-029`.

Cara memasang tanpa mengubah perilaku tes, dari yang paling aman:

1. **ID hanya di lampiran (tanpa menyentuh tes).** Skrip pembentuk CSV menambahkan kolom `id`. Agar stabil, kunci pemetaannya `berkas + fullName + indeks it.each` dan disimpan sebagai `tests/test-ids.json`. Kekurangannya: mengganti judul tes memutus pemetaan.
2. **Metadata tugas Vitest.** Di dalam tes, isi `task.meta.id = 'UT-FSM-001'` melalui konteks tes. Laporan JSON Vitest sudah memuat field `meta` (terlihat kosong `{}` pada run ini). Judul dan asersi tidak berubah.
3. **Prefiks pada judul**, misalnya `it('UT-FSM-001 DRAFT + SUBMIT → …')`. Cara ini paling terbaca di keluaran terminal, tetapi mengubah nama tes. Untuk `it.each`, gunakan `%#` (indeks) di judul.

---

## 6. Kesimpulan

### 6.1 Jawaban

**Belum cukup** untuk klaim "unit testing white-box dengan statement dan branch coverage" bila klaimnya mencakup aplikasi secara umum atau seluruh `src/`. Yang kurang:

1. **Tidak ada konfigurasi coverage** sebelum pemeriksaan ini. Klaim coverage di skripsi saat ini belum didukung data apa pun yang bisa direproduksi dari repo.
2. **Angka coverage seluruh `src/` rendah:** statement 30,42%, branch 24,93%. Penyebab utamanya, `src/components` dan `src/routes` (UI), praktis tidak diuji unit (≈ 2%).
3. **Ada dua mode pengukuran yang hasilnya sangat berbeda.** Angka branch mode bawaan (73,60%) tidak valid sebagai branch coverage. Skripsi harus menyebut mode dan definisi cabang yang dipakai (bagian 1.2), termasuk keterbatasan bahwa operand `&&`/`||` hanya diukur "pernah dievaluasi", bukan condition coverage.
4. **Lapisan API** baru 43,90% statement / 38,58% branch, dan 33 rute 0%.
5. **Penjaga sesi pusat (`local-server-auth.ts`) dan validasi rantai master (`kelengkapan-chain.ts`) 0%.**
6. **FSM hampir lengkap** (decision coverage 35/36, 10/11 jalur), tetapi belum ada kasus J7, cabang `default`, dan satu kombinasi kondisi. Asersi galat juga belum membuktikan jalur yang ditempuh.
7. **20 berkas (101 kasus) static source guard** ikut terhitung sebagai "white-box", padahal tidak mengeksekusi kode. Selain itu, setidaknya 7 berkas lebih tepat disebut integration test.

### 6.2 Pekerjaan yang disarankan (urut prioritas)

Jumlah kasus dan waktu di bawah adalah **perkiraan**, dihitung dari jumlah cabang yang belum tertutup dan pola kode. Dampaknya terhadap persentase coverage [BELUM TERVERIFIKASI] sampai tesnya ditulis dan dijalankan.

| No | Pekerjaan | Kasus baru (perkiraan) | Waktu (perkiraan) |
|---|---|---:|---|
| 1 | Tetapkan metrik dan lingkup: pakai mode AST (atau `@vitest/coverage-istanbul`) di `vitest.config.ts`; set `include` sesuai lingkup klaim; kecualikan modul klien dengan alasan tertulis; tambah `coverage/` ke `.gitignore`; tetapkan `maxWorkers` agar run stabil | 0 | 1–2 jam |
| 2 | Lengkapi FSM: 3–4 kasus (3.8) dan perkuat asersi pesan galat di ±22 kasus yang ada; susun flow graph + tabel jalur + tabel Myers untuk Bab Pengujian | 3–4 | 2–3 jam |
| 3 | `local-server-auth.ts` (±8), `password.ts` (3), `aksi-labels.ts` (4), `kegiatan-scope.ts` (7), `utils/format.ts` (4), `status-laporan.ts` (1), `berkas-arsip-page-format.ts` (±12), `local-submit-repository.ts` (±6), `dokumen/storage.ts` (±5); sebagian besar fungsi murni atau mock ringan | ±50 | 1 hari |
| 4 | `kelengkapan-chain.ts` (db di-mock) | 13 | 3–4 jam |
| 5 | Rute transisi dokumen (approve/reject PPK & PPSPM, kembalikan, `dokumen.$id.submit`, `ppk/resubmit/$id`): rantai penjaga 401/403/400/404/500, bisa memakai satu berkas tabel-driven untuk penjaga bersama | ±50–60 | 1,5–2 hari |
| 6 | Rute GET daftar/inbox (±16 berkas 0%): satu berkas `it.each` untuk 401/403/500 seperti pola `master-data-read-requires-session` | ±48 | 0,5 hari |
| 7 | `users/*` (8 rute) + `local-user-mutations.ts` + `local-user-passwords.ts` | ±40 | 1 hari |
| 8 | Master data CRUD (14 berkas); atau keluarkan dari lingkup klaim unit dan rujuk ke pengujian black-box/e2e | ±90 | 2 hari |
| 9 | Jalur galat yang tersisa di `src/lib/storage` dan `src/lib/archive` (`manual-arsip.ts`, `berkas-arsip-service.ts`, `read-model`, `document-file-access.ts`, dst.) | ±80 | 2–3 hari |
| 10 | Rapikan klasifikasi: perbarui `TEST-CLASSIFICATION.md` (90 → 115 berkas), pisahkan static source guard, tandai 7 berkas integration; tulis ulang atau hapus tes `it.skip` | 0 | 2–3 jam |

Butir 1–4 (±70 kasus, ±2 hari) sudah cukup untuk klaim sempit yang kuat (6.3). Butir 5–7 diperlukan bila `src/routes/api` ingin ikut diklaim.

### 6.3 Cakupan klaim yang paling aman

**Rekomendasi: klaim dibatasi pada `src/lib` sisi server (98 berkas, tanpa modul klien, konstanta, dan tipe) ditambah basis path testing untuk `src/lib/fsm.ts`.** Jangan klaim seluruh `src`, dan jangan klaim `src/lib + src/routes/api` sebelum butir 5–7 selesai.

Alasan:

- **Seluruh `src`** (30,42% / 24,93%) jelas tidak layak. Sebagian besar kode di sana adalah komponen dan halaman React yang memang tidak diuji unit; pengujiannya lewat e2e/black-box. Kode ini sebaiknya dikecualikan dengan menyebut alasannya, bukan dihitung sebagai kegagalan.
- **`src/lib` + `src/routes/api`** saat ini 59,65% / 54,38%. Angka ini bisa disajikan secara jujur, tetapi lemah untuk mendukung klaim. Lapisan API juga lebih banyak diuji di tingkat handler dengan banyak modul nyata (kategori B), yang lebih tepat disebut component/integration test.
- **`src/lib` sisi server** saat ini **74,45% statement / 69,28% branch** (mode AST). Lapisan ini berisi logika bisnis (FSM, validasi, arsip, storage, laporan) dan menjadi target utama 54 berkas tes unit terisolasi. Setelah butir 2–4, branch coverage-nya kemungkinan naik mendekati atau melewati 75% [BELUM TERVERIFIKASI].
- **Basis path hanya untuk `fsm.ts`** bisa dipertahankan karena V(G) per fungsi ≤ 15 dan 10/11 jalur sudah teruji. Setelah 3–4 kasus pada butir 2 ditambahkan, jalurnya 11/11 dan decision coverage 36/36.

Contoh rumusan untuk skripsi (sesuaikan angka setelah butir 1–4):

> Unit testing dilakukan dengan teknik white-box menggunakan Vitest 3.2.4. Kecukupan pengujian diukur dengan statement coverage dan branch coverage (penyedia V8 dengan pemetaan AST; cabang meliputi hasil benar/salah `if`, `switch`, operator ternary, dan setiap operand `&&`/`||`/`??`) pada lapisan logika sisi server (`src/lib`), dengan hasil statement X% dan branch Y%. Modul transisi status dokumen (`src/lib/fsm.ts`) diuji tambahan dengan basis path testing: V(G) = 11 untuk fungsi `transition()` dan 7 untuk `isActorValidForAction()`, dan seluruh jalur independen dipetakan ke kasus uji. Komponen antarmuka dan handler rute diuji di tingkat sistem (black-box/e2e) dan tidak termasuk dalam pengukuran coverage unit.

---

## Lampiran: cara mereproduksi

Coverage mode AST (angka yang disarankan dikutip):

```bash
pnpm test --coverage --coverage.experimentalAstAwareRemapping=true --coverage.reportsDirectory=./coverage-ast --maxWorkers=2
```

Coverage mode bawaan (pembanding):

```bash
pnpm test --coverage --maxWorkers=2
```

FSM saja:

```bash
npx vitest run tests/fsm.test.ts --coverage --coverage.include=src/lib/fsm.ts --coverage.experimentalAstAwareRemapping=true
```

Laporan HTML ada di `coverage-ast/index.html` dan `coverage/index.html`.

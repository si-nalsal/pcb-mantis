# Rencana re-placement SvayamV3 — MCU STM32H743IIT6 (LQFP176)

Dibuat 13 Nov 2026. Dokumen ini adalah instruksi kerja untuk sesi berikutnya (model Opus). Kerjakan urut dari atas; jangan mengulang analisis yang sudah ada di sini.

## 0. Status saat ini

- Skematik dan PCB sudah memakai `Svayam:STM32H743IITx` / `Package_QFP:LQFP-176_24x24mm_P0.5mm`. U1 di F.Cu sekitar (109.6, 79.9), rotasi 0°.
- Pemilik sudah menyimpan PCB dari KiCad setelah penggantian MCU. **Selalu baca file dari disk, jangan pakai salinan lama.**
- ERC sudah dijalankan pemilik di KiCad setelah penggantian MCU dan hasilnya bersih (konfirmasi pemilik, 13 Nov 2026).
- Gambar referensi: `docs/images/placement_plan_LQFP176.png` (zona placement + fanout PWM). Dibuat ulang dengan `python3 scripts/placement/pwm_route_preview.py` dari folder `scripts/placement/`.
- Rollback: `archive/SvayamV3_IIK_pre-LQFP176_20261113_0759.zip` (masih BGA). Sebelum mengubah apa pun, buat arsip baru `archive/SvayamV3_IIK_LQFP176_pre-replacement_<tanggal>.zip`.
- Folder proyek di komputer pemilik: `SvayamV3_IIK - v1edit`, dikerjakan lewat `device_bash` (`$HOME/mnt/SvayamV3_IIK - v1edit`). Hapus file tidak diizinkan; pindahkan sampah ke `_to_delete/`.
- KiCad harus **ditutup** selama file diedit. Periksa file `~*.lck`; kalau ada, minta pemilik menutup KiCad.

## 1. Alat yang sudah ada (jangan ditulis ulang)

`scripts/placement/`:
- `sx.py` — parser S-expression (`parse`, `Node.find/findall`, tiap node punya offset `.a/.b` untuk edit teks di tempat).
- `geo.py` — `ref(f)`, `pads(f)` (koordinat global), `crtyd(f)` (bbox courtyard).
- `render2.py <pcb> <png> x0,y0,x1,y1 [ref,ref,...]` — gambar F.Cu/B.Cu berdampingan, ref yang disebut diwarnai merah.
- `check.py <pcb> ref,ref,...` — cek tumpang tindih courtyard per layer.
- `pwm_route_preview.py` — hitung persilangan fanout PWM dan gambar `docs/images/placement_plan_LQFP176.png`.

`docs/lqfp176_pinmap.csv` — 176 pin: nomor, sisi, posisi lokal, pinfunction, net.

Cara edit yang terbukti aman: parse → kumpulkan `(a, b, teks_baru)` → terapkan dari offset terbesar ke terkecil. Saat memutar footprint, sudut `(at ...)` pada pad, property dan fp_text ikut ditambah selisih rotasinya. Setelah menulis, cek jumlah `(` sama dengan `)`.

## 2. Keputusan pemilik

- Cakupan: blok di sekitar MCU — resistor seri, pull-up, crystal, FRAM, transceiver CAN, decoupling, **dan blok OC tiap peripheral** (TPS2553 + kapasitor + pull-up flag).
- **Terkunci: hanya konektor servo J19–J34** (beserta zona +VSERVO). Konektor lain, tombol, dan outline board boleh disesuaikan.
- Tidak perlu rapat. Board tidak perlu kecil, tapi jangan membesar. Cari yang efisien.
- **Routing global dibuang.** Pemilik yang akan routing ulang.
- **Kompleks blok power (mux TPS2120 U10 + buck-boost TPS63070 U11, L1, D4/D5 dan semua pasifnya, sekitar x 59–81, y 44–71) tidak boleh disentuh sama sekali**: komponen tidak dipindah, track dan via di dalamnya tidak dihapus. Tentukan anggota blok dari net-nya (+VBRICK1/2, /Power/VIN_BB, Net-(U11-*), +5V_IN lokal) dan dari kotak koordinat; track yang keluar dari kotak dipotong di batas kotak hanya bila seluruh segmennya di luar.
- MCU boleh dirotasi.
- Urutan channel PWM boleh diubah. Label/penomoran konektor tidak penting bagi pemilik; yang penting routing mudah dan bersih. **Label "Channel N" tidak diubah**, cukup pemetaannya didokumentasikan.

## 3. Asumsi yang belum dikonfirmasi (pakai ini kecuali pemilik bilang lain)

1. Decoupling MCU tetap di B.Cu di bawah pin VDD, seperti desain BGA sebelumnya.
2. Outline board tidak diubah pada tahap ini. Setelah placement selesai, laporkan ruang kosong dan usulkan pemotongan tinggi board sebagai langkah terpisah.
3. Komponen tetap di sisi yang sekarang (konektor JST di B.Cu, IC dan pasif di F.Cu).

## 4. Analisis: sisi MCU pada rotasi 0°

Pin 1 di kiri atas, urutan berlawanan jarum jam: KIRI 1–44 (atas→bawah), BAWAH 45–88 (kiri→kanan), KANAN 89–132 (bawah→atas), ATAS 133–176 (kanan→kiri).

| Sisi | Fungsi (pin) |
|---|---|
| KIRI | FRAM SPI4 PE2–PE6 (1–5), LED (8, 9), PERI_EN (17), PWR_SEL (18), ADC3 (19, 24), CS_EXT1 (21), SPI5 (25–27), CS_EXT2 (28), OSC (29, 30), NRST (31), ADC1 (32, 33), BARO2 MISO (34), TIM2 CH1–3 (40–42) |
| BAWAH | PERI_OC (47), TIM3 (53, 56, 57), ADC2 (54, 55), I2C4 (64, 65), UART7 (68, 69, 70, 73), OC SERIAL5/I2C/CAN/CAN2/SPI (74–78), USART3 TX/RX (79, 80), VCAP (81), TIM5 (87, 88) |
| KANAN | FDCAN2 (92, 93), USART1 (94, 95), OC TELEM2/SERIAL4 (96, 97), GPIO buzzer (98), USART3 CTS/RTS (99, 100), TIM4 (101, 104), ALARM (105), SAFETY (107, 110), TIM8 CH1/2 (115, 116), SD D0/D1 (117, 118), TIM1 (119–121), USB (122, 123), SWDIO (124), VCAP (125), UART4 (128, 129), BARO2 CS/SCK (131, 132) |
| ATAS (kanan→kiri) | TIM8_CH4 (133), BARO2 MOSI (134), SWCLK (137), SD D2/D3/CK/CMD (139–141, 144), OC GPS/TELEM1 (142, 143), USART2 (145–147, 150), SPI1 MOSI (151), USART6_RX (152), CS BMI088 (153, 154), SPI6 (155–157), SPI1 SCK/MISO (161, 162), CS BMI270 (163), I2C1 (164, 165), BOOT0 (166), FDCAN1 (167, 168), PE1 (170), PDR_ON (171) |

**Kesimpulan rotasi: pertahankan 0°.** Sensor (SPI1, SPI6, I2C1) keluar dari sisi atas tepat ke arah kompleks sensor; PWM keluar dari kiri-bawah, bawah, dan kanan menuju rel servo di bawah; SD/USB/SWD keluar di kanan dan kanan-atas tempat J10/J1/J7 berada; SPI5/ADC/power-sense keluar di kiri tempat J3/J9/J16 berada. Rotasi 45° hanya berguna untuk BGA; pada LQFP malah mempersulit decoupling dan fanout.

Posisi U1: boleh digeser beberapa mm. Target pusat sekitar x 108–111, y 78–82; sisakan minimal 6 mm antara tepi atas LQFP dan IMU terdekat, dan minimal 14 mm antara tepi bawah LQFP dan baris servo atas untuk kanal konektor bawah.

## 5. Kompleks sensor — dinilai cukup baik, tidak dipindah

Posisi sekarang: U9 BMM150 (108.6, 46.3), U5 BMP388 (104.4, 49.6), U7 BMI088 (113.1, 51.0), U6 DPS368 (112.8, 55.6), U8 BMI270 (108.6, 58.8), LDO U13 (102.9, 53.4), pulau +3V3_SENS di In3.

- Terpusat di sumbu x (pusat board x ≈ 110.6), tetapi di sepertiga atas, bukan di tengah secara y (pusat y ≈ 88). Offset ini tidak melanggar aturan layout; offset IMU terhadap pusat massa dikompensasi di firmware.
- Yang penting sudah terpenuhi: jauh dari lubang mounting (≥ 60 mm), jauh dari arus servo (≥ 55 mm, baik untuk magnetometer), punya LDO dan pulau tegangan sendiri, dan bus-nya langsung berhadapan dengan sisi atas MCU.
- Catatan untuk pemilik (tidak dikerjakan otomatis): BMM150 hanya ±3 mm dari tepi atas board dan ±33 mm dari induktor L1; baro sebaiknya diberi busa/penutup terhadap cahaya dan aliran udara.
- Yang dikerjakan: kapasitor/resistor lokal sensor tidak disentuh. Hanya periksa jarak ke U1 setelah U1 final.

## 6. Perubahan skematik: urutan channel PWM

Tujuan: urutan pin timer mengelilingi MCU (kiri-bawah → bawah → kanan → atas) sama dengan urutan kolom konektor servo kiri → kanan, sehingga tidak ada persilangan. Posisi fisik konektor tidak berubah; yang ditukar hanya net timer mana yang masuk ke resistor seri mana.

Kolom servo (x): 69.5, 81.6, 94.1, 106.5, 118.9, 131.3, 143.8, 156.2. Baris atas y ≈ 112.6, baris bawah y ≈ 127.

| Kolom | Baris atas (konektor, R seri) | Net baru | Baris bawah (konektor, R seri) | Net baru |
|---|---|---|---|---|
| 1 | J19, R68 | TIM2_CH1 (40) | J21, R70 | TIM2_CH2 (41) |
| 2 | J23, R72 | TIM2_CH3 (42) | J25, R74 | TIM3_CH2 (53) |
| 3 | J27, R76 | TIM3_CH3 (56) | J29, R78 | TIM3_CH4 (57) |
| 4 | J31, R80 | TIM5_CH1 (87) | J33, R82 | TIM5_CH2 (88) |
| 5 | J20, R69 | TIM4_CH3 (104) | J22, R71 | TIM4_CH2 (101) |
| 6 | J24, R73 | TIM8_CH2 (116) | J26, R75 | TIM8_CH1 (115) |
| 7 | J28, R77 | TIM1_CH2 (120) | J30, R79 | TIM1_CH1 (119) |
| 8 | J32, R81 | TIM8_CH4 (133) | J34, R83 | TIM1_CH3 (121) |

Aturan di balik tabel: baris bawah dicapai lewat celah antar kolom baris atas. Kolom 1–4 (di kiri/bawah MCU) memakai celah di **kanan** kolomnya, jadi urutannya atas lalu bawah. Kolom 5–8 (di kanan MCU) memakai celah di **kiri** kolomnya, jadi urutannya bawah lalu atas. Celah antara kolom 4 dan 5 membawa dua jalur (TIM5_CH2 dan TIM4_CH2); celah lain masing-masing satu.

### Hasil verifikasi (13 Nov 2026, `pwm_route_preview.py`)

- Persilangan topologis (jumlah pasangan net yang urutannya terbalik antara pin MCU dan target): **pemetaan lama 32, pemetaan baru 0**.
- Rute ilustratif satu layer untuk ke-16 net digambar dan diuji dengan shapely: **0 persilangan**, jarak minimum antar jalur 0.5 mm. Artinya seluruh 16 PWM bisa dirutekan di satu layer tanpa via, sebagai tiga berkas: kiri-bawah (TIM2 + TIM3), tengah (TIM5), dan kanan (TIM4, TIM8, TIM1).
- Batasan verifikasi: ini uji topologi, bukan DRC. Lebar celah antar konektor servo dan lebar kanal di antara MCU dan rel servo belum dicek terhadap lebar track/clearance; konektor JST sisi bawah (B.Cu) berada di area yang sama dengan berkas PWM, jadi PWM sebaiknya di F.Cu atau layer dalam.
- TIM8_CH4 (pin 133, sisi atas) harus memutar lewat pojok kanan-atas MCU, di luar jalur TIM1/TIM8 lainnya; sisakan kanal ±5 mm di kanan MCU (x ≈ 123.5–128.5) bebas dari resistor seri dan konektor.

Langkah:
1. Di `connector.kicad_sch` (atau sheet tempat R68–R83 berada), tukar teks label net pada sisi MCU tiap resistor sesuai tabel. Jangan memindah simbol dan jangan mengubah label "Channel N".
2. Di PCB, perbarui `(net ...)` pad sisi MCU R68–R83 agar sama dengan skematik.
3. Jalankan ulang `pwm_route_preview.py` setelah U1 dan resistor final; hasilnya harus tetap 0 persilangan.
4. Tulis tabel final ke `docs/PWM_MAPPING.md` (konektor/label channel → timer → pin MCU) untuk hwdef firmware. Catat bahwa channel satu timer tidak lagi berurutan nomornya (mis. TIM2 = Channel 1, 9, 2).

Opsional, hanya dengan persetujuan pemilik:
- Flag OC adalah GPIO biasa. Lima flag di pin 74–78 boleh ditukar urutannya supaya sesuai urutan posisi load switch setelah placement.
- BARO2 MISO ada di pin 34 (kiri) sedangkan SCK/MOSI/CS di pojok kanan-atas (131, 132, 134). Memindahkannya ke PI2 (pin 133) akan mengumpulkan SPI2, tetapi pin itu dipakai TIM8_CH4; perlu cek alternate function di CubeMX. Kalau tidak diubah, MISO dirutekan lewat layer dalam.

## 7. Aturan placement

Jarak antar courtyard minimal 0.5 mm, target 1 mm. Pasif disejajarkan pada grid 0.25 mm dengan orientasi seragam per kelompok.

**a. Decoupling MCU (B.Cu)** — satu kapasitor per pin VDD, pad +3V3 tepat di bawah pin: C11→6 (VBAT), C2→15, C6→23, C20→36, C4→38 (VREF+), C1→39 (VDDA), C18→49, C21→62, C22→72, C23→82, C5→91, C12→103, C14→114 (VDD33_USB), C24→127, C7→136, C8→149, C9→159, C10→172. VCAP: C17→81, C19→125, sedekat mungkin. C3 (4u7 bulk) di bawah badan MCU. Hitung ulang posisi relatif terhadap pusat U1 yang final; jangan menabrak konektor B.Cu.

**b. Crystal** — Y1, C13, C16 di sisi kiri sejajar pin 29/30, jarak ≤ 5 mm dari pin, jalur simetris, tidak ada komponen lain di antaranya, sisakan ruang untuk guard GND.

**c. Resistor seri dan pull-up** — satu baris 2–4 mm di luar courtyard LQFP, tiap resistor sejajar dengan pinnya, pad sisi MCU menghadap MCU:
- KIRI: R54, R55 (ADC3), R63, R64 (ADC1), R2, R3 (LED) + D2, D3 di luarnya.
- BAWAH: R56, R57 (ADC2), R7, R8 (pull-up I2C4), R66, R67 (seri I2C4), R39–R42 (UART7), R58, R59 (USART3).
- KANAN: R33, R34 (USART1), R60, R61 (USART3 CTS/RTS), R51, R52 (UART4), R62, R65 (SAFETY).
- ATAS: R35–R38 (USART2), R10, R11 (pull-up I2C1), R5 (PDR_ON).
- R68–R83 (PWM) tetap di dekat konektor servo masing-masing.
- Pull-up SD (R32, R43, R44, R46, R48) tetap di dekat J10.

**d. FRAM U2** — di luar pojok kiri-atas (pin 1–5 dan 170), dengan R6, R9, R12 dan C26 menempel padanya.

**e. CAN** — U3 (CAN1, pin 167/168) di **kiri-atas** MCU, bukan kanan-atas seperti sekarang; J13 dan load switch U21 ikut pindah ke sisi kiri-atas, di antara blok power dan kompleks sensor (sekitar x 82–100, y 46–60; LDO U12 boleh digeser). U4 (CAN2, pin 92/93) di kanan-bawah; J18 dan U24 ikut ke sisi bawah-kanan. Terminasi R13/R14 dan decoupling C25/C27 menempel pada transceiver-nya.

**f. Konektor JST (B.Cu) dan blok OC-nya** — susun mengelilingi MCU sesuai sisi pinnya, dengan urutan sepanjang tiap sisi mengikuti urutan pin:
- Kiri (x ≈ 55–92), atas→bawah: J9 POWER1, J3 SPI, J16 POWER2, J17 I2C.
- Bawah (y ≈ 96–104, di antara MCU dan rel servo), kiri→kanan: J12 ADC, J5 TELEM2, J14 SERIAL5, J18 CAN2, J2 GPS, J11 BUZZER.
- Kanan (x ≈ 126–150), bawah→atas: J15 SAFETY, J8 SERIAL4, J4 TELEM1.
- Kanan jauh, tetap: J10 microSD, J1 USB + U15, J7 SWD, J6 SBUS.
- Kiri-atas: J13 CAN1.
- Tiap load switch TPS2553 (U16–U24) beserta dua kapasitor 1 µF dan pull-up flag-nya (R93–R101) diletakkan sebagai satu kelompok tepat di samping konektornya, dengan urutan aliran daya: +5V_PERIPH → C masuk → switch → C keluar → pin 1 konektor. Susunan internal kelompok yang sekarang sudah baik; pindahkan sebagai satu unit kaku.
- U14 (AP22804, PERI_EN/PERI_OC) di kiri-bawah dekat sumber +5V_IN.

**g. Lain-lain** — Q1/D7/R49/R50/R53 (buzzer) ikut J11. SW1 (BOOT) dan R4 di tepi kiri; SW2 (RESET) dan C15 di tepi kanan-atas dekat J7. LED D1/R1 tidak diubah.

## 8. Urutan kerja

1. Cek lock KiCad, buat arsip.
2. Terapkan perubahan skematik bagian 6, lalu samakan net pad di PCB.
3. Buang routing global: hapus semua `segment`, `arc`, `via` kecuali milik blok power (bagian 2). Zone dipertahankan. Laporkan jumlah yang dihapus dan yang dipertahankan, dan pastikan tidak ada footprint blok power yang berubah (bandingkan teks footprint sebelum/sesudah).
4. Tulis satu skrip `scripts/placement/replace.py` berisi tabel penempatan (ref → x, y, rotasi; kelompok kaku dipindah dengan satu offset). Jalankan, lalu `check.py` sampai 0 tumpang tindih, dan cek semua komponen berada di dalam Edge.Cuts dengan jarak ≥ 1 mm dari tepi.
5. Render dengan `render2.py`, lihat gambarnya, perbaiki yang janggal. Maksimal dua iterasi sebelum ditunjukkan ke pemilik.
6. Hitung persilangan ratsnest sederhana (garis lurus pin MCU → pad tujuan pertama) per sisi dan laporkan net yang masih bersilangan.
7. **Kirim gambar ke pemilik dan tunggu persetujuan sebelum menimpa `SvayamV3_IIK.kicad_pcb`.** Kerjakan dulu pada salinan.
8. Setelah disetujui: tulis file, perbarui `docs/STATUS.md`, laporkan ruang kosong dan usulan ukuran board. Jangan commit ke git kecuali diminta.

Yang harus dikerjakan pemilik di KiCad sesudahnya: ERC, Update PCB from Schematic, isi ulang zone (B), DRC.

## 9. Hemat pemakaian

- Jangan mencetak isi file `.kicad_pcb`/`.kicad_sch` ke output; selalu olah dengan skrip dan cetak ringkasan saja.
- Satu skrip untuk semua pemindahan, bukan satu panggilan per komponen.
- Render hanya area yang relevan (mis. `50,40,172,108`), paling banyak tiga kali.
- Tidak perlu mencari ulang data pin: pakai `docs/lqfp176_pinmap.csv` dan tabel bagian 4.

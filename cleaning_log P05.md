# Cleaning Log

**Tim:** [isi nama tim]
**Dataset:** transaksi_koperasi_2025.csv
**Baris mentah:** 1543 → **baris bersih:** 1493
**Terakhir diperbarui:** 2025-11-10 (tanggal pengerjaan: 10 November 2026)

---

## Urutan langkah

| # | Langkah | Kolom | Baris terdampak | Keputusan | Alasan |
|---|---|---|---|---|---|
| 1a | Perbaiki tipe data | total_bayar | 250 | Buang "Rp" dan titik pemisah ribuan, lalu `to_numeric(errors='coerce')`. Gagal konversi: 0. | Selama uang masih teks, tidak ada pemeriksaan angka yang bisa jalan. Titik di sini pemisah ribuan, bukan desimal. |
| 1b | Perbaiki tipe data | tanggal_transaksi | 1543 | Format ISO (`%Y-%m-%d %H:%M:%S`) dan format `%d/%m/%Y %H:%M` diurai terpisah. Gagal: 0. Rentang 1 Agu – 28 Nov 2025 (4 bulan). | `format="mixed"` dengan `dayfirst` merusak 483 baris berformat ISO tanpa memunculkan galat. Hasil lolos uji akal sehat (empat bulan, Agustus–November). |
| 2 | Buang spasi pada kolom teks | Seluruh kolom teks | 160 | `strip()` pada semua kolom teks sekaligus. | Spasi tersembunyi memecah `groupby` dan melambungkan `nunique` (mis. id_anggota 318 → 259 unik). |
| 3a | Placeholder menjadi kosong | Kolom teks | 19 | `n/a`, `na`, `-`, teks kosong, `null`, `none`, `tidak diketahui`, `unknown`, `?`, `#n/a` diubah menjadi NaN. | Itu bukan nilai sungguhan. Dilakukan sebelum penyeragaman agar tidak terhitung sebagai kategori. |
| 3b | Placeholder menjadi kosong | rating_layanan | 49 | `-999` diubah menjadi NaN (menyisakan 46 kosong pada data bersih setelah baris duplikat dibuang). | Angka -999 merusak rata-rata tanpa pernah terdeteksi `isna()`. |
| 4a | Seragamkan kategori | kategori | 2 label disatukan (bukan jumlah baris) | `strip().title()` oleh kode, lalu sinonim dan salah ketik lewat peta keputusan (lihat bawah). Kategori unik 18 → 5. | Huruf dan spasi bisa diputuskan mesin; sinonim dan salah ketik diputuskan manual dan dicatat di peta agar bisa ditinjau. |
| 4b | Seragamkan kategori | metode_bayar, kasir | belum dihitung | `strip().title()`; `Qris` → `QRIS`. metode_bayar: 4 nilai (QRIS, Saldo Koperasi, Transfer, Tunai); kasir: 4 nama. | Alasan sama dengan 4a: variasi huruf memecah pengelompokan. |
| 5a | Tangani duplikat | Seluruh kolom | 39 | `drop_duplicates()`, satu baris disimpan. | Seluruh kolom identik. Jumlahnya lebih besar daripada diagnosis pertemuan 4 (24) karena setelah teks diseragamkan, baris yang tadinya beda kapitalisasi menjadi identik. |
| 5b | Tangani duplikat | id_transaksi | 15 baris terlibat | Satu id satu baris; dipertahankan baris yang `catatan_kasir`-nya terisi. | Satu transaksi hanya boleh satu baris; baris dengan catatan lebih informatif. |
| 5c | Tangani duplikat | id_transaksi | 1 | Baris tanpa id_transaksi dibuang. | Kunci utama kosong tidak dapat ditelusuri kembali ke sumber. |
| 6a | Tangani nilai mustahil | jumlah | 29 | `jumlah <= 0` dijadikan kosong. | Belum jelas retur atau salah input; barisnya masih berguna untuk kolom lain. Alternatif membuang baris ditolak karena kehilangan transaksi yang mungkin sah. |
| 6b | Tangani nilai mustahil | diskon_persen | 9 | Nilai di luar 0–100 dijadikan kosong. | Nilai mustahil; menunggu konfirmasi pengurus. |
| 6c | Tangani nilai mustahil | total_bayar | 50 | Tidak ditimpa; ditandai di kolom `total_mencurigakan` (selisih dengan `jumlah × harga_satuan × (1 − diskon/100)` lebih dari Rp1). | Belum diketahui kolom mana yang keliru (total_bayar, jumlah, atau harga_satuan). Menimpa total_bayar sama dengan memutuskan tiga kolom lain pasti benar tanpa dasar. |
| 7a | Tangani nilai kosong | saldo_simpanan | 156 | Dibiarkan kosong, tidak diimputasi; keterbatasan dilaporkan. | Diduga MNAR (saldo besar lebih sering disembunyikan). Imputasi apa pun, juga membuang baris, membuat rata-rata saldo tampak lebih kecil daripada kenyataan. |
| 7b | Tangani nilai kosong | berat_kg | 290 | Ditandai di kolom `berlaku_berat` (False untuk kategori Pulsa dan Jasa); tidak diisi. | Kosong karena tidak berlaku (barang non-fisik tak punya berat), bukan karena hilang. |
| 7c | Tangani nilai kosong | catatan_kasir | 1004 | Dijadikan penanda biner `ada_catatan`. | Kosong berarti kasir tidak menulis apa pun; kekosongan itu sendiri sebuah kategori. |
| 8 | Buat kolom turunan | bulan, hari, jam, nilai_bersih, kelas_nilai | 1493 | `bulan` (periode bulanan), `hari`, `jam` dari tanggal; `nilai_bersih` = total_bayar; `kelas_nilai` dari `pd.cut` dengan batas 10.000 / 30.000 / 75.000 (Sangat kecil, Kecil, Sedang, Besar), dibuat berurutan (ordinal). | Kolom turunan hanya bermakna di atas data bersih. Batas bin berasal dari pengetahuan harga barang koperasi, bukan rumus; mengubah batas akan mengubah kesimpulan. One-hot tidak dipakai karena pada EDA kategori lebih mudah dibaca. |

---

## Peta keputusan manual

| Nilai asal | Diubah menjadi | Dasar keputusan |
|---|---|---|
| Atk | ATK | Penyeragaman penulisan. |
| Alat Tulis | ATK | Sinonim; dipastikan ke pengurus koperasi. |
| Minumam | Minuman | Salah ketik yang jelas. |
| Qris | QRIS | Penyeragaman penulisan singkatan. |

---

## Yang sengaja TIDAK diubah

| Masalah | Mengapa dibiarkan | Akibat bagi analisis |
|---|---|---|
| saldo_simpanan kosong (156 baris) | Diduga MNAR; tidak ada teknik statistik yang menyelamatkan, dan imputasi menggeser sebaran ke bawah. | Rata-rata saldo hanya batas bawah; angka sebenarnya lebih tinggi karena anggota bersaldo besar tidak menampilkan saldonya. |
| total_bayar tidak cocok (50 baris) | Ditandai, tidak ditimpa, karena kolom yang keliru belum diketahui. | Analisis perlu dijalankan dengan dan tanpa baris bertanda untuk melihat apakah kesimpulan berubah. |
| jumlah kosong hasil pembersihan (29 baris) | Belum jelas retur atau salah input; barisnya masih berguna untuk kolom lain. | Baris ini tidak ikut perhitungan yang memakai jumlah, dan tidak dapat diperiksa kecocokan totalnya. |
| diskon_persen kosong hasil pembersihan (9 baris) | Nilai mustahil, menunggu konfirmasi pengurus; tidak ditebak. | Sama seperti di atas: tidak dapat diperiksa kecocokan totalnya. |
| rating_layanan kosong (46 baris) | Tidak diimputasi agar rata-rata tidak dikarang. | Rata-rata rating dihitung hanya dari baris yang berisi. |
| metode_bayar kosong (28 baris) | Tidak diimputasi; nilai sebenarnya tidak diketahui. | Distribusi metode bayar dihitung tanpa baris ini (atau dilaporkan sebagai "tidak tercatat"). |

---

## Pertanyaan yang belum terjawab

- [ ] Apakah "Alat Tulis" memang sama dengan "ATK"? (tanyakan ke pengurus koperasi)
- [ ] Apakah `jumlah` nol atau negatif (29 baris) adalah retur atau salah input?
- [ ] Apa penyebab diskon di luar 0–100 (9 baris)?
- [ ] Pada 50 baris yang totalnya tidak cocok, kolom mana yang keliru: total_bayar, jumlah, atau harga_satuan?
- [ ] Mengapa saldo_simpanan kosong pada 156 baris, dan dapatkah sumber data melengkapinya?

---

## Dampak keseluruhan

| Ukuran | Sebelum | Sesudah | Catatan |
|---|---|---|---|
| Jumlah baris | 1543 | 1493 | 50 baris dibuang (duplikat dan id kosong). |
| Jumlah kolom | 16 | 26 | Bertambah 10 kolom: total_seharusnya, selisih_total, total_mencurigakan, berlaku_berat, ada_catatan, bulan, hari, jam, nilai_bersih, kelas_nilai. |
| Total sel kosong | [isi: jumlah NaN data mentah] | [isi: jumlah NaN data bersih, pada 16 kolom asli] | Bertambah pada rating_layanan (+46), jumlah (+29), metode_bayar (+19), diskon_persen (+9) karena placeholder dan nilai mustahil kini jujur mengaku kosong. Berkurang pada catatan_kasir (−40), berat_kg (−10), id_transaksi (−7), saldo_simpanan (−3) karena baris duplikat dibuang. |
| Baris duplikat | 24 (persis) | 0 | Lebih banyak terdeteksi setelah teks diseragamkan (39 dibuang pada langkah 5a). |
| Kategori pada kolom kunci | 18 (kategori); 318 (id_anggota) | 5 (kategori); 259 (id_anggota) | Penyeragaman, spasi, dan sinonim. |

---

## Reproduktibilitas

- [ ] Notebook berjalan dari awal sampai akhir setelah *Restart and run all*
- [ ] Pembersihan dibungkus dalam satu fungsi, bukan tersebar di banyak sel
- [ ] `data/raw/` tidak berubah dan tidak ikut diunggah
- [x] Hasil tersimpan di `data/processed/` (`transaksi_bersih.csv`)
- [ ] `python cek_m2.py` berjalan tanpa status GAGAL

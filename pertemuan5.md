# PERTEMUAN 5 – AGGREGATION FRAMEWORK
**Program Studi:** Teknologi Rekayasa Informatika Industri (TRIN)
**Alokasi Waktu:** 1 × 150 menit (praktikum)

---

## A. CAPAIAN PEMBELAJARAN PERTEMUAN 5
Setelah mengikuti perkuliahan ini, mahasiswa mampu:
1. Menjelaskan konsep aggregation pipeline dan cara kerjanya.
2. Menggunakan stage-stage utama: `$match`, `$group`, `$sort`, `$project`, `$unwind`, `$lookup`, `$addFields`, `$bucket`, `$facet`.
3. Menerapkan akumulator (`$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet`, `$first`, `$last`) untuk menghasilkan ringkasan data industri.
4. Memanfaatkan operator ekspresi (`$dateToString`, `$hour`, `$cond`, `$ifNull`, `$multiply`, `$divide`) untuk transformasi data di dalam pipeline.
5. Menggabungkan data dari beberapa koleksi menggunakan `$lookup` (join).
6. Merancang pipeline agregasi untuk menjawab pertanyaan bisnis nyata: total produksi per mesin, rata‑rata suhu per jam, reject rate, OEE, distribusi nilai sensor.
7. Mengoptimasi pipeline dengan menempatkan `$match` dan `$limit` di awal, serta memproyeksikan field yang diperlukan saja.

---

## B. ALAT DAN BAHAN
- Server MongoDB aktif.
- MongoDB Shell (`mongosh`).
- MongoDB Compass (sangat membantu untuk membangun pipeline secara visual dan melihat hasil tiap stage).
- Dataset: koleksi `log_produksi`, `sensor`, `maintenance` dari pertemuan sebelumnya (atau disediakan dataset baru). Minimal 1000 dokumen untuk agregasi bermakna.
- Script Python (opsional) untuk menambah volume data.

---

## C. URAIAN MATERI SUPER LENGKAP

### 1. Konsep Aggregation Pipeline
Aggregation framework adalah alat utama di MongoDB untuk memproses, mengubah, dan menganalisis data. Data mengalir melalui serangkaian **stage**, di mana output dari satu stage menjadi input stage berikutnya. Ini mirip dengan pipeline di Unix (`|`) atau operasi ETL.

```javascript
db.collection.aggregate([ stage1, stage2, stage3, ... ])
```

Setiap stage adalah dokumen yang mendeskripsikan operasi. MongoDB mengeksekusi pipeline secara berurutan, dan dapat mengoptimasi dengan menggabungkan `$match` dan `$sort` dengan indeks di awal pipeline.

**Keuntungan agregasi:**
- Semua pemrosesan terjadi di server, mengurangi data yang dikirim ke klien.
- Ekspresif dan lebih cepat dibandingkan banyak query di aplikasi.
- Mendukung operasi kompleks (group, unwind, join) dalam satu perintah.

### 2. Struktur Dasar Pipeline dan Stage
Pipeline adalah array. Stage paling umum:
- `$match` – filter dokumen (seperti WHERE).
- `$group` – mengelompokkan dan menghitung agregat.
- `$sort` – mengurutkan dokumen.
- `$project` – memilih, menambah, atau mengubah field.
- `$unwind` – memecah array menjadi dokumen individual.
- `$lookup` – join dengan koleksi lain.
- `$addFields` / `$set` – menambahkan field baru.
- `$limit` – membatasi jumlah dokumen.
- `$skip` – melewati sejumlah dokumen.
- `$count` – menghitung jumlah dokumen.
- `$bucket` / `$bucketAuto` – mengelompokkan dokumen ke dalam interval.
- `$facet` – menjalankan beberapa pipeline paralel pada data yang sama.

### 3. Stage dan Contoh Detail (dengan konteks industri)

#### a. `$match`
Memfilter dokumen, sama dengan query filter. Letakkan **paling awal** untuk mengurangi dokumen yang diproses selanjutnya, terutama jika didukung indeks.
```javascript
{ $match: { mesin: "M001", waktu: { $gte: ISODate("2026-05-01") } } }
```
**Tips:** Gunakan `$match` sedini mungkin untuk memanfaatkan indeks.

#### b. `$project`
Memilih field yang akan diteruskan, menambahkan field baru hasil komputasi, atau menghilangkan field. Bisa juga merubah struktur.
```javascript
{ $project: {
    _id: 0,
    mesin: 1,
    suhu: 1,
    tahun: { $year: "$timestamp" }  // menambah field tahun dari timestamp
}}
```
**Catatan:** Setelah `$project`, hanya field yang disebutkan yang lolos.

#### c. `$addFields` (alias `$set`)
Menambahkan field baru tanpa menghilangkan field yang sudah ada.
```javascript
{ $addFields: { reject_rate: { $divide: ["$reject", "$jumlah"] } } }
```
Ini berguna untuk menambahkan hasil kalkulasi.

#### d. `$group`
Stage paling penting. Mengelompokkan dokumen berdasarkan `_id` (ekspresi) dan menghitung agregat menggunakan **akumulator**.
- `$sum` – menjumlahkan nilai.
- `$avg` – rata‑rata.
- `$min`, `$max` – nilai minimum/maksimum.
- `$push` – membuat array seluruh nilai.
- `$addToSet` – membuat array nilai unik.
- `$first`, `$last` – mengambil nilai pertama/terakhir dalam kelompok (perlu `$sort` sebelumnya).
- `$count` – jumlah dokumen dalam grup (sama dengan `{ $sum: 1 }`).

**Contoh:** Total produksi per mesin.
```javascript
{ $group: {
    _id: "$mesin",
    total_produksi: { $sum: "$jumlah" },
    rata_reject: { $avg: "$reject" },
    max_reject: { $max: "$reject" }
}}
```
`_id` bisa berupa ekspresi, misal `{ mesin: "$mesin", shift: "$shift" }` untuk grup komposit.

#### e. `$sort`
Mengurutkan dokumen berdasarkan field. Arah: 1 (ascending), -1 (descending).
```javascript
{ $sort: { total_produksi: -1 } }
```
Jika diletakkan setelah `$group`, mengurutkan hasil agregasi. Bisa juga sebelum `$group` untuk memengaruhi `$first`/`$last`.

#### f. `$unwind`
Memecah field array menjadi beberapa dokumen, satu untuk tiap elemen. Setiap dokumen baru menggantikan array dengan nilai elemen.
```javascript
// Sebelum: { _id: 1, sensor: ["temp", "press"] }
// Sesudah $unwind "$sensor":
// { _id: 1, sensor: "temp" }
// { _id: 1, sensor: "press" }
```
Di industri: untuk menghitung rata‑rata dari array data sensor terkini, atau untuk merinci komponen.

**Hati‑hati:** Jika array kosong, dokumen akan hilang. Gunakan opsi `preserveNullAndEmptyArrays: true` untuk mempertahankannya.

#### g. `$lookup` (Join)
Melakukan left outer join dengan koleksi lain. Sintaks standar:
```javascript
{ $lookup: {
    from: "machines",            // koleksi asing
    localField: "machine_id",    // field di koleksi saat ini
    foreignField: "_id",         // field di koleksi asing
    as: "detail_mesin"          // nama field hasil
}}
```
Hasilnya adalah array `detail_mesin` di setiap dokumen. Untuk mengambil elemen pertama, gunakan `$unwind` atau `$first`. Pipeline `$lookup` (lebih fleksibel) memungkinkan sub-pipeline.

**Contoh:** Menampilkan data sensor dengan info mesin.

#### h. `$bucket` dan `$bucketAuto`
Mengelompokkan dokumen ke dalam interval (bucket). `$bucket` butuh batas yang eksplisit, `$bucketAuto` otomatis membagi rata.
```javascript
// Distribusi suhu
{ $bucket: {
    groupBy: "$suhu",
    boundaries: [ 0, 50, 70, 90, 100 ],
    default: ">100",
    output: { count: { $sum: 1 } }
}}
```
Berguna untuk histogram suhu, analisis distribusi kinerja.

#### i. `$facet`
Menjalankan beberapa pipeline secara paralel pada input yang sama dan menghasilkan satu dokumen dengan beberapa array hasil.
```javascript
{ $facet: {
    "total_per_mesin": [
      { $group: { _id: "$mesin", total: { $sum: "$jumlah" } } }
    ],
    "reject_tertinggi": [
      { $sort: { reject: -1 } },
      { $limit: 5 }
    ]
}}
```
Memberikan banyak laporan dalam satu query. Cocok untuk dashboard.

### 4. Ekspresi dalam Aggregation
Ekspresi digunakan di dalam `$project`, `$group`, `$addFields`, dll. Beberapa yang penting:

**String & Date:**
- `$concat`, `$toUpper`, `$substr`
- `$dateToString: { format: "%Y-%m-%d", date: "$timestamp" }`
- `$year`, `$month`, `$dayOfMonth`, `$hour`, `$minute`

**Aritmatika:**
- `$add`, `$subtract`, `$multiply`, `$divide`, `$mod`

**Logika:**
- `$cond: { if: { $gte: ["$suhu", 85] }, then: "tinggi", else: "normal" }`
- `$ifNull: ["$field", "default"]`
- `$and`, `$or`, `$not`

**Akumulator khusus `$group`:**
- `$accumulator` (custom JavaScript, hindari kecuali perlu)
- `$mergeObjects` (menggabungkan dokumen)

**Contoh ekspresi:** Menambahkan field `jam` dari timestamp.
```javascript
{ $addFields: { jam: { $hour: "$timestamp" } } }
```
Lalu group per jam.

### 5. Optimasi Aggregation Pipeline
- **Posisi `$match`**: Tempatkan `$match` sedini mungkin, terutama tepat setelah `$lookup` jika tidak ada dependency. MongoDB dapat mendorong `$match` ke depan secara otomatis, tetapi pastikan field yang difilter tersedia.
- **`$limit` awal**: Jika hanya butuh beberapa dokumen teratas, batasi sebelum operasi mahal.
- **Proyeksi awal**: `$project` untuk membuang field yang tidak diperlukan bisa mengurangi ukuran dokumen.
- **Indeks**: `$match` dan `$sort` di awal pipeline dapat memanfaatkan indeks.
- **Hindari `$unwind` besar-besaran**: Jika array besar, pertimbangkan untuk menyimpan data secara terpisah (reference) agar tidak perlu unwind.
- **Gunakan `$facet` dengan hati-hati**: karena menjalankan banyak pipeline, bisa berat.
- **Pantau dengan `explain()`**: `aggregate(...).explain("executionStats")` untuk melihat statistik dan indeks yang digunakan.

### 6. Pipeline Transformatif dalam Industri
Beberapa pola analisis:
- **Rata‑rata bergerak (moving avg):** Butuh window function, tidak ada stage bawaan. Bisa dilakukan dengan `$lookup` pada koleksi sendiri atau dengan aplikasi. Namun, kita bisa menghitung rata‑rata sederhana per jam dengan `$group`.
- **Pivot table:** Gunakan `$group` dengan multiple akumulator. Untuk pivot dinamis bisa menggunakan `$facet`.
- **Hierarki grouping:** Group bertingkat bisa dilakukan dengan beberapa `$group`, atau dengan sub-dokumen `_id`.

**Contoh menghitung OEE (Overall Equipment Effectiveness) sederhana:**
Formula: OEE = Availability × Performance × Quality. Dari data produksi, kita bisa menghitung:
- Availability: (waktu operasi aktual / waktu tersedia)
- Performance: (jumlah diproduksi / (waktu operasi aktual × kapasitas ideal))
- Quality: (jumlah ok / total jumlah)

Asumsikan koleksi `produksi` punya `mesin`, `start_time`, `end_time`, `jumlah_ok`, `jumlah_total`, `kapasitas_per_jam`. Maka pipeline kompleks menggunakan `$group` dengan `$sum` dan perhitungan di `$project`.

### 7. Penggunaan `$merge` dan `$out`
- `$out`: menulis hasil agregasi ke koleksi baru (menimpa jika ada). Berguna untuk laporan materialized.
- `$merge`: lebih fleksibel, dapat menambah, mengganti, atau memperbarui koleksi target berdasarkan kondisi.

```javascript
{ $merge: { into: "laporan_harian", on: "_id", whenMatched: "replace", whenNotMatched: "insert" } }
```
Ini memungkinkan pembuatan koleksi ringkasan yang terus diperbarui.

### 8. Validasi Skema dengan `$jsonSchema` (Opsional)
Aggregation juga bisa digunakan untuk memvalidasi atau membersihkan data.

---

## D. KEGIATAN PRAKTIKUM (120 MENIT)

### Fase 1: Pemanasan & Dataset (15 menit)
1. Pastikan server berjalan.
2. Gunakan database `praktikum5`.
3. Dosen menyediakan script untuk mengisi koleksi `log_produksi` (minimal 5000 dokumen) dengan field: `mesin` (M01-M10), `produk` (P1-P5), `jumlah` (100-500), `reject` (0-30), `waktu` (Date), `shift` (1-3). Atau mahasiswa mengimpor JSON.
4. Cek data: `db.log_produksi.countDocuments()`, `findOne()`.

### Fase 2: Eksplorasi `$match`, `$project`, `$addFields` (20 menit)
1. Pipeline sederhana:
   ```javascript
   db.log_produksi.aggregate([
     { $match: { mesin: "M01" } },
     { $project: { _id: 0, produk: 1, jumlah: 1, reject: 1, waktu: 1 } }
   ])
   ```
2. Tambahkan field `reject_rate` = reject / jumlah * 100:
   ```javascript
   { $addFields: { reject_rate: { $multiply: [ { $divide: ["$reject", "$jumlah"] }, 100 ] } } }
   ```
3. Ubah format tanggal menjadi string tanggal saja:
   ```javascript
   { $addFields: { tanggal: { $dateToString: { format: "%Y-%m-%d", date: "$waktu" } } } }
   ```

### Fase 3: Pengelompokan dengan `$group` (30 menit)
1. Total produksi dan rata‑rata reject per produk:
   ```javascript
   db.log_produksi.aggregate([
     { $group: {
         _id: "$produk",
         total_produksi: { $sum: "$jumlah" },
         rata_reject: { $avg: "$reject" },
         max_reject: { $max: "$reject" },
         count: { $sum: 1 }
     }},
     { $sort: { total_produksi: -1 } }
   ])
   ```
2. Group komposit: per `mesin` dan `shift`, total produksi.
   ```javascript
   { $group: { _id: { mesin: "$mesin", shift: "$shift" }, total: { $sum: "$jumlah" } } }
   ```
3. Hitung produksi per jam: gunakan `$hour` dari `waktu` sebagai bagian `_id`.
   ```javascript
   { $group: { _id: { mesin: "$mesin", jam: { $hour: "$waktu" } }, total: { $sum: "$jumlah" } } }
   ```

### Fase 4: `$unwind` untuk Data Array (20 menit)
1. Buat koleksi `mesin` (jika belum) dengan dokumen yang punya array `tag` (contoh: `["CNC", "presisi"]`).
2. Unwind tag untuk menghitung jumlah mesin per tag:
   ```javascript
   db.mesin.aggregate([
     { $unwind: "$tag" },
     { $group: { _id: "$tag", jumlah: { $sum: 1 } } }
   ])
   ```
3. Diskusikan dampak ukuran array besar.

### Fase 5: Join dengan `$lookup` (15 menit)
1. Buat koleksi `mesin_info` dengan `_id` kode mesin dan field `nama`, `lokasi`.
2. Jalankan agregasi dari `log_produksi` yang menggabungkan detail mesin:
   ```javascript
   db.log_produksi.aggregate([
     { $match: { waktu: { $gte: ISODate("2026-05-01") } } },
     { $lookup: {
         from: "mesin_info",
         localField: "mesin",
         foreignField: "_id",
         as: "info"
     }},
     { $unwind: "$info" },
     { $project: { _id: 0, mesin: 1, "info.nama": 1, jumlah: 1 } }
   ])
   ```

### Fase 6: `$bucket` dan `$facet` (10 menit)
- Distribusi jumlah produksi: bucket 0-200, 201-400, 401-600.
- `$facet` untuk menampilkan total per mesin dan top 5 reject secara bersamaan.

### Fase 7: Gunakan Compass Aggregation Builder (10 menit)
- Buka Compass, buka koleksi `log_produksi`, tab "Aggregations".
- Tambahkan stage secara visual, lihat output setiap stage, ekspor ke bahasa query.

---

## E. LATIHAN MANDIRI
1. Gunakan database `latihan5`, koleksi `data_sensor` dengan 5000 dokumen (field: `sensor_id`, `nilai`, `timestamp`, `category`).
2. Tulis pipeline untuk:
   - Menampilkan rata‑rata nilai per `sensor_id` pada 7 hari terakhir.
   - Menampilkan sensor yang memiliki nilai di atas 90 minimal 3 kali (gunakan `$group` dengan `$push` atau `$sum` kondisional).
   - Menambahkan field `jam` dan menghitung nilai rata‑rata per jam per sensor.
3. Buat koleksi `sensor_info` lalu join untuk menampilkan nama sensor dalam hasil.
4. Dokumentasikan setiap pipeline, jelaskan setiap stage.

---

## F. STUDI KASUS: ANALISIS EFEKTIVITAS PRODUKSI
**Latar Belakang:** Pabrik elektronik mencatat produksi di koleksi `produksi_harian` dengan struktur:
- `mesin`, `tanggal`, `shift`, `target` (target produksi),
- `actual_ok` (hasil baik), `actual_reject` (hasil cacat),
- `durasi_operasi_menit`, `durasi_tersedia_menit`.

Manajemen ingin:
1. Menghitung OEE secara kasar = (actual_ok / target) * (durasi_operasi / durasi_tersedia) * (actual_ok / (actual_ok+actual_reject)).
2. Menampilkan OEE per mesin per bulan.
3. Menemukan mesin dengan OEE di bawah 80% pada bulan tertentu.
4. Distribusi jumlah produksi per shift dalam bentuk bucket.

**Tugas:**
- Buat koleksi dan isi data simulasi minimal 200 dokumen.
- Tulis pipeline agregasi untuk menghitung OEE per mesin, per bulan. (Gunakan `$group` dengan `_id` { mesin, bulan: { $month: "$tanggal" } }).
- Tampilkan hanya mesin yang OEE < 0.8.
- Sajikan hasil dalam laporan, serta jelaskan setiap stage.

---

## G. TUGAS TERSTRUKTUR
### Deskripsi
Membangun pipeline agregasi untuk dashboard monitoring kualitas pabrik.

### Spesifikasi
1. Gunakan database `tugas5_<NIM>`. Buat koleksi `inspeksi` (min 1000 dok) dengan field:
   - `batch_id`, `tanggal` (Date), `mesin`, `inspektor`, `jumlah_diperiksa`, `cacat_ditemukan`, `jenis_cacat` (array string).
2. Buat koleksi `target_kualitas` berisi `batch_id`, `target_maks_cacat`.
3. Buat pipeline untuk:
   - Menghitung persentase cacat per batch.
   - Gabungkan dengan target kualitas menggunakan `$lookup`, lalu tandai batch dengan status "OK" atau "NOT OK" jika persentase melebihi target.
   - Hitung jumlah batch NOT OK per mesin per minggu (gunakan `$week`).
   - Tampilkan top 3 mesin dengan jumlah NOT OK terbanyak.
   - (Opsional) Gunakan `$facet` untuk memberikan ringkasan bulanan: total produksi, rata-rata cacat, dan mesin terburuk dalam satu query.
4. Ikuti prinsip optimasi: `$match` awal, indeks pada `tanggal` dan `mesin`, proyeksi minimal.
5. Sertakan perintah dan hasil di laporan PDF, serta interpretasi.

### Rubrik Penilaian
| Kriteria                        | Bobot |
|---------------------------------|-------|
| Kompleksitas & ketepatan pipeline | 30%   |
| Penggunaan $lookup & ekspresi   | 20%   |
| Optimasi dan indeks             | 20%   |
| Dokumentasi dan analisis        | 20%   |
| Kreativitas (facet, bucket)     | 10%   |

### Tenggat
Sebelum pertemuan ke-6, unggah di LMS.

---

## H. REFERENSI PERTEMUAN 5
1. MongoDB, Inc. *Aggregation Pipeline*. https://docs.mongodb.com/manual/aggregation/
2. MongoDB University. *M121: The MongoDB Aggregation Framework* (Free Course).
3. Hows, D., Membrey, P., Plugge, E. (2020). *MongoDB Basics*. Apress.
4. Copeland, R. (2020). *MongoDB Applied Design Patterns*. O'Reilly.

---

**Pesan Dosen:**
Aggregation framework adalah jantung analisis data di MongoDB. Kuasai, dan Anda dapat mengubah data mentah menjadi wawasan industri berharga. Jangan lupa selalu mengukur kinerja dengan `explain`. Selamat belajar!
